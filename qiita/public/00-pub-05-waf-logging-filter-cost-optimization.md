---
title: AWS WAF のログを Logging Filter で 整理してコスト削減しよう
publication_name: canly
private: false
tags:
  - AWS
  - WAF
  - コスト削減
updated_at: '2026-07-09T04:03:23.077Z'
id: null
organization_url_name: null
slide: false
---

## はじめに

こんにちは！カンリー SRE チームの有村です。
今回はWAFに関するAWSコスト削減の取り組みを本記事で紹介いたします。

## 背景

ある日、AWSのコストを確認してみると、`USE1-S3-Egress-Bytes-WAFLogs` のコストが上昇していました。

実態を調べるため、CloudFrontでホスティングしているWebサイトで利用中のWAFログを確認してみると、その大半が **静的ファイルへの ALLOW ログ** であり、これがコスト上昇の要因であることがわかりました。

WAFログを調査で利用するケースは

- 意図しないBLOCKが発生していないかを確認する
- 特定のルールにCOUNTモードを適用しており、ルールの評価状況を確認する

などといった点が考えられますが、静的ファイルに対するALLOWログは調査対象のログではないため、保持する必要性はそこまでありませんでした。

そこで WAF の Logging Filter を使って「必要性の低い ALLOW ログ」をS3に保持しないようにする(DROP)設定を導入することにしました。

### `S3-Egress-Bytes-WAFLogs` とは

**WAF → S3 への Vended Logs 配信** にかかるコストであり、**配信バイト数(≒ 配信ログレコード数 × 平均バイトサイズ)** に比例して課金されます。

## 実装

方針として、ログ記録から除外する対象のリクエストログに対し、**WebACL 側でラベルを付け、Logging Filter 側で対象のラベルがついたログをDROPする** という構成で実装します。

### 1. WebACL に「ラベル付けだけする」ルールを追加

静的ファイルの URI パターンを `regex_pattern_set` に切り出し、それにマッチしたら `static-asset` ラベルを付けて **`count`** としてリクエストを通過させるルールを追加します。

```hcl
resource "aws_wafv2_regex_pattern_set" "static_assets" {
  name     = "static-assets-pattern"
  scope    = "CLOUDFRONT"

  # Next.js ビルド成果物
  regular_expression {
    regex_string = "^/_next/static/[a-zA-Z0-9_/-]+\\.(js|css|woff2?|json)$"
  }
  # 公開画像（拡張子限定）
  regular_expression {
    regex_string = "^/(images|img|assets)/[a-zA-Z0-9_/-]+\\.(png|jpg|jpeg|gif|webp|svg|ico|avif)$"
  }
  regular_expression { regex_string = "^/favicon\\.ico$" }
  regular_expression { regex_string = "^/_next/data/[a-zA-Z0-9_/-]+\\.json$" }
  regular_expression { regex_string = "^/_next/image$" }
}

resource "aws_wafv2_web_acl" "main" { 
  name     = "waf-rule"
  scope    = "CLOUDFRONT"
  
  # ラベル付与をするためのルールを設定
  rule {
    name     = "LabelStaticAssets"
    priority = 200
    action { count {} } # 終端させず、ラベル付与とカウントだけ行う
  
    rule_label { name = "static-asset" }
  
    statement {
      regex_pattern_set_reference_statement {
        arn = aws_wafv2_regex_pattern_set.static_assets.arn
        field_to_match { uri_path {} }
        text_transformation { priority = 0; type = "NONE" }
      }
    }
	
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "LabelStaticAssets"
      sampled_requests_enabled   = true
    }
}
```

### 2. 除外対象のラベルがついたログをLogging FilterでDROP

Logging Filterで `static-asset` というラベルがつけられたログを破棄 (DROP)する設定をここで導入します。

```hcl
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  provider                = aws.virginia
  log_destination_configs = [aws_s3_bucket.waf_log.arn]
  resource_arn            = aws_wafv2_web_acl.main.arn

  logging_filter {
    default_behavior = "KEEP"
    
    # DROP：static-asset ラベル付きログを破棄
    filter {
      behavior    = "DROP"
      requirement = "MEETS_ALL"

      condition {
        label_name_condition {
          label_name = "awswaf:${data.aws_caller_identity.current.account_id}:webacl:${aws_wafv2_web_acl.main.name}:static-asset"
        }
      }
    }
  }
}
```

## Logging Filter利用におけるTipsと注意点

### 複数のラベルを利用する場合は、優先順位に気を付ける

DROPしたいログに付与するためのラベルのほかに、別のラベルも利用している場合にはLogging Filterの設定に注意が必要です。

例えば、

1. 特定ルールで `no-user-agent` というラベルを付与しており、調査用に保持しておきたい
2. 上記に該当せず、`static-asset` というラベルがついたログは保持しない

この場合、1. のラベルを優先的に保持（KEEP）するルールを先に定義した上で、DROPルールをその後に記述する必要があります。
（Logging Filter は記述順に評価する仕様となっているため）

例:

```hcl
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  provider                = aws.virginia
  log_destination_configs = [aws_s3_bucket.waf_log.arn]
  resource_arn            = aws_wafv2_web_acl.main.arn

  logging_filter {
    default_behavior = "KEEP"

    # KEEP 優先：no-user-agentラベルが付いたログは優先的に残す
    filter {
      behavior    = "KEEP"
      requirement = "MEETS_ANY"

      condition {
        label_name_condition {
          label_name = "awswaf:${data.aws_caller_identity.current.account_id}:webacl:${aws_wafv2_web_acl.main.name}:no-user-agent"
        }
      }
    }

    # DROP：上記 KEEP に該当しない static-asset ラベル付きを破棄
    filter {
      behavior    = "DROP"
      requirement = "MEETS_ALL"

      condition {
        label_name_condition {
          label_name = "awswaf:${data.aws_caller_identity.current.account_id}:webacl:${aws_wafv2_web_acl.main.name}:static-asset"
        }
      }
    }
  }
}
```

### Logging Filter には個数上限がある

`logging_filter` の `filter` は最大 3 件、`condition` は filter あたり最大 5 件です。URI を直接条件に並べていくと早晩詰まるので、regex_pattern_setに該当するリクエストパターンによって付与する**ラベル**を活用する方が運用がしやすいかと思います。

### DROP したログは復元不可

Logging FilterのDROPはS3やCloudWatchにログを残さないようにする設定のため、どこにも残りません。

DROPルールを本番環境に適用する前に、事前に不要なログにラベル付→DROPする予定のログのみにラベルが付与されているかを確認する方が安心でしょう。

## 3行まとめ

- `S3-Egress-Bytes-WAFLogs` は **「WAF → S3 へのログ配信」** にかかる Vended Logs のコストであり、配信バイト数に比例するため、コスト最適化のためにログとして記録するものを取捨選択する必要がある。
- DROPする対象ログにラベル付するルールを追加→Logging Filterで特定ラベルがついたリクエストをDROPするように設定 で実装するのがベター
- 他にもラベル運用をしている場合は、Logging Filterの評価の優先付に注意する必要がある。

今一度、WAFログに不要なログを溜めすぎてコストが嵩んでいないかを確認し、保持するログの棚卸しを実施するなど、コスト削減に本記事の内容が役に立てば幸いです。

## 参考リンク

- [AWS WAF AWS Shield Advanced AWS Shield ネットワークセキュリティディレクターとは何ですか](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/what-is-aws-waf.html)
- [保護パック (ウェブ ACL) のログ記録の設定](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-management-configure.html)
- [AWS WAFの料金](https://aws.amazon.com/waf/pricing/)
