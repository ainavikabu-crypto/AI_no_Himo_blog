# Tavily専用ノード版ワークフローガイド

## 概要

このガイドでは、**Tavily専用ノード**（`@tavily/n8n-nodes-tavily`）を使用したワークフローについて説明します。

## ワークフロー

- **ファイル名**: `seo-blog-aihimo-tavily-node.json`
- **推奨度**: ★★★★★（最も推奨）

## Tavily専用ノードとは

Tavily専用ノードは、n8nの公式マーケットプレイスからインストールできるTavily AI検索のための専用ノードです。

### インストール方法

1. n8nの管理画面を開く
2. 「Settings」→「Community Nodes」をクリック
3. 「Install a community node」をクリック
4. パッケージ名 `@tavily/n8n-nodes-tavily` を入力
5. 「Install」をクリック

または、n8nのコマンドラインから：

```bash
npm install @tavily/n8n-nodes-tavily
```

### 認証情報の設定

1. n8nの「Credentials」→「Add Credential」
2. 「Tavily」を選択
3. API Key: `tvly-YOUR_API_KEY` を入力
4. 「Save」をクリック

## Codeノードの修正内容

### 修正前（HTTP Request版）

```javascript
const tavilyData = $input.all()[0].json;

let researchText = '';

if (tavilyData.answer) {
  researchText += '## AI要約\n' + tavilyData.answer + '\n\n';
}

if (tavilyData.results && tavilyData.results.length > 0) {
  researchText += '## 調査結果\n\n';

  tavilyData.results.forEach((result, index) => {
    researchText += `### ${index + 1}. ${result.title}\n`;
    researchText += `${result.content}\n`;
    researchText += `出典: ${result.url}\n\n`;
  });
}

return [{
  json: {
    research: researchText,
    query: $input.all()[0].json.query || '',
    rawResults: tavilyData.results || []
  }
}];
```

### 修正後（Tavily専用ノード版）

```javascript
// Tavily専用ノードからの出力を処理
const items = $input.all();

// フォームからの入力を取得
const formData = $('フォーム送信時').first().json;
const searchQuery = formData['調査キーワード'] || '';

// Tavily検索結果を取得
const tavilyData = items[0].json;

let researchText = '';

// 検索クエリを追加
if (searchQuery) {
  researchText += `# 検索キーワード\n${searchQuery}\n\n`;
}

// AI要約があれば追加
if (tavilyData.answer) {
  researchText += `## AI要約\n${tavilyData.answer}\n\n`;
}

// 検索結果を整形
if (tavilyData.results && tavilyData.results.length > 0) {
  researchText += '## 調査結果\n\n';

  tavilyData.results.forEach((result, index) => {
    researchText += `### ${index + 1}. ${result.title || '(タイトルなし)'}\n`;

    if (result.content) {
      researchText += `${result.content}\n`;
    }

    if (result.url) {
      researchText += `出典: ${result.url}\n`;
    }

    if (result.score) {
      researchText += `関連性スコア: ${(result.score * 100).toFixed(1)}%\n`;
    }

    researchText += '\n';
  });
}

// 画像情報があれば追加（オプション）
if (tavilyData.images && tavilyData.images.length > 0) {
  researchText += '## 関連画像\n';
  tavilyData.images.slice(0, 5).forEach((image, index) => {
    researchText += `${index + 1}. ${image}\n`;
  });
  researchText += '\n';
}

// 結果を返す
return [{
  json: {
    research: researchText,
    query: tavilyData.query || searchQuery,
    searchKeyword: searchQuery,
    rawResults: tavilyData.results || [],
    answer: tavilyData.answer || '',
    resultCount: tavilyData.results ? tavilyData.results.length : 0
  }
}];
```

## 主な改善点

### 1. フォーム入力の取得

**修正前**:
```javascript
query: $input.all()[0].json.query || ''
```

**修正後**:
```javascript
const formData = $('フォーム送信時').first().json;
const searchQuery = formData['調査キーワード'] || '';
```

**理由**: Tavily専用ノードでは、フォームからの入力を直接取得する必要があるため。

### 2. 検索キーワードの表示

**追加機能**:
```javascript
if (searchQuery) {
  researchText += `# 検索キーワード\n${searchQuery}\n\n`;
}
```

**理由**: AIブログライターに検索意図を明確に伝えるため。

### 3. 関連性スコアの追加

**追加機能**:
```javascript
if (result.score) {
  researchText += `関連性スコア: ${(result.score * 100).toFixed(1)}%\n`;
}
```

**理由**: 検索結果の品質を可視化し、AIが重要な情報を優先できるようにするため。

### 4. 関連画像の抽出

**追加機能**:
```javascript
if (tavilyData.images && tavilyData.images.length > 0) {
  researchText += '## 関連画像\n';
  tavilyData.images.slice(0, 5).forEach((image, index) => {
    researchText += `${index + 1}. ${image}\n`;
  });
  researchText += '\n';
}
```

**理由**: Tavilyが提供する関連画像URLを活用できるようにするため（将来的にアイキャッチ画像の自動選択に使用可能）。

### 5. より詳細な出力データ

**修正後の出力**:
```javascript
{
  research: researchText,           // 整形済みテキスト
  query: tavilyData.query || searchQuery,  // 実際の検索クエリ
  searchKeyword: searchQuery,       // フォームからの入力
  rawResults: tavilyData.results || [],    // 生の検索結果
  answer: tavilyData.answer || '',         // AI要約
  resultCount: tavilyData.results ? tavilyData.results.length : 0  // 結果数
}
```

**理由**: 下流のノード（アイヒモブログライター）が、より豊富な情報を活用できるようにするため。

### 6. エラーハンドリングの改善

**改善点**:
- `result.title || '(タイトルなし)'`: タイトルがない場合のフォールバック
- `if (result.content)`: コンテンツがある場合のみ表示
- `if (result.url)`: URLがある場合のみ表示
- `if (result.score)`: スコアがある場合のみ表示

**理由**: Tavilyの検索結果が不完全な場合でもエラーを防ぐため。

## 使用方法

### 1. ワークフローをインポート

```bash
# n8nにインポート
n8n import:workflow --input=/path/to/seo-blog-aihimo-tavily-node.json
```

または、n8n UIから：
1. 「Workflows」→「Import from File」
2. `seo-blog-aihimo-tavily-node.json` を選択

### 2. 認証情報を設定

**Tavily認証情報**:
1. Tavily専用ノード（「Tavily検索」）を開く
2. 「Credential to connect with」で、作成したTavily認証情報を選択

**その他の認証情報**:
- OpenAI API（GPT-4o-mini、HTML作成、OpenAI Chat Model）
- WordPress（WordPress投稿、画像アップロード、画像設定）
- Telegram（完了通知、オプション）

### 3. テスト実行

1. 「フォーム送信時」ノードを開く
2. 「Test URL」をコピー
3. ブラウザで開く
4. キーワードを入力（例：「n8nで業務自動化を試してみた」）
5. 送信

### 4. デバッグ

各ノードの実行結果を確認：

1. **Tavily検索**ノード:
   - `answer`: AI要約が含まれているか
   - `results`: 検索結果が10件程度あるか
   - `images`: 関連画像URLがあるか

2. **Tavily結果を整形**ノード:
   - `research`: テキストが適切に整形されているか
   - `searchKeyword`: フォームからの入力が取得できているか
   - `resultCount`: 結果数が正しいか

3. **アイヒモブログライター**ノード:
   - `output`: アイヒモらしい記事が生成されているか

## トラブルシューティング

### エラー: "Cannot read property 'json' of undefined"

**原因**: フォーム送信時ノードのデータが取得できていない

**解決方法**:
```javascript
// エラーチェックを追加
const formData = $('フォーム送信時').first()?.json || {};
const searchQuery = formData['調査キーワード'] || '';
```

### エラー: "tavilyData.results is not iterable"

**原因**: Tavily検索結果が空

**解決方法**:
```javascript
// 結果の存在チェックを追加
if (tavilyData.results && Array.isArray(tavilyData.results) && tavilyData.results.length > 0) {
  // 処理
}
```

### Tavily検索結果が少ない

**原因**: 検索クエリが不適切

**解決方法**:
1. Tavily検索ノードで`Search Depth`を`advanced`に設定
2. `Max Results`を`10`以上に設定
3. フォームで具体的なキーワードを入力

## バージョン比較

| 項目 | HTTP Request版 | Tavily専用ノード版 |
|------|----------------|-------------------|
| **インストール** | 不要 | Tavily専用ノードが必要 |
| **認証設定** | HTTP Header Auth | Tavily認証情報 |
| **検索精度** | 標準 | 高精度 |
| **関連性スコア** | なし | あり |
| **関連画像** | なし | あり |
| **エラーハンドリング** | 基本 | 詳細 |
| **フォーム入力取得** | 自動 | 手動（Codeノードで取得） |
| **推奨度** | ★★★☆☆ | ★★★★★ |

## まとめ

**Tavily専用ノード版**は、以下の理由で最も推奨されます：

1. **高精度な検索結果**: Tavilyの最新機能をフル活用
2. **関連性スコア**: 重要な情報を優先
3. **関連画像**: 将来的にアイキャッチ画像の自動選択に活用可能
4. **詳細な出力**: AIブログライターがより豊富な情報を活用可能
5. **エラーハンドリング**: 不完全な検索結果でも安定動作

## 次のステップ

1. **画像自動選択**: Tavilyの`images`配列を使用してアイキャッチ画像を自動選択
2. **検索結果のフィルタリング**: スコアが低い結果を除外
3. **複数キーワード検索**: 関連キーワードでも検索して情報を充実化
4. **検索結果のキャッシュ**: 同じキーワードの再検索を避ける

---

**更新日**: 2025-01-17
**バージョン**: 1.0.0
**作成者**: Claude Code
