# WordPress認証テスト - ステップバイステップガイド

このガイドでは、curlコマンドを使ってWordPressのアプリケーションパスワード認証が正しく機能しているかテストします。

## 事前準備

### ステップ1: WordPressユーザー名を確認

1. WordPress管理画面にログイン
2. 右上のアカウント名をクリック → **「プロフィールを編集」**
3. **「ユーザー名」**の欄を確認（メールアドレスではありません）

例：
- ✅ ユーザー名：`admin` または `taro_yamada`
- ❌ メールアドレス：`admin@ainohimo.com`（これは使いません）

### ステップ2: アプリケーションパスワードを生成

1. 同じプロフィールページで下にスクロール
2. **「アプリケーションパスワード」**セクションを探す
3. 新しいアプリケーション名を入力（例：`n8n-test`）
4. **「新しいアプリケーションパスワードを追加」**ボタンをクリック
5. 生成されたパスワードが表示されます：

```
例：abcd efgh ijkl mnop qrst uvwx
```

6. このパスワードを**メモ帳などにコピー**してください
   - ⚠️ このパスワードは一度しか表示されません
   - ページを閉じると再表示できないので、必ずコピーしてください

---

## curlコマンドでのテスト

### 方法1: ターミナル/コマンドプロンプトで実行（推奨）

**Windowsの場合：**
1. `コマンドプロンプト`または`PowerShell`を開く
2. 以下のコマンドを実行

**Mac/Linuxの場合：**
1. `ターミナル`を開く
2. 以下のコマンドを実行

---

### テストコマンド（コピー&ペースト用）

以下のコマンドをコピーして、**3箇所を自分の情報に置き換えてから**実行してください：

```bash
curl -X POST https://ainohimo.com/wp-json/wp/v2/posts \
  -u "ユーザー名:アプリケーションパスワード" \
  -H "Content-Type: application/json" \
  -d '{"title":"テスト投稿","content":"これはテストです","status":"draft"}'
```

**置き換える箇所：**

1. `ユーザー名` → ステップ1で確認したユーザー名
2. `アプリケーションパスワード` → ステップ2で生成したパスワード（**スペースを全て削除**）

---

### 具体例

**生成されたアプリケーションパスワード：**
```
abcd efgh ijkl mnop qrst uvwx
```

**スペースを削除：**
```
abcdefghijklmnopqrstuvwx
```

**ユーザー名が`admin`の場合の完成したコマンド：**

```bash
curl -X POST https://ainohimo.com/wp-json/wp/v2/posts \
  -u "admin:abcdefghijklmnopqrstuvwx" \
  -H "Content-Type: application/json" \
  -d '{"title":"テスト投稿","content":"これはテストです","status":"draft"}'
```

---

### Windows PowerShellの場合の注意

PowerShellの場合、シングルクォート`'`とダブルクォート`"`の扱いが異なるため、以下のように書き換えてください：

```powershell
curl.exe -X POST https://ainohimo.com/wp-json/wp/v2/posts `
  -u "admin:abcdefghijklmnopqrstuvwx" `
  -H "Content-Type: application/json" `
  -d '{\"title\":\"テスト投稿\",\"content\":\"これはテストです\",\"status\":\"draft\"}'
```

または、1行で：

```powershell
curl.exe -X POST https://ainohimo.com/wp-json/wp/v2/posts -u "admin:abcdefghijklmnopqrstuvwx" -H "Content-Type: application/json" -d "{\"title\":\"テスト投稿\",\"content\":\"これはテストです\",\"status\":\"draft\"}"
```

---

## 結果の見方

### ✅ 成功した場合

以下のようなJSON形式のレスポンスが返ってきます：

```json
{
  "id": 2,
  "date": "2025-11-18T10:30:00",
  "date_gmt": "2025-11-18T10:30:00",
  "guid": {
    "rendered": "https://ainohimo.com/?p=2"
  },
  "modified": "2025-11-18T10:30:00",
  "modified_gmt": "2025-11-18T10:30:00",
  "slug": "",
  "status": "draft",
  "type": "post",
  "link": "https://ainohimo.com/?p=2",
  "title": {
    "rendered": "テスト投稿"
  },
  "content": {
    "rendered": "<p>これはテストです</p>\n",
    "protected": false
  },
  ...
}
```

この場合、**認証は成功しています！** 🎉

WordPress管理画面 → 投稿 → 下書き を確認すると、「テスト投稿」という記事が作成されているはずです。

---

### ❌ 失敗した場合

#### エラー1: `401 Unauthorized`

```json
{
  "code": "rest_forbidden",
  "message": "申し訳ありません。あなたはこの投稿を作成できません。",
  "data": {
    "status": 401
  }
}
```

**原因：**
- アプリケーションパスワードが間違っている
- ユーザー名が間違っている
- スペースが残っている

**解決方法：**
1. アプリケーションパスワードを再生成
2. スペースを完全に削除したか確認
3. ユーザー名（メールアドレスではない）を再確認

---

#### エラー2: `403 Forbidden`

```json
{
  "code": "rest_cannot_create",
  "message": "申し訳ありません。あなたには投稿を作成する権限がありません。",
  "data": {
    "status": 403
  }
}
```

**原因：**
- ユーザーに投稿権限がない
- セキュリティプラグインでブロックされている

**解決方法：**
1. WordPress管理画面 → ユーザー で、ユーザーが「管理者」または「編集者」権限を持っているか確認
2. セキュリティプラグイン（Wordfence等）を一時的に無効化してテスト

---

#### エラー3: タイムアウト（応答なし）

コマンド実行後、何も返ってこず、長時間待たされる。

**原因：**
- サーバーのファイアウォールでブロックされている
- WordPressサイトが落ちている
- タイムアウト設定が短すぎる

**解決方法：**
1. ブラウザで https://ainohimo.com にアクセスできるか確認
2. サーバーのファイアウォール設定を確認
3. curlコマンドにタイムアウト設定を追加：

```bash
curl -X POST https://ainohimo.com/wp-json/wp/v2/posts \
  --max-time 60 \
  -u "admin:abcdefghijklmnopqrstuvwx" \
  -H "Content-Type: application/json" \
  -d '{"title":"テスト投稿","content":"これはテストです","status":"draft"}'
```

---

#### エラー4: SSL証明書エラー

```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

**原因：**
SSL証明書の検証に失敗

**解決方法（テスト目的のみ）：**
```bash
curl -k -X POST https://ainohimo.com/wp-json/wp/v2/posts \
  -u "admin:abcdefghijklmnopqrstuvwx" \
  -H "Content-Type: application/json" \
  -d '{"title":"テスト投稿","content":"これはテストです","status":"draft"}'
```

（`-k`オプションは証明書検証をスキップします）

⚠️ 本番環境では、SSL証明書を正しく設定してください。

---

## トラブルシューティングチェックリスト

curlコマンドを実行する前に、以下を確認してください：

- [ ] ユーザー名は正しいか（メールアドレスではない）
- [ ] アプリケーションパスワードのスペースを全て削除したか
- [ ] WordPressサイトURL（https://ainohimo.com）は正しいか
- [ ] インターネット接続は正常か
- [ ] ブラウザでWordPressサイトにアクセスできるか

---

## 次のステップ

### curlコマンドで成功した場合

n8nの認証情報設定に問題があります。以下を確認：

1. n8n → Credentials → WordPress
2. 以下を正確に入力：
   ```
   Site URL: https://ainohimo.com
   Username: [curlで成功したユーザー名]
   Password: [curlで成功したアプリケーションパスワード（スペースなし）]
   ```
3. 「Test Credential」をクリック
4. 成功したら「Save」

### curlコマンドでも失敗した場合

WordPress側の設定に問題があります：

1. WordPressのバージョンが5.6以上か確認
2. セキュリティプラグインを一時的に無効化してテスト
3. 別のユーザーでアプリケーションパスワードを生成してテスト
4. WordPressホスティング会社に連絡（ファイアウォール設定の確認）

---

## よくある質問

### Q: アプリケーションパスワードが表示されません

A: 以下の条件が必要です：
- WordPress 5.6以上
- HTTPS接続（本番環境の場合）
- ローカル環境の場合、`wp-config.php`に以下を追加：
  ```php
  define('WP_ENVIRONMENT_TYPE', 'local');
  ```

### Q: コマンドプロンプトが見つかりません（Windows）

A:
1. Windowsキー + R を押す
2. `cmd` と入力してEnter
3. コマンドプロンプトが開きます

### Q: curlコマンドが見つかりません（Windows）

A: Windows 10/11の場合、curlは標準搭載されています。
PowerShellで `curl.exe` と入力してみてください。

古いバージョンの場合、Git Bashをインストールするか、オンラインツールを使用：
- https://reqbin.com/ （オンラインでcurlコマンドを実行できる）

---

## 実行例のスクリーンショット

### 成功した場合のイメージ

```
$ curl -X POST https://ainohimo.com/wp-json/wp/v2/posts \
  -u "admin:abcdefghijklmnopqrstuvwx" \
  -H "Content-Type: application/json" \
  -d '{"title":"テスト投稿","content":"これはテストです","status":"draft"}'

{"id":2,"date":"2025-11-18T10:30:00",...}
```

大量のJSONが返ってきたら成功です！

---

このガイドに従ってテストを実行し、結果を教えてください。エラーメッセージが表示された場合は、そのメッセージ全体をコピーして共有してください。
