# WordPressタイムアウトエラーの解決方法

## 🚨 問題

n8nのWordPressノードでタイムアウトエラーが発生し、WordPress投稿が作成されない。

## 📋 解決手順（この順番で実行してください）

---

### ✅ 手順1: WordPress REST APIの確認

**1-1. ブラウザでREST APIにアクセス**

以下のURLをブラウザで開いてください（`あなたのサイト.com`を実際のサイトURLに置き換え）：

```
https://あなたのサイト.com/wp-json/wp/v2/posts
```

**期待される結果：**
```json
[
  {
    "id": 1,
    "date": "2025-01-18T00:00:00",
    "title": {"rendered": "..."},
    ...
  }
]
```

**❌ エラーが表示される場合：**

| エラーメッセージ | 原因 | 対策 |
|----------------|------|------|
| `REST API is disabled` | REST APIが無効化されている | 手順1-2へ |
| `404 Not Found` | パーマリンク設定の問題 | 手順1-3へ |
| `403 Forbidden` | セキュリティプラグインがブロック | 手順1-4へ |
| ページが読み込まれない | サイトURLが間違っている、またはサイトがダウン | WordPress管理画面にアクセスできるか確認 |

---

**1-2. REST APIを有効化**

WordPress管理画面にログインして確認：

1. **プラグイン** → **インストール済みプラグイン** を開く
2. 以下のプラグインが有効になっている場合は無効化：
   - `Disable REST API`
   - `Disable WP REST API`
   - `WP REST API Controller`

3. `.htaccess`ファイルを確認（サーバーのルートディレクトリ）：
   ```apache
   # 以下の行があれば削除
   # <Files wp-json>
   #   Order Deny,Allow
   #   Deny from all
   # </Files>
   ```

---

**1-3. パーマリンク設定を修正**

WordPress管理画面で：

1. **設定** → **パーマリンク** を開く
2. **投稿名** を選択（推奨）
   ```
   /%postname%/
   ```
3. **変更を保存** をクリック

**重要：** これだけでも多くの問題が解決します！

---

**1-4. セキュリティプラグインの設定**

セキュリティプラグインがインストールされている場合：

**Wordfenceの場合：**
1. **Wordfence** → **Firewall** → **All Firewall Options**
2. **Rate Limiting** で `REST API` を探す
3. `Allow REST API` を有効化

**iThemes Securityの場合：**
1. **Security** → **Settings** → **REST API**
2. **Restrict Access to REST API** を無効化

**SiteGuard WP Pluginの場合（日本で人気）：**
1. **SiteGuard** → **REST API アクセス制限**
2. **OFF** に設定

---

### ✅ 手順2: WordPress認証情報の設定

**2-1. アプリケーションパスワードを生成**

1. WordPress管理画面にログイン
2. **ユーザー** → **プロフィール** を開く
3. 下にスクロールして **「アプリケーションパスワード」** セクションを探す

**❌ 「アプリケーションパスワード」が表示されない場合：**

以下の条件を確認：
- WordPressバージョン5.6以降であること
- HTTPSを使用していること（必須）
- ユーザーが管理者権限を持っていること

**HTTPSでない場合の対処法：**
`wp-config.php`に以下を追加（開発環境のみ、本番環境では非推奨）：
```php
define('WP_ENVIRONMENT_TYPE', 'local');
```

4. 新しいアプリケーション名を入力（例：`n8n-integration`）
5. **「新しいアプリケーションパスワードを追加」** をクリック
6. 生成されたパスワードをコピー（**スペースを含めてそのまま**コピー）

**例：**
```
abcd efgh ijkl mnop qrst uvwx
```

---

**2-2. n8nで認証情報を設定**

1. n8nワークフローエディタを開く
2. **「Create a post」（WordPress投稿）** ノードをクリック
3. **Credential for Wordpress** の隣の鉛筆アイコンをクリック
4. 以下を入力：

| フィールド | 入力内容 | 例 |
|-----------|---------|-----|
| **Site URL** | WordPressサイトのURL（**末尾のスラッシュなし**） | `https://example.com` |
| **Username** | WordPressのユーザー名 | `admin` |
| **Password** | コピーしたアプリケーションパスワード | `abcd efgh ijkl mnop qrst uvwx` |

**⚠️ 重要な注意点：**
- Site URLに `/wp-admin` や `/wp-json` は含めない
- 末尾のスラッシュ `/` は削除
- `http://` ではなく `https://` を使用（HTTPSが必須）

5. **保存** をクリック

---

### ✅ 手順3: 接続テスト

**3-1. テストワークフローを実行**

1. `wordpress-connection-test.json` をn8nにインポート
2. **「REST API接続テスト」** ノードを開く
3. URLを **あなたのWordPressサイトURL** に変更：
   ```
   https://あなたのサイト.com/wp-json/wp/v2/posts
   ```
4. **認証情報** を選択（手順2で作成したもの）
5. **「Execute Node」** をクリック

**✅ 成功した場合：**
- 投稿のリストが表示される
- → WordPress接続が正常に機能しています！

**❌ エラーが表示される場合：**

| エラーコード | 原因 | 対策 |
|------------|------|------|
| `401 Unauthorized` | 認証情報が間違っている | 手順2を再度実行 |
| `403 Forbidden` | アクセス権限がない | セキュリティプラグインを確認（手順1-4） |
| `404 Not Found` | URLが間違っている | Site URLとパーマリンクを確認 |
| `Timeout` | サイトが応答しない | サーバーの負荷、ファイアウォール、プラグインの競合を確認 |

---

**3-2. 実際の投稿作成をテスト**

接続テストが成功したら、元のワークフローを実行：

1. **「When clicking 'Execute workflow'」** をクリック
2. **「Execute Workflow」** をクリック
3. **「Create a post」** ノードを確認

**✅ 成功した場合：**
- WordPressの **投稿 → 下書き** に新しい投稿が作成される
- ノードに緑のチェックマークが表示される

**❌ まだエラーが出る場合：**
- 手順4のデバッグ方法へ

---

### ✅ 手順4: 詳細デバッグ

**4-1. WordPressのデバッグログを有効化**

`wp-config.php` に以下を追加（`/* That's all, stop editing! */` の前）：

```php
// デバッグモードを有効化
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
```

**4-2. ログを確認**

WordPressのデバッグログを確認：
```
/wp-content/debug.log
```

**4-3. n8nのエラーログを確認**

1. n8nの **Executions** を開く
2. 失敗した実行をクリック
3. エラーメッセージをコピー

**4-4. よくあるエラーと対策**

| エラーメッセージ | 原因 | 対策 |
|----------------|------|------|
| `ECONNREFUSED` | WordPressサーバーが応答しない | サーバーが起動しているか確認 |
| `ETIMEDOUT` | タイムアウト | ファイアウォール、プラグインの競合を確認 |
| `SSL Error` | SSL証明書の問題 | HTTPSが正しく設定されているか確認 |
| `429 Too Many Requests` | レート制限 | セキュリティプラグインのレート制限を緩和 |

---

## 🎯 よくある原因トップ5

実際のトラブルシューティングで最も多い原因：

1. **パーマリンク設定が「基本」になっている** → 手順1-3で修正
2. **HTTPSを使用していない** → アプリケーションパスワードにはHTTPS必須
3. **セキュリティプラグインがREST APIをブロック** → 手順1-4で設定変更
4. **Site URLに末尾のスラッシュがある** → `https://example.com/` ではなく `https://example.com`
5. **アプリケーションパスワードのコピーミス** → スペースを含めてコピー

---

## 🧪 curlでテストする方法（上級者向け）

ターミナルで以下のコマンドを実行（Macの場合）：

```bash
curl -X GET "https://あなたのサイト.com/wp-json/wp/v2/posts" \
  -u "ユーザー名:アプリケーションパスワード"
```

**Windows（PowerShell）の場合：**
```powershell
curl -Method GET `
  -Uri "https://あなたのサイト.com/wp-json/wp/v2/posts" `
  -Headers @{ Authorization = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("ユーザー名:アプリケーションパスワード")) }
```

**期待される結果：** 投稿のリストがJSON形式で返される

---

## 📝 チェックリスト

すべて確認したかチェックしてください：

| チェック項目 | 完了 |
|------------|------|
| ✅ ブラウザで `/wp-json/wp/v2/posts` にアクセスできる | ⬜ |
| ✅ パーマリンク設定を「投稿名」に変更した | ⬜ |
| ✅ セキュリティプラグインでREST APIを許可した | ⬜ |
| ✅ HTTPSを使用している | ⬜ |
| ✅ アプリケーションパスワードを生成した | ⬜ |
| ✅ n8nで認証情報を正しく設定した（スペース含む） | ⬜ |
| ✅ Site URLの末尾にスラッシュがない | ⬜ |
| ✅ 接続テストワークフローが成功した | ⬜ |

---

## 🆘 まだ解決しない場合

以下の情報を集めて質問してください：

1. **WordPressのバージョン** → WordPress管理画面の右下に表示
2. **n8nのバージョン** → n8n画面の左下に表示
3. **エラーメッセージの全文** → n8nのExecutionsから
4. **使用しているプラグイン一覧** → WordPress → プラグイン
5. **ホスティング環境** → レンタルサーバー、VPS、ローカル環境など
6. **`/wp-json/wp/v2/posts` にアクセスした結果** → スクリーンショット

---

## 📚 参考リンク

- [WordPress REST API ハンドブック](https://developer.wordpress.org/rest-api/)
- [n8n WordPress ノード公式ドキュメント](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/)
- [アプリケーションパスワードの使用方法](https://make.wordpress.org/core/2020/11/05/application-passwords-integration-guide/)

---

**最終更新:** 2025-01-18
**作成者:** アイヒモ
