# プライベートリポジトリ取得機能および差分同期の改修設計書

## 1. 概要と背景
GitHub Pages（`https://t2k2pp.github.io/repositories_search/`）において、GitHub Personal Access Token（PAT）を設定しているにもかかわらず、Publicリポジトリ（110件）しか取得・表示されず、Privateリポジトリ（34件、全144件）が表示されない問題の修正およびリアルタイム診断機能の追加。

## 2. 徹底的な原因分析

### 原因1: 認証時の厳格すぎるユーザー名フィルタ（決定打）
- **現象**: `/user/repos` でリポジトリを取得した直後に、`fetchedRepos.filter(r => !r.owner || r.owner.login.toLowerCase() === targetOwner ...)` を実行していた。
- **原因**: ユーザーがアクセス権を持つプライベートリポジトリのうち、Organization（組織）所有のものや別名義のリポジトリがある場合、このフィルタにより34件のプライベートリポジトリが全件除外され、結果として Public 110件のみになっていた。
- **対策**: 認証エンドポイント `/user/repos` で取得されたリポジトリは、ユーザーが正当な権限を持つ全リポジトリであるため、フィルタを完全に撤廃して全件保持する。

### 原因2: トークン種別に応じたAuthorizationヘッダーの最適化
- **現象**: Classic PAT（`ghp_...`）とFine-grained PAT（`github_pat_...`）で最適なプレフィックスが異なる。
- **対策**: `ghp_` で始まる Classic PAT には `Authorization: token ${rawToken}` を、Fine-grained PAT やその他には `Authorization: Bearer ${rawToken}` を割り当てる。

### 原因3: `/user/repos` エラー時のサイレントフォールバック
- **現象**: `/user/repos` で万一エラー（401/403/422）が発生した場合、公開エンドポイント（`/users/t2k2pp/repos`）にフォールバックし、エラー内容がユーザーに伝わらず「なぜかPublicしか出ない」状態になっていた。
- **対策**: エラー発生時はHTTPステータスコードおよびGitHub APIのエラーメッセージ（message）を取得し、トースト通知および設定モーダル上で明示する。

### 原因4: トークンの権限（スコープ）の可視化不足
- **対策**: 設定モーダルに「**接続テスト**」ボタンを新設。ボタン1クリックで `/user` および `/user/repos` を叩き、
  - 認証ユーザー名
  - トークンのスコープ（`repo` スコープの有無など）
  - 1ページ目に検出されたプライベートリポジトリの件数
  をリアルタイムに診断表示する。

## 3. 改修内容
1. `index.html`: `getGitHubHeaders` の最適化（Classicならtoken、Fine-grainedならBearer）
2. `index.html`: `syncDiffRepositories` での認証時フィルタ撤廃（全プライベートリポジトリを保持）
3. `index.html`: 設定モーダルへの「接続テスト（Stethoscopeアイコン）」ボタンの追加
4. `index.html`: `testTokenConnection` によるリアルタイム診断機能の実装
