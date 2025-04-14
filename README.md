# GKE上に検証環境を自動作成する仕組みの構築タスク一覧

## ✅ 1. 前提準備フェーズ（初回のみ）

### GKE クラスタ関連

- [ ] GKEクラスタの作成（もしくは既存クラスタの確認）
- [ ] 適切なノードプール構成（Auto-scalingやコスト制御含む）の設計

### GCP 認証まわり

- [x] GitHub Actions から GKE に接続するための認証方式を決定  
  - Workload Identity Federation（推奨） or サービスアカウントキー
- [x] GCP IAMの設定（GitHub Actions用サービスアカウントに権限を付与）

---

## ✅ 2. GitHub Actions ワークフロー作成フェーズ

### ワークフロー設計

- [x] トリガー設定（例：`on: push`、ブランチ名フィルタ、manifestファイル変更検知など）
- [x] 使用するYAMLファイルのパスの取り扱い設計（e.g. `./k8s/branch-name/*.yaml`）

### ワークフローステップ

- [ ] `gcloud` / `kubectl` のセットアップ
- [x] GCPへの認証（Workload Identity連携 or サービスアカウントキーの読み取り）
- [ ] GKEクラスタへの接続（`gcloud container clusters get-credentials`）
- [ ] ブランチに基づいたNamespaceの作成（例：`kubectl create namespace <branch>`）
- [ ] `kubectl apply -f` でmanifestをデプロイ

---

## ✅ 3. manifest構成ルールの整備

- [x] manifestの格納パスの統一ルールを決定（例：`k8s/<branch>`）
- [ ] Namespace、Service名などにブランチ名を取り入れる方法をルール化
- [ ] 必要に応じてConfigMapやSecretのテンプレートも準備

---

## ✅ 4. 動作確認・テストフェーズ

- [ ] テスト用のサンプルmanifestを用意
- [ ] テストブランチでのmanifest push → GKE反映の流れを検証
- [ ] 失敗パターンの確認（無効なmanifestや認証エラー時の動作確認）

---

## ✅ 5. オプション（必要に応じて）

### クリーンアップ機構（推奨）

- [ ] ブランチ削除に連動してNamespace/PODを自動削除する処理の追加  
  - `delete`イベントをGitHub Actionsでハンドリング or 定期バッチ
