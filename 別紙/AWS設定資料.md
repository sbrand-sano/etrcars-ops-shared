---
title: ref_aws_settings
source: 別紙/別紙aws 設定資料 25_1031_JP修正.docx
status: draft
---

ETRCARS AWS設計書
作成日： 2025年10月31日
版数： Ver.1.0
作成者： CODIUM Co., Ltd.
共同作成： CODIUM Co., Ltd.
機密区分： 関係者限定／外部開示禁止

### 改訂履歴


| 改訂日 | 版数 | 改訂内容 | 作成者 |
|---|---|---|---|
| 2025/10/3１ | Ver.1.0 | 初版作成 | CODIUM Co., Ltd |
|  |  |  |  |

本ドキュメントは必要に応じて内容を見直し、最新の運用に合わせて随時更新する
目次
システム概要
構成図
開発サーバ構成図
AWSリソース構成
ネットワーク許可設定
権限設定
自動タスク設定
監視およびアラーム設定
緊急対応手順
運用手順
その他
1. システム概要
ETRCARSシステムは以下の構成要素から成るWebベースの業務アプリケーションです。
- バックエンドアプリケーション: Django（Python Web フレームワーク）
- フロントエンドアプリケーション: Angular（TypeScript / JavaScript フレームワーク）
- モバイルアプリケーション: Swift（iOS ネイティブアプリケーション）
- データベース: AWS RDS PostgreSQL（マネージドリレーショナルデータベース）
- ファイルストレージ: AWS S3（オブジェクトストレージサービス）、Glacier（アーカイブ用）
- コンテナレジストリ: Amazon ECR（Docker イメージを保存する Elastic Container Registry）
- キャッシュ: AWS ElastiCache（パフォーマンス最適化のためのインメモリキャッシュ）
- ログおよびモニタリング: AWS CloudWatch および AWS CloudTrail
- セキュリティおよびコンプライアンス: AWS GuardDuty、AWS Inspector、AWS Config、AWS Security Hub
- ロードバランシング: Application Load Balancer（ALB）を利用した AWS ECS
- ネットワーキング: Amazon VPC（プライベートおよびパブリックサブネットを使用）
- シークレット管理: AWS Secrets Manager
- リモートアクセス: AWS Systems Manager Session Manager
- メール通知: Amazon SES（Simple Email Service）
●　ID およびアクセス管理: AWS IAM、AWS IAM Identity Center
2. 構成図
2.1 本番環境構成
本番環境はCloudflare経由でALBを通過し、プライベートサブネット内のECSサービスへ到達する構成です。
![ref_aws_settings_img001.png](images/ref_aws_settings/ref_aws_settings_img001.png)
  主要コンポーネント:
  ●　Cloudflare CDN: グローバルなコンテンツ配信、DDoS 保護、エッジキャッシュを提供
  ●　Application Load Balancer (ALB): ECS タスクへのトラフィックを分散し、スケーラビリティと耐障害性を確保
  ●　ECS Fargate: サーバーレスで Django およびバックグラウンドワーカーアプリケーションを実行
  ●　Amazon ECR: GitLab CI/CD パイプラインでビルドおよびデプロイされた Docker イメージを保存・管理
●　RDS PostgreSQL: 高可用性と自動バックアップを備えたマルチ AZ のマネージドデータベース
●　ElastiCacheRedis:エクスポートやインポートなどの長時間処理を高速化するキャッシュレイヤー
●　Amazon S3: 画像、ファイル、その他の静的アセットを保存・取得
●　Amazon S3 Glacier: 長期的なバックアップや低頻度アクセスデータの低コスト保管
●　Amazon SES: トランザクションメールおよび通知メールの送信を担当
●　Amazon VPC（複数 AZ 構成）: 冗長性を備えた安全で隔離されたネットワーク環境
●　セキュリティグループ: インスタンスレベルの仮想ファイアウォールとして通信制御を実施
●　プライベート / パブリックサブネット: 公開システムと内部システムを分離し、セキュリティを強化
●　Network ACL: サブネットレベルでの通信制御を行い、外部とのトラフィックを制限
●　AWS Config / Inspector / GuardDuty / Security Hub: 構成の監視、脆弱性検出、脅威の特定、コンプライアンス遵守を継続的に実施
追加構成項目
●	別リージョンへのリアルタイムデータレプリケーション
●	AWS Auto Scaling による自動スケーリング機能
●	AWS Organizations による複数アカウントのガバナンス管理
●	長期保持を含む集中ログ管理
●	アクセスキーを使用しない安全なアクセス方式
●	全アカウントおよびデータベースアクセスに MFA（多要素認証）を適用
●	包括的な監査ログおよびモニタリング
●	VPC エンドポイントの利用によるプライベート通信
3. 開発サーバ構成
開発環境は本番と同一構造を維持しつつ、コスト最適化された設定。
•	Cloudflare統合
•	AWS VPCによる分離環境
•	EC2で以下をホスト：
•	Djangoアプリ（バックエンド）
•	PostgreSQL
•	Redis
•	Nginxリバースプロキシ
![ref_aws_settings_img002.png](images/ref_aws_settings/ref_aws_settings_img002.png)
• 簡易ネットワーク構成：開発環境におけるネットワーク構成を簡素化し、運用負荷を軽減。
• Cloudflare連携：DDoS攻撃対策およびCDN（コンテンツ配信ネットワーク）として利用。
• AWS VPC：AWS内における分離された開発専用環境を構築。
Amazon EC2：以下のコンテナ化された開発用サービスを実行：
• Djangoアプリケーション（バックエンド） — アプリケーション層。
• PostgreSQLデータベース — データベース層。
• Redisサービス — キャッシュおよびバックグラウンドタスク処理に使用。
• Nginxウェブサーバー — リバースプロキシとして機能し、リクエストをDjangoバックエンドへルーティング。

### 4. AWSリソース構成


#### 4.1 VPC


| Parameter | Value |
|---|---|
| VPC ID | vpc-053c4748907edd0d1 |
| VPC CIDR | 10.0.0.0/16 |
| Tenancy | Default |


#### 4.2 サブネット


| Subnet ID | Name | CIDR | Availability Zone |
|---|---|---|---|
| subnet-0aa5adfc689afea95 | etrcars-app-subnet-ap-northeast-1a | 10.0.8.0/24 | ap-northeast-1a |
| subnet-0b40ebef2c6a1429e | etrcars-app-subnet-ap-northeast-1c | 10.0.11.0/24 | ap-northeast-1c |
| subnet-08905df2c05442aa9 | etrcars-data-subnet-ap-northeast-1a | 10.0.9.0/24 | ap-northeast-1a |
| subnet-04b70b7bf86d76c5f | etrcars-data-subnet-ap-northeast-1c | 10.0.12.0/24 | ap-northeast-1c |
| subnet-06507180b947e2ac4 | etrcars-public-subnet-ap-northeast-1a | 10.0.7.0/24 | ap-northeast-1a |
| subnet-03478f822b47c38f9 | etrcars-public-subnet-ap-northeast-1c | 10.0.10.0/24 | ap-northeast-1c |


#### 4.3 ECS

アプリケーションは、サーバーレスコンテナ管理に Fargate 起動タイプを使用して AWS ECS で実行されます.

| Parameter | Value |
|---|---|
| Cluster Name | etrcars-ecs-cluster |
| Launch Type | FARGATE |

ECS タスク定義

| Task Name | CPU | Memory | Port |
|---|---|---|---|
| etrcars-django | 2 vCPU | 4GB | 8000 |
| etrcars-rq-worker | 0.5 vCPU | 1GB | N/A |

Auto Scaling の設定

| Parameter | Value |
|---|---|
| Minimum Capacity | 3 |
| Maximum Capacity | 6 |
| CPU Threshold | 50% |
| Memory Threshold | 75% |


#### 4.4 RDS


| Parameter | Value |
|---|---|
| Database Name | etrcars |
| Engine | PostgreSQL |
| Version | 16 |
| Instance Type | db.m7i.large |
| Multi-AZ | Yes |
| Storage | 200GB |

5. ネットワーク権限設定
オープンアクセスなし: セキュリティグループ内のどのポートでも0.0.0.0/0を許可しないようにし、サービスが公開されないようにします.
内部通信用のVPCエンドポイント: サービスは、パブリックインターネットを使用せずにVPC内で安全に通信します.
ECR VPC エンドポイント: ECS はコンテナイメージを安全にプルします。
S3 & Secrets Manager VPC エンドポイント: ECS は、データまたはシークレットを安全にプッシュおよびプルできます。
ログ VPC エンドポイント: サービスが VPC を離れることなく CloudWatch と CloudTrail にログを送信できるようにします.
5.1 セキュリティグループ構成
セキュリティグループは仮想ファイアウォールとして入出力通信を制御し、ネットワークの分離とアクセス制御を実現します。

##### 5.1.1 受信 (Inbound) ルール

ロードバランサー: Cloudflare の IP 範囲からのみ通信を許可
ECS: ALB のセキュリティグループからの通信のみ許可
RDS: Bastion および ECS セキュリティグループからの 5432/TCP 通信を許可
Redis: ECS セキュリティグループからの通信のみ許可
Bastion: Session Manager 経由のアクセス（受信ルール不要）
VPC Endpoint: ECS からの通信を許可

| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 172.64.0.0/13 |  |  |  |
|---|---|---|---|---|---|---|---|---|---|
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 103.21.244.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 190.93.240.0/20 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 103.22.200.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 104.24.0.0/14 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 141.101.64.0/18 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 108.162.192.0/18 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 188.114.96.0/20 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 103.31.4.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 162.158.0.0/15 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 173.245.48.0/20 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 198.41.128.0/17 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 104.16.0.0/13 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 131.0.72.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 80 | 80 | 197.234.240.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 190.93.240.0/20 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 103.22.200.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 197.234.240.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 103.31.4.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 141.101.64.0/18 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 131.0.72.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 103.21.244.0/22 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 104.16.0.0/13 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 188.114.96.0/20 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 162.158.0.0/15 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 108.162.192.0/18 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 172.64.0.0/13 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 104.24.0.0/14 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 173.245.48.0/20 |  |  |  |
| sg-0101df0db3a448778 | etrcars-alb-sg | Inbound/Ingress | tcp | 443 | 443 | 198.41.128.0/17 |  |  |  |

• ECS セキュリティグループ: Load Balancer セキュリティグループからのトラフィックのみを受け入れます。

| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Inbound/Ingress | tcp | 8000 | 8000 |  |  |  | sg-0101df0db3a448778 |
|---|---|---|---|---|---|---|---|---|---|

• RDSセキュリティ・グループ: ポート5432でBastionおよびECSセキュリティ・グループからのトラフィックを受け入れます。

| sg-0e4dd94f15fec9901 | etrcars-rds-sg | Inbound/Ingress | tcp | 5432 | 5432 |  |  |  | sg-003084ddfa6689fe5 |
|---|---|---|---|---|---|---|---|---|---|
| sg-0e4dd94f15fec9901 | etrcars-rds-sg | Inbound/Ingress | tcp | 5432 | 5432 |  |  |  | sg-02c0992b8c03f20aa |

• Redisセキュリティグループ:ECSセキュリティグループからのトラフィックのみを受け入れます

| sg-0266d0c3a850a31f6 | etrcars-redis-sg | Inbound/Ingress | tcp | 6379 | 6379 |  |  |  | sg-02c0992b8c03f20aa |
|---|---|---|---|---|---|---|---|---|---|

• 要塞セキュリティ・グループ: セッション・マネージャ・ロールを使用します(インバウンド・ルールは不要)
• VPC エンドポイントセキュリティグループ: VPC エンドポイントとのバインドは、ECS からのトラフィックを受け入れます。

| sg-069286d41fa2e62be | ecr-vpc-sg | Inbound/Ingress | tcp | 587 | 587 |  |  |  | sg-02c0992b8c03f20aa |
|---|---|---|---|---|---|---|---|---|---|
| sg-069286d41fa2e62be | ecr-vpc-sg | Inbound/Ingress | tcp | 443 | 443 |  |  |  | sg-02c0992b8c03f20aa |
| sg-069286d41fa2e62be | ecr-vpc-sg | Inbound/Ingress | tcp | 443 | 443 |  |  |  | sg-003084ddfa6689fe5 |


##### 5.1.2 送信 (Outbound) ルール

ALB: ECS への 8000/TCP 通信を許可
ECS: RDS、Redis、Google API、Cloudflare、SES IP への通信を許可
Bastion: RDS への 5432/TCP 通信を許可
RDS: 送信通信なし（制限）
Redis: RDS への通信を許可

| sg-0101df0db3a448778 | etrcars-alb-sg | Outbound/Egress | tcp | 8000 | 8000 | 10.0.11.0/24 |  |  |  |
|---|---|---|---|---|---|---|---|---|---|
| sg-0101df0db3a448778 | etrcars-alb-sg | Outbound/Egress | tcp | 8000 | 8000 | 10.0.8.0/24 |  |  |  |

• ECSセキュリティグループ:RDSおよびRedisセキュリティグループ、Google API、cloudflare、SES IPへのアウトバウンドを許可します。

| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Inbound/Ingress | tcp | 8000 | 8000 |  |  |  | sg-0101df0db3a448778 |
|---|---|---|---|---|---|---|---|---|---|
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 5432 | 5432 | 10.0.9.0/24 |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 5432 | 5432 | 10.0.12.0/24 |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 6379 | 6379 | 10.0.9.0/24 |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 6379 | 6379 | 10.0.12.0/24 |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 587 | 587 |  |  |  | sg-069286d41fa2e62be (SES) |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 122.8.152.121/32 (CODIUM Sentry) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 103.21.244.0/22 (cloud 1) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 103.22.200.0/22 (cloud 2) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 103.31.4.0/22 (cloud 3) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 104.16.0.0/13 (cloud 4) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 104.24.0.0/14 (cloud 5) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 108.162.192.0/18 (cloud 6) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 131.0.72.0/22 (cloud 7) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 141.101.64.0/18 (cloud 8) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 162.158.0.0/15 (cloud 9) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 172.64.0.0/13 (cloud 10) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 173.245.48.0/20 (cloud 11) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 188.114.96.0/20 (cloud 12) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 190.93.240.0/20 (cloud 13) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 197.234.240.0/22 (cloud 14) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 198.41.128.0/17 (cloud 15) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 172.217.0.0/16 (google 1) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 142.250.0.0/16 (google 2) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 74.125.0.0/16 (google 3) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 209.85.128.0/17 (google 4) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 64.233.160.0/19 (google 5) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 66.102.0.0/20 (google 6) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 66.249.80.0/20 (google 7) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 72.14.192.0/18 (google 8) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 108.177.8.0/21 (google 9) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 173.194.0.0/16 (google 10) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 207.126.144.0/20 (google 11) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 216.58.192.0/19 (google 12) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 216.239.32.0/19 (google 13) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 52.192.0.0/15 (ses 1) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 54.248.0.0/15 (ses 2) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 54.250.0.0/16 (ses 3) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 57.181.215.6/32 (ses 4) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 54.238.213.192/32 (ses 5) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 | 54.249.105.210/32 (ses 6) |  |  |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 |  |  | pl-61a54008 |  |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 |  |  |  | sg-069286d41fa2e62be |
| sg-02c0992b8c03f20aa | etrcars-ecs-sg | Outbound/Egress | tcp | 443 | 443 |  |  |  | sg-0a8f1378b49c9d7dc (guardduty) |

• 要塞セキュリティグループ: RDSセキュリティグループへのアクセスを許可します

| sg-003084ddfa6689fe5 | etrcars-bastion-sg | Outbound/Egress | tcp | 5432 | 5432 | 10.0.9.0/24 |  |  |  |
|---|---|---|---|---|---|---|---|---|---|
| sg-003084ddfa6689fe5 | etrcars-bastion-sg | Outbound/Egress | tcp | 5432 | 5432 | 10.0.12.0/24 |  |  |  |
| sg-003084ddfa6689fe5 | etrcars-bastion-sg | Outbound/Egress | tcp | 443 | 443 |  |  | pl-61a54008 |  |
| sg-003084ddfa6689fe5 | etrcars-bastion-sg | Outbound/Egress | tcp | 443 | 443 |  |  |  | sg-069286d41fa2e62be |
| sg-003084ddfa6689fe5 | etrcars-bastion-sg | Outbound/Egress | tcp | 443 | 443 |  |  |  | sg-0a8f1378b49c9d7dc (GuardDuty) |

• RDS セキュリティグループ: アウトバウンドネットワークは許可されません
• Redisセキュリティグループ:RDSへのアウトバウンドネットワークを許可します

| sg-0266d0c3a850a31f6 | etrcars-redis-sg | Outbound/Egress | tcp | 5432 | 5432 | 10.0.9.0/24 |  |  |  |
|---|---|---|---|---|---|---|---|---|---|
| sg-0266d0c3a850a31f6 | etrcars-redis-sg | Outbound/Egress | tcp | 5432 | 5432 | 10.0.12.0/24 |  |  |  |

5.2 ネットワーク ACL
セキュリティ グループは、受信トラフィックと送信トラフィックを制御する仮想ファイアウォールとして機能します。次のセキュリティ グループ構成により、適切なネットワーク分離とアクセス制御が保証されます。

| Name | ID | Associated with | Description |
|---|---|---|---|
| etrcars-data-acl | acl-04e7494427034c121 | etrcars-data-subnet-ap-northeast-1c, etrcars-data-subnet-ap-northeast-1a | More secure ACL for Database |
| etrcars-public-acl | acl-05484679f6e65dbbe | etrcars-public-subnet-ap-northeast-1a, etrcars-public-subnet-ap-northeast-1c | Allow some connections for internet access |
| etrcars-app-acl | acl-084c4fedbd5db26f4 | etrcars-app-subnet-ap-northeast-1a, etrcars-app-subnet-ap-northeast-1c | Only allow some IPs Cidr which are needed |

5.2.1 Inbound ルール
• etrcars-private-acl:2 つのアプリ サブネットからのトラフィックのみを受け入れます

| 120 | Custom TCP | TCP (6) | 1024 - 65535 | 10.0.8.0/24 | Allow |
|---|---|---|---|---|---|
| 130 | Custom TCP | TCP (6) | 1024 - 65535 | 10.0.11.0/24 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

• etrcars-public-acl:一部のポートからのトラフィックのみを許可します。

| 130 | HTTP (80) | TCP (6) | 80 | 0.0.0.0/0 | Allow |
|---|---|---|---|---|---|
| 140 | HTTPS (443) | TCP (6) | 443 | 0.0.0.0/0 | Allow |
| 150 | Custom TCP | TCP (6) | 1024 - 65535 | 0.0.0.0/0 | Allow |
| 160 | Custom TCP | TCP (6) | 587 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

Etrcars-app-acl: 特定の Cidr を持つ一部のポートからのトラフィックのみを許可する

| 2 | Custom TCP | TCP (6) | 1 - 65535 | 173.194.0.0/16 | Allow |
|---|---|---|---|---|---|
| 3 | Custom TCP | TCP (6) | 1 - 65535 | 207.126.144.0/20 | Allow |
| 4 | Custom TCP | TCP (6) | 1 - 65535 | 216.58.192.0/19 | Allow |
| 5 | Custom TCP | TCP (6) | 1 - 65535 | 216.239.32.0/19 | Allow |
| 6 | Custom TCP | TCP (6) | 1 - 65535 | 142.250.0.0/16 | Allow |
| 7 | Custom TCP | TCP (6) | 1 - 65535 | 74.125.0.0/16 | Allow |
| 8 | Custom TCP | TCP (6) | 1 - 65535 | 209.85.128.0/17 | Allow |
| 9 | Custom TCP | TCP (6) | 1 - 65535 | 64.233.160.0/19 | Allow |
| 10 | Custom TCP | TCP (6) | 1 - 65535 | 66.102.0.0/20 | Allow |
| 11 | Custom TCP | TCP (6) | 1 - 65535 | 66.249.80.0/20 | Allow |
| 12 | Custom TCP | TCP (6) | 1 - 65535 | 72.14.192.0/18 | Allow |
| 13 | Custom TCP | TCP (6) | 1 - 65535 | 108.177.8.0/21 | Allow |
| 14 | Custom TCP | TCP (6) | 1 - 65535 | 172.217.0.0/16 | Allow |
| 15 | Custom TCP | TCP (6) | 1 - 65535 | 103.21.244.0/22 | Allow |
| 16 | Custom TCP | TCP (6) | 1 - 65535 | 103.22.200.0/22 | Allow |
| 17 | Custom TCP | TCP (6) | 1 - 65535 | 108.162.192.0/18 | Allow |
| 18 | Custom TCP | TCP (6) | 1 - 65535 | 131.0.72.0/22 | Allow |
| 19 | Custom TCP | TCP (6) | 1 - 65535 | 141.101.64.0/18 | Allow |
| 20 | Custom TCP | TCP (6) | 1 - 65535 | 162.158.0.0/15 | Allow |
| 21 | Custom TCP | TCP (6) | 1 - 65535 | 172.64.0.0/13 | Allow |
| 22 | Custom TCP | TCP (6) | 1 - 65535 | 173.245.48.0/20 | Allow |
| 23 | Custom TCP | TCP (6) | 1 - 65535 | 188.114.96.0/20 | Allow |
| 24 | Custom TCP | TCP (6) | 1 - 65535 | 190.93.240.0/20 | Allow |
| 25 | Custom TCP | TCP (6) | 1 - 65535 | 197.234.240.0/22 | Allow |
| 26 | Custom TCP | TCP (6) | 1 - 65535 | 198.41.128.0/17 | Allow |
| 27 | Custom TCP | TCP (6) | 1 - 65535 | 54.0.0.0/8 | Allow |
| 28 | Custom TCP | TCP (6) | 1 - 65535 | 57.181.215.6/32 | Allow |
| 29 | Custom TCP | TCP (6) | 1 - 65535 | 52.192.0.0/15 | Allow |
| 30 | Custom TCP | TCP (6) | 1 - 65535 | 103.31.4.0/22 | Allow |
| 31 | Custom TCP | TCP (6) | 1 - 65535 | 104.16.0.0/13 | Allow |
| 32 | Custom TCP | TCP (6) | 1 - 65535 | 104.24.0.0/14 | Allow |
| 100 | Custom TCP | TCP (6) | 1 - 65535 | 3.5.152.0/21 | Allow |
| 101 | Custom TCP | TCP (6) | 1 - 65535 | 52.219.0.0/16 | Allow |
| 110 | Custom TCP | TCP (6) | 1 - 65535 | 10.0.0.0/16 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

5.2.2 アウトバウンドルール
• etrcars-private-acl:2 つのアプリ サブネットからのトラフィックのみを受け入れます

| 120 | Custom TCP | TCP (6) | 1024 - 65535 | 10.0.8.0/24 | Allow |
|---|---|---|---|---|---|
| 130 | Custom TCP | TCP (6) | 1024 - 65535 | 10.0.11.0/24 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

• etrcars-public-acl:一部のポートからのトラフィックのみを許可します。

| 130 | HTTP (80) | TCP (6) | 80 | 0.0.0.0/0 | Allow |
|---|---|---|---|---|---|
| 140 | HTTPS (443) | TCP (6) | 443 | 0.0.0.0/0 | Allow |
| 150 | Custom TCP | TCP (6) | 1024 - 65535 | 0.0.0.0/0 | Allow |
| 160 | Custom TCP | TCP (6) | 587 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

Etrcars-app-acl: 特定の Cidr を持つ一部のポートからのトラフィックのみを許可する

| 2 | Custom TCP | TCP (6) | 1 - 65535 | 173.194.0.0/16 | Allow |
|---|---|---|---|---|---|
| 3 | Custom TCP | TCP (6) | 1 - 65535 | 207.126.144.0/20 | Allow |
| 4 | Custom TCP | TCP (6) | 1 - 65535 | 216.58.192.0/19 | Allow |
| 5 | Custom TCP | TCP (6) | 1 - 65535 | 216.239.32.0/19 | Allow |
| 6 | Custom TCP | TCP (6) | 1 - 65535 | 142.250.0.0/16 | Allow |
| 7 | Custom TCP | TCP (6) | 1 - 65535 | 74.125.0.0/16 | Allow |
| 8 | Custom TCP | TCP (6) | 1 - 65535 | 209.85.128.0/17 | Allow |
| 9 | Custom TCP | TCP (6) | 1 - 65535 | 64.233.160.0/19 | Allow |
| 10 | Custom TCP | TCP (6) | 1 - 65535 | 66.102.0.0/20 | Allow |
| 11 | Custom TCP | TCP (6) | 1 - 65535 | 66.249.80.0/20 | Allow |
| 12 | Custom TCP | TCP (6) | 1 - 65535 | 72.14.192.0/18 | Allow |
| 13 | Custom TCP | TCP (6) | 1 - 65535 | 108.177.8.0/21 | Allow |
| 14 | Custom TCP | TCP (6) | 1 - 65535 | 172.217.0.0/16 | Allow |
| 15 | Custom TCP | TCP (6) | 1 - 65535 | 103.21.244.0/22 | Allow |
| 16 | Custom TCP | TCP (6) | 1 - 65535 | 103.22.200.0/22 | Allow |
| 17 | Custom TCP | TCP (6) | 1 - 65535 | 108.162.192.0/18 | Allow |
| 18 | Custom TCP | TCP (6) | 1 - 65535 | 131.0.72.0/22 | Allow |
| 19 | Custom TCP | TCP (6) | 1 - 65535 | 141.101.64.0/18 | Allow |
| 20 | Custom TCP | TCP (6) | 1 - 65535 | 162.158.0.0/15 | Allow |
| 21 | Custom TCP | TCP (6) | 1 - 65535 | 172.64.0.0/13 | Allow |
| 22 | Custom TCP | TCP (6) | 1 - 65535 | 173.245.48.0/20 | Allow |
| 23 | Custom TCP | TCP (6) | 1 - 65535 | 188.114.96.0/20 | Allow |
| 24 | Custom TCP | TCP (6) | 1 - 65535 | 190.93.240.0/20 | Allow |
| 25 | Custom TCP | TCP (6) | 1 - 65535 | 197.234.240.0/22 | Allow |
| 26 | Custom TCP | TCP (6) | 1 - 65535 | 198.41.128.0/17 | Allow |
| 27 | Custom TCP | TCP (6) | 1 - 65535 | 3.5.152.0/21 | Allow |
| 28 | Custom TCP | TCP (6) | 1 - 65535 | 52.219.0.0/16 | Allow |
| 29 | Custom TCP | TCP (6) | 1 - 65535 | 54.0.0.0/8 | Allow |
| 30 | Custom TCP | TCP (6) | 1 - 65535 | 57.181.215.6/32 | Allow |
| 31 | Custom TCP | TCP (6) | 1 - 65535 | 52.192.0.0/15 | Allow |
| 32 | Custom TCP | TCP (6) | 1 - 65535 | 103.31.4.0/22 | Allow |
| 33 | Custom TCP | TCP (6) | 1 - 65535 | 104.16.0.0/13 | Allow |
| 34 | Custom TCP | TCP (6) | 1 - 65535 | 104.24.0.0/14 | Allow |
| 100 | Custom TCP | TCP (6) | 1 - 65535 | 10.0.0.0/16 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

6. 権限設定
ユーザーアクセスは、最小権限の原則で AWS IAM を通じて管理されます。次のユーザーと権限が構成されています。
IAM User Configuration

| Username | Group | Permissions |
|---|---|---|
| ETRCARS (root user) | None | AdministratorAccess |
| will@codium.co | None | EC2, ECS, <br>CODIUM_WIFI_ONLY |
| liang@codium.co | None | EC2, ECS, IAM ReadOnly, <br>CODIUM_WIFI_ONLY |

![ref_aws_settings_img003.png](images/ref_aws_settings/ref_aws_settings_img003.png)
7. タスク設定の自動化
以下のプロセスが定期的に実行されます。
自動化されたタスク

| タスク名 | 概要 | 頻度 |
|---|---|---|
| RDSデータベースバックアップ | 災害復旧用に RDS PostgreSQL データベースの自動スナップショットを取得 | リアルタイム（1日1回のスナップショット保持ポリシー） |
| ECS オートスケーリング | CPU / メモリ使用率に基づき ECS タスク数を動的に調整 | 常時 / オンデマンド |
| CloudWatch ログローテーション | ECS コンテナ、アプリケーション、システムプロセスのログを収集・保持・自動削除 | 7日間保持 |
| S3 バックアップ（大阪リージョン） | 重要な S3 データをセカンダリリージョン（大阪）にリアルタイムでレプリケーション | リアルタイム |
| S3 Glacier アーカイブ | 古いバックアップや低頻度アクセスデータを S3 Glacier に移動してコスト最適化 | 7年間保持 |
| Security Hub コンプライアンススキャン | GuardDuty、Inspector、Config からの結果を集約しコンプライアンス状況を評価 | リアルタイム |
| GuardDuty 脅威検出 | 不正アクセスや悪意ある活動を継続的に監視 | リアルタイム |
| AWS Config 構成変更検出 | AWS リソースの設定変更を監視し、非準拠変更を報告 | リアルタイム |
| Inspector 脆弱性スキャン | ECS および EC2 インスタンスの脆弱性を自動スキャン | リアルタイム |
| システムヘルスチェック | システム全体のパフォーマンスおよび AWS サービスの稼働状態を監視 | リアルタイム |

8. 監視とアラームの設定
The following process are monitored and alarmed using these tools:

| ツール | 名前 | 条件 | 詳細 |
|---|---|---|---|
| CloudWatch | TargetTracking-service/etrcars-ecs-cluster/etrcars-django-service-AlarmLow-f45b3b32... | MemoryUtilization < 67.5（15分間に15データポイント） | ECS の負荷低下検知 |
| CloudWatch | TargetTracking-service/...AlarmLow-07cbdf4a... | CPUUtilization < 45（15分間に15データポイント） | ECS の負荷低下検知 |
| CloudWatch | TargetTracking-service/...AlarmHigh-9358c52a... | MemoryUtilization > 75（3分間に3データポイント） | ECS の高負荷検知 |
| CloudWatch | TargetTracking-service/...AlarmHigh-842d5658... | CPUUtilization > 50（3分間に3データポイント） | ECS の高負荷検知 |
| CloudWatch | Session manager connected | CallCount >= 1（5分間に1データポイント） | Session Manager 経由の DB 接続 |
| CloudWatch | RootAccountLogin | RootAccountLogin >= 1（5分間に1データポイント） | ルートアカウントでのログイン検知 |
| CloudWatch | RDS Memory | FreeableMemory <= 1GB（1分間に1データポイント） | RDS メモリ不足 |
| CloudWatch | RDS CPU | CPUUtilization >= 50（1分間に1データポイント） | RDS 高負荷 |
| CloudWatch | Network Change | NetworkConfigModificationCount > 0（1分間に1データポイント） | VPC ネットワーク設定の変更検知 |
| CloudWatch | IAM Policy Changed | IAMMonitoring >= 1（1分間に1データポイント） | IAM ポリシー変更検知 |
| Security Hub |  | GuardDuty, Inspector, Config などのセキュリティツールからの監視結果を統合 | システム全体のセキュリティ状態を可視化 |


| ツール | 名前 | 条件 | 詳細 |
|---|---|---|---|
| Cloudwatch | BackupSettingsChanged | BackupSettingsChanged > 1 for 1 datapoints within 5 minutes | Backup settings changed in AWS with these rules { ($.eventSource = "backup.amazonaws.com") && ( $.eventName = "CreateBackupPlan" \|\| $.eventName = "UpdateBackupPlan" \|\| $.eventName = "DeleteBackupPlan" \|\| $.eventName = "PutBackupVaultNotifications" \|\| $.eventName = "PutBackupVaultAccessPolicy" ) } |
| Cloudwatch | RDS settings changes alarm | RDS Changes >= 1 for 1 datapoints within 5 minutes | Changes about RDS instance or settings with these rules<br>{ ($.eventSource = "rds.amazonaws.com") && ( $.eventName = "CreateDBInstance" \|\| $.eventName = "ModifyDBInstance" \|\| $.eventName = "DeleteDBInstance" ) } |
| Cloudwatch | S3 settings Changes | S3 Changes >= 1 for 1 datapoints within 5 minutes | Changes about S3 bucket and settings with these rules<br>{ ($.eventSource = "s3.amazonaws.com") && ( $.eventName = "CreateBucket" \|\| $.eventName = "DeleteBucket" \|\| $.eventName = "PutBucketAcl" \|\| $.eventName = "PutBucketPolicy" \|\| $.eventName = "PutBucketPublicAccessBlock" ) } |
| Cloudwatch | TargetTracking-service/etrcars-ecs-cluster/etrcars-django-service-AlarmLow-f45b3b32-dfaa-4eab-a850-43cdcf3a53ac | MemoryUtilization < 67.5 for 15 datapoints within 15 minutes | ECS underload |
| Cloudwatch | TargetTracking-service/etrcars-ecs-cluster/etrcars-django-service-AlarmLow-07cbdf4a-01c7-48e7-8ce4-e922bc95b365 | CPUUtilization < 45 for 15 datapoints within 15 minutes | ECS underload |
| Cloudwatch | TargetTracking-service/etrcars-ecs-cluster/etrcars-django-service-AlarmHigh-9358c52a-9551-4504-8233-dd6dbd45fe53 | MemoryUtilization > 75 for 3 datapoints within 3 minutes | ECS overload |
| Cloudwatch | TargetTracking-service/etrcars-ecs-cluster/etrcars-django-service-AlarmHigh-842d5658-dd48-4e9c-bc3a-129f01e0da2b | CPUUtilization > 50 for 3 datapoints within 3 minutes | ECS overload |
| Cloudwatch | Session manager connected | CallCount >= 1 for 1 datapoints within 5 minutes | Database connected from session manager |
| Cloudwatch | RootAccountLogin | RootAccountLogin >= 1 for 1 datapoints within 5 minutes | Login with Root Account |
| Cloudwatch | RDS Memory | FreeableMemory <= 1073741824 for 1 datapoints within 1 minute | RDS overload |
| Cloudwatch | RDS CPU | CPUUtilization >= 50 for 1 datapoints within 1 minute | RDS overload |
| Cloudwatch | Network Change | NetworkConfigModificationCount > 0 for 1 datapoints within 1 minute | Network settings changed in VPC |
| Cloudwatch | IAM Policy Changed | IAMMonitoring >= 1 for 1 datapoints within 1 minute | IAM Policy changed |
| Security Hub |  |  | Gather all the results monitoring from security tools such as GuardDuty, Inspector, Config |

アラームメールは、以下のアドレスに送信されます。

| name | post | Contact Email Address |
|---|---|---|
| Wai Yan Aung (Will) | SeniorSoftware Developer | will@codium.co |
| 佐野　泰輔 | SBRAND | sano@sbrandg.com |

監査は、3ヶ月に1回、パッチリストやログリストなどの外部チェックリスト文書により、日本側が実施しています。
9. 緊急時対応手順
アラート発生時の流れは、以下のとおりです。
![ref_aws_settings_img004.png](images/ref_aws_settings/ref_aws_settings_img004.png)


![ref_aws_settings_img005.png](images/ref_aws_settings/ref_aws_settings_img005.png)


![ref_aws_settings_img006.png](images/ref_aws_settings/ref_aws_settings_img006.png)


![ref_aws_settings_img007.png](images/ref_aws_settings/ref_aws_settings_img007.png)


![ref_aws_settings_img008.png](images/ref_aws_settings/ref_aws_settings_img008.png)

| フロー | 手順 |
|---|---|
| CloudWatchにてGuardDuty からのアラート検知 | 設定条件が満たされた場合に自動的にトリガー |
| 影響範囲の分析 | 影響を受けた AWS サービス（ECS、RDS など）を特定し、関連ログを確認 |
| 対応・復旧 | SBRAND と連携して対応方針を決定し、必要に応じてサービス再起動・タスクスケーリング・バックアップ復元を実施 |

10. 運用手順

#### アクセスキーの管理と認証


##### 10.1. アクセスキーポリシー

静的アクセスキーに依存している AWS のサービスやユーザーはいません。
アクセスキーは、アプリケーションコード、構成ファイル、またはCI/CDパイプラインに保存されたり埋め込まれたりしません。すべてのサービス間通信は、最小権限のアクセス許可を持つ IAM ロールを使用して処理されます。

##### 10.2. IAM ロールベースのアクセス

各 AWS サービス (ECS、S3、SES、SSM、Secret Manager など)  は、専用の IAM ロールを使用して動作します。ロールは信頼関係で構成されているため、資格情報を公開することなく、安全なサービス間通信が可能になります。
例: ECS タスクは、S3 または Session Manager に安全にアクセスするロールを引き受けます。
ロールのアクセス許可は、露出を最小限に抑え、権限の昇格を防ぐために、範囲が狭くなります。

##### 10.3. 外部サービス認証(GitLab CI/CD)

GitLab などの外部 CI/CD パイプラインは、OpenID Connect (OIDC) を使用して AWS に対して認証されます。
OIDC を使用すると、GitLab ランナーは、保存されている AWS 認証情報を使用せずに一時的に IAM ロールを引き受けることができます。
このメカニズムにより、パイプラインは Docker イメージを Amazon ECR に安全にプッシュ し、デプロイオペレーションを実行できます。

##### 10.4. セッションアクセスおよび管理操作

この通信経路は、CODIUM社が開発/運用にてデータベース参照時に利用する経路とする。
AWS リソースまたは本番環境への直接管理アクセスでは、アクセスキーは使用されません。
AWS Session Manager は、SSH キーなしでインスタンスに安全にアクセスするために使用されます。
Session Manager へのアクセスは、AWS IAM Identity Center と統合されています。
ユーザーは、SSOポータルURLを介してログインし、企業資格証明とMFAを使用して認証し、セッションのIAMロールを引き受けてデータベースに接続します。
MFA（この接続専用アカウント）については、IP制限をかけているが、エスブランド開発保守責任者へ通知されるため、アクセス時にはエスブランドの許可を得ている前提となる。また、セッションについては、コンソールから停止することが可能となっている。
これにより、セッション マネージャーを介して、RDS データベースなどの機密システムへの追跡可能で、有効期間が短く、監査可能なアクセスが保証されます。
1１. その他
１、AWSサポートプランについては、ビジネスプランに加入しサポート体制を整える
2、CODIUM社の使用するIAMユーザーのAWSコンソールアクセスは通常CODIM社のIP制限をかけている。AWS設定内に（CODIUM_WIFI_Only）の設定を作成しIPを設定している。
緊急時に自宅でアクセスするさいは、エスブランド開発保守責任者により、IP制限を解除し、自宅での使用を許可する。
作業終了後、エスブランド開発保守責任者はAWSコンソールの設定を元に戻し、CODIUM社内のIP制限を設定する
