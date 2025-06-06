---
title: "ECS Fargate上のWebアプリでマルチステージング環境を実現する"
emoji: "🤹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ECS", "AWS", "Tech", "マルチステージング"]
published: true
---

## やりたいこと

ECS Fargate でホスティングしているアプリケーションでマルチステージング環境を構築します。

`[commit_hash].domain.jp` といった形で、コミットハッシュをサブドメインとしたドメインにアクセスするとそれに対応したバージョンのアプリケーションにアクセスできるようになります。

それによって複数人開発で開発環境をコミットごとに分けて利用できるようになります！

## 実際に試してみる

### 前提

- CloudFront → ALB → ECS で構築されている
- domain.jp というドメインでホスティングされる
- ドメインのレコード等は Route53 で管理しており、 `domain.jp` `*.domain.jp` それぞれの A レコードが登録されている。
- A レコードに対するエイリアスとして CloudFront のディストリビューションが設定されている

### アプリケーションの中身(Nginx)を用意する

今回 ECS Fargate 上で動作させるアプリケーションの中身は簡単な HTML を表示する Nginx の Docker イメージとします。

アプリケーションリポジトリのディレクトリ

```
.
├── .github
│   ├── terraform
│   │   ├── data.tf
│   │   ├── locals.tf
│   │   ├── main.tf
│   │   ├── provider.tf
│   │   └── variables.tf
│   └── workflows
│       └── deploy.yml
├── Dockerfile
├── nginx.conf
└── task_definitions
    └── sample-app.json
```

**nginx.conf**

```conf
events {}

http {
  server {
    listen 80;
    location / {
    default_type text/plain;
    return 200 "Hello, Nginx!\n";
    }
  }
}
```

**Dockerfile**

```dockerfile
FROM nginx:alpine

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**タスク定義**

```json
{
  "containerDefinitions": [
    {
      "name": "sample-app",
      "image": "000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/sample-app:latest",
      "portMappings": [
        {
          "name": "sample-app--80-tcp",
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [],
      "mountPoints": [],
      "volumesFrom": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/sandbox-rails-app",
          "mode": "non-blocking",
          "awslogs-create-group": "true",
          "max-buffer-size": "25m",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        },
        "secretOptions": []
      },
      "systemControls": []
    }
  ],
  "family": "sample-app",
  "taskRoleArn": "arn:aws:iam::000000000000:role/EcsRole",
  "executionRoleArn": "arn:aws:iam::000000000000:role/EcsRole",
  "networkMode": "awsvpc",
  "volumes": [],
  "placementConstraints": [],
  "compatibilities": ["EC2", "FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "tags": []
}
```

### デプロイ用のパイプライン (GitHub Actions)を用意する

GitHub Actions で以下のようなデプロイのパイプラインを構築します。

- 特定のブランチ上の変更内容をビルドして ECR にイメージをプッシュする
- ALB のリスナールール・ターゲットグループ・ECS サービスを既存の ALB, ECS クラスター上に作成する (Terraform を実行)
- 上記で作成した環境に変更内容を反映した ECS タスク定義をデプロイする

```yaml
name: Deploy ECS Service for multi-staging

on:
  workflow_dispatch:

env:
  AWS_ROLE_ARN: arn:aws:iam::000000000000:role/deploy-role
  AWS_REGION: ap-northeast-1
  ECR_REPOSITORY: sample-app
  ECS_TASK_DEF_FILE: sample-app.json
  ECS_CLUSTER: sample-app
  CONTAINER_NAME: sample-app
  TERRAFORM_VERSION: 1.10.5

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push-to-ecr:
    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ steps.set_image_tag.outputs.IMAGE_TAG }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: deploy-session

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
        with:
          mask-password: "true"

      - name: Set IMAGE_TAG
        id: set_image_tag
        run: echo "IMAGE_TAG=$(date +%Y%m%d)-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Build and push
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker buildx build \
          -f Dockerfile \
          -t ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ steps.set_image_tag.outputs.IMAGE_TAG }} \
          --push \
          --provenance=false \
          .

  create-multi-staging-resources:
    needs: build-and-push-to-ecr
    runs-on: ubuntu-latest
    outputs:
      service_name: ${{ steps.define-service-name.outputs.SERVICE_NAME }}
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Format Subdomain
        id: format-subdomain
        run: |
          SUBDOMAIN=$(git rev-parse --short HEAD)
          echo "SUBDOMAIN=$SUBDOMAIN" >> "$GITHUB_OUTPUT"

      - name: Define Service Name
        id: define-service-name
        run: |
          echo "SERVICE_NAME=app-${{ steps.format-subdomain.outputs.SUBDOMAIN }}" >> $GITHUB_OUTPUT

      - name: Echo Subdomain
        run: echo "Subdomain is ${{ steps.format-subdomain.outputs.SUBDOMAIN }}"

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Initialize Terraform
        working-directory: .github/terraform/
        run: terraform init

      - name: Apply Terraform Configuration
        working-directory: .github/terraform/
        run: |
          terraform apply -auto-approve \
            -var="subdomain=${{ steps.format-subdomain.outputs.SUBDOMAIN }}" \
            -var="listener_priority=$((100 + RANDOM % 900))" \
            -var="service_name=${{ steps.define-service-name.outputs.SERVICE_NAME }}"

  deploy-to-ecs:
    needs: [build-and-push-to-ecr, create-multi-staging-resources]
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: deploy-session

      - uses: aws-actions/amazon-ecr-login@v1
        id: login-ecr
        with:
          mask-password: "true"

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def-be
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ./task_definitions/${{ env.ECS_TASK_DEF_FILE }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{needs.build-and-push-to-ecr.outputs.image_tag}}

      - name: Registers ECS task definition and deploys ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def-be.outputs.task-definition }}
          service: ${{ needs.create-multi-staging-resources.outputs.SERVICE_NAME }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

### アプリケーションのリポジトリ内に terraform ファイルを用意する

:::message
ECS クラスター・タスク定義・ALB・リスナー・VPC・セキュリティグループはすでに別途構築されているものとします。
:::

**data.tf**

```hcl
data "aws_ecs_cluster" "existing_cluster" {
  cluster_name = "sample-app"
}

data "aws_ecs_task_definition" "existing_task" {
  task_definition = "sample-app"
}

data "aws_lb" "existing_lb" {
  name = "sample-app"
}

data "aws_lb_listener" "https_443" {
  load_balancer_arn = data.aws_lb.existing_lb.arn
  port              = 443
}

data "aws_security_group" "alb_sg" {
  id = "sg-XXXXXXXXX" # 既存のALB用SG
}

data "aws_security_group" "ecs_sg" {
  id = "sg-YYYYYYYYY" # 既存のECS用SG
}

data "aws_vpc" "default" {
  filter {
    name   = "tag:Name"
    values = ["default"]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }

  tags = {
    Tier = "private"
  }
}

```

**main.tf**

```hcl
# ECS service
resource "aws_ecs_service" "default" {
  name            = var.service_name
  cluster         = data.aws_ecs_cluster.existing_cluster.id
  task_definition = data.aws_ecs_task_definition.existing_task.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    assign_public_ip = true # 検証用ではpublic subnet
    subnets          = data.aws_subnets.private.ids
    security_groups  = [data.aws_security_group.ecs_sg.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.http_80.arn
    container_name   = "sample-app"
    container_port   = 80
  }
}

# ALB
resource "aws_lb_target_group" "http_80" {
  name        = "multi-staging-${var.subdomain}"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = data.aws_vpc.default.id
  target_type = "ip"

  health_check {
    enabled = true
    path    = "/"
  }
}

resource "aws_lb_listener_rule" "host_based" {
  listener_arn = data.aws_lb_listener.https_443.arn
  priority     = var.listener_priority

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.http_80.arn
  }

  condition {
    host_header {
      values = ["${var.subdomain}.${var.domain_name}"]
    }
  }
}

```

**variables.tf**

```hcl
variable "listener_priority" {
  type        = number
  description = "Priority for the ALB listener rule"
  default     = 100
}

variable "domain_name" {
  default = "domain.jp"
}

variable "subdomain" {
  type = string
  description = "Commit Hash"
}

variable "service_name" {
  type = string
}

```

上記 Terraform を GitHub Actions で実施してマルチステージングかに必要なリソースをプロビジョニングします。

最新のコミットハッシュが `a123456` だとしたら、以下のリソースが作成されます

- ECS サービス
  - `a123456-sample-app`
- ALB ターゲットグループ
  - `multi-staging-a123456`
- リスナールール
  - サブドメインが`a123456` の場合、上記のターゲットグループにルーティングする

## 実際に動かして試してみる

適当にブランチを切って、nginx.conf の内容を変えてみます

```conf
events {}

http {
  server {
    listen 80;
    location / {
      default_type text/plain;
      return 200 "This is another version!\n";
    }
  }
}
```

表示される文字列を Hello, Nginx!から変えてみました。

これで GitHub Actions のワークフローを実行してみます。

AWS コンソール > ECS から名称にコミットハッシュのついたサービスがデプロイされていることを確認

![](/images/ecs-app-multi-staging/ecs_service.png)

`domain.jp` にアクセスしてみる

![](/images/ecs-app-multi-staging/hello_nginx.png)

`0adda9f.domain.jp` にアクセスすると

![](/images/ecs-app-multi-staging/another_version.png)
別バージョンであることが確認できます！

## 課題点

### リソースが増える分コストが増える

マルチステージング化によって ECS サービスが増えるので、その分追加のコストがかかってしまいます。

Fargate Spot という余剰リソースを使ってコンテナサービスをプロビジョニングできるので、それを利用するのが良いかなと思います

https://aws.amazon.com/jp/blogs/news/aws-fargate-spot-now-generally-available/

### リソースを手動で削除する必要がある

GitHub Actions 上で Terraform を実行し、state は保存しないようにしているため、不要になったリソースは都度手動で削除する必要があります。

マルチステージングで作成されるリソースに特定のタグをつけて、Lambda + EventBridge で定期的に削除する設定などを追加で入れてあげると良いかもしれません。
