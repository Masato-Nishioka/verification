# verification

検証用リポジトリ

## GitHub to GKEについて

ディレクトリ構成

```tree
.
├── terraform/
│   ├── github-to-gsa/
│   │   ├── main.tf
│   │   ├── provider.tf
│   │   └── terraform.tfvars
│   ├── gsa-to-ksa/
│   │   ├── iam.tf
│   │   ├── provider.tf
│   │   ├── terraform.tfvars
│   │   └── variables.tf
│   └── gke/
│        ├── main.tf
│        ├── terraform.tfvars
│        └── variables.tf
├── manifest/
│    ├── namespace.yaml
│    ├── role.yaml
│    ├── rolebinding.yaml
│    └── serviceaccount.yaml
```

## 実行順序

- GitHub から GSA への接続設定 (github-to-gsa)
- GKE クラスタの作成 (gke)
- Kubernetes RBAC 設定 (manifest)
- GSA と KSA の接続設定 (gsa-to-ksa)

```bash
cd terraform/github-to-gsa
terraform init
terraform apply

cd terraform/gke
terraform init
terraform apply

cd manifest
kubectl apply -f namespace.yaml
kubectl apply -f role.yaml
kubectl apply -f serviceaccount.yaml
kubectl apply -f rolebinding.yaml

cd terraform/gsa-to-ksa
terraform init
terraform apply
```
