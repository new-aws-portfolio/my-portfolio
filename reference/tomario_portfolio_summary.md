# Tomario - AWSポートフォリオ 設計まとめ

## 背景・出発点

転職活動（3ヶ月以内に開始予定）に向けて、AWSポートフォリオを制作することにした。

- 現職ではインフラ・ネットワーク系システムの基本設計から実装まで携わっている
- 会社の進め方（非機能要件ヒアリング → 要件定義 → 基本設計書 → 環境定義書）が自分の軸になっていた
- AWS経験はあるが、個人でポートフォリオを作るのは初めて
- 転職希望ポジション：インフラ・クラウドエンジニア

---

## 制作方針

### ドキュメントより先に構築する

会社の進め方をそのまま適用すると、ドキュメント作成だけで1〜2ヶ月消費するリスクがある。
手を動かしながら考えるタイプ・3ヶ月以内に転職活動を始めたいという条件から、以下の順序で進める。

1. ゴール（構成図）を先に決める
2. Terraformで構築する
3. 構築後にREADME・設計書を逆算して作成する

> 構築後にドキュメントを書いても内容は変わらない。むしろ構築後の方が設計意図を正確に言語化できる。

### アピールポイント

- **IaC（Terraform）** による再現性・設計力の証明
- **コスト意識**のある設計（NATゲートウェイなし・VPCエンドポイント活用）
- **セキュリティ設計**（SSM Session Manager・最小権限IAM）
- **環境分離**（dev / stg / prd の3面構成）

---

## サービス概要

| 項目 | 内容 |
|------|------|
| サービス名 | **Tomario**（「泊まりお」から。造語で被りにくい） |
| アプリ種別 | ホテル・宿泊予約システム |
| ドメイン候補 | tomario.jp / tomario.io |

### アプリをホテル予約にした理由

TODOアプリなどシンプルな構成だと「サーバーレスで十分では？」と面接で突っ込まれるリスクがある。
ホテル予約システムはEC2・ALB・RDSの必然性が自然に説明できる。

- **RDSが必要な理由**：チェックイン重複防止のトランザクション制御が必要なため、DynamoDBではなくRDBが適切
- **ALB + Auto Scalingが必要な理由**：年末年始・GWなど繁忙期の同時予約アクセス集中に対応するため

### 主な機能

| エンドポイント | 機能 |
|---|---|
| GET /rooms | 部屋一覧取得 |
| GET /rooms/available | 空き部屋確認 |
| POST /bookings | 予約作成 |
| GET /bookings/:id | 予約確認 |
| DELETE /bookings/:id | 予約キャンセル |

### DBテーブル設計

- **users**：id / name / email / password
- **rooms**：id / room_number / type / price
- **bookings**：id / user_id / room_id / check_in / check_out

---

## AWSサービス構成

### 採用サービス一覧

| レイヤー | サービス | 用途 | コスト |
|---|---|---|---|
| DNS | Route53 | 独自ドメイン管理 | ドメイン代のみ（$12/年〜） |
| エッジ | CloudFront | CDN / HTTPS終端 / 振り分け | ほぼ無料 |
| ストレージ | S3 | 静的ファイル配信 / tfstate保管 | ほぼ無料 |
| ネットワーク | VPC / Subnet / SG / IGW | パブリック+プライベート / マルチAZ | 無料 |
| コンピュート | EC2 + ALB + Auto Scaling | API処理 / 負荷分散 / 自動スケール | 〜$20/月 |
| DB | RDS MySQL | データ永続化 | 〜$15/月 |
| 運用 | SSM Session Manager | SSH不要でのEC2接続 | 無料 |
| 監視 | CloudWatch | メトリクス / アラーム | ほぼ無料 |
| 権限 | IAM | 最小権限ロール管理 | 無料 |

**月額概算：$35〜40**（使わないときEC2・RDS停止で $5以下も可能）

### 非採用サービスと理由

| サービス | 理由 |
|---|---|
| NATゲートウェイ | 月$30〜のコストがかかるため禁止。VPCエンドポイントで代替 |
| ECS / EKS | 次フェーズ。今回はEC2ベースの設計・構築力をアピール |

### 構成フロー

```
User
 └─ Route53（独自ドメイン）
     └─ CloudFront（HTTPS終端 / CDN）
         ├─ S3（静的ファイル配信）
         └─ ALB（動的リクエスト）
             ├─ EC2 AZ-a（t3.micro / Auto Scaling）
             └─ EC2 AZ-b（t3.micro / Auto Scaling）
                 └─ RDS MySQL（プライベートサブネット）

SSM Session Manager ──→ EC2（SSH・キーペアなし）
CloudWatch ──────────→ VPC全体の監視
```

---

## Terraform構成

### ディレクトリ構成（ディレクトリ分割方式）

workspaceではなくディレクトリ分割を採用。環境ごとに独立したtfstateを管理する。

```
tomario/
├── envs/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   ├── stg/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prd/
│       ├── main.tf
│       └── terraform.tfvars
└── modules/
    ├── vpc/
    ├── ec2/
    ├── alb/
    ├── rds/
    ├── cloudfront/
    ├── route53/
    ├── ssm/
    └── cloudwatch/
```

### 環境別スペック

| 項目 | dev | stg | prd |
|---|---|---|---|
| EC2 | t3.micro | t3.micro | t3.small〜 |
| RDS | db.t3.micro / シングルAZ | db.t3.micro / シングルAZ | db.t3.small / Multi-AZ |
| Auto Scaling | 最小1台 | 最小1台 | 最小2台 |

> devとstg/prdでRDSのMulti-AZ設定を変えることで「環境ごとに設定を使い分ける意識」をアピールできる

---

## 今後の制作フロー

### フェーズ1：AWS・ローカル環境準備（1日）

- [ ] IAMユーザー作成（Terraform実行用）
- [ ] アクセスキー発行
- [ ] S3バケット作成（tfstate保管用）
- [ ] Terraformインストール
- [ ] AWS CLIインストール・認証設定
- [ ] ディレクトリ作成（envs/dev から着手）

### フェーズ2：dev環境の構築（2〜3週間）

- [ ] VPC / サブネット / SG / IGW
- [ ] EC2 + ALB + Auto Scaling
- [ ] RDS MySQL（シングルAZ）
- [ ] SSM Session Manager設定
- [ ] S3 + CloudFront
- [ ] Route53（独自ドメイン）
- [ ] CloudWatch アラーム設定
- [ ] IAMロール・ポリシー整備

### フェーズ3：アプリ実装（1〜2週間）

- [ ] バックエンドAPI（Python Flask / Node.js）
- [ ] フロントエンド（静的HTML + JS）
- [ ] EC2へのデプロイ
- [ ] 動作確認（予約フロー一通り）

### フェーズ4：stg / prd環境への展開（1週間）

- [ ] envs/stg・envs/prd のtfvars作成
- [ ] RDSをMulti-AZに変更（prd）
- [ ] 各環境でapply・動作確認

### フェーズ5：ドキュメント整備（1週間）

- [ ] README作成（構成図・設計意図・構築手順）
- [ ] 基本設計書（逆算して作成）
- [ ] 環境定義書
- [ ] GitHubに公開

### フェーズ6：転職活動開始

- [ ] ポートフォリオをGitHubに公開した状態で応募開始
- [ ] 面接での説明ポイントを言語化しておく

---

## 面接での説明ポイント

| 質問 | 答え方 |
|---|---|
| なぜEC2か | 繁忙期の同時予約アクセス集中を想定し、ALBで負荷分散・Auto Scalingでスケールアウト設計にした |
| なぜRDSか | チェックイン重複防止のトランザクション制御が必要なため。DynamoDBでは対応しにくい |
| NATゲートウェイは？ | コスト最適化のため不採用。SSMはVPCエンドポイント経由でアクセスし、コストゼロを実現 |
| SSMを使った理由 | SSH・キーペアを廃止しセキュリティグループのインバウンドを最小化するため |
| 環境分離は？ | dev / stg / prdの3面をTerraformのディレクトリ分割で管理。RDSのMulti-AZ設定など環境ごとに変数で制御 |
