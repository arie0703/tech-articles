---
title: ECSタスク上でTerraform × MySQL Providerを実行できる環境を作ってみる
private: false
tags:
  - ECS
  - AWS
  - Tech
  - SessionManager
  - Fargate
  - Terraform
updated_at: '2025-06-06T22:39:42.902Z'
id: null
organization_url_name: null
slide: false
---

## 概要

TerraformでMySQL Providerを用いてデータベースやユーザ管理をしており、`terraform apply`はローカル環境から踏み台サーバ(EC2)をポートフォワーディングでRDSに接続して実行しているとします。
その場合、ポートフォワーディング時にSSHでの通信を介すため、よりセキュアな環境でMySQL Providerを使いたいです。
また、EC2ではなくECS Fargateを使用することで運用コストの削減も図りたいです。

そうした課題を解決するため、今回はTerraform, MySQLプロバイダーを利用できるECSコンテナ環境を構築してみます。

## いざ構築

### 構成図

![構成図](/images/bastion-terraform-mysql/bastion_ecs.drawio.png)

今回はECS + 関連リソース(ECR, IAM)を構築します。

:::note

- VPC, サブネットの作成は省略します
- サブネットはパブリック・プライベートどちらかは問いません
  - アウトバウンド通信ができる必要があります(パブリックIPの割り当てorNATゲートウェイの利用が必須)  

:::

### Terraformでリソース構築

```hcl
resource "aws_ecr_repository" "bastion" {
  name = "bastion-container"
}

resource "aws_ecs_cluster" "bastion" {
  name = "bastion-cluster"
}

resource "aws_ecs_task_definition" "bastion_task" {
  family                   = "bastion-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = templatefile("${path.module}/container_definition.json", {
    ecr_image_url = aws_ecr_repository.bastion.repository_url
    log_group     = aws_cloudwatch_log_group.bastion.name
  })

  runtime_platform {
    operating_system_family = "LINUX"
  }
}

# ECSサービス
resource "aws_ecs_service" "bastion_service" {
  name            = "bastion-service"
  cluster         = aws_ecs_cluster.bastion.id
  task_definition = aws_ecs_task_definition.bastion_task.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = [data.aws_subnet.selected.id]
    security_groups  = [aws_security_group.bastion_sg.id]
    assign_public_ip = true # NATゲートウェイを使用する場合はfalseでOK
  }

  enable_execute_command = true
}

# セキュリティグループ
resource "aws_security_group" "bastion_sg" {
  name   = "bastion-sg"
  vpc_id = data.aws_vpc.selected.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Log Group (Optional)
resource "aws_cloudwatch_log_group" "bastion" {
  name              = "/ecs/bastion"
  retention_in_days = 7
}

```

### コンテナ定義

```json
[
  {
    "name": "bastion",
    "image": "${ecr_image_url}:latest",
    "essential": true,
    "command": [
      "sleep",
      "infinity"
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "${log_group}",
        "awslogs-region": "ap-northeast-1",
        "awslogs-stream-prefix": "bastion",
        "awslogs-create-group": "true"
      }
    }
  }
]
```

### IAMロール (タスクロール・タスク実行ロール)

```hcl
# ECS タスクロール
resource "aws_iam_role" "ecs_task_role" {
  name = "bastion-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

# ReadOnlyAccess ポリシー付与
resource "aws_iam_role_policy_attachment" "ecs_task_readonly" {
  role       = aws_iam_role.ecs_task_role.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

# CloudWatch Logsポリシー
resource "aws_iam_policy" "cloudwatch_logs" {
  name = "ecs-task-logs-policy"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogStreams"
        ],
        Resource = "${aws_cloudwatch_log_group.bastion.arn}:*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "cloudwatch_logs_attach" {
  role       = aws_iam_role.ecs_task_role.name
  policy_arn = aws_iam_policy.cloudwatch_logs.arn
}

# Terraform State更新・参照ポリシー
resource "aws_iam_policy" "ecs_task_s3_terraform" {
  name = "ecs-task-tfstate-policy"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ],
        Resource = ["arn:aws:s3:::arie-terraform-states", "arn:aws:s3:::arie-terraform-states/*"]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_s3_attach" {
  role       = aws_iam_role.ecs_task_role.name
  policy_arn = aws_iam_policy.ecs_task_s3_terraform.arn
}

# タスク実行ロール
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "bastion-ecs-task-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_exec_attach" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Task RoleにSSMのポリシーをアタッチする
resource "aws_iam_role_policy_attachment" "ecs_task_ssm_attach" {
  role       = aws_iam_role.ecs_task_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

```

タスク実行ロールはAWS管理のAmazonECSTaskExecutionRolePolicyを利用しておけばOK

タスクロールにはstateを保存しているS3バケット内のオブジェクト参照・追加できる権限をアタッチする必要があります。

MySQL Providerを利用する上ではその他の権限は不要です。

## Dockerfile

今回はAmazon Linux 2023のイメージを使用

```dockerfile
FROM amazonlinux:2023

ENV HOME=/root

RUN yum install -y \
    git \
    tar \
    gzip \
    make \
    unzip \
    shadow-utils && \
    usermod -aG wheel root && \
    git clone https://github.com/tfutils/tfenv.git ~/.tfenv && \
    echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc && \
    echo 'eval "$(tfenv init -)"' >> ~/.bashrc

# tfenv useを使う際にfindコマンドが必要
RUN dnf install -y findutils

WORKDIR /root/bastion-container

CMD [ "sleep", "infinity" ]
```

## ECSコンテナ上で`terraform apply`してみる

構築が完了してECSタスクが起動できたら、SessionManagerでECSコンテナに接続→terraform実行を試してみようと思います。（AWS SSM Agentは導入済みとします）

### ECS Execを実行

```bash
TASK_ID=xxxxxxxx
AWS_REGION=ap-northeast-1

aws ecs execute-command \
  --cluster bastion-cluster \
  --region ${AWS_REGION} \
  --task ${TASK_ID} \
  --container bastion \
  --interactive \
  --command "/bin/bash"
```

### tfenvの設定

```bash
# 好きなバージョンをインストールする
tfenv install 1.12.0

tfenv use 1.12.0
```

### mysql構築用のterraformファイルを作成

```hcl
# SecretsManagerにMySQLの接続情報を保存するものとします。
resource "aws_secretsmanager_secret" "mysql" {
  name = "mysql-secrets"
}

data "aws_secretsmanager_secret_version" "secret-version" {
  secret_id = aws_secretsmanager_secret.mysql.id
}

provider "mysql" {
  endpoint = jsondecode(data.aws_secretsmanager_secret_version.secret-version.secret_string)["MYSQL_HOST"]
  username = jsondecode(data.aws_secretsmanager_secret_version.secret-version.secret_string)["MYSQL_USERNAME"]
  password = jsondecode(data.aws_secretsmanager_secret_version.secret-version.secret_string)["MYSQL_PASSWORD"]
}

terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
    mysql = {
      source  = "petoju/mysql"
      version = "3.0.23"
    }
  }
  backend "s3" {
    bucket = "{Stateを保存しているS3バケット名}"
    region = "ap-northeast-1"
    key    = "mysql/terraform.tfstate"
  }
}

resource "mysql_database" "app" {
  name = "sample_db"
}

```

`terraform apply`してApply CompleteできればOK

## 試してみた中での気づきメモ

- CloudShellでも良さそうだが今回はFargateを採用
  - CloudShellのストレージ容量は1GBなので, terraformの実行環境としては物足りないため
- mysql providerは `mysql_native_password` プラグインを使っているぽい。
  - 現行のAurora MySQL 3系だとデフォルトの認証プラグインは`mysql_native_password`なので特に問題なさそう
  - RDS for MySQLは`caching_sha2_password`なので、認証プラグインの設定を変更する必要がありそう
  - EC2でMySQL 8.4をホスティングして検証した際はmy.cnfを編集してデフォルト認証プラグインを変更する必要があった。
