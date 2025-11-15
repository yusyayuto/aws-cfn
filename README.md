# AWS CloudFormation Templates

AWS CloudFormationテンプレート集です。各種AWSリソースを効率的にデプロイするためのテンプレートを提供します。

## 概要

このリポジトリには、以下のAWSリソースを作成するためのCloudFormationテンプレートが含まれています：

- **ネットワーク**: VPC, サブネット, ルートテーブル
- **コンピューティング**: EC2, ECS Cluster, ECS Service, Auto Scaling
- **データベース**: RDS
- **ロードバランサー**: ELB/ALB
- **コンテナ**: ECR, ECS
- **セキュリティ**: IAM Role, IAM User, IAM Access Analyzer
- **DNS**: Route53
- **証明書**: ACM (Certificate Manager)
- **CDN**: CloudFront
- **ログ**: CloudWatch Logs

## テンプレート一覧

### 基盤リソース

| テンプレート | 説明 | 依存関係 |
|------------|------|---------|
| `vpc.yml` | VPC、サブネット、インターネットゲートウェイ、ルートテーブル、VPCエンドポイントなどを作成 | なし |
| `iam-role.yml` | IAMロールを作成 | なし |
| `iam-user.yml` | IAMユーザーを作成 | なし |
| `iam-accessanalyzer.yml` | IAM Access Analyzerを作成 | なし |
| `route53.yml` | Route53ホストゾーンとレコードを作成 | なし |
| `certificatemanager.yml` | ACM証明書を作成 | なし |
| `ecr.yml` | ECRリポジトリを作成 | なし |

### アプリケーションリソース

| テンプレート | 説明 | 依存関係 |
|------------|------|---------|
| `rds.yml` | RDS DBクラスターとインスタンスを作成 | `iam-role`, `vpc` |
| `elb.yml` | Application Load Balancerを作成 | `vpc`, `certificatemanager` |
| `ec2.yml` | EC2インスタンスを作成 | `route53`, `vpc`, `rds`, `elb` |
| `ec2-asg.yml` | EC2 Auto Scalingグループを作成 | `iam-role`, `vpc`, `rds`, `elb` |
| `ecs-cluster.yml` | ECSクラスター（EC2起動テンプレートとAuto Scalingグループ）を作成 | `iam-role`, `vpc`, `rds`, `elb` |
| `ecs-service.yml` | ECSサービスとタスク定義を作成 | `vpc`, `elb`, `ecs-cluster` |
| `cloudfront.yml` | CloudFrontディストリビューションを作成 | 環境による |
| `logs-delivery_us-east-1.yml` | CloudWatch Logsの配信設定（us-east-1リージョン用） | なし |

## 使用方法

### 前提条件

- AWSアカウント
- AWS CLIまたはマネジメントコンソールへのアクセス
- 適切なIAM権限

### デプロイ順序

リソース間に依存関係があるため、以下の順序でデプロイすることを推奨します：

1. **基盤リソース**
   ```bash
   # IAMロール
   aws cloudformation create-stack \
     --stack-name iam-role \
     --template-body file://iam-role.yml \
     --capabilities CAPABILITY_NAMED_IAM

   # VPC
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-vpc \
     --template-body file://vpc.yml \
     --parameters ParameterKey=SystemName,ParameterValue=awsmaster \
                  ParameterKey=Environment,ParameterValue=prod

   # Route53
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-route53 \
     --template-body file://route53.yml

   # Certificate Manager
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-acm \
     --template-body file://certificatemanager.yml
   ```

2. **データベース**
   ```bash
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-rds \
     --template-body file://rds.yml \
     --parameters ParameterKey=SystemName,ParameterValue=awsmaster \
                  ParameterKey=Environment,ParameterValue=prod
   ```

3. **ロードバランサー**
   ```bash
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-elb \
     --template-body file://elb.yml \
     --parameters ParameterKey=SystemName,ParameterValue=awsmaster \
                  ParameterKey=Environment,ParameterValue=prod
   ```

4. **コンピューティングリソース**
   ```bash
   # ECS Cluster
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-ecs-cluster \
     --template-body file://ecs-cluster.yml \
     --parameters ParameterKey=SystemName,ParameterValue=awsmaster \
                  ParameterKey=Environment,ParameterValue=prod

   # ECS Service
   aws cloudformation create-stack \
     --stack-name awsmaster-prod-ecs-service \
     --template-body file://ecs-service.yml \
     --parameters ParameterKey=SystemName,ParameterValue=awsmaster \
                  ParameterKey=Environment,ParameterValue=prod
   ```

### パラメータのカスタマイズ

各テンプレートには、以下の共通パラメータがあります：

- **SystemName**: システム名（デフォルト: `awsmaster`）
- **Environment**: 環境名（`prod`, `stg`, `dev`）

テンプレートファイル内の `[Change System Name]` コメントを参考に、システム名を変更してください。

### 環境別設定

テンプレートは3つの環境をサポートしています：

- **prod**: 本番環境
- **stg**: ステージング環境
- **dev**: 開発環境

環境ごとに異なるリソースサイズやCIDRブロックが設定されています。

## ネットワーク構成

### VPC CIDR設計

各環境には専用のCIDRブロックが割り当てられています：

- **prod**: `10.0.0.0/19`
  - Public: `10.0.0.0/24` - `10.0.3.0/24`
  - Protected: `10.0.4.0/24` - `10.0.7.0/24`
  - Private: `10.0.8.0/24` - `10.0.11.0/24`

- **stg**: `10.0.32.0/19`
  - Public: `10.0.32.0/24` - `10.0.35.0/24`
  - Protected: `10.0.36.0/24` - `10.0.39.0/24`
  - Private: `10.0.40.0/24` - `10.0.43.0/24`

- **dev**: `10.0.64.0/19`
  - Public: `10.0.64.0/24` - `10.0.67.0/24`
  - Protected: `10.0.68.0/24` - `10.0.71.0/24`
  - Private: `10.0.72.0/24` - `10.0.75.0/24`

## 主な機能

### Auto Scaling

ECSクラスターには以下のスケジュールアクションが設定されています：

- **本番環境**
  - スケールアウト: 月-金 07:00 JST
  - スケールイン: 月-金 23:00 JST

- **非本番環境**
  - 起動: 月-金 07:50 JST
  - 停止: 月-金 22:00 JST

### セキュリティ

- IAMロールとポリシーによるアクセス制御
- セキュリティグループによるネットワーク制御
- ACM証明書によるSSL/TLS通信
- IAM Access Analyzerによるアクセス分析

## 注意事項

1. **AZリバランス**: ECS ClusterのAuto Scalingグループでは、以下のコマンドでAZリバランスを無効化することを推奨します：
   ```bash
   aws autoscaling suspend-processes \
     --scaling-processes AZRebalance \
     --auto-scaling-group-name <auto-scaling-group-name>
   ```

2. **依存関係**: 各テンプレートのコメントに記載されている依存関係を確認し、正しい順序でスタックを作成してください。

3. **リソース削除**: スタックを削除する際は、依存関係の逆順で削除してください。

4. **コスト**: リソースを起動すると課金が発生します。不要なリソースは削除してください。

## ライセンス

MITライセンス

## 貢献

プルリクエストを歓迎します。大きな変更の場合は、まずissueを開いて変更内容を議論してください。
