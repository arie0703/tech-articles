---
title: Metric Streamsを用いたDatadogへのAWSメトリクス送信コスト削減
publication_name: canly
private: false
tags:
  - AWS
  - コスト削減
  - CloudWatch
  - Metric Streams
updated_at: '2025-12-10T00:03:47.429Z'
id: null
organization_url_name: null
slide: false
---

みなさんこんにちは！

カンリーでSREを担当している有村です。

今回は、CloudWatch Metric Streamsを導入してAWSのCloudWatchコストを削減した事例を紹介します。

## 背景と課題

カンリーでは、Datadogを監視ツールとして利用しており、AWSのインフラリソースのメトリクスをDatadogに送信するためにCloudWatchの`GetMetricData API`を利用していました。

しかし、CloudWatch APIは以下のようなコスト構造となっており、メトリクス取得回数に応じてコストが発生します：

- 0.01USD / 1000メトリクス

Datadogへのメトリクス送信をGMD APIで行っている箇所が、CloudWatchコストの主要な要因となっていました。

## Metric Streamsの導入

### Metric Streamsとは

CloudWatch Metric Streamsは、CloudWatchメトリクスをリアルタイムでストリーミング配信するサービスです。Kinesis Data Firehoseを経由して、Datadogなどの外部監視ツールにメトリクスを配信できます。

CloudWatch APIとの違い
- APIポーリングではなくPush形式での配信
- APIポーリング（10分間隔）よりも早い間隔での配信が可能
- メトリクスあたりのコストが低い(1,000メトリクス更新ごとに0.003USD)

API経由でのメトリクス取得と比較して、コスト効率が高いことから、今回導入に至りました。

## 抑えておきたいポイント

### ① Metric Streamsで必ずしもコスト減になるとは限らない

API経由でメトリクス送信している部分を**全てMetric Streamsに置き換えれば良いわけではありません**。

リソースの種類によっては、Datadog IntegrationのTag Filter機能で**タグによる絞り込み**ができる場合があります。タグによる絞り込みは**CloudWatch API経由のみ可能**です。(ALB, EC2, RDS, SQS, Lambdaなどが対象)

同一アカウント内に複数のサービスや環境が混在しており、**特定のサービスタグを持つリソースのみメトリクスをDatadogに送りたい場合**は、全リソースのメトリクスをMetric Streamsで送信するより、API経由でタグによる絞り込みをしたほうがコスト効率が良い場合があります。

また、Metric Streams経由で必要ないリソースのメトリクスを一括で送信するとDatadogのHost数が増えることが起因でDatadogのコスト増になる可能性もあります。

そのため、特定のAWSリソースにおいて一部リソースのみメトリクス監視が必要な場合は、CloudWatch API経由でのメトリクス送信を検討すると良いでしょう。


### ② 送信するメトリクスを絞り込む

Metric Streamsでは送信するメトリクスの種類を絞り込むことが可能です。

例えばEC2であればCPU使用率(CPUUtilization)やインスタンスストアボリュームでの読み取り操作数(DiskReadOps)などがあります。

Datadog上での監視が必要なリソースに絞り込んでメトリクス送信するように設定すると、コスト効率を上げることができます。

### ③ Metric Streamsだと一部取得できないメトリクスが存在する

ECSのサービス・タスク単位のステータスなど、一部メトリクスはCloudWatch API経由でしか取得できないケースもあります。（例: `ecs.tasks.running`）

これはAWSがCloudWatchメトリクスとして公開しているものではなく、DatadogがCloudWatch APIを使ってリソースの状態を能動的に取得・計算しているメトリクスです。

そうした制約があることも踏まえ、API経由でしか取得できないメトリクスが必要な場合はCloudWatch APIでのメトリクス送信を検討すると良いでしょう。

## 導入に必要なリソース

Metric Streamsを導入するために、以下のリソースを作成します。（Terraformの実装例など、詳細な実装方法は今回割愛します）

![構成図](https://raw.githubusercontent.com/arie0703/tech-articles/main/images/00-pub-02-metric-stream/metricstream.drawio.png)

### ① Kinesis Data Firehose Delivery Stream

Datadogへのメトリクス配信を行うためのFirehose Delivery Streamを作成します。

基本的な設定は以下となります。

- HTTPエンドポイント名: Datadog
- HTTPエンドポイントURL: `https://awsmetrics-intake.datadoghq.com/api/v2/awsmetrics?dd-protocol=aws-kinesis-firehose`
- コンテンツエンコーディング: `GZIP`

### ② CloudWatch Metric Streams

CloudWatchメトリクスをFirehoseに配信するためのMetric Streamsを作成します。


:::note
出力フォーマットは`OpenTelemetry v0.7`を指定する必要があります。

[参考](https://docs.datadoghq.com/ja/integrations/guide/aws-cloudwatch-metric-streams-with-kinesis-data-firehose/?tab=aws%E3%82%B3%E3%83%B3%E3%82%BD%E3%83%BC%E3%83%AB)
:::

### ③ IAMロール

Metric StreamsとFirehoseが適切に動作するために、以下のIAMロールを作成します：
- Metric Streams用のIAMロール（Firehoseへの書き込み権限）
- Firehose用のIAMロール（S3バケットへのログ書き込み権限、Secrets ManagerからのAPIキー取得権限）

### ④ S3バケット

Firehoseの配信失敗ログを格納するためのS3バケットを作成します。

### ⑤ Secrets Manager

FirehoseからDatadogにメトリクスを送信する際、必要となるDatadogのAPIキーを格納します。

:::note
FirehoseでSecrets ManagerからDatadog APIキーを取得する際は、**キー名が`api_key`である必要があります**
:::


## 導入効果

![コスト削減効果](https://raw.githubusercontent.com/arie0703/tech-articles/main/images/00-pub-02-metric-stream/costs.png)

本記事で紹介したポイントを押さえつつ、一部リソースのメトリクスをMetric Streamsに置き換えたところ、約半分のコスト削減を実現できました。(`APN1-CW-GMD-Metrics`がCloudWatch APIの利用料金に該当します)

## まとめ

CloudWatch Metric Streamsを導入することで、CloudWatch API経由でのメトリクス取得コストを大幅に削減することができました。

ただし、全てのケースでMetric Streamsが最適とは限りません。AWSアカウントにおけるサービス・環境の構成や必要なメトリクスの種類によって、CloudWatch APIとMetric Streamsの使い分けを検討する必要があります。

本記事がMetric Streamsの導入を検討されている方の参考になれば幸いです！

ご質問やご意見がございましたら、コメントいただけると幸いです。
