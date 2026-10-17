# プライベートリポジトリ取得機能の改修設計書

## 1. 概要と背景
GitHub Pages（`https://t2k2pp.github.io/repositories_search/`）において、GitHub Personal Access Token（PAT）を設定しているにもかかわらず、Publicリポジトリしか取得・表示されず、Privateリポジトリが表示されない不具合が発生している。
本改修では、Classic PAT（`ghp_...`）およびFine-grained PAT（`github_pat_...`）の双方で、Privateリポジトリを正常に取得・表示・検索・閲覧できるように修正する。

## 2. 原因分析

### 原因1: Authorization ヘッダー形式が `token` 固定になっている
- **現象**: `getGitHubHeaders()` で `Authorization: token ${token}` を送信している。
- **原因**: Classic PAT では `token` 構文が受け入れられるが、Fine-grained PAT（`github_pat_...`）では `Bearer ${token}` 形式が必須であり、`token` プレフィックスを指定すると `401 Unauthorized (Bad credentials)` となり認証に失敗する。
- **対策**: `Authorization: Bearer ${token}` に統一する（GitHub API では Classic PAT も Fine-grained PAT も `Bearer` 認証をサポート）。

### 原因2: `GET /user` による事前判定によるフォールバックの失敗
- **現象**: トークン検証のために `GET https://api.github.com/user` を実行し、成功した場合のみ `useAuthenticatedEndpoint = true` として `/user/repos` を叩き、失敗した場合は `/users/{username}/repos` へフォールバックしている。
- **原因**: Fine-grained PAT では、ユーザーがリポジトリ操作権限（All repositories など）を付与していても、Account permissions の「User profile: Read-only」を付与していない場合、`GET /user` は `403 Forbidden (Resource not accessible by personal access token)` を返す。この結果、有効なトークンであるにもかかわらず `useAuthenticatedEndpoint = false` と判定されてしまう。
- **対策**: トークンが設定されている場合は、`GET /user` の成功成否にかかわらず、まずプライベートリポジトリを取得可能な `GET /user/repos` を優先して呼び出す。

### 原因3: `/users/{username}/repos` は Private リポジトリを返さない仕様
- **現象**: `useAuthenticatedEndpoint = false` になった場合、`GET https://api.github.com/users/{username}/repos` が呼び出される。
- **原因**: GitHub REST API の仕様により、`/users/{username}/repos` は Authorization ヘッダーを付けていても **Public リポジトリしか返さない**。
- **対策**: ユーザー自身のトークンが提供されている場合は、必ず `GET /user/repos` からリポジトリを取得する。

### 原因4: `/user/repos` のクエリパラメータとページネーション
- **現象**: ページネーションが3ページ（最大300件）で打ち切られ、また `visibility` パラメータが明示されていない。
- **対策**: `visibility=all` を明示し、ページネーションを最大10ページ（最大1000件）まで取得可能とする。

### 原因5: キャッシュのTTLおよびトークン設定時の再取得
- **現象**: 以前にトークン未設定（または上記不具合）で取得されたキャッシュ（Publicのみ）が IndexedDB / localStorage に残っていると、トークンを設定しても古いキャッシュが表示され続ける。
- **対策**: トークン保存時に既存リポジトリキャッシュをリフレッシュし、強制的に最新のデータを取得し直す。

## 3. 改修設計

### 3.1 ヘッダー生成処理 (`getGitHubHeaders`)
```javascript
function getGitHubHeaders(acceptRaw = false, includeToken = true) {
    const headers = {
        'Accept': acceptRaw ? 'application/vnd.github.v3.raw' : 'application/vnd.github.v3+json'
    };
    if (includeToken && state.token && state.token.trim() !== '') {
        headers['Authorization'] = `Bearer ${state.token.trim()}`;
    }
    return headers;
}
```

### 3.2 リポジトリ取得ロジック (`loadUserAndRepositories`)
1. トークンが設定されている場合:
   - まず `GET /user` でユーザープロフィールの取得を試みる（403/エラーでも中断しない）。
   - エンドポイントとして `https://api.github.com/user/repos?visibility=all&affiliation=owner&sort=updated&per_page=100&page=${page}` を使用し、プライベートリポジトリを含む一覧を取得する。
   - 取得成功時は、その中から指定オーナーのリポジトリを取り出し（または本人所有のリポジトリ）、キャッシュに保存して画面描画する。
2. トークンが設定されていない場合:
   - 従来通り公開エンドポイント `https://api.github.com/users/${username}/repos` を使用する。

### 3.3 設定保存モーダル (`saveTokenOnly` / `applySettingsAndReload`)
- トークン設定更新時、既存のキャッシュに関わらず `loadUserAndRepositories(true)` を確実に実行する。
