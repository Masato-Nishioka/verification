# GitHub Actions + Helm vs Argo CD + Helm 比較

本ドキュメントは、GitHub Actions を用いて Helm チャートを GKE などの Kubernetes クラスタにデプロイする方法と、GitOps ツールである Argo CD を用いた場合の違いをまとめたものです。

---

## 🎯 比較の目的

Git リポジトリ上の Helm チャートをソース・オブ・トゥルース（信頼できる唯一の情報源）として、  
「Git の状態 = クラスタの状態」に保つ、いわゆる **GitOps を実現したい**という観点から比較を行います。

---

## ⚙️ 比較表

| 項目 | GitHub Actions + Helm | Argo CD + Helm |
|------|------------------------|----------------|
| 自動同期 | ❌ 手動トリガー必要（例：push） | ✅ Git 監視で自動同期 |
| prune（不要リソースの削除） | ❌ 原則できない（後述） | ✅ `prune: true` で削除 |
| 差分の可視化 | ❌ なし | ✅ Web UI で確認可能 |
| 手動変更の修復（Self-heal） | ❌ なし | ✅ `selfHeal: true` で自動修復 |
| Helm の状態管理機能 | ✅ 使用可能（リリースあり） | ✅ 使用可能（Helm サポートあり） |
| ダウンタイムのリスク | ⛔ uninstall → install で発生可能 | ✅ 差分適用で最小限に |
| マニフェスト管理単位 | チャート単位 | アプリ単位（Git repo + path） |
| 導入の手間 | 少ない（GitHub Actions に統合） | 初期セットアップが必要 |
| CI/CD との統合 | ✅ GitHub Actions ネイティブ | ✅ 外部連携で可能（Webhook等） |
| セキュリティ制御（RBAC） | GitHub 側に依存 | Argo CD 側で細かく制御可能 |

---

## 🔍 GitHub Actions + Helm の動作と制約

### デプロイフロー例

```yaml
helm upgrade --install "$RELEASE_NAME" "$CHART_PATH" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  --wait
```

- `helm upgrade` は **チャートで定義されたリソースを作成/更新**する
- ただし、**templates ディレクトリから削除されたリソースはクラスタに残ったまま**

### リソース削除（prune）ができない理由

Helm は「現在のチャートに含まれる定義だけ」を見て状態を変更する。  
過去のリソースがチャートから消えても、それがクラスタ上から消されることはない。

### 回避策：uninstall → install

```bash
# 既存リリースを削除
helm uninstall $RELEASE_NAME -n $NAMESPACE

# 再度インストール
helm install $RELEASE_NAME $CHART_PATH -n $NAMESPACE
```

これにより「削除されたリソース」もクラスタから消えるが、**すべて再作成されるためダウンタイムが発生する可能性あり**。

### 代替案（非推奨）

```bash
helm template my-app ./charts/my-app > manifest.yaml
kubectl apply -f manifest.yaml --prune ...
```

この方法で `--prune` によるリソース削除は可能だが、Helm のリリース管理機能（アップグレード、ロールバック等）が使えなくなる。

---

## 🚀 Argo CD + Helm の利点

Argo CD は GitOps ツールとして、以下のような機能を持つ：

```yaml
syncPolicy:
  automated:
    prune: true       # Git に存在しないリソースを自動削除
    selfHeal: true    # クラスタ上の手動変更をGitに合わせて修復
```

- `prune: true`：Git 上から削除されたリソースはクラスタからも削除
- `selfHeal: true`：クラスタ側で手動変更された内容は Git ベースで上書きされる
- UI や CLI で差分・同期状態を可視化できる

### Helm チャートの扱い

Argo CD は Helm チャートを直接 Git リポジトリから参照して同期・デプロイが可能。

```yaml
source:
  repoURL: https://github.com/your-org/your-repo
  path: charts/my-app
  targetRevision: HEAD
  helm:
    values: values-production.yaml
```

---

## ✅ 結論

| 要件 | 推奨手段 |
|------|-----------|
| 手軽に CI/CD に Helm を使いたい | GitHub Actions + Helm |
| GitOps を本格的に導入したい | ✅ Argo CD + Helm |
| クラスタの状態と Git を完全に同期させたい | ✅ Argo CD |
| 差分表示・自動修復・安全な適用が必要 | ✅ Argo CD |

GitHub Actions + Helm は「CI/CD による Helm ベースの手動運用」には便利ですが、  
「Git の状態 = クラスタの状態」を保証するには限界があり、**本格的な GitOps を目指すなら Argo CD の導入が強く推奨されます**。

---
