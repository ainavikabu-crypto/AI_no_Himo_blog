# アイヒモのSEOブログ自動生成 - Google Drive対応版 セットアップガイド

このワークフローは、Tavilyで調査した情報をもとにアイヒモのブログ記事を自動生成し、**WordPress投稿前にGoogle Driveにバックアップ**を保存します。

## 📋 目次

1. [ワークフローの概要](#ワークフローの概要)
2. [必要な準備](#必要な準備)
3. [セットアップ手順](#セットアップ手順)
4. [Google Drive設定](#google-drive設定)
5. [ワークフローの使い方](#ワークフローの使い方)
6. [トラブルシューティング](#トラブルシューティング)

---

## ワークフローの概要

### 処理フロー

```
フォーム入力
  ↓
Tavily検索 → 結果整形
  ↓
アイヒモブログ生成
  ↓
HTML変換 + メタデータ生成
  ↓
結合・クリーンアップ
  ↓
【NEW】Google Driveに保存 ← ここが追加ポイント！
  ↓
WordPress投稿（下書き）
  ↓
画像アップロード・設定
  ↓
Telegram通知
```

### 新機能：Google Drive保存

- **保存内容**: 完全なHTMLドキュメント（メタデータ付き）
- **ファイル名形式**: `YYYY-MM-DD_HH-MM-SS_slug.html`
  - 例: `2025-01-17_14-30-45_ai-automation-trial.html`
- **メタデータ**: タイトル、スラッグ、メタディスクリプション、作成日時
- **形式**: HTMLファイル（ブラウザで直接開いて確認可能）

---

## 必要な準備

### 1. n8nアカウント
- n8n Cloud または セルフホスティング環境

### 2. API・サービスアカウント

#### 必須
- **Tavily API** ([https://tavily.com](https://tavily.com))
- **OpenAI API** (GPT-4o-mini使用)
- **Google Drive API** ← 新規追加
- **WordPress REST API**

#### オプション
- **Telegram Bot** (完了通知用)

---

## セットアップ手順

### ステップ1: ワークフローのインポート

1. n8nにログイン
2. 左メニューから「Workflows」を選択
3. 右上の「Add Workflow」→「Import from File」をクリック
4. `アイヒモのSEOブログ自動生成_Tavily版_GoogleDrive対応.json` を選択
5. インポート完了

### ステップ2: Google Drive認証設定

#### 2-1. Google Cloud Consoleでプロジェクト作成

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. 新しいプロジェクトを作成（または既存プロジェクトを選択）
3. 「APIとサービス」→「ライブラリ」から **Google Drive API** を有効化

#### 2-2. OAuth 2.0認証情報の作成

1. 「APIとサービス」→「認証情報」を開く
2. 「認証情報を作成」→「OAuth クライアント ID」を選択
3. アプリケーションの種類：**ウェブアプリケーション**
4. 承認済みのリダイレクト URI に以下を追加：
   ```
   https://n8n.your-domain.com/rest/oauth2-credential/callback
   ```
   ※ n8n Cloudの場合は `https://app.n8n.cloud/rest/oauth2-credential/callback`
5. クライアントIDとクライアントシークレットをコピー

#### 2-3. n8nでGoogle Drive認証を追加

1. n8nのワークフローエディタで「Google Driveにアップロード」ノードを選択
2. 「Credential to connect with」の横の鉛筆アイコンをクリック
3. 「Create New Credential」を選択
4. 以下を入力：
   - **Client ID**: Google CloudでコピーしたクライアントID
   - **Client Secret**: Google Cloudでコピーしたクライアントシークレット
5. 「Connect my account」をクリックしてGoogleアカウントで認証
6. 保存

### ステップ3: その他の認証情報設定

#### Tavily API

1. 「Tavily検索」ノードを選択
2. Credential設定でTavily APIキーを入力

#### OpenAI API

1. 「GPT-4o-mini」ノードと「HTML作成」ノードを選択
2. それぞれにOpenAI APIキーを設定

#### WordPress API

1. 「WordPress投稿」ノードを選択
2. WordPress サイトのURL、ユーザー名、アプリケーションパスワードを設定

#### Telegram (オプション)

1. 「完了通知」ノードを選択
2. Telegram Bot TokenとチャットIDを設定

### ステップ4: Google Drive保存先フォルダの設定

1. Google Driveで記事保存用のフォルダを作成（例：「アイヒモブログ記事」）
2. フォルダを右クリック→「共有」→「リンクを取得」
3. URLの最後の部分がフォルダID
   ```
   https://drive.google.com/drive/folders/[ここがフォルダID]
   ```
4. n8nの「Google Driveにアップロード」ノードで以下を設定：
   - **Drive**: My Drive
   - **Folder**: フォルダIDを入力（または選択）

### ステップ5: WordPress URL設定（環境変数を使用 - 推奨）

**v2.0.0以降、WordPress URLは環境変数から自動取得されます。**

#### 環境変数の設定方法

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

#### 旧バージョン（v1.x）からの移行

v1.xを使用している場合、以下のノードでWordPress URLを手動で更新する必要があります：

1. **WordPressに画像アップロード**ノード
2. **WordPress投稿に画像設定**ノード

**v2.0.0にアップグレードすることを強く推奨します。**

---

## Google Drive設定

### 保存されるファイルの内容

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>記事タイトル</title>
    <meta name="description" content="メタディスクリプション">
</head>
<body>
    <!-- メタデータセクション -->
    <div class="metadata">
        作成日時: 2025-01-17 14:30:45
        タイトル: 記事タイトル
        スラッグ: article-slug
        メタディスクリプション: ...
    </div>

    <!-- 記事コンテンツ -->
    <div class="content">
        <!-- HTML形式の記事本文 -->
    </div>
</body>
</html>
```

### フォルダ構造の推奨

```
Google Drive
└── アイヒモブログ記事/
    ├── 2025-01-17_14-30-45_ai-automation-trial.html
    ├── 2025-01-18_10-15-30_blog-writing-tips.html
    └── ...
```

### アクセス権限

- デフォルトでは、認証したGoogleアカウントのみがアクセス可能
- 必要に応じて、フォルダの共有設定を変更

---

## ワークフローの使い方

### 基本的な使い方

1. n8nでワークフローを開く
2. 「フォーム送信時」ノードの「Test URL」をコピー
3. ブラウザでURLを開く
4. 「調査キーワード」を入力して送信
5. 処理完了まで待機（5-10分程度）

### 処理完了後の確認

#### Google Drive
1. 設定したフォルダにHTMLファイルが保存されていることを確認
2. ファイルをダブルクリックしてブラウザで開く
3. メタデータと記事内容を確認

#### WordPress
1. WordPressダッシュボードにログイン
2. 「投稿」→「下書き」を確認
3. 記事を編集・公開

#### Telegram（設定済みの場合）
- 通知メッセージにGoogle DriveのリンクとWordPress下書きのリンクが含まれます

---

## トラブルシューティング

### Google Driveアップロードが失敗する

**エラー**: `Authentication failed`

**解決策**:
1. Google Drive認証情報を再設定
2. Google Cloud ConsoleでGoogle Drive APIが有効化されているか確認
3. OAuth同意画面の設定を確認

**エラー**: `Folder not found`

**解決策**:
1. フォルダIDが正しいか確認
2. 認証したGoogleアカウントがフォルダにアクセスできるか確認

### ファイル名が文字化けする

**解決策**:
- 「Google Drive用ドキュメント作成」ノードのJavaScriptコードで、タイムスタンプとスラッグが正しく生成されているか確認

### WordPress投稿が失敗する

**解決策**:
1. WordPress REST APIが有効か確認
2. アプリケーションパスワードが正しいか確認
3. WordPressのURLが正しいか確認（`https://`を含む）

### HTMLファイルがGoogle Driveで正しく表示されない

**解決策**:
- Google Driveでは、HTMLファイルを直接プレビューできません
- 「ダウンロード」してからローカルでブラウザで開いてください
- または、Google Driveの「アプリで開く」→「Google ドキュメント」を選択

---

## カスタマイズ例

### 保存先フォルダを日付ごとに分ける

「Google Drive用ドキュメント作成」ノードのコードを以下のように修正：

```javascript
// 日付フォルダ名を作成
const folderName = jstDate.toISOString().split('T')[0]; // YYYY-MM-DD

// fileName変数の前に追加
const fileName = `${folderName}/${timestamp}_${metadata.slug}.html`;
```

その後、「Google Driveにアップロード」ノードで動的にフォルダを作成する処理を追加します。

### ファイル名のカスタマイズ

「Google Drive用ドキュメント作成」ノードの`fileName`変数を変更：

```javascript
// 例：日付 + タイトル
const fileName = `${timestamp}_${metadata.title}.html`;

// 例：スラッグのみ
const fileName = `${metadata.slug}.html`;
```

---

## まとめ

このワークフローにより、以下が実現できます：

✅ **バックアップの自動化**: すべての記事がGoogle Driveに保存される
✅ **履歴管理**: タイムスタンプ付きファイル名で時系列管理
✅ **オフライン確認**: HTMLファイルとしてダウンロード可能
✅ **メタデータ保存**: SEO情報も一緒に保存
✅ **WordPress連携**: 下書きとして自動投稿

問題が発生した場合は、各ノードの出力を確認し、エラーメッセージを参照してください。

---

## 変更履歴

### v2.0.0 (2025-01-18) - 環境変数対応版

**重要な変更:**
- ✅ WordPress URLを環境変数から取得するように改善
  - `WORDPRESS_SITE_URL` 環境変数をサポート
  - ハードコードされたURLを削除
- ✅ Telegram認証情報のプレースホルダーを削除
  - 手動で設定が必要（オプション）
- ✅ トラブルシューティングガイドを追加
  - `WORDPRESS_TROUBLESHOOTING.md` を参照

**移行方法:**
1. 環境変数 `WORDPRESS_SITE_URL` を設定
2. ワークフローを再インポート
3. 各認証情報を再設定

### v1.0.0 (2025-01-17)
- 初回リリース
- Google Drive保存機能
- WordPress自動投稿
- Telegram通知
