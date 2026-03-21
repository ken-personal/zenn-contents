title: “ECS Fargate + Terraform + GitHub Actions で本番想定のCI/CDパイプラインをゼロから構築した話【OIDC・Trivy・監視基盤まで】”
emoji: “🚀”
type: “tech”
topics: [“aws”, “terraform”, “githubactions”, “ecs”, “nestjs”]
published: true
はじめに
本記事では、Next.js / NestJS / PostgreSQL で構成した社内ポータルシステムをAWS上にデプロイするまでの設計・構築・自動化の全工程を解説します。
この記事で学べること：
	∙	ECS Fargate をPrivate Subnetに配置したセキュアな3層アーキテクチャの設計
	∙	TerraformによるAWSインフラのモジュール分割IaC化
	∙	GitHub ActionsでのOIDC認証（アクセスキー不要）の実装
	∙	Trivyセキュリティスキャンをパイプラインに組み込む方法
	∙	CloudWatch / EventBridge / SNSによる監視・アラート基盤の構築
対象読者：
	∙	AWSでコンテナ環境を構築したい方
	∙	GitHub ActionsでECSへのCI/CDを実装したい方
	∙	Terraformをモジュール分割で管理したい方
