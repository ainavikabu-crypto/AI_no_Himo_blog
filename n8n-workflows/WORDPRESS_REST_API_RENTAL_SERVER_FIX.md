# レンタルサーバーでのWordPress REST API完全解決ガイド

## 🎯 このガイドの対象者

- レンタルサーバーでWordPressを運用している方
- パーマリンク設定、アプリケーションパスワード再生成を試しても解決しない方
- エックスサーバー、さくらインターネット、ロリポップ、ConoHa WINGなどを使用している方

---

## 📊 診断フローチャート

まず、現在の状況を診断しましょう。

### ステップ1: REST APIにアクセスできるか確認

ブラウザで以下のURLを開く：
```
https://あなたのサイト.com/wp-json/wp/v2/posts
```

**結果によって対処法が変わります：**

| 表示される内容 | 診断結果 | 次のステップ |
|--------------|---------|------------|
| JSON形式のデータ（投稿リスト） | ✅ REST APIは正常 | → **診断A** へ |
| `401 Unauthorized` | 🔑 認証エラー | → **診断B** へ |
| `403 Forbidden` | 🚫 アクセス拒否 | → **診断C** へ |
| `404 Not Found` | ❓ パーマリンク問題 | → **診断D** へ |
| `500 Internal Server Error` | ⚠️ サーバーエラー | → **診断E** へ |
| 何も表示されない（真っ白） | 🛡️ WAFブロック | → **診断F** へ |

---

## 診断A: REST APIは動くが、n8nで401エラー

### 原因
- n8nの認証情報が間違っている
- アプリケーションパスワードのコピーミス

### 解決方法

#### A-1. アプリケーションパスワードを完全に再生成

1. WordPress管理画面 → **ユーザー** → **プロフィール**
2. 既存のアプリケーションパスワードを**すべて削除**
3. 新しく生成：
   - 名前: `n8n-integration`
   - **新しいアプリケーションパスワードを追加**をクリック
4. 生成されたパスワードを**スペースも含めて**メモ帳にコピー

**例:**
```
abcd efgh ijkl mnop qrst uvwx
```

#### A-2. n8nで認証情報を設定し直す

1. n8nのWordPressノードを開く
2. 既存の認証情報を**削除**
3. 新しい認証情報を作成：

| フィールド | 入力内容 | ❌ 間違い例 | ✅ 正しい例 |
|-----------|---------|-----------|-----------|
| Site URL | WordPressのURL | `https://example.com/` | `https://example.com` |
| Username | WordPressのユーザー名 | `Admin`（大文字小文字注意） | `admin` |
| Password | アプリケーションパスワード | `abcdefghijklmnopqrstuvwx`（スペース削除） | `abcd efgh ijkl mnop qrst uvwx` |

**重要チェックポイント:**
- [ ] Site URLの末尾に `/` がない
- [ ] Site URLに `/wp-admin` や `/wp-json` が含まれていない
- [ ] Usernameが**完全に一致**（大文字小文字も）
- [ ] Passwordに**スペースが含まれている**

#### A-3. 接続テスト

n8nでテスト実行して、WordPressに投稿が作成されるか確認。

---

## 診断B: 401 Unauthorized（ブラウザでもエラー）

### 原因
- Basic認証がかかっている
- アプリケーションパスワード機能が無効

### 解決方法

#### B-1. レンタルサーバーのBasic認証を確認

**エックスサーバーの場合:**
1. サーバーパネルにログイン
2. **アクセス制限** → **Basic認証設定**
3. WordPressのディレクトリに認証がかかっていれば**一時的に解除**

**さくらインターネットの場合:**
1. サーバーコントロールパネルにログイン
2. **ファイルマネージャー**
3. `.htaccess` を確認（後述）

**ロリポップの場合:**
1. ユーザー専用ページにログイン
2. **セキュリティ** → **アクセス制限**
3. 設定を確認

#### B-2. .htaccessのBasic認証を解除

サーバーのファイルマネージャーまたはFTPで `.htaccess` を確認：

**削除または一時的にコメントアウト:**
```apache
# 以下の行をコメントアウト（行頭に # を追加）
# AuthType Basic
# AuthName "Private"
# AuthUserFile /path/to/.htpasswd
# Require valid-user
```

#### B-3. HTTPSを使用しているか確認

アプリケーションパスワードは**HTTPS必須**です。

**HTTPの場合の対処法（開発環境のみ）:**

`wp-config.php` に以下を追加（`/* That's all, stop editing! */` の前）：
```php
define('WP_ENVIRONMENT_TYPE', 'local');
```

**⚠️ 本番環境では必ずHTTPSを使用してください**

---

## 診断C: 403 Forbidden（最も多い問題）

### 原因
レンタルサーバーのWAF（Webアプリケーションファイアウォール）やセキュリティ機能がREST APIをブロックしています。

### 解決方法（レンタルサーバー別）

---

#### 🔷 エックスサーバー

**C-1. WAF設定を確認**

1. **サーバーパネル**にログイン
2. **セキュリティ** → **WAF設定**
3. 対象ドメインの**ONにする**をクリック（OFFにする）
   - ⚠️ WAFを完全にOFFにするのではなく、次の手順へ

**C-2. WAF除外ルールを追加（推奨）**

完全にOFFにするのはセキュリティリスクがあるため、REST APIだけ許可：

1. サーバーパネル → **WAF設定** → **ログ照会**
2. ブロックされたログを確認
3. REST APIのルールIDを確認
4. そのルールだけを**除外設定**

または、`.htaccess`に以下を追加：
```apache
# WordPress REST API を WAF 除外
<IfModule mod_siteguard.c>
    SiteGuard_User_ExcludeSig wp-json
</IfModule>
```

**C-3. mod_securityを確認**

エックスサーバーの場合、`.htaccess` に以下を追加：
```apache
# REST API のみ mod_security を無効化
<IfModule mod_security.c>
    <FilesMatch "wp-json">
        SecRuleEngine Off
    </FilesMatch>
</IfModule>
```

---

#### 🔷 さくらインターネット

**C-1. 国外IPアクセス制限を確認**

1. **サーバーコントロールパネル**にログイン
2. **セキュリティ設定** → **国外IPアドレスフィルタ**
3. **WordPress管理画面へのアクセス**が有効になっている場合
4. **REST APIも制限対象**になっている可能性

**対処法:**
- n8nが国内サーバーで動いている場合 → 問題なし
- n8nがクラウド（海外IP）の場合 → 制限を一時的に解除

**C-2. WAF（Webアプリケーションファイアウォール）**

1. サーバーコントロールパネル → **WAF設定**
2. 対象ドメインのWAFを**OFF**に設定（一時的）
3. REST APIが動作するか確認
4. 動作する場合、WAFのログを確認して除外設定を追加

**C-3. .htaccessに除外設定を追加**

```apache
# さくらインターネット WAF除外
<FilesMatch "wp-json">
    SecFilterEngine Off
    SecFilterScanPOST Off
</FilesMatch>
```

---

#### 🔷 ロリポップ

**C-1. WAF設定を確認**

1. **ユーザー専用ページ**にログイン
2. **セキュリティ** → **WAF設定**
3. 対象ドメインを**無効にする**

**重要:** ロリポップのWAFは**ドメイン全体**に適用されます。

**C-2. ログを確認**

1. WAF設定ページで**ログ参照**
2. ブロックされた`wp-json`のアクセスを確認
3. 該当するシグネチャを除外リストに追加

**C-3. .htaccessで除外（非推奨）**

ロリポップの場合、`.htaccess`での除外は効かないことが多いため、管理画面から設定してください。

---

#### 🔷 ConoHa WING

**C-1. WAF設定**

1. **コントロールパネル**にログイン
2. **サイト管理** → **サイトセキュリティ** → **WAF**
3. 対象ドメインのWAFを**OFF**

**C-2. WAFログを確認**

1. **WAF** → **ログ**
2. `/wp-json/` がブロックされているか確認
3. 除外ルールを設定

**C-3. .htaccessに除外設定**

```apache
# ConoHa WING WAF除外
<IfModule mod_siteguard.c>
    SiteGuard_User_ExcludeSig wp-json
</IfModule>
```

---

#### 🔷 カラフルボックス

**C-1. Imunify360（WAF）を確認**

1. cPanelにログイン
2. **Imunify360**
3. **Firewall** → **Whitelist**
4. REST APIのパス `/wp-json/` を追加

---

#### 🔷 mixhost

**C-1. LiteSpeed WAFを確認**

1. cPanelにログイン
2. **LiteSpeed Web Server**
3. **WAF** を確認
4. REST APIを除外リストに追加

---

### C-X. すべてのレンタルサーバー共通の対処法

#### .htaccessで直接REST APIを許可

WordPressのルートディレクトリの `.htaccess` に以下を追加：

```apache
# WordPress REST API を保護しつつ許可
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /

    # wp-json へのアクセスを許可
    RewriteCond %{REQUEST_URI} ^/wp-json/
    RewriteRule ^ - [L]
</IfModule>

# セキュリティ設定を一時的に緩和（REST APIのみ）
<FilesMatch "wp-json">
    # mod_security を無効化
    <IfModule mod_security.c>
        SecRuleEngine Off
    </IfModule>

    # SiteGuard を無効化
    <IfModule mod_siteguard.c>
        SiteGuard_User_ExcludeSig wp-json
    </IfModule>
</FilesMatch>
```

**⚠️ 重要な注意点:**
- `.htaccess` の編集前に**必ずバックアップ**を取る
- 構文エラーがあるとサイト全体が表示されなくなる可能性あり
- 編集後、すぐにサイトが表示されるか確認

---

## 診断D: 404 Not Found

### 原因
- パーマリンク設定の問題
- `.htaccess` が正しく生成されていない

### 解決方法

#### D-1. パーマリンク設定を再保存

1. WordPress管理画面 → **設定** → **パーマリンク**
2. **投稿名** を選択
3. **変更を保存**をクリック
4. もう一度**変更を保存**をクリック（2回保存が重要）

#### D-2. .htaccessが書き込み可能か確認

**FTPまたはファイルマネージャーで確認:**

1. WordPressのルートディレクトリの `.htaccess` を確認
2. パーミッションを`644`に設定
3. 所有者がWebサーバーのユーザー（`www-data`, `apache`, `nginx`など）であることを確認

**レンタルサーバーの場合:**
- エックスサーバー: ファイルマネージャーで権限変更可能
- さくらインターネット: 通常は自動設定
- ロリポップ: FTPで`644`に設定

#### D-3. .htaccessを手動で編集

パーマリンク保存しても404が出る場合、`.htaccess` に以下を追加：

```apache
# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress
```

**重要:** `RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]` の行が認証に必要です。

---

## 診断E: 500 Internal Server Error

### 原因
- PHPエラー
- メモリ不足
- プラグインの競合

### 解決方法

#### E-1. WordPressデバッグモードを有効化

`wp-config.php` に以下を追加（`/* That's all, stop editing! */` の前）：

```php
// デバッグモード有効化
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
@ini_set('display_errors', 0);
```

#### E-2. デバッグログを確認

FTPまたはファイルマネージャーで以下を確認：
```
/wp-content/debug.log
```

**よくあるエラーと対処法:**

| エラーメッセージ | 原因 | 対処法 |
|----------------|------|--------|
| `Allowed memory size exhausted` | メモリ不足 | `wp-config.php`に`define('WP_MEMORY_LIMIT', '256M');`を追加 |
| `Fatal error in /wp-content/plugins/...` | プラグインエラー | 該当プラグインを無効化 |
| `max_execution_time exceeded` | 実行時間超過 | PHPの`max_execution_time`を増やす |

#### E-3. メモリ制限を増やす

**wp-config.phpに追加:**
```php
define('WP_MEMORY_LIMIT', '256M');
define('WP_MAX_MEMORY_LIMIT', '512M');
```

**レンタルサーバーのPHP設定で増やす:**

- **エックスサーバー**: サーバーパネル → PHP設定 → memory_limit を 256M に設定
- **さくらインターネット**: コントロールパネル → PHPバージョン設定
- **ロリポップ**: ユーザー専用ページ → PHP設定

#### E-4. すべてのプラグインを無効化

1. FTPでWordPressディレクトリにアクセス
2. `/wp-content/plugins/` を `/wp-content/plugins-disabled/` にリネーム
3. REST APIにアクセスしてみる
4. 動作する場合、プラグインの競合が原因
5. 一つずつプラグインを有効化して原因を特定

---

## 診断F: 何も表示されない（真っ白）

### 原因
- レンタルサーバーのWAFが完全にブロック
- PHPの致命的エラー

### 解決方法

#### F-1. サーバーのエラーログを確認

**レンタルサーバーの管理画面でエラーログを確認:**

- **エックスサーバー**: サーバーパネル → エラーログ
- **さくらインターネット**: コントロールパネル → アクセスログ・エラーログ
- **ロリポップ**: ユーザー専用ページ → ログ → エラーログ

#### F-2. WAFを完全に無効化（テスト用）

一時的にWAFを無効化してテスト：

1. レンタルサーバーの管理画面でWAFを無効化
2. REST APIにアクセス
3. 動作する場合、WAFが原因
4. WAFを有効化して除外設定を追加（診断C参照）

#### F-3. PHPバージョンを確認

**推奨バージョン:** PHP 7.4以上（WordPress 6.0以降）

**レンタルサーバーでPHPバージョンを変更:**

- **エックスサーバー**: サーバーパネル → PHP Ver.切替
- **さくらインターネット**: コントロールパネル → PHPバージョン設定
- **ロリポップ**: ユーザー専用ページ → PHP設定

---

## 🔧 最終手段：すべて試しても解決しない場合

### 最終チェックリスト

すべて確認したかチェック：

- [ ] パーマリンク設定を「投稿名」に変更し、**2回保存した**
- [ ] アプリケーションパスワードを**完全に再生成した**
- [ ] n8nの認証情報を**完全に削除して再作成した**
- [ ] レンタルサーバーのWAFを**一時的に無効化してテストした**
- [ ] `.htaccess`のパーミッションを確認した（`644`）
- [ ] WordPressのデバッグログを確認した
- [ ] レンタルサーバーのエラーログを確認した
- [ ] すべてのプラグインを一時的に無効化してテストした
- [ ] PHPバージョンが7.4以上であることを確認した
- [ ] HTTPSを使用していることを確認した

### プラグインで強制的にREST APIを有効化

**最終手段として、プラグインをインストール:**

#### 方法1: REST API Enablerプラグイン

1. WordPress管理画面 → **プラグイン** → **新規追加**
2. `REST API Enabler` を検索
3. インストールして有効化

#### 方法2: カスタムコードで強制有効化

WordPressテーマの `functions.php` に以下を追加：

```php
// REST APIを強制的に有効化
add_filter('rest_authentication_errors', function($result) {
    if (!empty($result)) {
        return $result;
    }
    if (!is_user_logged_in()) {
        return true;
    }
    return $result;
});

// REST APIのCORS許可（n8n用）
add_action('rest_api_init', function() {
    remove_filter('rest_pre_serve_request', 'rest_send_cors_headers');
    add_filter('rest_pre_serve_request', function($value) {
        header('Access-Control-Allow-Origin: *');
        header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
        header('Access-Control-Allow-Credentials: true');
        header('Access-Control-Allow-Headers: Authorization, Content-Type');
        return $value;
    });
}, 15);
```

**⚠️ 注意:** これはセキュリティリスクがあるため、テスト環境でのみ使用してください。

### レンタルサーバーのサポートに問い合わせ

**問い合わせ時に伝える情報:**

```
件名: WordPress REST APIへのアクセスがブロックされている

本文:
WordPressサイトのREST API（/wp-json/wp/v2/posts）にアクセスすると
403 Forbiddenエラーが発生します。

WAFまたはセキュリティ設定でREST APIがブロックされていると思われますが、
REST APIを許可する方法を教えていただけますでしょうか。

サイトURL: https://あなたのサイト.com
エラー内容: 403 Forbidden
試したこと: WAF設定の確認、.htaccessの編集、パーマリンク再保存
```

---

## 📝 curlコマンドでのテスト方法

### Macの場合

```bash
# REST APIにアクセス
curl -X GET "https://あなたのサイト.com/wp-json/wp/v2/posts" \
  -u "ユーザー名:アプリケーションパスワード" \
  -v
```

`-v` オプションで詳細なログが表示されます。

### Windowsの場合（PowerShell）

```powershell
$user = "ユーザー名"
$pass = "アプリケーションパスワード"
$base64 = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("${user}:${pass}"))

Invoke-WebRequest -Uri "https://あなたのサイト.com/wp-json/wp/v2/posts" `
  -Headers @{ Authorization = "Basic $base64" } `
  -Verbose
```

### 期待される結果

```json
[
  {
    "id": 1,
    "date": "2025-01-18T00:00:00",
    "title": {
      "rendered": "投稿タイトル"
    },
    "content": {
      "rendered": "投稿内容..."
    },
    ...
  }
]
```

---

## 🎯 最も多い解決パターン（レンタルサーバー別）

### エックスサーバー
**原因:** WAF（SiteGuard）がREST APIをブロック
**解決:** サーバーパネル → WAF設定 → 一時的にOFF → 除外ルール追加

### さくらインターネット
**原因:** 国外IPアクセス制限 + WAF
**解決:** 国外IPアクセス制限を解除 または WAFを調整

### ロリポップ
**原因:** WAFがドメイン全体に厳しく適用
**解決:** WAF設定 → 対象ドメインを無効化 → ログから除外設定

### ConoHa WING
**原因:** WAF（SiteGuard）がブロック
**解決:** サイト管理 → サイトセキュリティ → WAF → OFF

### カラフルボックス / mixhost
**原因:** Imunify360 または LiteSpeed WAF
**解決:** cPanel → Imunify360/LiteSpeed → 除外設定

---

## 🆘 それでも解決しない場合の情報収集

以下の情報を集めてください：

### 1. WordPress情報
```bash
# WordPress管理画面 → ダッシュボード → サイトヘルスで確認
- WordPressバージョン
- PHPバージョン
- データベースバージョン
```

### 2. サーバー情報
- レンタルサーバー名（エックスサーバー、さくらなど）
- プラン名
- PHPバージョン
- メモリ制限

### 3. エラー情報
- ブラウザで `/wp-json/wp/v2/posts` にアクセスした結果（スクリーンショット）
- n8nのエラーメッセージ（全文）
- `/wp-content/debug.log` の内容
- レンタルサーバーのエラーログ

### 4. プラグイン情報
```bash
# WordPress管理画面 → プラグイン → インストール済みプラグイン
有効化されているプラグインのリスト
```

### 5. セキュリティ設定
- WAFの状態（ON/OFF）
- Basic認証の有無
- IP制限の有無
- SSL証明書の状態

---

## 📚 参考リンク

- [WordPress REST APIハンドブック](https://developer.wordpress.org/rest-api/)
- [n8n WordPress公式ドキュメント](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/)
- [エックスサーバー WAF設定](https://www.xserver.ne.jp/manual/man_server_waf.php)
- [さくらインターネット WAF設定](https://help.sakura.ad.jp/360000226461/)
- [ロリポップ WAF設定](https://lolipop.jp/manual/user/waf/)

---

**最終更新:** 2025-01-18
**対象:** レンタルサーバーユーザー向け
