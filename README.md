# Finance Project CI/CD

CloudFormationによるFinance ProjectのCodeBuild定義です。

## 🎯 目的

CloudFormationは**環境構築不要**という強みがあるため、CI/CDパイプラインのみCloudFormationで管理します。

- ✅ Python/Node.js不要
- ✅ CDK CLI不要
- ✅ AWS CLIまたはコンソールのみでデプロイ可能
- ✅ 環境依存なし

## 📁 構成

```
FinanceProject_CICD/
├── codebuild-backend.yaml   # Backend (SAM) 用 CodeBuild
├── codebuild-frontend.yaml  # Frontend (S3) 用 CodeBuild
├── codebuild-infra.yaml     # Infrastructure (CDK) 用 CodeBuild
└── README.md
```

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

### Backend CodeBuild のデプロイ

```bash
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-backend.yaml \
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

### Infrastructure CodeBuild のデプロイ

```bash
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-infra.yaml \
  --stack-name stack-finance-dashboard-cicd-infra \
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

### Frontend CodeBuild のデプロイ

```bash
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-frontend.yaml \
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
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-backend.yaml \
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

---

## 🆚 Terraform/CDK版との違い

| 項目 | CloudFormation | Terraform/CDK |
|------|----------------|---------------|
| 環境構築 | 不要 | 必要（Python/Node.js/Terraform CLI） |
| デプロイ | AWS CLIのみ | Terraform CLI / CDK CLI |
| 学習コスト | 低（YAML） | 中〜高（HCL/Python） |
| 再利用性 | 低 | 高 |
| 適用範囲 | CodeBuildのみ | インフラ全体 |

**結論**: CI/CDパイプラインのみCloudFormationで管理し、その他のインフラ（Cognito、S3、CloudFront等）はCDK/Terraformで管理する。

---

## 📝 カスタマイズ

### プロジェクト名を変更

デフォルトのパラメータを変更するには、YAMLファイルの`Parameters`セクションを編集：

```yaml
Parameters:
  ProjectName:
    Type: String
    Default: your-custom-project-name  # ここを変更
```

### IAM権限を追加

BackendまたはFrontendで追加の権限が必要な場合、該当するPolicyに追加：

```yaml
- Sid: YourCustomPermission
  Effect: Allow
  Action:
    - service:Action
  Resource:
    - arn:aws:service:region:account:resource
```

---

## 🔗 関連リポジトリ

- **Backend**: [FinanceDashboardProject_Backend](https://github.com/h-akira/FinanceDashboardProject_Backend)
- **Frontend**: [FinanceDashboardProject_Frontend](https://github.com/h-akira/FinanceDashboardProject_Frontend)
- **Infra (CDK)**: [FinanceProject_Infra_CDK](../FinanceProject_Infra_CDK)
- **Infra (Terraform)**: [FinanceProject_Infra](../FinanceProject_Infra)

---

## 💡 Tips

### CodeStar Connection ARNの取得

```bash
aws codeconnections list-connections --region ap-northeast-1
```

### S3バケット名の確認

```bash
aws s3 ls | grep finance
```

### CodeBuildログの確認

```bash
# Backend
aws logs tail /aws/codebuild/build-finance-dashboard-backend --follow

# Frontend
aws logs tail /aws/codebuild/build-finance-dashboard-frontend --follow
```

### 手動ビルド実行

```bash
# Backend
aws codebuild start-build \
  --project-name build-finance-dashboard-backend \
  --region ap-northeast-1

# Frontend
aws codebuild start-build \
  --project-name build-finance-dashboard-frontend \
  --region ap-northeast-1
```

---

## ❓ トラブルシューティング

### エラー: "User is not authorized to perform: codeconnections:UseConnection"

**原因**: CodeStar Connectionが承認されていない

**解決**: AWSコンソールで接続を承認してください（上記「前提条件」参照）

### エラー: "Stack with id XXX does not exist"

**原因**: スタック名が間違っている

**解決**: `aws cloudformation list-stacks`でスタック名を確認

### エラー: "Bucket does not exist"

**原因**: S3バケットが作成されていない（Frontend）

**解決**: Main Stack（CDK/Terraform）でS3バケットを先に作成してください

---

**Happy Building! 🚀**
