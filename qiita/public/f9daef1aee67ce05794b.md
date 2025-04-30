---
title: 【CodePipeline】ECRにpushしてECSに最新イメージをデプロイするパイプラインを構築
tags:
  - AWS
  - ECS
  - CodePipeline
private: false
updated_at: '2022-12-17T14:04:38+09:00'
id: f9daef1aee67ce05794b
organization_url_name: null
slide: false
ignorePublish: false
---
## 概要
ECRにイメージをpushしたら、ECSサービスが更新され、最新イメージがデプロイされるパイプラインをCodePipelineで実装します。
Code Build, Code Commitなどは今回は使わず、シンプルな構成で作ってみます。


## ECR / コンテナの用意
### ECRでリポジトリを作成する

### ローカルでDockerfileを作成
（今回はapacheのイメージを使って簡単に作成します）

```dockerfile
FROM httpd:latest
```

ECRにプッシュ
プッシュコマンドはECRのリポジトリで見られます。

![スクリーンショット 2022-12-17 12.57.59.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/36e8952d-e598-7c3a-21f7-5781553f5cf8.png)

``` 
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin [AWSアカウントID].dkr.ecr.ap-northeast-1.amazonaws.com
docker build -t [リポジトリ名] .
docker tag [リポジトリURI]
docker push [リポジトリURI]
```

(M1 Macを使っている場合、うまくデプロイできないことがあるので`--platform`オプションをつける)

dockerファイルを編集
```dockerfile
FROM --platform=amd64 httpd:latest
```
もしくはビルドコマンドにオプションをつける
```
docker build -t [リポジトリ名] . --platform amd64
```
## ECS
クラスター・サービス・タスクを作成する

Terraformでパパッと作成すると楽です

```tf
# タスク定義
resource "aws_ecs_task_definition" "task" {
  family                   = "sandbox-cicd-task"
  #0.25vCPU
  cpu                      = "256"
  #0.5GB
  memory                   = "512"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  container_definitions    = file("./container_definitions.json")
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
}

# クラスター
resource "aws_ecs_cluster" "cluster" {
  name = "sandbox-cicd-cluster"
}

# サービス
resource "aws_ecs_service" "service" {
  name                              = "sandbox-cicd-service"
  cluster                           = aws_ecs_cluster.cluster.arn
  task_definition                   = aws_ecs_task_definition.task.arn
  desired_count                     = 1
  launch_type                       = "FARGATE"
  platform_version                  = "1.4.0"

  network_configuration {
    assign_public_ip = true
    security_groups  = [aws_security_group.service.id]
    subnets = module.vpc.public_subnets
  }

  lifecycle {
    ignore_changes = [task_definition]
  }
}

resource "aws_security_group" "service" {
  name        = "httpd-sg"
  description = "httpd-sg"
  vpc_id      = module.vpc.vpc_id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "httpd-sg"
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.10.0"

  name = "sandbox-cicd-vpc"
  cidr                 = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  azs             = ["ap-northeast-1a", "ap-northeast-1c"]
  public_subnets  = ["10.0.11.0/24", "10.0.12.0/24"]

  manage_default_security_group  = true
  default_security_group_ingress = []
  default_security_group_egress  = []
}

```

コンテナ定義

```json
[
    {
        "name": "httpd-container",
        "image": "httpd:latest",
        "essential": true,
        "memory": 128,
        "portMappings": [
            {
                "protocol": "tcp",
                "containerPort": 80
            }
        ],
        "logConfiguration": {
            "logDriver": "awslogs",
            "options": {
                "awslogs-group": "sandbox-cicd-container-log",
                "awslogs-region": "ap-northeast-1",
                "awslogs-create-group": "true",
                "awslogs-stream-prefix": "firelens"
            }
        }
    }
]

```

タスク実行ロールの作成

```tf
data "aws_iam_policy" "ecs_task_execution_role_policy_source" {
    arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ロググループを生成するポリシーをアタッチする
data "aws_iam_policy_document" "ecs_task_execution_role_policy_document" {
    source_json = data.aws_iam_policy.ecs_task_execution_role_policy_source.policy
    statement {
        effect    = "Allow"
        actions   = ["logs:CreateLogGroup"]
        resources = ["*"]
    }
}


resource "aws_iam_role" "ecs_task_execution_role" {
    name               = "sandbox-cicd-task-execution-role"
    assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

resource "aws_iam_policy" "ecs_task_execution_role_policy" {
    name   = "sandbox-cicd-task-execution-role-policy"
    policy = data.aws_iam_policy_document.ecs_task_execution_role_policy_document.json
}

data "aws_iam_policy_document" "assume_role" {
    statement {
        actions = ["sts:AssumeRole"]
        principals {
        type        = "Service"
        identifiers = ["ecs-tasks.amazonaws.com"]
        }
    }
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_attachment" {
    role       = aws_iam_role.ecs_task_execution_role.name
    policy_arn = aws_iam_policy.ecs_task_execution_role_policy.arn
}

```


## CodePipeline

ECRへのプッシュをトリガーに作動するパイプラインを作成します。

### サービスロールの作成
今回作成するパイプラインはS3バケットのオブジェクトにアクセスする必要があるため、S3へのアクセスポリシーを付与したサービスロールを作成します。



### Source

![スクリーンショット 2022-12-17 13.06.58.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/7a3a6199-96bb-64de-9b92-7f8f4c9f5bae.png)

SourceとしてECRのリポジトリを指定します。
また、デプロイ対象のコンテナとイメージURIを紐づけるために`imagedefinition.json`を用意します。
```imagedefinition.json
[
    {
        "name":"httpd-container",
        "imageUri":"XXXXXXXX.dkr.ecr.ap-northeast-1.amazonaws.com/cicd-sandbox-ecr:latest"
    }
]

```


上記のjsonファイルを格納する用のS3バケットを作成し、上記jsonファイルをzipで圧縮してアップロードします。
この時作成するバケットは**バージョニングを有効化**してください。

### Deploy
![スクリーンショット 2022-12-17 13.15.32.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/bdf39cc1-beb7-d34b-58b6-ac912ee6a1b4.png)

Deploy StageではECSで最新イメージをデプロイするアクションを定義し、以下の設定を行います。

**入力アーティファクト**
SourceステージのS3バケットの出力アーティファクト

**イメージ定義ファイル**
imagedefinitions.json


## イメージを更新してECRにpush
`docker-compose.yml`を作っておくとリビルドができて便利

```docker-compose.yml
services:
  httpd:
    image: XXXXXXX.dkr.ecr.ap-northeast-1.amazonaws.com/cicd-sandbox-ecr # ECRのリポジトリURI
    platform: amd64
    build:
      context: .
      dockerfile: Dockerfile
```

ECRにpushするとパイプラインが作動するのを確認できる
![スクリーンショット 2022-12-17 13.40.24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/3f3cf361-6c3f-1d73-006a-7230084e459f.png)

ECS > サービスで新規リビジョンが作成されているのを確認
![スクリーンショット 2022-12-17 11.39.07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/328a6fbd-185a-51c8-5a4d-e408faf84570.png)


サービスが新規リビジョンに切り替わったらデプロイ成功

![スクリーンショット 2022-12-17 12.09.35.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/689205/fab6eef3-14a3-ae8f-6006-6f59f1bab41f.png)


## うまくいかない時確認すること

- imagedefinitions.jsonが正しく圧縮されているか
- 権限周りのエラー
    - CodePipelineのサービスロールにS3バケットのオブジェクトに対するアクセス権限が付与されているか
    - ECRへのアクセス権限を持つ**タスク実行ロール**がタスク定義にアタッチされているか
- デプロイに何度も失敗する
    - コンテナログを確認する


## 参考
https://dev.classmethod.jp/articles/ecr_push_ecs_simple_codepipeline/

https://hikari-blog.com/exec-format-error/
