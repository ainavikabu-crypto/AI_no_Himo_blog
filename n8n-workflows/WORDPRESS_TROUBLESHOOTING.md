# WordPressとn8n連携のトラブルシューティングガイド

## 🔍 問題の概要

WordPressとn8nの連携がうまくいかない場合、以下の主な原因が考えられます。

---

## 📋 チェックリスト

このチェックリストを順番に確認してください。

### ✅ 1. WordPress REST APIが有効か確認

**確認方法:**

ブラウザで以下のURLにアクセス:
```
https://your-wordpress-site.com/wp-json/wp/v2/posts
```

**期待される結果:**
- JSON形式のデータが表示される
- または「REST API is disabled」以外のメッセージ

**エラーの場合:**
- WordPressのREST APIが無効化されている可能性
- プラグイン「Disable REST API」などが有効になっていないか確認
- `.htaccess`でREST APIへのアクセスがブロックされていないか確認

---

### ✅ 2. WordPress アプリケーションパスワードを確認

**問題:** アプリケーションパスワードが無効、または誤っている

**解決方法:**

1. **WordPress管理画面**にログイン
2. **ユーザー** → **プロフィール** に移動
3. 下にスクロールして **「アプリケーションパスワード」** セクションを探す
4. 新しいアプリケーション名（例: `n8n-integration`）を入力
5. **「新しいアプリケーションパスワードを追加」** をクリック
6. 生成されたパスワードをコピー（スペースを含めてそのままコピー）

**n8nでの設定:**
1. n8nワークフローエディタを開く
2. **「WordPress投稿」** ノードをクリック
3. Credentialの鉛筆アイコンをクリック
4. 以下を入力:
   - **Site URL**: `https://your-site.com` (末尾のスラッシュなし)
   - **Username**: WordPressユーザー名
   - **Password**: コピーしたアプリケーションパスワード

---

### ✅ 3. WordPress URLがハードコードされている問題

**問題の特定:**

現在のワークフロー（`アイヒモのSEOブログ自動生成_Tavily版_GoogleDrive対応_修正版.json`）では、以下のノードでWordPress URLがハードコードされています:

- **「WordPressに画像アップロード」** ノード（行356）
  ```
  https://ainohimo.com/wp-json/wp/v2/media
  ```

- **「WordPress投稿に画像設定」** ノード（行391）
  ```
  https://ainohimo.com/wp-json/wp/v2/posts/{{ $('WordPress投稿').item.json.id }}
  ```

**解決方法:**

#### オプションA: ワークフローJSONを直接編集

1. n8nでワークフローをエクスポート
2. テキストエディタで開く
3. `https://ainohimo.com` を **あなたのWordPressサイトURL** に置き換え
4. n8nにインポート

#### オプションB: n8nエディタで手動変更

1. n8nワークフローエディタを開く
2. **「WordPressに画像アップロード」** ノードをクリック
3. **URL** フィールドを編集:
   ```
   https://YOUR-SITE.com/wp-json/wp/v2/media
   ```
4. **「WordPress投稿に画像設定」** ノードをクリック
5. **URL** フィールドを編集:
   ```
   https://YOUR-SITE.com/wp-json/wp/v2/posts/{{ $('WordPress投稿').item.json.id }}
   ```

---

### ✅ 4. n8n環境変数を使用する（推奨）

**メリット:**
- URLを一箇所で管理
- ワークフローの再利用性が向上
- 環境ごとに異なるURLを簡単に設定可能

**設定方法:**

#### 4-1. n8nに環境変数を追加

**Docker の場合:**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e WORDPRESS_SITE_URL=https://your-site.com \
  -e TELEGRAM_CHAT_ID=your-chat-id \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

**npm の場合:**
```bash
export WORDPRESS_SITE_URL=https://your-site.com
export TELEGRAM_CHAT_ID=your-chat-id
n8n start
```

**n8n Cloud の場合:**
1. n8n Cloud ダッシュボードにログイン
2. **Settings** → **Environments** に移動
3. 環境変数を追加:
   - Key: `WORDPRESS_SITE_URL`
   - Value: `https://your-site.com`

#### 4-2. ワークフローで環境変数を使用

**「WordPressに画像アップロード」** ノードのURLを変更:
```
={{ $env.WORDPRESS_SITE_URL }}/wp-json/wp/v2/media
```

**「WordPress投稿に画像設定」** ノードのURLを変更:
```
={{ $env.WORDPRESS_SITE_URL }}/wp-json/wp/v2/posts/{{ $('WordPress投稿').item.json.id }}
```

---

### ✅ 5. Telegram通知の認証情報を設定（オプション）

**問題:** 完了通知ノードの認証情報が未設定

**現在の状態:**
```json
"credentials": {
  "telegramApi": {
    "id": "YOUR_TELEGRAM_CREDENTIAL_ID",
    "name": "Telegram account"
  }
}
```

**解決方法:**

Telegram通知が不要な場合は、**「完了通知」ノードを削除**するか無効化してください。

Telegram通知が必要な場合:

1. **Telegram Bot を作成**
   - Telegramで `@BotFather` を検索
   - `/newbot` コマンドで新しいボットを作成
   - Bot Tokenをコピー

2. **Chat ID を取得**
   - Telegramで `@userinfobot` を検索
   - メッセージを送信してChat IDを取得

3. **n8nで認証情報を設定**
   - **「完了通知」** ノードをクリック
   - Credentialの鉛筆アイコンをクリック
   - **Access Token** に Bot Token を入力

4. **環境変数にChat IDを設定**
   ```bash
   export TELEGRAM_CHAT_ID=your-chat-id
   ```

---

## 🐛 よくあるエラーと解決方法

### エラー 1: `401 Unauthorized`

**原因:**
- WordPress アプリケーションパスワードが無効
- ユーザー名またはパスワードが間違っている

**解決方法:**
1. WordPress アプリケーションパスワードを再生成
2. n8nの認証情報を更新
3. ユーザー名とパスワードにスペースや特殊文字が含まれていないか確認

---

### エラー 2: `403 Forbidden`

**原因:**
- WordPressのパーマリンク設定が正しくない
- REST APIへのアクセスがブロックされている
- セキュリティプラグインがREST APIをブロックしている

**解決方法:**
1. **WordPress管理画面** → **設定** → **パーマリンク** を開く
2. 「投稿名」または「カスタム構造」を選択
3. **「変更を保存」** をクリック
4. セキュリティプラグイン（例: Wordfence、iThemes Security）の設定を確認
5. REST APIへのアクセスを許可

---

### エラー 3: `404 Not Found`

**原因:**
- WordPress サイトのURLが間違っている
- REST APIエンドポイントが存在しない

**解決方法:**
1. WordPress サイトのURLが正しいか確認（`https://`を含む）
2. 末尾のスラッシュを削除
3. REST APIエンドポイントにアクセスできるか確認:
   ```
   https://your-site.com/wp-json/wp/v2/posts
   ```

---

### エラー 4: `500 Internal Server Error`

**原因:**
- WordPressのPHPエラー
- プラグインの競合
- サーバーの設定問題

**解決方法:**
1. WordPressのデバッグモードを有効化:
   - `wp-config.php` に以下を追加:
     ```php
     define('WP_DEBUG', true);
     define('WP_DEBUG_LOG', true);
     ```
2. `/wp-content/debug.log` を確認
3. 問題のあるプラグインを無効化
4. PHPのメモリ制限を増やす（最低256MB推奨）

---

### エラー 5: 画像アップロードが失敗する

**原因:**
- 画像URLが無効
- WordPressのメディアアップロード制限
- ファイルサイズが大きすぎる

**解決方法:**
1. **画像URL設定** ノードで別の画像URLを試す
2. WordPress管理画面 → **メディア** → **新規追加** で手動アップロードを試す
3. `php.ini` で以下を確認:
   ```ini
   upload_max_filesize = 10M
   post_max_size = 10M
   ```
4. `.htaccess` で制限を増やす:
   ```apache
   php_value upload_max_filesize 10M
   php_value post_max_size 10M
   ```

---

## 🔬 デバッグ方法

### n8nのワークフロー実行履歴を確認

1. n8nの左メニューから **「Executions」** をクリック
2. 失敗した実行を選択
3. エラーが発生したノードを確認
4. エラーメッセージを読む

### 各ノードの出力を確認

1. ワークフローエディタで各ノードをクリック
2. 右側のパネルで **「Output」** タブを選択
3. データの構造を確認
4. エラーがある場合は **「Error」** タブを確認

### WordPress側のログを確認

1. WordPressのデバッグログを有効化
2. `/wp-content/debug.log` を確認
3. REST APIリクエストのエラーを確認

---

## 📝 テスト手順

### ステップ1: REST APIをテスト

**curlでテスト:**
```bash
curl -X GET "https://your-site.com/wp-json/wp/v2/posts" \
  -u "username:application-password"
```

**期待される結果:** 投稿のリストが返される

---

### ステップ2: 投稿作成をテスト

**curlでテスト:**
```bash
curl -X POST "https://your-site.com/wp-json/wp/v2/posts" \
  -u "username:application-password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "テスト投稿",
    "content": "これはテストです",
    "status": "draft"
  }'
```

**期待される結果:** 新しい投稿が作成される

---

### ステップ3: n8nでテスト実行

1. n8nワークフローエディタを開く
2. **「フォーム送信時」** ノードの **「Test URL」** をコピー
3. ブラウザで開く
4. シンプルなキーワードを入力（例: 「テスト」）
5. 送信して結果を確認

---

## 🎯 推奨される対策

### 対策1: 環境変数を使用

**ファイル:** `.env`（n8nのルートディレクトリ）
```bash
WORDPRESS_SITE_URL=https://your-site.com
TELEGRAM_CHAT_ID=your-chat-id
```

**メリット:**
- URLを一箇所で管理
- 環境ごとに異なる設定が可能
- セキュリティが向上

---

### 対策2: 設定用Setノードを追加

ワークフローの最初に「設定」ノードを追加し、すべての設定を一箇所で管理:

**ノード名:** `設定`
**タイプ:** Set
**フィールド:**
- `wordpressSiteUrl`: `https://your-site.com`
- `telegramChatId`: `your-chat-id`
- `authorId`: `2`

その後、各ノードで以下のように参照:
```
={{ $('設定').item.json.wordpressSiteUrl }}/wp-json/wp/v2/media
```

---

### 対策3: エラーハンドリングを追加

各重要なノードに「On Error」設定を追加:

1. ノードの **「Settings」** タブを開く
2. **「On Error」** で **「Continue」** を選択
3. エラー通知用のノードを追加（例: Telegramで通知）

---

## 📊 診断チェックシート

| チェック項目 | 状態 | メモ |
|------------|------|------|
| WordPress REST APIにアクセス可能 | ⬜ | |
| アプリケーションパスワードを生成済み | ⬜ | |
| n8nでWordPress認証情報を設定済み | ⬜ | |
| ワークフロー内のWordPress URLを更新済み | ⬜ | |
| 環境変数を設定済み（推奨） | ⬜ | |
| Telegram認証情報を設定済み（オプション） | ⬜ | |
| テスト実行が成功 | ⬜ | |

---

## 🆘 まだ解決しない場合

以下の情報を集めて、GitHubのIssuesで報告してください:

1. **n8nのバージョン**
2. **WordPressのバージョン**
3. **エラーメッセージの全文**
4. **n8n実行履歴のスクリーンショット**
5. **WordPressのdebug.logの関連部分**
6. **使用しているWordPressプラグイン一覧**

---

## 📚 参考リソース

- [WordPress REST API ハンドブック](https://developer.wordpress.org/rest-api/)
- [n8n公式ドキュメント](https://docs.n8n.io/)
- [n8n WordPressノード](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/)
- [Telegram Bot API](https://core.telegram.org/bots/api)

---

**最終更新:** 2025-01-18
