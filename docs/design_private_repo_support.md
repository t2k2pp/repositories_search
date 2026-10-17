# プライベートリポジトリ取得機能および差分同期の改修設計書

## 1. 概要と背景
GitHub Pages（`https://t2k2pp.github.io/repositories_search/`）において、GitHub Personal Access Token（PAT）を設定しているにもかかわらず、Publicリポジトリしか取得・表示されず、Privateリポジトリが表示されない不具合が発生していた。
また、最新のGit版で実装された「差分更新（非破壊マージ・新リポジトリ検出）」「トークン診断機能」を100%保持した上で、Classic PAT（`ghp_...`）およびFine-grained PAT（`github_pat_...`）の双方でPrivateリポジトリを確実に取得・表示・検索・閲覧できるように修正する。

## 2. 原因分析

### 原因1: Authorization ヘッダー形式
- Fine-grained PAT（`github_pat_...`）では `Bearer ${token}` 形式が必須。`replace(/^(Bearer|token)\s+/i, '')` により接頭辞を除去して `Bearer` 形式に統一。

### 原因2: `GET /user` の403（User Profile権限不足）によるフォールバック失敗
- Fine-grained PAT でリポジトリ操作権限（All repositories など）が付与されていても、Account permissions の「User profile: Read-only」がない場合、`GET /user` は `403 Forbidden` を返す。
- 従来の判定では 403 の場合に `useAuthenticatedEndpoint = false` となり、公開エンドポイント `/users/{username}/repos` にフォールバックしてしまっていた。
- GitHub API の仕様上、公開エンドポイント `/users/{username}/repos` は **Public リポジトリしか返さない**。

## 3. 改修内容

### 3.1 トークン検証と認証エンドポイントの優先 (`syncDiffRepositories`)
- トークンが設定されている場合は、まず `useAuthenticatedEndpoint = true` を設定する。
- `GET /user` が 200 OK の場合はユーザープロフィールとスコープ情報を取得して表示。
- `GET /user` が 403 の場合（Fine-grained PAT のアカウント権限なし）でも、諦めずに `useAuthenticatedEndpoint = true` を維持し、リポジトリ取得エンドポイント `https://api.github.com/user/repos?visibility=all&affiliation=owner,collaborator,organization_member` を呼び出す。
- `GET /user` が 401（無効なトークン文字列）の場合のみ、公開エンドポイントにフォールバックする。

### 3.2 ページネーションとフィルタ
- ページ上限を最大10ページ（最大1000件）に拡張。
- ユーザー本人名義のリポジトリフィルタにおいて、`state.authenticatedUser` と `state.username` の双方で大文字小文字を許容してマッチング。

### 3.3 既存機能の完全保持
- 差分更新（`syncDiffRepositories`）
- 非破壊的マージ（`mergeRepositoryLists`）
- 未検出リポジトリの検出・通知
- README解析
- キャッシュ管理（IndexedDB / localStorage）
