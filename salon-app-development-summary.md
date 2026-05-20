# サロン帳簿アプリ開発 完了記録

## プロジェクト概要
娘さんの美容サロン向け青色申告帳簿PWAアプリ

**アプリURL:** https://salon-book.pages.dev/

**開発環境:**
- スマートフォンのみ（PC不使用）
- StackBlitz (bolt.new) でアプリ開発
- VSCode + Claude拡張機能でコード編集
- Cloudflare Pages でホスティング
- Cloudflare Workers でAIプロキシ

---

## 実装した機能

### 1. 基本機能
- 売上・経費の入力フォーム
- 月次・年間の損益集計
- データのローカルストレージ保存
- PWA対応（ホーム画面追加可能）

### 2. AIレシート読み取り機能
- カメラでレシート撮影
- Anthropic Claude APIで自動読み取り
- 仕入高・消費税を自動分類して経費入力

### 3. AI相談機能
- 帳簿・税金に関する質問応答
- よくある質問ボタン

### 4. デザイン
- ダークパープル基調
- ピンクのアクセントカラー
- キティちゃんアイコン（favicon）

---

## 技術スタック

### フロントエンド
- **HTML/CSS/JavaScript** (バニラJS)
- **PWA** manifest.json + service-worker.js
- **LocalStorage** でデータ永続化

### バックエンド
- **Cloudflare Workers** (AIプロキシ)
- **Anthropic Claude API** (Haiku 4.5)

### デプロイ
- **Cloudflare Pages** (GitHub連携)
- **GitHubリポジトリ:** keikahime-ux

---

## Cloudflare Worker設定

### Worker名
`salon-ai-proxy`

### URL
```
https://salon-ai-proxy.yubbb14.workers.dev
```

### 環境変数
| 変数名 | 値 |
|--------|-----|
| `ANTHROPIC_API_KEY` | Anthropic APIキー (秘密) |

### Workerコードの役割
1. アプリからのPOSTリクエストを受信
2. Anthropic APIにリクエストを転送
3. APIキーを隠蔽（セキュリティ）
4. CORS対応

---

## 重要な修正ポイント

### ① messagesフィールドの正しい形式
Anthropic APIへのリクエストは以下の形式が必須:

```javascript
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "base64画像データ"
          }
        },
        {
          "type": "text",
          "text": "プロンプト"
        }
      ]
    }
  ]
}
```

### ② Worker.jsのコード構造

**正しいコード:**
```javascript
export default {
  async fetch(request, env) {
    // CORS設定
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    if (request.method !== 'POST') {
      return new Response(JSON.stringify({ error: 'Method not allowed' }), {
        status: 405,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });
    }

    try {
      const body = await request.json();

      const claudeResp = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': env.ANTHROPIC_API_KEY,
          'anthropic-version': '2023-06-01',
        },
        body: JSON.stringify({
          model: body.model || 'claude-haiku-4-5-20251001',
          max_tokens: body.max_tokens || 1024,
          messages: body.messages || [{ role: 'user', content: body.content || '' }],
        }),
      });

      const data = await claudeResp.json();

      return new Response(JSON.stringify(data), {
        status: claudeResp.status,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });

    } catch (err) {
      return new Response(JSON.stringify({ error: err.message }), {
        status: 500,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });
    }
  },
};
```

---

## トラブルシューティング記録

### エラー1: `messages: Field required`
**原因:** Workerに `messages` フィールドが送られていない  
**解決:** Worker.jsで `messages` フィールドを構築

### エラー2: `image: Extra inputs are not permitted`
**原因:** 画像データの形式が間違っている  
**解決:** `content` を配列形式に修正（`type` と `source` を含める）

### エラー3: Worker URLで405エラー
**原因:** GETリクエストを拒否する設計  
**解決:** 正常動作（POSTのみ受け付ける）

---

## VSCode + Claude拡張機能の活用

### 使い方
1. VSCodeでプロジェクトフォルダを開く
2. 下部のClaude拡張チャットでコード修正を依頼
3. 自動でコードを検索・修正してくれる

### 便利なプロンプト例
```
「index.htmlの中で、レシート画像をWorker URLにPOSTしている関数を見つけて表示してください」

「このコードを修正してください。Anthropic APIのmessages形式に合わせて、contentを配列にして、typeとsourceを含めた形式で画像を送信するようにしてください。」

「この修正をworker.jsに適用してください」
```

---

## まだやっていないタスク

- [ ] 売上フォームのメニュー構造変更（料金表の整理）
- [ ] 配色変更
- [ ] 年間トータル表示（損益タブ） ← **完了済み**

---

## 次のプロジェクト（カウンセリングシート）への引き継ぎ

### 再利用できる知識
1. **bolt.newでの開発フロー**
   - アプリのベースをbolt.newで生成
   - VSCodeで細かい修正
   - GitHubにpush → Cloudflare Pagesで自動デプロイ

2. **Cloudflare Workerの設定方法**
   - Workers作成 → コード貼り付け → 環境変数設定 → Deploy

3. **Anthropic API利用パターン**
   - 画像認識（レシート読み取り）
   - テキスト生成（AI相談）

4. **PWA化の手順**
   - manifest.json
   - service-worker.js
   - アイコン設定

### カウンセリングシート開発のヒント
- **フォーム入力UI**: 今回の売上/経費フォームを参考
- **AI機能**: 質問応答やデータ解析に同じWorker構造を流用可能
- **データ保存**: LocalStorageまたはCloudflare KVを検討
- **デザイン**: サロン向けの柔らかい雰囲気を継承

---

## 開発に使用したアカウント

### Cloudflare
- **メインアカウント:** Yubbb14@yahoo.co.jp
- **別アカウント:** 別のGoogleアカウント（keikahimeのWorkerはこちら）

### GitHub
- **リポジトリ:** keikahime-ux (プライベート)

### Anthropic
- **APIキー:** 新しいキーを生成済み（古いキーは漏洩したため無効化）

---

## 参考資料

- Anthropic API公式ドキュメント: https://docs.anthropic.com
- Cloudflare Workers公式: https://developers.cloudflare.com/workers/
- PWA公式ガイド: https://web.dev/progressive-web-apps/

---

## 完成日
2026年5月16日

お疲れさまでした！🎉
