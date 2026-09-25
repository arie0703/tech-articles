---
title: "Terraform リポジトリの Renovate 設定を見直して PR の滞留を解消した話"
emoji: "♻️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "Renovate", "tfaction", "GitHubActions"]
published: false
publication_name: "canly"
---

## はじめに

こんにちは！カンリー SRE チームの有村です。

Renovate を導入すると依存関係の更新を自動化できる一方で、設定次第では PR が大量に作成され、誰もマージしないまま滞留してしまうことがあります。
実際にカンリーで利用しているTerraformリポジトリにおいても、**74 件の Renovate PR が滞留** している状態になっていました。

本記事では、Renovate の設定を見直して滞留を解消した際に得られた Tips を紹介します。

## 前提

対象のリポジトリは以下のような運用になっています。

- CI に [tfaction](https://github.com/suzuki-shunsuke/tfaction) (v1) を利用しており、PR 作成時に `terraform plan`、マージ時に `terraform apply` が実行される
- GitHub のブランチ保護で **マージ前にブランチが最新であること (Require branches to be up to date before merging)** を必須にしている

## 滞留の要因と行った変更

滞留の要因を調べたところ、大きく以下の 3 つがありました。

1. tfstate を持つディレクトリごとに PR が作成され、PR の数そのものが多すぎた
2. PR のブランチが古い base のまま放置され、マージできない状態で積み重なっていた
3. アップデートによって必ず発生する plan 差分で CI が失敗し、自動マージが止まっていた

それぞれについて行った変更を紹介します。

### 1. Branch prefix の見直しで PR の粒度を調整する

`hashicorp/terraform` (Terraform 本体) の更新 PR が、**tfstate を持つディレクトリごと** に作成される設定になっていました。

たとえば以下のディレクトリ構成だと、tfstate を持つディレクトリが 9 つあるため、Terraform のバージョン更新のたびに **9 つの PR** が作成されます。

```plain text
terraform
├── service-1
│   ├── service-1-a
│   ├── service-1-b
│   └── service-1-c
├── service-2
│   ├── service-2-a
│   ├── service-2-b
│   └── service-2-c
└── service-3
    ├── service-3-a
    ├── service-3-b
    └── service-3-c
```

原因は `renovate.json` の `additionalBranchPrefix` に `{{baseDir}}` を指定していたことです。
`additionalBranchPrefix` は Renovate が作成するブランチ名に付与する prefix です。Renovate は **ブランチ名が同じ更新を 1 つの PR にまとめる** ため、ここに `baseDir` (tfstate を持つディレクトリのパス) を含めると、**ディレクトリごとに別ブランチ = 別 PR** として作成されます。
（`additionalBranchPrefix` を単純に削除すると、リポジトリ全体の更新が 1 つの PR にまとまってしまい、PRの粒度が大きくなってしまいます。）

今回のケースでは、**`terraform/` 直下のディレクトリ (上記の例では `service-1` 〜 `service-3`) 単位** で PR をまとめるように、以下の変更を行いました。

- `matchFileNames` で対象を `terraform/` 配下に限定する
- `additionalBranchPrefix` を `baseDir` から「`terraform/` 直下のディレクトリ名」に変更する

before:

```json
{
  "matchPackageNames": ["hashicorp/terraform"],
  "additionalBranchPrefix": "{{baseDir}}-"
}
```

after:

```json
{
  "matchPackageNames": ["hashicorp/terraform"],
  "matchFileNames": ["terraform/**"],
  "additionalBranchPrefix": "{{{ replace 'terraform/([^/]+).*' '$1' packageFileDir }}}-"
}
```

`additionalBranchPrefix` の変更内容について:

- `packageFileDir` は更新対象ファイルが置かれているディレクトリのパス (例: `terraform/service-1/service-1-a`)
- Renovate のテンプレートで使える `replace` ヘルパーで、正規表現 `terraform/([^/]+).*` にマッチさせ、キャプチャした `terraform/` 直下のディレクトリ名 (例: `service-1`) だけを取り出す

これにより、`service-1-a` 〜 `service-1-c` の更新はすべて `service-1-` という prefix を持つ同じブランチに集約され、上記の構成であれば **9 PR → 3 PR** にまとまります。

PR の数が減ることで、レビュー・マージの負荷が下がるだけでなく、後述する rebase による CI 実行回数の削減にもつながります。

### 2. `rebaseWhen` で古い base のまま滞留する PR を解消する

`rebaseWhen` が `never` に設定されていました。
PR が頻繁にマージされるリポジトリでは Renovate PR のブランチがすぐに古くなり、ブランチ保護 (Require branches to be up to date before merging) によって **マージできない PR が積み重なる** ことが常態化していました。

そこで `rebaseWhen` を `behind-base-branch` に変更しました。

```json
{
  "rebaseWhen": "behind-base-branch"
}
```

`rebaseWhen` に指定できる値と挙動は以下の通りです。

| 値 | 挙動 |
| --- | --- |
| `auto` (既定) | automerge 設定時、またはリポジトリが「PR が最新であること」を要求している場合は `behind-base-branch`。それ以外は `conflicted` |
| `behind-base-branch` | base より 1 コミット以上遅れたら常に rebase |
| `conflicted` | コンフリクトしたときのみ rebase |
| `never` | 手動で要求しない限り rebase しない |
| `automerging` | automerge 設定時は `behind-base-branch`、それ以外は `never` |

:::message alert
自動 rebase を有効にした状態で Renovate PR が大量に滞留していると、base ブランチが更新されるたびに全 PR が rebase され、**CI (terraform plan) が大量に実行されてコストが跳ね上がる** 可能性があります。
:::

`rebaseWhen` の変更とあわせて、以下のオプションで PR の作成数を制限し、rebase 起因で同時に走る CI の数を抑えるとよいでしょう。
例えば、`prConcurrentLimit`というオプションを設定することで、同時に open できる PR 数に上限を設けることができます。

```json
{
  "rebaseWhen": "behind-base-branch",
  "prConcurrentLimit": 5
}
```

### 3. アップデートで必ず発生する plan 差分を許容する

Lambda (AWS) や Cloud Functions (Google Cloud) のソースコードを `archive_file` でパッケージングして管理している場合、ソースコードのライブラリをアップデートするとアーカイブのハッシュが変わるため、**毎回 `terraform plan` に差分が発生** します。

tfaction には、Renovate が作成した PR で plan 結果に差分がある場合に CI を失敗させ、意図しない変更が自動マージされないようにする仕組みがあります。
これ自体は安全のための仕組みですが、ライブラリ更新で必ず差分が出るディレクトリでは毎回 CI が失敗し、自動マージが止まって滞留の要因になっていました。

tfaction では、PR に **`renovate-change` ラベル** を付与すると plan 差分を許容して CI を通す仕様になっています。
この仕様を利用し、特定のアップデートで必ず発生する差分については、Renovate の `addLabels` で `renovate-change` ラベルを付与するようにしました。

```json
// 例: Cloud Functions の archive_file を含むディレクトリを指定し、
//     ライブラリ (Go) のアップデート PR に renovate-change ラベルを付与する
{
  "matchManagers": ["gomod"],
  "matchFileNames": ["terraform/google-cloud/**"],
  "addLabels": ["renovate-change"]
}
```

:::message
`renovate-change` ラベルを付与すると plan 差分があっても CI が通るようになるため、対象は「許容可能な差分が発生することが明らかなアップデート」に絞り込むのが安全です。
`matchManagers` や `matchFileNames` で範囲を限定し、Terraform のリソース定義そのものに影響するアップデートには付与しないようにしましょう。
:::

## まとめ

- **PR の粒度**: `additionalBranchPrefix` に `baseDir` を含めるとディレクトリごとに PR が作成される。`additionalBranchPrefix` に `replace` ヘルパーで抽出した上位ディレクトリ名を指定し、適切な単位にまとめる
- **rebase の方針**: ブランチ保護で「最新であること」を要求している場合は `rebaseWhen: behind-base-branch` にする。ただし CI コスト増を避けるため `prConcurrentLimit` / `prHourlyLimit` もあわせて設定する
- **plan 差分の許容**: `archive_file` などで必ず差分が出るアップデートには、tfaction の `renovate-change` ラベルを `addLabels` で付与する

本記事がRenovate PR の滞留に悩んでいる方の参考になれば幸いです。

## 参考資料

- [Configuration Options - Renovate Docs](https://docs.renovatebot.com/configuration-options/)
- [Noise Reduction - Renovate Docs](https://docs.renovatebot.com/noise-reduction/)
- [suzuki-shunsuke/tfaction - GitHub](https://github.com/suzuki-shunsuke/tfaction)
