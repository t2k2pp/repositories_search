# タスク ToDo リスト: プライベートリポジトリ取得対応

- [x] 1. 設計書・ToDoの作成と構成確認
- [x] 2. `index.html` の `getGitHubHeaders` を `Bearer` 認証に対応
- [x] 3. `index.html` の `loadUserAndRepositories` で `/user/repos?visibility=all` を優先し、Fine-grained PAT / Privateリポジトリを確実に取得するよう改修
- [x] 4. ページネーション上限の拡張（1000件まで）とキャッシュ更新処理の改善
- [x] 5. 設定保存・トークン入力時の再取得処理の改善
- [x] 6. リモートリポジトリ（mainブランチ）へコミット・プッシュ
- [x] 7. ローカルHTTPサーバーを起動してEdge等で動作確認
