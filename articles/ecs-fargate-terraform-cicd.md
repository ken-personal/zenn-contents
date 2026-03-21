---
title: "ECS Fargate + Terraform + GitHub Actions で本番想定のCI/CDパイプラインをゼロから構築した話【OIDC・Trivy・監視基盤まで】"
emoji: "🚀"
type: "tech"
topics: ["aws", "terraform", "githubactions", "ecs", "nestjs"]
published: true
---

## はじめに

本記事では、Next.js / NestJS / PostgreSQL で構成した社内ポータルシステムをAWS上にデプロイするまでの設計・構築・自動化の全工程を解説します。

**この記事で学べること：**
- ECS Fargate をPrivate Subnetに配置したセキュアな3層アーキテクチャの設計
- TerraformによるAWSインフラのモジュール分割IaC化
- GitHub ActionsでのOIDC認証（アクセスキー不要）の実装
- Trivyセキュリティスキャンをパイプラインに組み込む方法
- CloudWatch / EventBridge / SNSによる監視・アラート基盤の構築

**対象読者：**
- AWSでコンテナ環境を構築したい方
- GitHub ActionsでECSへのCI/CDを実装したい方
- Terraformをモジュール分割で管理したい方

---

## アーキテクチャ全体図

**技術スタック：**

| カテゴリ | 技術 |
|---------|------|
| Frontend | Next.js / TypeScript / Tailwind CSS |
| Backend | NestJS / Prisma ORM / JWT認証 |
| DB | RDS PostgreSQL Multi-AZ |
| IaC | Terraform（モジュール分割） |
| CI/CD | GitHub Actions（OIDC / Trivy / ECSデプロイ） |
| 監視 | CloudWatch / EventBridge / SNS / SQS |

---

## インフラ設計のポイント

### 1. セキュリティ設計

**ECS FargateをすべてPrivate Subnetに配置しました。**
パブリックIPを持たせず、ALB経由のみアクセス可能にすることで、AWS Well-Architectedフレームワークのセキュリティの柱に準拠した構成にしています。

Security Groupは用途別に5つ作成し、最小権限の通信制御を実現しました。

**GitHub ActionsのAWS認証にOIDCを採用した理由：**

従来のアクセスキー方式は、キーが漏洩すると即座に不正アクセスされるリスクがあります。OIDCを使うことで一時的な認証情報のみ発行され、アクセスキーをコードやSecretsに保存する必要がなくなります。

### 2. 可用性設計

- ALBによるロードバランシング（ヘルスチェック付き）
- RDS PostgreSQL Multi-AZ構成（自動フェイルオーバー）
- ECS Auto Scaling（CPU使用率70%超で自動スケールアウト）
- ECSローリングデプロイ（ダウンタイムなし）

### 3. コスト最適化

NAT GatewayはAZごとに作ると費用がかさむため、今回は1つに絞りました。またVPC Endpoint（ECR / CloudWatch / S3）を導入することでNAT Gateway経由の通信を削減しています。
---

## Terraform実装：モジュール分割構成

インフラ全体をTerraformでコード管理しています。モジュール分割により、再利用性・保守性を高めました。
terraform/
└── modules/
├── vpc/            # VPC・Subnet×6・IGW・NAT GW・VPC Endpoint
├── security_group/ # SG × 5
├── iam/            # ECS実行ロール・タスクロール
├── ecr/            # コンテナイメージリポジトリ
├── acm/            # SSL/TLS証明書（DNS検証）
├── route53/        # DNSレコード管理
├── alb/            # ALB・ターゲットグループ・Listener
├── ecs/            # Cluster・TaskDef・Service・AutoScaling
├── rds/            # PostgreSQL Multi-AZ・Secrets Manager
└── monitoring/     # CloudWatch・CloudTrail・EventBridge・SNS・SQS

DBパスワードはSecrets Managerで管理し、Terraformのtfstateにパスワードが残らないようにしています。

---

## GitHub Actions CI/CDパイプライン

コードの変更から本番デプロイまでを完全自動化しています。変更があったフォルダのみをビルド・デプロイするモノリポ対応設計にしました。

### CIの流れ（PRを作成したとき）

PRを作成するとESLint・TypeScript型チェック・Jest・Prismaスキーマ検証が自動で走ります。

### CDの流れ（mainへのマージ時）

mainへのマージ後、OIDC認証→Dockerビルド→Trivyスキャン→ECSデプロイ→SNS通知の順で自動実行されます。

### Trivyセキュリティスキャンをパイプラインに組み込む

HIGH・CRITICALの脆弱性が検出された場合、デプロイを自動停止します。

---

## 監視基盤の構築

| アラーム | 閾値 | 通知先 |
|---------|------|--------|
| ECS CPU使用率 | 80%超 | SNS |
| RDS CPU使用率 | 80%超 | SNS |
| RDS空きメモリ | 256MB以下 | SNS |
| ALB 5xxエラー | 1分間に10件超 | SNS |

EventBridgeでECSタスクの異常停止を検知し、SNS+SQSでアラートメール通知とキューイングを行っています。

---

## 詰まったポイントと解決策

### 1. Docker内部ネットワークとPrismaの接続エラー

Next.jsがブラウザとSSRの両方からAPIを叩くため、環境変数でホスト名を使い分けることで解決しました。

### 2. パスワード二重ハッシュ化の問題

bcryptによるハッシュ化がフロントとバックの両方で行われていたため、バックエンドのみに統一しました。

### 3. Prisma Schema・型・実DBの不整合

`npx prisma generate`→`npx prisma migrate dev`の順番を徹底することで解決しました。

---

## まとめ

- セキュリティはアーキテクチャで担保する（OIDC・Private Subnet）
- IaCは最初からモジュール分割する
- CI/CDにセキュリティスキャンを組み込む

---

## ソースコード・デモ

- GitHub：https://github.com/ken-personal/fullstack-internal-portal
- デモ：https://frontend-seven-mu-71.vercel.app

AWSインフラ・CI/CD・フルスタック開発に関するご質問があればコメントください。
