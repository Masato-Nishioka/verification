# GitHub Actions ワークフロー: マニフェストのプッシュで GKE にデプロイ

このリポジトリには、`k8s/` ディレクトリに Kubernetes マニフェストが push された際に、Google Kubernetes Engine (GKE) に自動デプロイを行う GitHub Actions ワークフローが含まれています。

## 📂 ワークフローファイル

```
.github/workflows/deploy-to-gke.yaml
```

## 🚀 トリガー

このワークフローは、以下の条件でトリガーされます。

```yaml
on:
  push:
    branches:
      - "**"
    paths:
      - "k8s/**"
```

- **すべてのブランチ**への push
- `k8s/` ディレクトリ配下のファイルが変更された場合

## 🔧 処理内容

このワークフローでは、以下の処理を行います。

1. ソースコードのチェックアウト
2. Workload Identity Federation を使って Google Cloud に認証
3. `gcloud` CLI のセットアップ
4. GKE クラスタの認証情報を取得
5. ブランチ名に応じた Kubernetes namespace を作成（存在しない場合）
6. 該当 namespace にマニフェストを適用

## 🌐 Namespace によるブランチごとのレビュー環境

Namespace は以下の形式で自動生成されます：

```
<ブランチ名>
```

例：`feature/api-refactor` というブランチなら、

```
Namespace: feature-api-refactor
```

> ⚠ Kubernetes の namespace には `/` や `_` を含められないため、以下のような変換が必要です。

## 🔤 Namespace 命名変換処理

以下のように `github.ref_name` を安全な形式に変換して namespace に使用します。

```yaml
env:
  RAW_BRANCH_NAME: ${{ github.ref_name }}
  SAFE_BRANCH_NAME: ${{ github.ref_name }}
  NAMESPACE: ${{ env.SAFE_BRANCH_NAME }}

# 変換例（ステップ内で実行）:
- name: Sanitize branch name
  run: |
    SANITIZED=$(echo "$RAW_BRANCH_NAME" | tr '/_' '--')
    echo "SANITIZED_BRANCH_NAME=$SANITIZED" >> $GITHUB_ENV
```

> `tr` コマンドで `/` や `_` を `-` に変換しています。

## 🔐 必要な GitHub シークレット

以下のシークレットを GitHub リポジトリに設定してください：

| シークレット名 | 説明 |
|----------------|------|
| `GCP_PROJECT_ID` | GCP プロジェクト ID |
| `GKE_CLUSTER_NAME` | GKE クラスタ名 |
| `GKE_CLUSTER_LOCATION` | GKE クラスタのリージョンまたはゾーン |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Workload Identity Federation プロバイダ名 |
| `GCP_SERVICE_ACCOUNT` | サービスアカウントのメールアドレス |

## 📦 ディレクトリ構成の例

ブランチごとのマニフェストは `k8s/` 配下にブランチ名のディレクトリを作って格納します：

```
k8s/
├── main/
│   └── deployment.yaml
├── feature-xyz/
│   └── deployment.yaml
```

## 📄 ワークフロー定義の概要（デプロイ）

```yaml
name: Deploy to GKE on manifest push

on:
  push:
    branches:
      - "**"
    paths:
      - "k8s/**"

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    env:
      PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      CLUSTER_NAME: ${{ secrets.GKE_CLUSTER_NAME }}
      CLUSTER_LOCATION: ${{ secrets.GKE_CLUSTER_LOCATION }}
      RAW_BRANCH_NAME: ${{ github.ref_name }}
    steps:
      - uses: actions/checkout@v4

      - name: Sanitize branch name
        run: |
          SANITIZED=$(echo "$RAW_BRANCH_NAME" | tr '/_' '--')
          echo "SANITIZED_BRANCH_NAME=$SANITIZED" >> $GITHUB_ENV

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - uses: google-github-actions/setup-gcloud@v2

      - run: |
          gcloud container clusters get-credentials "$CLUSTER_NAME" \
            --region "$CLUSTER_LOCATION" \
            --project "$PROJECT_ID"

      - run: |
          kubectl get namespace "${SANITIZED_BRANCH_NAME}" || \
          kubectl create namespace "${SANITIZED_BRANCH_NAME}"

      - run: |
          kubectl apply -n "${SANITIZED_BRANCH_NAME}" -f "k8s/${RAW_BRANCH_NAME}"
```

## 🧹 削除用ワークフローの例（ブランチ削除時）

以下のワークフローは、ブランチが削除された際に対応する namespace を削除します。

```yaml
# .github/workflows/delete-from-gke.yaml

name: Delete namespace on branch deletion

on:
  delete:
    branches:
      - "**"

jobs:
  delete:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    env:
      PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      CLUSTER_NAME: ${{ secrets.GKE_CLUSTER_NAME }}
      CLUSTER_LOCATION: ${{ secrets.GKE_CLUSTER_LOCATION }}
      RAW_BRANCH_NAME: ${{ github.event.ref }}
    steps:
      - name: Sanitize branch name
        run: |
          SANITIZED=$(echo "$RAW_BRANCH_NAME" | tr '/_' '--')
          echo "SANITIZED_BRANCH_NAME=$SANITIZED" >> $GITHUB_ENV

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - uses: google-github-actions/setup-gcloud@v2

      - run: |
          gcloud container clusters get-credentials "$CLUSTER_NAME" \
            --region "$CLUSTER_LOCATION" \
            --project "$PROJECT_ID"

      - run: |
          kubectl delete namespace "${SANITIZED_BRANCH_NAME}" || echo "Namespace not found"
```

## 🛠 前提条件

- GKE クラスタが事前に作成されていること
- サービスアカウントに以下の権限があること：
  - GKE クラスタの認証情報の取得
  - Namespace の作成 / 削除
  - Kubernetes マニフェストの適用
- GitHub と GCP 間で Workload Identity Federation が正しく設定されていること

## 📘 参考リンク

- [google-github-actions/auth](https://github.com/google-github-actions/auth)
- [google-github-actions/setup-gcloud](https://github.com/google-github-actions/setup-gcloud)
- [GKE ドキュメント](https://cloud.google.com/kubernetes-engine/docs)
