# エックスサーバー + n8n Cloud：ETIMEDOUTエラー完全解決ガイド

## 🚨 この問題の症状

- ブラウザで `/wp-json/wp/v2/posts` にアクセスすると**JSON形式のデータが表示される**（正常）
- n8n Cloudで実行すると `ETIMEDOUT` エラーが発生
- エラーメッセージ：`connect ETIMEDOUT 85.131.213.82:443`
- REST API制限をOFFにしても解決しない
- WAFもOFFにしても解決しない

## 🔍 根本原因

**エックスサーバーのサーバーレベルのファイアウォールが、n8n Cloud（海外サーバー）からのアクセスをブロックしています。**

エックスサーバーには以下の複数のセキュリティ層があります：

1. **WordPress セキュリティ設定**（国外IP制限）← 設定でOFF可能
2. **WAF**（Webアプリケーションファイアウォール）← 設定でOFF可能
3. **サーバーレベルのファイアウォール** ← **ここが原因**

Cloudronフォーラムの同様の事例では、「ホスティングプロバイダーがn8nサーバーのIPをホワイトリストに追加する」ことで解決しました。

参考：https://forum.cloudron.io/topic/12461/n8n-times-out-for-wordpress-api

---

## ✅ 解決方法（3つの選択肢）

### 🎯 方法1: .htaccessで強制的にREST APIを許可（最速）

エックスサーバーのすべてのセキュリティ層を回避する設定を`.htaccess`に追加します。

#### 手順：

**1. .htaccessのバックアップを取る**

エックスサーバーのファイルマネージャーまたはFTPで、WordPressのルートディレクトリの `.htaccess` をダウンロードしてバックアップを保存。

**2. .htaccessを編集**

`.htaccess` の**一番上**（`# BEGIN WordPress` の前）に以下を追加：

```apache
# ========================================
# n8n Cloud用：REST APIアクセスを完全に許可
# エックスサーバーの全セキュリティ層をバイパス
# ========================================

# mod_securityを無効化（REST APIのみ）
<IfModule mod_security.c>
    <If "%{REQUEST_URI} =~ m#^/wp-json/#">
        SecRuleEngine Off
    </If>
</IfModule>

# SiteGuard（エックスサーバーのWAF）を無効化（REST APIのみ）
<IfModule mod_siteguard.c>
    SiteGuard_User_ExcludeSig wp-json
</IfModule>

# 認証ヘッダーを通過させる（重要！）
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteCond %{HTTP:Authorization} ^(.+)$
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%1]
</IfModule>

# REST APIへのアクセスを完全に許可
<IfModule mod_rewrite.c>
    RewriteCond %{REQUEST_URI} ^/wp-json/ [NC]
    RewriteRule ^ - [L,E=noconntimeout:1]
</IfModule>

# REST APIへのアクセス制御を上書き
<FilesMatch "^(wp-json)">
    <IfModule mod_authz_core.c>
        Require all granted
    </IfModule>
    <IfModule !mod_authz_core.c>
        Order allow,deny
        Allow from all
    </IfModule>
</FilesMatch>

# ========================================
```

**3. 保存してテスト**

- `.htaccess` を保存
- **すぐにブラウザでサイトが表示されるか確認**（構文エラーチェック）
- サイトが表示されたら、n8nでテスト実行

**✅ 成功した場合：**
- `ETIMEDOUT` エラーが消える
- WordPressに投稿が作成される

**❌ サイトが表示されなくなった場合：**
- バックアップした`.htaccess`を元に戻す
- 方法2または方法3へ

---

### 🎯 方法2: エックスサーバーサポートに問い合わせ（推奨・根本解決）

エックスサーバーのサポートに、n8n CloudのIPアドレスをホワイトリストに追加してもらうよう依頼します。

#### 問い合わせテンプレート：

```
件名: WordPress REST APIへの海外サーバーからのアクセス許可について

本文:
お世話になっております。

WordPressサイトでREST APIを使用し、外部の自動化サービス（n8n Cloud）から
投稿を自動作成する仕組みを構築しております。

以下の設定をすべて確認しましたが、タイムアウトエラー（ETIMEDOUT）が発生します：

【確認済みの設定】
✅ WordPress セキュリティ設定 → 国外IPアクセス制限 → REST API制限：OFF
✅ WAF設定：OFF
✅ パーマリンク設定：投稿名
✅ アプリケーションパスワード：設定済み

【現在の状況】
- ブラウザから直接 /wp-json/wp/v2/posts にアクセス：正常（JSON表示）
- n8n Cloud（海外サーバー）からアクセス：タイムアウト

【エラー詳細】
エラーコード: ETIMEDOUT
エラーメッセージ: connect ETIMEDOUT 85.131.213.82:443

サーバーレベルのファイアウォールまたはIP制限が原因と思われます。

つきましては、以下の対応が可能かご教示いただけますでしょうか：

1. n8n Cloud（海外サーバー）からのREST APIアクセスを許可する方法
2. サーバーレベルのファイアウォールでREST APIを除外する設定
3. 特定のIPアドレスをホワイトリストに追加する方法

サイトURL: https://あなたのサイト.com
対象パス: /wp-json/wp/v2/*

お手数をおかけしますが、ご確認のほどよろしくお願いいたします。
```

#### 問い合わせ先：

**エックスサーバー サポート：**
- https://www.xserver.ne.jp/support/
- サーバーパネル → サポート → お問い合わせ

**期待される回答：**
- サーバーレベルの設定方法の案内
- またはエックスサーバー側で設定を変更してくれる可能性

---

### 🎯 方法3: n8nを日本国内でセルフホスト（長期的な根本解決）

n8n Cloudではなく、日本国内のサーバーでn8nをセルフホストすることで、
エックスサーバーの国外IP制限の問題を完全に回避できます。

#### メリット：

- ✅ エックスサーバーの国外IP制限を完全に回避
- ✅ より高速な接続（日本国内サーバー同士）
- ✅ タイムアウトエラーが発生しない
- ✅ カスタマイズの自由度が高い
- ✅ 長期的なコスト削減（n8n Cloudの月額料金不要）

#### セルフホストの選択肢：

##### 選択肢A: 日本国内VPS

**推奨VPSプロバイダー：**

| プロバイダー | 最低料金 | メモリ | おすすめ度 |
|------------|---------|--------|-----------|
| **ConoHa VPS** | 678円/月 | 512MB | ⭐⭐⭐⭐⭐ |
| **さくらのVPS** | 643円/月 | 512MB | ⭐⭐⭐⭐ |
| **Kagoya CLOUD VPS** | 550円/月 | 1GB | ⭐⭐⭐⭐ |
| **ABLENET VPS** | 580円/月 | 1GB | ⭐⭐⭐ |

**推奨：ConoHa VPS（エックスサーバーと同じ会社）**
- エックスサーバーと同じGMOグループ
- 日本国内データセンター
- 初期費用なし
- 簡単セットアップ

**ConoHa VPSでのn8nセットアップ手順：**

1. **ConoHa VPSアカウント作成**
   - https://www.conoha.jp/vps/

2. **VPSを作成**
   - プラン: 512MB以上推奨
   - OS: Ubuntu 22.04
   - リージョン: 東京

3. **SSHでVPSに接続**
   ```bash
   ssh root@あなたのVPSのIPアドレス
   ```

4. **Dockerをインストール**
   ```bash
   # システムアップデート
   apt update && apt upgrade -y

   # Dockerをインストール
   curl -fsSL https://get.docker.com -o get-docker.sh
   sh get-docker.sh

   # Docker Composeをインストール
   apt install docker-compose -y
   ```

5. **n8nをDockerで起動**
   ```bash
   # n8n用ディレクトリを作成
   mkdir -p ~/.n8n

   # n8nを起動
   docker run -d \
     --name n8n \
     --restart always \
     -p 5678:5678 \
     -e N8N_HOST="あなたのVPSのIPアドレス" \
     -e N8N_PORT=5678 \
     -e N8N_PROTOCOL=http \
     -e WEBHOOK_URL="http://あなたのVPSのIPアドレス:5678/" \
     -v ~/.n8n:/home/node/.n8n \
     n8nio/n8n
   ```

6. **ファイアウォールを設定**
   ```bash
   # UFWをインストール（未インストールの場合）
   apt install ufw -y

   # SSH（22）とn8n（5678）を許可
   ufw allow 22/tcp
   ufw allow 5678/tcp
   ufw enable
   ```

7. **ブラウザでアクセス**
   ```
   http://あなたのVPSのIPアドレス:5678
   ```

8. **n8nの初期設定**
   - ブラウザでn8nにアクセス
   - 管理者アカウントを作成
   - ワークフローをインポート

9. **SSL証明書を設定（推奨）**

   無料のLet's Encryptを使用：

   ```bash
   # ドメインを取得してDNS設定を完了させておく
   # 例: n8n.yourdomain.com → VPSのIPアドレス

   # Nginxをインストール
   apt install nginx certbot python3-certbot-nginx -y

   # Nginx設定ファイルを作成
   cat > /etc/nginx/sites-available/n8n <<EOF
   server {
       listen 80;
       server_name n8n.yourdomain.com;

       location / {
           proxy_pass http://localhost:5678;
           proxy_http_version 1.1;
           proxy_set_header Upgrade \$http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host \$host;
           proxy_cache_bypass \$http_upgrade;
       }
   }
   EOF

   # 設定を有効化
   ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
   nginx -t
   systemctl restart nginx

   # SSL証明書を取得
   certbot --nginx -d n8n.yourdomain.com

   # n8nを再起動（HTTPS用）
   docker stop n8n
   docker rm n8n
   docker run -d \
     --name n8n \
     --restart always \
     -p 5678:5678 \
     -e N8N_HOST="n8n.yourdomain.com" \
     -e N8N_PORT=5678 \
     -e N8N_PROTOCOL=https \
     -e WEBHOOK_URL="https://n8n.yourdomain.com/" \
     -v ~/.n8n:/home/node/.n8n \
     n8nio/n8n
   ```

10. **HTTPSでアクセス**
    ```
    https://n8n.yourdomain.com
    ```

---

##### 選択肢B: Dockerデスクトップ（ローカル開発・テスト用）

**Macの場合：**

1. Docker Desktopをインストール
   - https://www.docker.com/products/docker-desktop

2. ターミナルで実行：
   ```bash
   docker run -d \
     --name n8n \
     -p 5678:5678 \
     -v ~/.n8n:/home/node/.n8n \
     n8nio/n8n
   ```

3. ブラウザでアクセス：
   ```
   http://localhost:5678
   ```

**⚠️ 注意：**
- ローカルで動作させる場合、外部からのWebhookアクセスには別途トンネリング（ngrokなど）が必要
- 本番運用には不向き（PCを常時起動させる必要がある）

---

##### 選択肢C: Railway / Render（海外サービス）

**⚠️ これらは海外サーバーのため、エックスサーバーの国外IP制限問題は解決しません。**

---

## 📊 各方法の比較

| 方法 | 難易度 | 費用 | 解決速度 | 推奨度 |
|-----|--------|------|---------|--------|
| **方法1: .htaccess** | ⭐ 簡単 | 無料 | 5分 | ⭐⭐⭐⭐ |
| **方法2: サポート問い合わせ** | ⭐ 簡単 | 無料 | 1-3日 | ⭐⭐⭐⭐⭐ |
| **方法3: セルフホスト（VPS）** | ⭐⭐⭐ 中級 | 550円/月〜 | 1-2時間 | ⭐⭐⭐⭐⭐ |
| **方法3: セルフホスト（ローカル）** | ⭐⭐ 簡単 | 無料 | 10分 | ⭐⭐（テストのみ） |

---

## 🎯 推奨される対処の流れ

### ステップ1: まず方法1を試す（5分）

`.htaccess`の設定を追加して、すぐにテスト。

**成功した場合：**
- ✅ すぐに運用開始可能
- エックスサーバーのサポートに問い合わせる必要なし

**失敗した場合：**
- ステップ2へ

---

### ステップ2: 方法2（サポート問い合わせ）と方法3（セルフホスト）を並行

**方法2（サポート問い合わせ）：**
- 問い合わせを送信
- 回答を待つ（1-3日）
- 根本的な解決につながる可能性

**方法3（セルフホスト）：**
- 待ち時間を利用してVPSをセットアップ
- すぐに運用開始可能
- 長期的には最もコスパが良い

---

## 🔧 トラブルシューティング

### Q1: 方法1を試したがまだタイムアウトする

**A:** 以下を確認してください：

1. `.htaccess`の構文エラーがないか確認
   ```bash
   # エックスサーバーのSSHまたはファイルマネージャーで確認
   ```

2. `.htaccess`が正しく読み込まれているか確認
   - WordPress管理画面でパーマリンク設定を再保存
   - これで`.htaccess`が再生成される

3. ブラウザのキャッシュをクリア

4. n8nのキャッシュをクリア（ワークフローを再保存）

---

### Q2: セルフホストの場合、既存のワークフローを移行できますか？

**A:** はい、可能です。

**n8n Cloudからエクスポート：**
1. n8n Cloudでワークフローを開く
2. 右上の「...」→ 「Export」
3. JSONファイルをダウンロード

**セルフホストn8nにインポート：**
1. セルフホストn8nにアクセス
2. 「Import from File」
3. ダウンロードしたJSONファイルを選択
4. 認証情報を再設定

---

### Q3: セルフホストのコストはどのくらい？

**A:** 月額550円〜1,000円程度です。

**内訳：**
- VPS: 550円〜1,000円/月
- ドメイン（オプション）: 100円〜1,500円/年
- SSL証明書: 無料（Let's Encrypt使用）

**比較：**
- n8n Cloud Starter: $20/月（約3,000円）
- セルフホスト: 550円/月〜

→ **年間約30,000円の節約**

---

## 📝 まとめ

エックスサーバー + n8n CloudのETIMEDOUTエラーは、以下の3つの方法で解決できます：

1. **すぐに試す：** `.htaccess`設定（5分）
2. **サポートに依頼：** エックスサーバーに問い合わせ（1-3日）
3. **根本解決：** n8nを日本国内VPSでセルフホスト（1-2時間）

**最も推奨される方法：**
- 短期：方法1（.htaccess）
- 長期：方法3（セルフホスト）+ 方法2（サポート問い合わせ）

---

**最終更新:** 2025-01-18
**対象:** エックスサーバー + n8n Cloudユーザー
**参考:** Cloudron Forum - https://forum.cloudron.io/topic/12461/n8n-times-out-for-wordpress-api
