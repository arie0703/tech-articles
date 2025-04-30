---
title: 【Terraform】色々な条件分岐方法メモ
tags:
  - Terraform
private: false
updated_at: '2022-12-16T19:43:05+09:00'
id: 254e2adcade3bbac1156
organization_url_name: null
slide: false
ignorePublish: false
---
## 概要


Terraformにはif文がない。
しかし、環境ごとに値を変えたいなど条件分岐的な記述をしたい時がある。
Terraformでは、三項演算子と特定のプロパティを組み合わせることによって、条件分岐的な記述が可能になる。
## count + 三項演算子
countの数だけリソースが作成され、0ならリソースを作成しない。  
開発環境によってcountの値を変化させ、リソース作成の有無を分岐させるなどといった使い方ができる。

(例)
```tf
resource "aws_s3_bucket" "b" {

    count  = var.env == "prod" ? 0 : 1
    bucket = "s3-website-test.hashicorp.com"
    acl    = "public-read"
    policy = file("policy.json")

}
```


## dynamic + for_each

一部リソースではcountが使えないこともある。
そこで`dynamic`ブロックと`for_each`を使ってリソースの作成数を分岐させる方法もある。

```tf
variable list = []
```

```tf
resource "example" "hoge" {

    dynamic attribute {
        for_each = var.list
        
        content {
            attribute1 = "hoge"
            ...
        }
    }
}

```

三項演算子と組み合わせる場合

```tf
resource "example" "hoge" {

    dynamic attribute {
        for_each = var.fuga ? [] : [1]
        
        content {
            attribute1 = "hoge"
            ...
        }
    }
}

```

for_eachが空配列なら、countが0の時のようにリソースが作成されない。
## contains + 三項演算子

配列`list`の中に`value`が含まれているかで分岐
```tf
attribute = cointains(list, value) : valiue ? null
```


#### map内に特定のkeyがあるかどうかで判定

以下のような変数があるとする

```tf
variable item {
    hoge = "fuga"
}
```

```tf
attribute = contains(keys(item.value), "hoge") ? item.value["hoge"] : null
```

変数itemの中にhogeというkeyが存在するかどうかで判定
→ 存在しなければ**nullを代入**


## 参考
https://developer.hashicorp.com/terraform/language/meta-arguments/count

https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks

https://developer.hashicorp.com/terraform/language/functions/contains
