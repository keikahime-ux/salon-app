# 美容サロンカウンセリングPWA 引継ぎドキュメント

## プロジェクト概要

美容サロン向けカウンセリングカルテ管理PWAアプリ。

- **アプリURL**: https://beauty-salon-counsel-hl30.bolt.host
- **bolt.newプロジェクト**: https://bolt.new/~/sb1-zjepkvdc
- **Cloudflare Worker API**: https://salon-karte-api.yubbb14.workers.dev

---

## 構成

| レイヤー | 技術 | 詳細 |
|----------|------|------|
| フロントエンド | Vanilla JS + Vite + PWA | bolt.newで作成・ホスティング |
| ホスティング | bolt.host | beauty-salon-counsel-hl30.bolt.host |
| API | Cloudflare Workers | salon-karte-api（Worker名） |
| テキストDB | Cloudflare KV | salon-karte-kv |
| 画像ストレージ | Cloudflare R2 | salon-karte-images-yubbb14 |

---

## Cloudflare設定

### Workerバインディング（重要）

| Type | Variable Name | Value |
|------|--------------|-------|
| KV namespace | `CARTE_KV` | salon-karte-kv |
| R2 bucket | `GAZO_BUCKET` | salon-karte-images-yubbb14 |

> ⚠️ Cloudflareダッシュボードは**英語UI**で操作すること（日本語UIだと変数名が自動翻訳される）

### Worker APIエンドポイント

| メソッド | パス | 説明 |
|---------|------|------|
| GET | /customers | 顧客一覧取得 |
| POST | /customers | 顧客作成（同名・フリガナが既存の場合は既存を返す） |
| GET | /kartes/:customerId | 顧客のカルテ一覧取得 |
| POST | /kartes | カルテ保存 |
| POST | /images | 画像アップロード（base64） |
| GET | /images/:imageId | 画像取得 |

---

## ファイル構成（bolt.new）

```
/
├── api.js          # Cloudflare Workers APIクライアント
├── app.js          # ルーティング・レンダリング
├── views.js        # 各画面のHTML生成
├── main.js         # エントリーポイント
├── index.html      # PWAのHTMLベース
├── style.css       # スタイル
├── worker.js       # Cloudflare Worker本体（別途デプロイ）
├── wrangler.toml   # Wrangler設定
└── package.json
```

---

## worker.js（Cloudflare Worker本体）

```javascript
const CORS_HEADERS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type',
};

function corsResponse(body, status = 200, extraHeaders = {}) {
  return new Response(body, {
    status,
    headers: { ...CORS_HEADERS, 'Content-Type': 'application/json', ...extraHeaders },
  });
}

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const path = url.pathname;

    if (request.method === 'OPTIONS') {
      return new Response(null, { status: 200, headers: CORS_HEADERS });
    }

    try {
      if (path === '/customers' && request.method === 'GET') {
        return await handleGetCustomers(env);
      }
      if (path === '/customers' && request.method === 'POST') {
        return await handleCreateCustomer(request, env);
      }
      if (path.startsWith('/kartes/') && request.method === 'GET') {
        const customerId = path.split('/')[2];
        return await handleGetKartes(customerId, env);
      }
      if (path === '/kartes' && request.method === 'POST') {
        return await handleSaveKarte(request, env);
      }
      if (path === '/images' && request.method === 'POST') {
        return await handleUploadImage(request, env);
      }
      if (path.startsWith('/images/') && request.method === 'GET') {
        const imageId = path.split('/')[2];
        return await handleGetImage(imageId, env);
      }
      return corsResponse(JSON.stringify({ error: 'Not Found' }), 404);
    } catch (err) {
      return corsResponse(JSON.stringify({ error: err.message }), 500);
    }
  },
};

async function handleGetCustomers(env) {
  const list = await env.CARTE_KV.list({ prefix: 'customer:' });
  const customers = [];
  for (const key of list.keys) {
    const raw = await env.CARTE_KV.get(key.name);
    if (raw) customers.push(JSON.parse(raw));
  }
  return corsResponse(JSON.stringify(customers));
}

async function handleCreateCustomer(request, env) {
  const data = await request.json();
  // 同名・フリガナが既存の場合は既存顧客を返す
  if (data.name) {
    const list = await env.CARTE_KV.list({ prefix: 'customer:' });
    for (const key of list.keys) {
      const raw = await env.CARTE_KV.get(key.name);
      if (!raw) continue;
      const existing = JSON.parse(raw);
      if (existing.name === data.name && existing.kana === data.kana) {
        return corsResponse(JSON.stringify(existing), 200);
      }
    }
  }
  const id = crypto.randomUUID();
  const customer = { id, ...data };
  await env.CARTE_KV.put(`customer:${id}`, JSON.stringify(customer));
  return corsResponse(JSON.stringify(customer), 201);
}

async function handleGetKartes(customerId, env) {
  const list = await env.CARTE_KV.list({ prefix: `karte:${customerId}:` });
  const kartes = [];
  for (const key of list.keys) {
    const raw = await env.CARTE_KV.get(key.name);
    if (raw) kartes.push(JSON.parse(raw));
  }
  return corsResponse(JSON.stringify(kartes));
}

async function handleSaveKarte(request, env) {
  const data = await request.json();
  const customerId = data.customerId;
  const karteId = crypto.randomUUID();
  const karte = { id: karteId, ...data };
  await env.CARTE_KV.put(`karte:${customerId}:${karteId}`, JSON.stringify(karte));
  const custRaw = await env.CARTE_KV.get(`customer:${customerId}`);
  if (custRaw) {
    const cust = JSON.parse(custRaw);
    if (karte.createdAt) cust.lastVisit = karte.createdAt;
    if (karte.type) cust.lastType = karte.type;
    await env.CARTE_KV.put(`customer:${customerId}`, JSON.stringify(cust));
  }
  return corsResponse(JSON.stringify(karte), 201);
}

async function handleUploadImage(request, env) {
  const { image } = await request.json();
  if (!image) {
    return corsResponse(JSON.stringify({ error: 'image field is required' }), 400);
  }
  const base64 = image.replace(/^data:image\/[a-zA-Z+]+;base64,/, '');
  const binaryStr = atob(base64);
  const bytes = new Uint8Array(binaryStr.length);
  for (let i = 0; i < binaryStr.length; i++) {
    bytes[i] = binaryStr.charCodeAt(i);
  }
  const contentTypeMatch = image.match(/^data:(image\/[a-zA-Z+]+);base64,/);
  const contentType = contentTypeMatch ? contentTypeMatch[1] : 'image/png';
  const imageId = crypto.randomUUID();
  await env.GAZO_BUCKET.put(imageId, bytes, {
    httpMetadata: { contentType },
  });
  return corsResponse(JSON.stringify({ imageId }), 201);
}

async function handleGetImage(imageId, env) {
  const object = await env.GAZO_BUCKET.get(imageId);
  if (!object) {
    return corsResponse(JSON.stringify({ error: 'Image not found' }), 404);
  }
  const headers = {
    ...CORS_HEADERS,
    'Content-Type': object.httpMetadata?.contentType || 'image/png',
    'Cache-Control': 'public, max-age=31536000, immutable',
  };
  return new Response(object.body, { status: 200, headers });
}
```

---

## api.js（フロントエンドAPIクライアント）

```javascript
export const WORKER_URL = 'https://salon-karte-api.yubbb14.workers.dev';

async function request(path, options = {}) {
  const res = await fetch(`${WORKER_URL}${path}`, {
    headers: { 'Content-Type': 'application/json', ...options.headers },
    ...options,
  });
  if (!res.ok) {
    const text = await res.text().catch(() => 'Unknown error');
    throw new Error(`API error ${res.status}: ${text}`);
  }
  return res.json();
}

export async function getCustomers() {
  return request('/customers');
}
export async function createCustomer(data) {
  return request('/customers', { method: 'POST', body: JSON.stringify(data) });
}
export async function getKartes(customerId) {
  return request(`/kartes/${customerId}`);
}
export async function saveKarte(data) {
  return request('/kartes', { method: 'POST', body: JSON.stringify(data) });
}
export async function uploadImage(base64Data) {
  const res = await request('/images', {
    method: 'POST',
    body: JSON.stringify({ image: base64Data }),
  });
  return res.imageId;
}
export function getImageUrl(imageId) {
  if (!imageId) return '';
  return `${WORKER_URL}/images/${imageId}`;
}
```

---

## wrangler.toml

```toml
name = "salon-karte-api"
main = "worker.js"
compatibility_date = "2024-01-01"

[[kv_namespaces]]
binding = "CARTE_KV"
id = "（CloudflareダッシュボードのKV namespace ID）"

[[r2_buckets]]
binding = "GAZO_BUCKET"
bucket_name = "salon-karte-images-yubbb14"
```

---

## 完成済み機能

- ✅ Eyelash・LashLiftカルテ（新規）
- ✅ Eyebrowカルテ（新規）
- ✅ 再来店カルテ（施術状態10段階評価・ビフォーアフター画像）
- ✅ 手書きサイン機能
- ✅ 顧客一覧・来店履歴表示
- ✅ クラウド保存（KV + R2）
- ✅ 画像付きカルテ保存（R2）

---

## VSCodeでの開発手順

### 初回セットアップ

```bash
# bolt.newからGitHubにエクスポート（bolt.new右上のGitHubアイコン）
git clone https://github.com/（あなたのリポジトリ）
cd salon-karte
npm install

# Wranglerインストール
npm install -g wrangler
wrangler login
```

### Workerのデプロイ

```bash
# wrangler.tomlのKV IDを実際のIDに変更してから
wrangler deploy
```

### フロントエンドのデプロイ

```bash
npm run build
# bolt.hostへはbolt.newのPublishボタンから
```

---

## 新ルームへの引継ぎプロンプト

以下を新しいClaudeのチャットに貼り付けてください：

---

```
美容サロン向けカウンセリングカルテPWAアプリの開発を続けています。

【アプリURL】
https://beauty-salon-counsel-hl30.bolt.host

【構成】
- フロントエンド：Vanilla JS + Vite + PWA
- ホスティング：bolt.host
- データ保存：Cloudflare Workers KV（salon-karte-kv）
- 画像保存：Cloudflare R2（salon-karte-images-yubbb14）
- API：Cloudflare Workers（salon-karte-api.yubbb14.workers.dev）

【Cloudflare Workerバインディング】
- KV: CARTE_KV → salon-karte-kv
- R2: GAZO_BUCKET → salon-karte-images-yubbb14
※ Cloudflareダッシュボードは英語UIで操作すること

【完成済み機能】
- Eyelash・LashLiftカルテ（新規）
- Eyebrowカルテ（新規）
- 再来店カルテ（施術状態10段階評価・ビフォーアフター画像）
- 手書きサイン機能
- 顧客一覧・来店履歴表示
- クラウド保存（KV + R2）
- 画像付きカルテ保存

【開発環境】
- VSCode使用
- GitHubリポジトリ：keikahime-ux（プライベート）
- bolt.newプロジェクト：https://bolt.new/~/sb1-zjepkvdc

【今回お願いしたいこと】
（ここに次にやりたいことを書く）
```
