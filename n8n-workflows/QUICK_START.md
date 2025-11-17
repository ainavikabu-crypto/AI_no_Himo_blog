# クイックスタートガイド

このガイドでは、最短でワークフローを動かすための手順を説明します。

## 前提条件

- n8nがインストールされている（Dockerまたはnpm）
- 以下のAPIキーを取得済み：
  - OpenAI API Key
  - Perplexity API Key
  - WordPressサイトとアプリケーションパスワード

## 5ステップで開始

### ステップ1: ワークフローをインポート

1. n8nを開く（http://localhost:5678）
2. 「Import from File」をクリック
3. `seo-blog-auto-publisher-ja.json`を選択
4. 「Import」をクリック

### ステップ2: OpenAI認証情報を設定

1. 左メニューの「Credentials」をクリック
2. 「Add Credential」→「OpenAI」を選択
3. API Keyを入力
4. 「Save」をクリック

### ステップ3: Perplexity認証情報を設定

1. 「Credentials」→「Add Credential」→「HTTP Header Auth」を選択
2. 以下を入力：
   - Name: `Perplexity`
   - Header Name: `Authorization`
   - Header Value: `Bearer YOUR_PERPLEXITY_API_KEY`
3. 「Save」をクリック

### ステップ4: WordPress認証情報を設定

1. WordPress管理画面でアプリケーションパスワードを生成：
   - ユーザー → プロフィール
   - 「アプリケーションパスワード」セクション
   - アプリケーション名（例：n8n）を入力
   - 「新しいアプリケーションパスワードを追加」をクリック
   - パスワードをコピー

2. n8nで設定：
   - 「Credentials」→「Add Credential」→「WordPress」
   - Site URL: `https://your-site.com`（あなたのWordPressサイトURL）
   - Username: WordPressユーザー名
   - Password: アプリケーションパスワード
   - 「Save」をクリック

### ステップ5: ワークフローをアクティブ化して実行

1. ワークフローに戻る
2. 各ノードの認証情報を選択（赤いマークがなくなるまで）
3. 右上のトグルスイッチをONにする
4. 「フォーム送信時」ノードのTest URLをコピー
5. ブラウザで開いてキーワードを入力
6. 送信！

## テスト実行

最初のテストには以下のキーワードを使用してください：

```
2025年のAI技術トレンドと日本企業への影響について調査してください
```

実行後、WordPress管理画面の「投稿」→「下書き」に新しい記事が追加されます。

## トラブルシューティング

### 認証エラーが出る場合

1. APIキーをコピペし直す（余分なスペースがないか確認）
2. APIキーの有効期限を確認
3. 各サービスのダッシュボードで使用状況を確認

### WordPressに投稿されない場合

1. WordPressサイトURLを確認（末尾のスラッシュなし）
2. アプリケーションパスワードを再生成
3. WordPressのREST APIが有効か確認

### それでも解決しない場合

詳細なトラブルシューティングはREADME.mdを参照してください。

## 次のステップ

- カスタマイズ方法を学ぶ（README.md参照）
- Telegram通知を設定する（オプション）
- スケジュール実行を設定する
- 複数のWordPressサイトに対応させる

---

質問や問題がある場合は、GitHubのIssuesで報告してください。
