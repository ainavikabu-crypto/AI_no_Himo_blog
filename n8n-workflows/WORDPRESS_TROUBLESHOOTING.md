# WordPress接続トラブルシューティングガイド

## タイムアウトエラーの原因と解決方法

n8nとWordPressの接続でタイムアウトが発生する場合、以下の原因が考えられます。

## 1. WordPress REST APIの問題

### 原因
- REST APIが無効化されている
- セキュリティプラグインによるブロック
- .htaccessでのブロック

### 確認方法

**ステップ1: REST APIが有効か確認**

ブラウザで以下のURLにアクセス：
```
https://your-wordpress-site.com/wp-json/wp/v2/posts
```

✅ **正常な場合**: JSON形式でデータが返ってくる
❌ **問題がある場合**: 404エラー、アクセス拒否、または何も表示されない

**ステップ2: REST APIエンドポイント一覧を確認**

```
https://your-wordpress-site.com/wp-json/
```

これでWordPress REST APIのルート情報が表示されれば、APIは有効です。

### 解決方法

#### A. セキュリティプラグインの設定を確認

**Wordfenceの場合:**
1. WordPress管理画面 → Wordfence → Firewall
2. 「Manage Rate Limiting」を確認
3. REST APIへのアクセスが制限されていないか確認

**iThemes Securityの場合:**
1. WordPress管理画面 → Security → Settings
2. 「REST API」セクションを確認
3. n8nのIPアドレスをホワイトリストに追加

#### B. .htaccessの確認

WordPress root の `.htaccess` ファイルを確認し、以下のような記述がないかチェック：

```apache
# 問題のある設定例
<Files "wp-json">
  Order allow,deny
  Deny from all
</Files>
```

上記のような記述がある場合は削除またはコメントアウト。

#### C. functions.phpでの無効化を確認

テーマの `functions.php` で以下のようなコードがないかチェック：

```php
// 問題のあるコード例
add_filter('rest_authentication_errors', function($result) {
    if (!is_user_logged_in()) {
        return new WP_Error('rest_disabled', 'REST API disabled', array('status' => 401));
    }
    return $result;
});
```

---

## 2. アプリケーションパスワードの問題

### 原因
- アプリケーションパスワードが正しく設定されていない
- スペースやハイフンが含まれている
- HTTPSが必要だがHTTPで接続している
- WordPressのバージョンが古い（5.6未満）

### 確認方法

**ステップ1: WordPressのバージョンを確認**

アプリケーションパスワードはWordPress 5.6以降で利用可能です。

WordPress管理画面 → ダッシュボード → 更新 でバージョンを確認

**ステップ2: アプリケーションパスワードが利用可能か確認**

WordPress管理画面 → ユーザー → プロフィール

ページ下部に「アプリケーションパスワード」セクションがあるか確認。

❌ **表示されない場合の原因:**
- HTTPSではなくHTTPでアクセスしている（本番環境の場合）
- WordPress 5.6未満
- プラグインやテーマで機能が無効化されている

### 解決方法

#### A. アプリケーションパスワードを再生成

1. WordPress管理画面 → ユーザー → プロフィール
2. 「アプリケーションパスワード」セクションまでスクロール
3. 既存のn8n用パスワードがあれば削除
4. 新しいアプリケーション名（例：`n8n-blog`）を入力
5. 「新しいアプリケーションパスワードを追加」をクリック
6. 生成されたパスワードを**スペースなし**でコピー

例：
```
# 生成されたパスワード（スペース入り）
abcd efgh ijkl mnop qrst uvwx

# n8nに入力する際はスペースを削除
abcdefghijklmnopqrstuvwx
```

#### B. n8n認証情報の再設定

1. n8n → Credentials → WordPress
2. 以下を正確に入力：
   - **Site URL**: `https://your-site.com` （末尾のスラッシュなし）
   - **Username**: WordPressログインユーザー名（メールアドレスではない）
   - **Password**: アプリケーションパスワード（スペースなし）
3. 「Test Credential」をクリックして接続テスト
4. 「Save」をクリック

#### C. HTTPSの確認

本番環境の場合、WordPressサイトがHTTPSで動作している必要があります。

```bash
# n8nで設定するURL
❌ http://your-site.com
✅ https://your-site.com
```

---

## 3. WordPress URLの設定ミス

### 原因
- 末尾のスラッシュの有無
- wwwの有無
- http vs https
- サブディレクトリの間違い

### 確認方法

WordPress管理画面 → 設定 → 一般

「WordPressアドレス (URL)」と「サイトアドレス (URL)」を確認

### 解決方法

n8nの認証情報で、WordPressの設定と完全に一致するURLを使用：

```bash
# WordPress設定が以下の場合
https://www.example.com

# n8nでも同じURLを使用（末尾スラッシュなし）
https://www.example.com

# サブディレクトリにインストールしている場合
https://example.com/blog
```

---

## 4. ファイアウォール・セキュリティ設定

### 原因
- サーバーのファイアウォールでn8nのIPがブロックされている
- Cloudflareなどのプロキシでブロックされている
- レート制限に引っかかっている

### 解決方法

#### A. n8nのIPアドレスをホワイトリスト化

**n8nのIPアドレスを確認：**

n8nが動作しているサーバーで：
```bash
curl ifconfig.me
```

**WordPressサーバーでホワイトリスト化：**

cPanel、Plesk、またはサーバー管理画面で、n8nのIPアドレスをホワイトリストに追加。

#### B. Cloudflareを使用している場合

1. Cloudflare → Security → WAF
2. n8nのIPアドレスをIPアクセスルールで許可
3. または、API Shieldで特定のエンドポイントを保護解除

#### C. WordPressプラグインでのIP制限を確認

**Wordfenceの場合：**
1. Wordfence → Firewall → All Firewall Options
2. 「Whitelisted IPs」にn8nのIPを追加

**Limit Login Attemptsなどの場合：**
1. プラグイン設定でn8nのIPをホワイトリストに追加

---

## 5. タイムアウト設定

### 原因
- WordPressサーバーのタイムアウト設定が短すぎる
- n8nのタイムアウト設定が短すぎる
- PHPの実行時間制限

### 解決方法

#### A. WordPressサーバーの設定

**php.ini または .user.ini を編集：**

```ini
max_execution_time = 300
max_input_time = 300
```

**WordPress wp-config.php に追加：**

```php
define('WP_HTTP_TIMEOUT', 60);
```

#### B. n8nノードのタイムアウト設定

1. WordPress投稿ノードを開く
2. 「Settings」タブ
3. 「Timeout」を60000（60秒）以上に設定

---

## 6. WordPress XMLRPCの確認

### 原因
一部のホスティング環境では、セキュリティのためにXMLRPCが無効化されており、それがREST APIにも影響することがあります。

### 確認方法

```bash
curl -X POST https://your-site.com/xmlrpc.php
```

### 解決方法

XMLRPCを無効化している場合は、REST APIは別のものなので、セキュリティプラグインの設定でREST APIのみを有効化。

---

## トラブルシューティング手順

以下の順番で確認してください：

### ステップ1: 基本的な接続確認

```bash
# REST APIが有効か確認
curl https://your-wordpress-site.com/wp-json/

# 投稿エンドポイントが有効か確認
curl https://your-wordpress-site.com/wp-json/wp/v2/posts
```

### ステップ2: 認証情報の確認

```bash
# 認証付きでテスト（アプリケーションパスワードを使用）
curl -X POST \
  https://your-wordpress-site.com/wp-json/wp/v2/posts \
  -u "username:application_password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Test Post",
    "content": "Test content",
    "status": "draft"
  }'
```

✅ **成功した場合**: JSON形式で投稿データが返ってくる
❌ **失敗した場合**: エラーメッセージを確認

### ステップ3: n8nでのテスト

1. WordPress投稿ノードを開く
2. 「Execute Node」をクリック
3. エラーメッセージを確認

### ステップ4: n8n実行ログの確認

n8n → Executions → 失敗した実行を選択

エラーメッセージから原因を特定：

| エラーメッセージ | 原因 | 解決方法 |
|----------------|------|---------|
| `Timeout` | タイムアウト | タイムアウト設定を延長 |
| `401 Unauthorized` | 認証エラー | アプリケーションパスワードを確認 |
| `403 Forbidden` | アクセス拒否 | ファイアウォール、セキュリティプラグインを確認 |
| `404 Not Found` | URL間違い | WordPress URLを確認 |
| `Connection refused` | サーバー到達不可 | ネットワーク、ファイアウォールを確認 |
| `SSL error` | SSL証明書の問題 | SSL証明書を確認 |

---

## よくある質問

### Q1: ローカル環境（localhost）でテストできますか？

A: はい、できます。ただし以下の点に注意：

- アプリケーションパスワードは本番環境（HTTPS）では必須ですが、ローカル（HTTP）では以下の設定を`wp-config.php`に追加することで使用可能：

```php
define('WP_ENVIRONMENT_TYPE', 'local');
```

- または、Basic認証を使用

### Q2: n8nクラウド版を使用していますが、接続できません

A: n8nクラウドのIPアドレスをWordPressサーバーのファイアウォールでホワイトリスト化する必要がある場合があります。n8nのドキュメントでクラウド版のIPアドレス範囲を確認してください。

### Q3: 「rest_cookie_invalid_nonce」エラーが出ます

A: これはCookie認証の問題です。n8nではアプリケーションパスワード認証を使用するため、このエラーは通常発生しません。WordPress認証情報の設定を確認してください。

---

## チェックリスト

接続できない場合、以下をすべて確認：

- [ ] WordPressバージョンが5.6以上
- [ ] HTTPSでアクセスしている（本番環境の場合）
- [ ] アプリケーションパスワードが生成されている
- [ ] アプリケーションパスワードをスペースなしでコピーした
- [ ] WordPress URLが正確（末尾スラッシュなし）
- [ ] REST APIが有効（`/wp-json/` にアクセスできる）
- [ ] セキュリティプラグインでREST APIがブロックされていない
- [ ] ファイアウォールでn8nのIPがブロックされていない
- [ ] n8n認証情報でユーザー名（メールアドレスではない）を使用
- [ ] タイムアウト設定が十分（60秒以上）

---

## それでも解決しない場合

以下の情報を収集してサポートに問い合わせ：

1. WordPressバージョン
2. 使用しているセキュリティプラグイン
3. ホスティング環境（共用サーバー、VPS、クラウドなど）
4. n8nのエラーメッセージ全文
5. `curl`コマンドでのテスト結果

```bash
# 以下のコマンド結果を共有
curl -I https://your-wordpress-site.com/wp-json/
curl https://your-wordpress-site.com/wp-json/ | jq
```

---

**参考リンク：**
- [WordPress REST API Handbook](https://developer.wordpress.org/rest-api/)
- [Application Passwords Documentation](https://make.wordpress.org/core/2020/11/05/application-passwords-integration-guide/)
- [n8n WordPress Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/)
