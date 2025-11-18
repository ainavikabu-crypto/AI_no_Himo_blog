# クイックセットアップガイド - WordPress連携を5分で解決

## 🚀 最短ルート

WordPressとn8nの連携が動かない？このガイドで5分で解決します。

---

## ステップ1: 環境変数を設定（1分）

### Dockerの場合

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e WORDPRESS_SITE_URL=https://your-site.com \
  -e TELEGRAM_CHAT_ID=your-chat-id \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

`https://your-site.com` を **あなたのWordPressサイトのURL** に変更してください。

### npmの場合

```bash
export WORDPRESS_SITE_URL=https://your-site.com
export TELEGRAM_CHAT_ID=your-chat-id
n8n start
```

### n8n Cloudの場合

1. n8n Cloudダッシュボードにログイン
2. **Settings** → **Environments**
3. 環境変数を追加:
   - **Key**: `WORDPRESS_SITE_URL`
   - **Value**: `https://your-site.com`

---

## ステップ2: WordPress アプリケーションパスワードを取得（2分）

1. **WordPress管理画面**にログイン
2. **ユーザー** → **プロフィール**
3. 下にスクロールして **「アプリケーションパスワード」** を探す
4. 名前を入力（例: `n8n-integration`）
5. **「新しいアプリケーションパスワードを追加」** をクリック
6. パスワードをコピー

---

## ステップ3: n8nに認証情報を設定（2分）

1. n8nワークフローエディタを開く
2. **「WordPress投稿」** ノードをクリック
3. **Credential** の横の鉛筆アイコンをクリック
4. 以下を入力:
   - **Site URL**: `https://your-site.com` （末尾のスラッシュなし）
   - **Username**: WordPressユーザー名
   - **Password**: ステップ2でコピーしたパスワード
5. **保存**

---

## ✅ 動作確認

### テスト1: REST APIにアクセス

ブラウザで以下を開く:
```
https://your-site.com/wp-json/wp/v2/posts
```

JSON データが表示されればOK！

### テスト2: ワークフローを実行

1. n8nで**「フォーム送信時」**ノードの **Test URL** をコピー
2. ブラウザで開く
3. キーワードを入力（例: 「テスト」）
4. 送信

5-10分後、WordPressの**「投稿」→「下書き」**に記事が作成されていればOK！

---

## 🆘 まだエラーが出る場合

### エラー: `401 Unauthorized`

→ アプリケーションパスワードが間違っています。ステップ2をもう一度やり直してください。

### エラー: `403 Forbidden`

→ WordPressのREST APIがブロックされています。
1. **WordPress管理画面** → **設定** → **パーマリンク**
2. **「投稿名」** を選択
3. **「変更を保存」**

### エラー: `404 Not Found`

→ WordPress URLが間違っています。
- `https://` を含めていますか？
- 末尾のスラッシュ `/` を削除していますか？
- 正しいドメインですか？

### その他のエラー

詳細なトラブルシューティングは [`WORDPRESS_TROUBLESHOOTING.md`](./WORDPRESS_TROUBLESHOOTING.md) を参照してください。

---

## 📚 完全なドキュメント

- [GoogleDrive対応ワークフロー_セットアップガイド.md](./GoogleDrive対応ワークフロー_セットアップガイド.md)
- [WORDPRESS_TROUBLESHOOTING.md](./WORDPRESS_TROUBLESHOOTING.md)
- [README-AIHIMO.md](./README-AIHIMO.md)

---

**所要時間:** 約5分
**最終更新:** 2025-01-18
