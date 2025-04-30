---
title: Slack + Lambda + NewRelicでコーヒー摂取量を記録・可視化できるコマンドを作ってみた
tags:
  - AWS
  - NewRelic
  - lambda
  - APIGateway
  - slack-api
private: false
updated_at: '2022-12-01T09:43:35+09:00'
id: 7f638e6047fedac1acbd
organization_url_name: null
slide: false
ignorePublish: false
---
[新卒エンジニアによる全部俺カレンダー2022](https://qiita.com/advent-calendar/2022/arie_onlyme2022)　1日目投稿記事です。


## 概要

作業に集中する時やリラックスする時によく飲むコーヒー・・。
頭が冴えてスッキリするのですが、ついつい飲み過ぎてカフェイン依存症にならないか健康面的に心配になってきました。

そこで自分がどのくらいコーヒーを飲んだか可視化してみようと思い、Slackコマンドを作成してみました。

## 使用技術

Slack API
AWS Lambda + API Gateway
NewRelic

## 用意するもの
- Slackのアカウント / ワークスペース
- AWSアカウント
- NewRelicのアカウント（無料版でOK）

## Slack APIでの設定

Command: 好きなコマンド名（英字）

Request URL: API Gatewayのエンドポイントを設定

![スクリーンショット 2022-11-23 11.01.54.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/bef2a5ea-bac0-b2d7-f845-14699d4da141.png)

今回は`/newrelic`という名前でコマンドを作成していきます。
## Lambda関数の作成

今回ランタイムはPython3.9を使用。

設定 > 環境変数で　NewRelicのAPI KeyとEvent APIのエンドポイントURLをあらかじめ設定しておく。
![スクリーンショット 2022-12-01 9.34.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/a042d77a-76d9-7dff-bf0e-254c8136fb75.png)


```python
import json
import os
from urllib.parse import parse_qs
import requests
import boto3

NEWRELIC_KEY = os.environ["NEWRELIC_KEY"]
NEWRELIC_ENDPOINT = os.environ["NEWRELIC_ENDPOINT"]

def lambda_handler(event, context):

    headers = {"X-Insert-Key": NEWRELIC_KEY}
    d = {}
    # NRQLで指定するイベント名
    d["eventType"] = "health"
    # NRQLで取得する属性名
    d["count_coffee"] = 1

    # NewRelicにイベントを送信
    res = requests.post(NEWRELIC_ENDPOINT, headers=headers, json=d)
    
    # コマンド実行時に返すメッセージ
    message = '記録完了！ \n コーヒーの飲み過ぎはほどほどに。'

    return {
        "response_type": "ephemeral",
        "text": message
    }
```

## API Gatewayの設定

Lambda関数のトリガーとしてAPI Gatewayを作成する。

（※ API Gatewayの概要・基本的な使い方は割愛します）

APIの作成から REST API を選択

![スクリーンショット 2022-11-23 10.36.14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/9f40e01b-1bed-3a96-3d87-54675065f354.png)


アクション → メソッドの作成からPOSTメソッドを作成

統合リクエストの設定を行う。

- 統合タイプ: Lambda関数
- Lambda関数: 今回の手順で作成するLambda関数名

![スクリーンショット 2022-11-23 10.34.41.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/8143761d-847e-4562-9782-9351f78743fa.png)


続いてマッピングテンプレートの設定を行う。

Slack APIのContent-Typeである`**application/x-www-form-urlencoded**`を指定する。

![スクリーンショット 2022-11-23 10.58.46.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/610079dd-a305-aeb5-8d22-3701608f5193.png)


テンプレートの生成から「メソッドリクエストのパススルー」を選択



## Slackコマンドを使ってみる

早速コマンドを使ってみます。
![スクリーンショット 2022-12-01 9.05.13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/0b6c3a32-ace1-2e15-966d-e3f079c02fd0.png)

Lambda関数で設定した、コマンド実行時のメッセージが返って来れば成功です。
![スクリーンショット 2022-10-23 9.41.10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/55dc075c-0315-3595-fedd-7bbea9957c73.png)

## NewRelicでコーヒー摂取量を可視化

Slackコマンドによって、NewRelic上の`health`というイベントに`count_coffee`という属性でコーヒー摂取量を記録しました。

実際に自分がどのくらいコーヒーを飲んでいるか、NewRelic上で可視化してみます。


NewRelic > Dashboardに行き、ダッシュボードを作成します。
![スクリーンショット 2022-12-01 9.11.20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/b60868db-8e07-1325-d9f7-2537ff282832.png)

作成したダッシュボードの右上の+ボタンをクリックし、Add Chartでグラフを作成します。

![スクリーンショット 2022-12-01 9.12.15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/32a7c61d-d4a1-9313-5690-542932ad1be8.png)

![スクリーンショット 2022-12-01 9.12.05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/c33da186-7384-c3d2-d67e-e4536d665721.png)


以下のNRQLを貼り付けてグラフを作成します。

```
SELECT sum(count_coffee) from health since 7 week ago TIMESERIES 1 week
```


![スクリーンショット 2022-12-01 9.14.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/c21486bb-6f81-b42c-e287-056285294db2.png)

こんな感じでグラフで自分のコーヒー摂取量がグラフで可視化されます・・・！

Slack API, Lambda, NewRelicを組み合わせると、このようなちょっとしたライフログ的なものが作れます。

## 参考
https://dev.classmethod.jp/articles/lambda-api-slack-command/

https://docs.newrelic.com/jp/docs/data-apis/ingest-apis/event-api/introduction-event-api/
