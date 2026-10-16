---
title: ECS Fargate タスクがスケールアウト直後にすぐスケールインしてしまう問題に立ち向かう
publication_name: canly
private: false
tags:
  - AWS
  - ECS
  - Fargate
  - CloudWatch
  - AutoScaling
updated_at: '2026-10-16T01:35:10.973Z'
id: null
organization_url_name: null
slide: false
---

## はじめに

こんにちは！カンリー SRE チームの有村です。

ECS Fargate でオートスケーリングを設定していると、「スケールアウトしたと思ったらすぐにスケールインしてしまう」という現象に出会うことがあります。
本記事では、実際に私たちのサービスで発生したタスク数のフラッピングについて、**スケールインポリシーのトリガーとなる CloudWatch Alarm の設定を見直すことでスケールインのタイミングを遅延させ、問題を解消した事例** を負荷試験の実測値とともに紹介します。

## TL;DR

- スケールインポリシーのトリガーとなる CloudWatch Alarm の **評価期間 (`evaluation_periods`) を長めに設定** し、ALARM 状態への遷移を遅らせることでスケールインを遅延させる
- スケールインの閾値に CPU 使用率を使っている場合、**閾値が高すぎないか (= スケールアウト後すぐに下回ってしまわないか)** を確認する

## 発生していた課題

ECS Fargate で運用していたアプリケーションにおいて、アクセススパイク時に **スケールアウト → すぐにスケールイン** が繰り返され、タスク数のフラッピングが発生していました。

その結果、スパイクが続いているにもかかわらずスケールインしてしまい、残ったタスクが負荷に耐えきれずヘルスチェックに失敗してしまうケースもありました。
スケールアウトで負荷に耐えられるようにしたはずが、直後のスケールインで再び過負荷に陥る、という本末転倒な状態です。

## 構成

対象サービスでは、CloudWatch Alarm をトリガーとしたステップスケーリングポリシーでスケールアウト / スケールインを制御していました。

```text
CloudWatch Alarm (CPUUtilization 高) ──▶ スケールアウトポリシー
CloudWatch Alarm (CPUUtilization 低) ──▶ スケールインポリシー
```

## 原因

原因は大きく 2 つありました。

### 1. スケールインの閾値が高すぎた

スケールインの閾値として「**平均 CPU 使用率 30% 以下**」を設定していました。

ECS サービスの `CPUUtilization` はサービス内の **全タスクの平均値** です。
スケールアウトでタスク数が増えると負荷が分散されるため、スパイク自体はまだ続いていても平均 CPU 使用率は下がります。
たとえば 2 タスクで平均 60% だったものが 4 タスクに増えれば、単純計算で平均 30% 前後まで下がります。

つまり、「スケールアウトしたこと自体によってスケールインの条件を満たしてしまう状態」になっていました。

### 2. アラームの評価期間が短すぎた

スケールイン用の CloudWatch Alarm の評価条件が「**1 分間 × 1 データポイント**」になっていました。

CloudWatch Alarm は、`evaluation_periods` (評価期間の数) と `datapoints_to_alarm` (ALARM に必要な閾値超過データポイント数) で ALARM 状態への遷移を判定します。
この設定だと、**閾値を 1 回下回った時点で即 ALARM (= 即スケールイン)** となってしまいます。

1 の問題と組み合わさることで、「スケールアウト → 平均 CPU が下がる → 1 分後にはスケールイン」というフラッピングが起きていました。

## 解決策

### 1. スケールインの閾値を見直す

スケールインの閾値を **30% → 15%** に引き下げました。

値の根拠としては、アプリケーションの CPU 使用率メトリクスを確認し、**アクセスが少ない時間帯の CPU 使用率** を採用しています。
「スケールインしてよい = 本当に負荷が落ち着いている」状態を表す値にすることで、スケールアウト直後の平均値低下では閾値を下回りにくくなります。

### 2. CloudWatch Alarm の評価期間を長くする

スケールイン用アラームの `evaluation_periods` を長く設定し、**閾値を下回ってから ALARM 状態に遷移するまでに猶予を持たせる** ようにしました。

スパイクが続いている間は CPU 使用率が閾値付近を上下するため、評価期間中に一度でも閾値を上回れば ALARM には遷移しません。
これにより、「負荷が十分な時間落ち着いてから」スケールインされるようになります。

Terraform での設定例

```hcl
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_low" {
  alarm_name          = "${var.service_name}-cpu-low"
  namespace           = "AWS/ECS"
  metric_name         = "CPUUtilization"
  statistic           = "Average"
  period              = 60
  comparison_operator = "LessThanOrEqualToThreshold"
  threshold           = 15 # 30 -> 15

  # 1 -> 10: 10 分間連続で閾値を下回った場合のみ ALARM に遷移させる
  evaluation_periods  = 10
  datapoints_to_alarm = 10

  dimensions = {
    ClusterName = var.cluster_name
    ServiceName = var.service_name
  }

  alarm_actions = [aws_appautoscaling_policy.scale_in.arn]
}

resource "aws_appautoscaling_policy" "scale_in" {
  name               = "${var.service_name}-scale-in"
  policy_type        = "StepScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  step_scaling_policy_configuration {
    adjustment_type         = "ChangeInCapacity"
    cooldown                = 300
    metric_aggregation_type = "Average"

    step_adjustment {
      metric_interval_upper_bound = 0
      scaling_adjustment          = -1
    }
  }
}
```

:::note
`datapoints_to_alarm` を `evaluation_periods` より小さくすると「N 個中 M 個」のアラームになり、閾値を下回るデータポイントが連続していなくても ALARM に遷移します。
スケールインを確実に遅らせたい場合は、両者を同じ値にして「N 分間連続で下回ったら」という条件にするのがわかりやすいです。
:::

## 実測

設定変更の効果を確認するため、負荷試験を実施し、`evaluation_periods` ごとに **スケールアウトしてからスケールインするまでの時間** を計測しました。

| `evaluation_periods` | CPU 低下から ALARM まで | scale-out → scale-in |
| --- | --- | --- |
| **1** | 約 1 分 | **77〜148 秒** |
| **5** | 約 7 分 20 秒 | 10 分 |
| **10** | 約 12 分 | **13 分 30 秒** |

`evaluation_periods = 1` の場合、スケールアウトからわずか 1〜2 分半ほどでスケールインしてしまっていました。
一方、`evaluation_periods = 10` にすると、スケールアウトからスケールインまで **約 13 分 30 秒** の猶予が生まれ、評価期間の長さに応じてスケールインのタイミングを遅延できていることがわかります。

※なお、CPU 低下から ALARM までの時間が `evaluation_periods` × 1 分よりやや長くなっているのは、CloudWatch へのメトリクス反映の遅延やアラームの評価タイミングの影響によるものです。

評価時間を長くしすぎると、余剰なタスクが残り続けることになるので、スパイクの傾向とコストのバランスを考慮してちょうど良い値に設定すると良いでしょう。

## まとめ

- スケールアウト直後のスケールインは、**スケールインの閾値** と **アラームの評価期間** の組み合わせで起こりやすい
- CPU 使用率はタスク数が増えると平均値が下がるため、スケールインの閾値は「本当に負荷が低いとき」の実績値をもとに決める
- CloudWatch Alarm の `evaluation_periods` / `datapoints_to_alarm` を長めに設定することで、ALARM への遷移を遅らせ、スケールインのタイミングをコントロールできる

本記事が同じような事象に悩んでいる方の参考になれば幸いです！

## 参考資料

- [Evaluating an alarm - Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html)
- [Use step scaling policies to scale Amazon ECS services](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-autoscaling-stepscaling.html)
- [Step scaling policies for Application Auto Scaling](https://docs.aws.amazon.com/autoscaling/application/userguide/application-auto-scaling-step-scaling-policies.html)
