# Simple AWS PCS Cluster

AWS Parallel Computing Service (PCS) を使用したシンプルなクラスター環境の CloudFormation テンプレートです。

## 概要

このテンプレートは、検証用のシンプルな AWS PCS クラスター環境を構築します。

- ログインノード: 1台（t3.micro）
- コンピュートノード: 最大4台（m7i.large）
- 共有ストレージ: EFS
- スケジューラー: Slurm 24.11

## アーキテクチャ

### ネットワーク構成
- VPC: 10.0.0.0/16
- パブリックサブネット: 3つのAZ（10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24）
- プライベートサブネット: 3つのAZ（10.0.11.0/24, 10.0.12.0/24, 10.0.13.0/24）
- アイソレートサブネット: 3つのAZ（10.0.21.0/24, 10.0.22.0/24, 10.0.23.0/24）
- NAT Gateway: 1つ（プライベートサブネットのインターネットアクセス用）

### コンピュート構成
- ログインノードグループ: パブリックサブネット内のt3.microインスタンス
- コンピュートノードグループ: プライベートサブネット内のm7i.largeインスタンス
- Auto Scaling: コンピュートノードは0～4台まで自動スケーリング

### ストレージ構成
- EFS: プライベートサブネット全体にマウントターゲットを配置
- 暗号化: EFSは暗号化有効
- 共有ディレクトリ: `/shared` にマウント

## パラメータ

| パラメータ名 | デフォルト値 | 説明 |
|-------------|-------------|------|
| ClusterName | simple-pcs-cluster | PCSクラスターの名前 |

## デプロイ方法

### AWS CLI を使用
```bash
aws cloudformation create-stack \
  --stack-name simple-pcs-cluster \
  --template-body file://simple-pcs-cluster.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=ClusterName,ParameterValue=my-cluster
```

### AWS マネジメントコンソールを使用
1. CloudFormation コンソールを開く
2. 「スタックの作成」を選択
3. テンプレートファイル `simple-pcs-cluster.yaml` をアップロード
4. パラメータを設定
5. IAM リソースの作成を許可
6. スタックを作成

## 作成されるリソース

### ネットワーク
- VPC、インターネットゲートウェイ、NAT Gateway
- パブリック・プライベート・アイソレートサブネット（各3つ）
- ルートテーブルとセキュリティグループ

### ストレージ
- EFS ファイルシステム（暗号化済み）
- EFS マウントターゲット（3つのAZ）

### コンピュート
- PCS クラスター（SLURM スケジューラー）
- ログインノードグループ（t3.micro × 1）
- コンピュートノードグループ（m7i.large × 0-4）
- コンピュートキュー

### IAM
- PCS インスタンス用 IAM ロール
- Session Manager アクセス用ポリシー
- PCS 登録用ポリシー

## 出力値

| 出力名 | 説明 |
|--------|------|
| ClusterId | PCS クラスターの ID |
| VPCId | VPC の ID |
| EFSFileSystemId | EFS ファイルシステムの ID |
| PCSConsoleUrl | PCS コンソールへの直接リンク |
| EC2ConsoleUrl | ログインノードへのSession Manager アクセス用リンク |
| PCSInstanceRoleArn | PCS インスタンスロールの ARN |

## 使用方法

### ログインノードへのアクセス
1. EC2 コンソールで Session Manager を使用してログインノードに接続
2. または出力値の `EC2ConsoleUrl` から直接アクセス

### ジョブの実行
```bash
# Slurm コマンドの確認
sinfo
squeue

# サンプルジョブの実行
sbatch -p compute-queue [実行スクリプト]
```

### 共有ストレージの使用
```bash
# 共有ディレクトリの確認
ls -la /shared

# ファイルの作成・共有
echo "test" > /shared/test.txt
```

## 料金

### 主要コスト要素
- EC2 インスタンス: t3.micro（ログイン）+ m7i.large（コンピュート）
- EFS: ストレージ使用量に応じた課金
- NAT Gateway: 固定料金 + データ転送料金
- PCS: クラスター管理サービス料金

### コスト最適化
- コンピュートノードは最小0台で自動スケーリング
- 不要時はスタック全体を削除可能

## 注意事項

- AMI ID（ami-004a4ae6bc74a1a1a）は特定リージョン用のため、他リージョンでの使用時は変更が必要
- Session Manager 経由でのアクセスを前提としているため、SSH キーペアは不要
- セキュリティグループは PCS クラスター内での通信を許可する設定

## トラブルシューティング

### よくある問題
1. AMI が見つからない: デプロイリージョンに対応したAMI IDに変更