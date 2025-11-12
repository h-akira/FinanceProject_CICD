# Finance Project CI/CD

Finance ProjectのCI/CDパイプライン（CodeBuild）をCloudFormationで管理します。

## 📁 構成

```
FinanceProject_CICD/
├── common/
│   └── codebuild-infra.yaml     # Infrastructure (CDK) 用 CodeBuild
├── dashboard/
│   ├── codebuild-backend.yaml   # Dashboard Backend (SAM) 用 CodeBuild
│   └── codebuild-frontend.yaml  # Dashboard Frontend (S3) 用 CodeBuild
└── README.md
```

**ディレクトリ構成の意図:**
- `common/` - 全サブシステム共通のインフラCI/CD
- `dashboard/` - FinanceDashboardProjectサブシステム専用のCI/CD
- 将来サブシステムが増えた場合は、同様のディレクトリを追加（例: `analytics/`）

## 🚀 デプロイ方法

### 前提条件

1. **CodeStar Connection作成**（初回のみ）

GitHub接続を作成します：

```bash
# AWS CLI v2が必要
aws codeconnections create-connection \
  --provider-type GitHub \
  --connection-name connect-github-main \
  --region ap-northeast-1
```

出力されたARNをメモしてください：
```
arn:aws:codeconnections:ap-northeast-1:XXXXXXXXXXXX:connection/xxx
```

次に、AWSコンソールで接続を承認します：
1. CodePipeline → 設定 → 接続
2. 作成した接続を選択
3. 「保留中の接続を更新」→ GitHubで認証

2. **S3バケット作成**（Frontend用、Main Stackで作成）

Frontend用のS3バケットは別途作成してください（CDK/Terraformで管理）。

---

### Infrastructure CodeBuild のデプロイ（最初にデプロイ）

```bash
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file common/codebuild-infra.yaml \
  --stack-name stack-finance-cicd-infra \
  --parameter-overrides \
    ProjectName=build-finance-infra \
    GitHubOwner=h-akira \
    GitHubRepo=FinanceProject_Infra \
    GitHubBranch=main \
    CodeStarConnectionArn=arn:aws:codeconnections:ap-northeast-1:XXXXXXXXXXXX:connection/xxx \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

**パラメータ**:
- `ProjectName`: CodeBuildプロジェクト名
- `GitHubOwner`: GitHubリポジトリオーナー
- `GitHubRepo`: GitHubリポジトリ名
- `GitHubBranch`: ビルド対象ブランチ（mainなど）
- `CodeStarConnectionArn`: 上記で作成したCodeStar Connection ARN

---

### Backend CodeBuild のデプロイ

```bash
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file dashboard/codebuild-backend.yaml \
  --stack-name stack-finance-dashboard-cicd-backend \
  --parameter-overrides \
    ProjectName=build-finance-dashboard-backend \
    GitHubOwner=h-akira \
    GitHubRepo=FinanceDashboardProject_Backend \
    GitHubBranch=main \
    CodeStarConnectionArn=arn:aws:codeconnections:ap-northeast-1:XXXXXXXXXXXX:connection/xxx \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

**パラメータ**:
- `ProjectName`: CodeBuildプロジェクト名
- `GitHubOwner`: GitHubリポジトリオーナー
- `GitHubRepo`: GitHubリポジトリ名
- `GitHubBranch`: ビルド対象ブランチ（mainなど）
- `CodeStarConnectionArn`: 上記で作成したCodeStar Connection ARN

---

### Frontend CodeBuild のデプロイ

```bash
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file dashboard/codebuild-frontend.yaml \
  --stack-name stack-finance-dashboard-cicd-frontend \
  --parameter-overrides \
    ProjectName=build-finance-dashboard-frontend \
    GitHubOwner=h-akira \
    GitHubRepo=FinanceDashboardProject_Frontend \
    GitHubBranch=main \
    CodeStarConnectionArn=arn:aws:codeconnections:ap-northeast-1:XXXXXXXXXXXX:connection/xxx \
    S3BucketName=s3-finance-dashboard-contents \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

**パラメータ**:
- `ProjectName`: CodeBuildプロジェクト名
- `GitHubOwner`: GitHubリポジトリオーナー
- `GitHubRepo`: GitHubリポジトリ名
- `GitHubBranch`: ビルド対象ブランチ（mainなど）
- `CodeStarConnectionArn`: CodeStar Connection ARN
- `S3BucketName`: デプロイ先S3バケット名（Main Stackで作成済み）

---

## 🔄 更新

CloudFormationテンプレートを修正した後、同じコマンドで更新できます：

```bash
# Infrastructure更新例
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file common/codebuild-infra.yaml \
  --stack-name stack-finance-cicd-infra \
  --parameter-overrides ...

# Backend更新例
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file dashboard/codebuild-backend.yaml \
  --stack-name stack-finance-dashboard-cicd-backend \
  --parameter-overrides ... \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

---

## 🗑️ 削除

```bash
# Backend CodeBuild削除
AWS_PROFILE=finance aws cloudformation delete-stack \
  --stack-name stack-finance-dashboard-cicd-backend \
  --region ap-northeast-1

# Infrastructure CodeBuild削除
AWS_PROFILE=finance aws cloudformation delete-stack \
  --stack-name stack-finance-dashboard-cicd-infra \
  --region ap-northeast-1

# Frontend CodeBuild削除
AWS_PROFILE=finance aws cloudformation delete-stack \
  --stack-name stack-finance-dashboard-cicd-frontend \
  --region ap-northeast-1
```

---

## 📊 スタック状態確認

```bash
# Backend
AWS_PROFILE=finance aws cloudformation describe-stacks \
  --stack-name stack-finance-dashboard-cicd-backend \
  --region ap-northeast-1

# Infrastructure
AWS_PROFILE=finance aws cloudformation describe-stacks \
  --stack-name stack-finance-dashboard-cicd-infra \
  --region ap-northeast-1

# Frontend
AWS_PROFILE=finance aws cloudformation describe-stacks \
  --stack-name stack-finance-dashboard-cicd-frontend \
  --region ap-northeast-1
```

---

## 🔍 Backend CodeBuild の詳細

### 作成されるリソース

1. **CodeBuild Project** (`build-finance-dashboard-backend`)
   - SAMアプリケーションのビルドとデプロイ
   - GitHubリポジトリと連携（WebHook）
   - mainブランチへのPushで自動ビルド

2. **IAM Role** (`codebuild-build-finance-dashboard-backend-service-role`)
   - SAMデプロイに必要な権限
   - CloudFormation、Lambda、API Gateway、S3、IAM等

3. **IAM Managed Policy** (`CodeBuildCodeConnectionsPolicy-build-finance-dashboard-backend`)
   - CodeConnections（GitHub連携）権限

### 権限

Backend CodeBuildには以下の権限が付与されます：

- **CloudFormation**: SAM Transformとスタック管理
- **Lambda**: 関数の作成/更新/削除
- **API Gateway**: APIの作成/更新/削除
- **S3**: SAMアーティファクト用バケット操作
- **IAM**: Lambda実行ロールの作成
- **CloudWatch Logs**: ロググループ管理
- **SSM Parameter Store**: パラメータ読み取り

---

## 🔍 Frontend CodeBuild の詳細

### 作成されるリソース

1. **CodeBuild Project** (`build-finance-dashboard-frontend`)
   - フロントエンドアプリケーションのビルドとS3デプロイ
   - GitHubリポジトリと連携（WebHook）
   - mainブランチへのPushで自動ビルド

2. **IAM Role** (`codebuild-build-finance-dashboard-frontend-service-role`)
   - S3デプロイに必要な権限

3. **IAM Managed Policy** (`CodeBuildCodeConnectionsPolicy-build-finance-dashboard-frontend`)
   - CodeConnections（GitHub連携）権限

### 権限

Frontend CodeBuildには以下の権限が付与されます：

- **S3**: 指定バケットへのオブジェクト操作（PutObject、DeleteObject等）
- **CloudWatch Logs**: ビルドログ出力

### 環境変数

- `S3_BUCKET`: デプロイ先S3バケット名（パラメータから自動設定）

