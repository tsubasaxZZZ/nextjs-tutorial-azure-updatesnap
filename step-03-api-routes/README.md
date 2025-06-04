# Step 3: API ルートの実装

このステップでは、Next.js の Route Handlers を使って API エンドポイントを実装します。

## 🎯 このステップで学ぶこと

- Route Handlers の基本
- 外部 API との連携
- エラーハンドリング
- レート制限の実装
- キャッシュ制御

## 📖 解説

### Route Handlers とは

Next.js 14 の API ルートは Route Handlers と呼ばれ、`route.ts` ファイルで定義します。

```
app/
├── api/
│   ├── refresh/
│   │   └── route.ts      # /api/refresh
│   └── cache-url/
│       └── route.ts      # /api/cache-url
```

### HTTP メソッドのサポート

```ts
export async function GET(request: Request) {}
export async function POST(request: Request) {}
export async function PUT(request: Request) {}
export async function DELETE(request: Request) {}
```

## 🛠️ ハンズオン

### 1. Microsoft API クライアントの作成

`src/lib/azureFetch.ts`:

```ts
const AZURE_API_BASE = 'https://techcommunity.microsoft.com/plugins/custom/microsoft/o365/api-v2';

export interface AzureUpdate {
  id: number;
  guid: string;
  title: string;
  description: string;
  impactDescription?: string;
  publicDisclosureDate: string;
  tags: string[];
  detailsUrl: string;
}

export async function fetchAzureUpdate(id: string): Promise<AzureUpdate | null> {
  try {
    // 数値 ID の場合
    if (/^\d+$/.test(id)) {
      const response = await fetch(
        `${AZURE_API_BASE}/releasecommunications/${id}`,
        {
          headers: {
            'Accept': 'application/json',
          },
          next: {
            revalidate: 3600, // 1時間キャッシュ
          },
        }
      );

      if (!response.ok) {
        if (response.status === 404) {
          return null;
        }
        throw new Error(`API error: ${response.status}`);
      }

      const data = await response.json();
      
      return {
        id: data.id,
        guid: data.guid,
        title: data.title,
        description: data.description,
        impactDescription: data.impactDescription,
        publicDisclosureDate: data.publicDisclosureDate,
        tags: data.tags || [],
        detailsUrl: `https://admin.microsoft.com/adminportal/home#/MessageCenter/${data.guid}`,
      };
    }
    
    // GUID の場合は別の処理（簡略化のため省略）
    return null;
  } catch (error) {
    console.error('Failed to fetch Azure update:', error);
    throw error;
  }
}
```

### 2. レート制限の実装

`src/lib/rateLimiter.ts`:

```ts
interface RateLimitEntry {
  count: number;
  resetTime: number;
}

class RateLimiter {
  private limits = new Map<string, RateLimitEntry>();
  private readonly maxRequests: number;
  private readonly windowMs: number;

  constructor(maxRequests = 10, windowMs = 60000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
  }

  check(identifier: string): { allowed: boolean; remaining: number } {
    const now = Date.now();
    const entry = this.limits.get(identifier);

    if (!entry || now > entry.resetTime) {
      this.limits.set(identifier, {
        count: 1,
        resetTime: now + this.windowMs,
      });
      return { allowed: true, remaining: this.maxRequests - 1 };
    }

    if (entry.count >= this.maxRequests) {
      return { allowed: false, remaining: 0 };
    }

    entry.count++;
    return { allowed: true, remaining: this.maxRequests - entry.count };
  }

  // 古いエントリをクリーンアップ
  cleanup() {
    const now = Date.now();
    for (const [key, entry] of this.limits.entries()) {
      if (now > entry.resetTime) {
        this.limits.delete(key);
      }
    }
  }
}

export const rateLimiter = new RateLimiter();

// 定期的なクリーンアップ
if (typeof window === 'undefined') {
  setInterval(() => rateLimiter.cleanup(), 60000);
}
```

### 3. リフレッシュ API の実装

`src/app/api/refresh/route.ts`:

```ts
import { NextRequest, NextResponse } from 'next/server';
import { fetchAzureUpdate } from '@/lib/azureFetch';
import { rateLimiter } from '@/lib/rateLimiter';

export async function GET(request: NextRequest) {
  try {
    // レート制限チェック
    const ip = request.headers.get('x-forwarded-for') || 'unknown';
    const { allowed, remaining } = rateLimiter.check(ip);
    
    if (!allowed) {
      return NextResponse.json(
        { error: 'Rate limit exceeded' },
        { 
          status: 429,
          headers: {
            'X-RateLimit-Remaining': '0',
            'Retry-After': '60',
          },
        }
      );
    }

    // クエリパラメータから ID を取得
    const searchParams = request.nextUrl.searchParams;
    const id = searchParams.get('id');

    if (!id) {
      return NextResponse.json(
        { error: 'ID parameter is required' },
        { status: 400 }
      );
    }

    // Azure Update を取得
    const update = await fetchAzureUpdate(id);

    if (!update) {
      return NextResponse.json(
        { error: 'Update not found' },
        { status: 404 }
      );
    }

    return NextResponse.json(
      { 
        success: true,
        data: update,
        cached: false,
      },
      {
        headers: {
          'X-RateLimit-Remaining': remaining.toString(),
          'Cache-Control': 'no-cache',
        },
      }
    );

  } catch (error) {
    console.error('Refresh API error:', error);
    
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### 4. URL キャッシュ API の実装

`src/app/api/cache-url/route.ts`:

```ts
import { NextRequest, NextResponse } from 'next/server';
import crypto from 'crypto';

// 簡単なメモリキャッシュ（本番環境では Redis など使用）
const urlCache = new Map<string, { shortId: string; createdAt: number }>();

function generateShortId(url: string): string {
  const hash = crypto.createHash('sha256').update(url).digest('hex');
  return hash.substring(0, 8);
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { url } = body;

    if (!url || typeof url !== 'string') {
      return NextResponse.json(
        { error: 'Valid URL is required' },
        { status: 400 }
      );
    }

    // URL の検証
    try {
      new URL(url);
    } catch {
      return NextResponse.json(
        { error: 'Invalid URL format' },
        { status: 400 }
      );
    }

    // キャッシュをチェック
    for (const [cachedUrl, data] of urlCache.entries()) {
      if (cachedUrl === url) {
        return NextResponse.json({
          shortId: data.shortId,
          cached: true,
        });
      }
    }

    // 新しい短縮 ID を生成
    const shortId = generateShortId(url);
    urlCache.set(url, {
      shortId,
      createdAt: Date.now(),
    });

    // 古いエントリをクリーンアップ（15分以上前）
    const fifteenMinutesAgo = Date.now() - 15 * 60 * 1000;
    for (const [cachedUrl, data] of urlCache.entries()) {
      if (data.createdAt < fifteenMinutesAgo) {
        urlCache.delete(cachedUrl);
      }
    }

    return NextResponse.json({
      shortId,
      cached: false,
    });

  } catch (error) {
    console.error('Cache URL API error:', error);
    
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### 5. API クライアントフック（Client Component 用）

`src/hooks/useAzureUpdate.ts`:

```ts
'use client';

import { useState, useEffect } from 'react';
import { AzureUpdate } from '@/lib/azureFetch';

export function useAzureUpdate(id: string) {
  const [data, setData] = useState<AzureUpdate | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchData() {
      try {
        setLoading(true);
        const response = await fetch(`/api/refresh?id=${id}`);
        
        if (!response.ok) {
          throw new Error(`Failed to fetch: ${response.status}`);
        }

        const result = await response.json();
        setData(result.data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    }

    if (id) {
      fetchData();
    }
  }, [id]);

  return { data, loading, error };
}
```

## 📚 重要な概念

### Route Handlers の設計思想と Request/Response の仕組み

#### NextRequest と NextResponse の拡張機能

```ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  // NextRequest は標準の Request オブジェクトを拡張
  
  // 1. URLSearchParams へのアクセス
  const searchParams = request.nextUrl.searchParams;
  const id = searchParams.get('id');
  const page = searchParams.get('page') || '1';
  
  // 2. Cookie へのアクセス
  const sessionToken = request.cookies.get('session-token')?.value;
  
  // 3. 地理的情報（Vercel でホストする場合）
  const country = request.geo?.country;
  const city = request.geo?.city;
  
  // 4. IP アドレスの取得
  const ip = request.headers.get('x-forwarded-for') || 
             request.headers.get('x-real-ip') ||
             request.ip ||
             'unknown';
  
  // 5. User-Agent のパース
  const userAgent = request.headers.get('user-agent') || '';
  const isMobile = /Mobile|Android|iPhone/i.test(userAgent);
  
  return NextResponse.json({ 
    id, 
    page: parseInt(page),
    country,
    isMobile 
  });
}

export async function POST(request: NextRequest) {
  // リクエストボディの処理方法
  
  // JSON の場合
  const jsonData = await request.json();
  
  // FormData の場合
  const formData = await request.formData();
  const file = formData.get('file') as File;
  
  // テキストの場合
  const textData = await request.text();
  
  // ArrayBuffer の場合（バイナリデータ）
  const bufferData = await request.arrayBuffer();
  
  return NextResponse.json({ success: true });
}
```

#### NextResponse の高度な機能

```ts
export async function GET(request: NextRequest) {
  const data = await fetchData();
  
  // 基本的なJSONレスポンス
  const basicResponse = NextResponse.json(data);
  
  // カスタムヘッダー付きレスポンス
  const responseWithHeaders = NextResponse.json(data, {
    status: 200,
    headers: {
      'Cache-Control': 'public, s-maxage=3600, stale-while-revalidate=86400',
      'X-Custom-Header': 'custom-value',
      'Access-Control-Allow-Origin': '*',
    },
  });
  
  // Cookie を設定するレスポンス
  const responseWithCookie = NextResponse.json(data);
  responseWithCookie.cookies.set('user-preference', 'dark-mode', {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7, // 1週間
  });
  
  // リダイレクトレスポンス
  const redirectResponse = NextResponse.redirect(
    new URL('/new-path', request.url),
    { status: 307 } // 一時的なリダイレクト
  );
  
  // Rewrite（内部的に別のパスを表示）
  const rewriteResponse = NextResponse.rewrite(
    new URL('/internal-path', request.url)
  );
  
  return responseWithHeaders;
}
```

### Edge Runtime vs Node.js Runtime の詳細比較

#### Edge Runtime の制約と利点

```ts
// route.ts で Runtime を指定
export const runtime = 'edge';

// ✅ Edge Runtime で利用可能
export async function GET(request: NextRequest) {
  // Web標準API
  const url = new URL(request.url);
  const headers = new Headers();
  
  // Crypto API
  const uuid = crypto.randomUUID();
  const hash = await crypto.subtle.digest('SHA-256', 
    new TextEncoder().encode('data')
  );
  
  // Fetch API
  const response = await fetch('https://api.example.com/data', {
    headers: {
      'Authorization': `Bearer ${process.env.API_KEY}`,
    },
  });
  
  // 軽量な JavaScript 処理
  const data = await response.json();
  const processed = data.map(item => ({
    ...item,
    processedAt: new Date().toISOString(),
  }));
  
  return NextResponse.json(processed);
}

// ❌ Edge Runtime で利用不可能な例
/*
import fs from 'fs'; // Node.js ファイルシステム
import { createConnection } from 'mysql2'; // ネイティブデータベース接続
import sharp from 'sharp'; // 重いImageライブラリ

export async function POST(request: NextRequest) {
  // これらはエラーになる
  const fileContent = fs.readFileSync('./data.json'); // ❌
  const connection = createConnection({ ... }); // ❌
  const image = sharp(buffer); // ❌
}
*/
```

#### Node.js Runtime のフル機能

```ts
// route.ts
export const runtime = 'nodejs'; // デフォルト

import fs from 'fs/promises';
import path from 'path';
import { createHash } from 'crypto';

export async function GET(request: NextRequest) {
  // ファイルシステムアクセス
  const filePath = path.join(process.cwd(), 'data', 'cache.json');
  const fileExists = await fs.access(filePath).then(() => true).catch(() => false);
  
  if (fileExists) {
    const content = await fs.readFile(filePath, 'utf-8');
    return NextResponse.json(JSON.parse(content));
  }
  
  // データベース接続（例：Prisma）
  const prisma = new PrismaClient();
  const data = await prisma.user.findMany();
  
  // 重い処理
  const processedData = await heavyComputation(data);
  
  // ファイルにキャッシュ
  await fs.writeFile(filePath, JSON.stringify(processedData));
  
  return NextResponse.json(processedData);
}

export async function POST(request: NextRequest) {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  
  if (file) {
    // ファイルの保存
    const buffer = await file.arrayBuffer();
    const filename = `${Date.now()}-${file.name}`;
    const uploadPath = path.join(process.cwd(), 'public', 'uploads', filename);
    
    await fs.writeFile(uploadPath, Buffer.from(buffer));
    
    return NextResponse.json({
      message: 'File uploaded successfully',
      filename,
      path: `/uploads/${filename}`,
    });
  }
  
  return NextResponse.json({ error: 'No file provided' }, { status: 400 });
}
```

### ストリーミングレスポンスの詳細実装

#### Server-Sent Events (SSE) の実装

```ts
export async function GET(request: NextRequest) {
  const encoder = new TextEncoder();
  
  const stream = new ReadableStream({
    async start(controller) {
      // SSE ヘッダーの送信
      const sendEvent = (data: any, event?: string) => {
        const formatted = `${event ? `event: ${event}\n` : ''}data: ${JSON.stringify(data)}\n\n`;
        controller.enqueue(encoder.encode(formatted));
      };
      
      // 初期データ送信
      sendEvent({ message: 'Connection established', timestamp: Date.now() }, 'connect');
      
      // 定期的なデータ送信
      let counter = 0;
      const interval = setInterval(() => {
        sendEvent({
          counter: ++counter,
          timestamp: Date.now(),
          randomValue: Math.random(),
        }, 'update');
        
        // 10回で終了
        if (counter >= 10) {
          clearInterval(interval);
          sendEvent({ message: 'Stream completed' }, 'complete');
          controller.close();
        }
      }, 1000);
      
      // クライアントが接続を切断した場合のクリーンアップ
      request.signal.addEventListener('abort', () => {
        clearInterval(interval);
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'Access-Control-Allow-Origin': '*',
    },
  });
}
```

#### リアルタイムデータのストリーミング

```ts
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const topic = searchParams.get('topic') || 'general';
  
  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();
      
      // 外部APIからのデータを定期的に取得してストリーム
      const fetchAndStream = async () => {
        try {
          const response = await fetch(`https://api.example.com/live/${topic}`);
          const data = await response.json();
          
          controller.enqueue(encoder.encode(
            `data: ${JSON.stringify(data)}\n\n`
          ));
        } catch (error) {
          controller.enqueue(encoder.encode(
            `data: ${JSON.stringify({ error: 'Failed to fetch data' })}\n\n`
          ));
        }
      };
      
      // 即座に最初のデータを送信
      await fetchAndStream();
      
      // 5秒ごとに更新
      const interval = setInterval(fetchAndStream, 5000);
      
      // クリーンアップ
      request.signal.addEventListener('abort', () => {
        clearInterval(interval);
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### 高度なキャッシュ制御戦略

#### 多層キャッシュ制御

```ts
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const id = searchParams.get('id');
  const forceRefresh = searchParams.get('refresh') === 'true';
  
  // 1. メモリキャッシュ
  if (!forceRefresh && memoryCache.has(id)) {
    const cached = memoryCache.get(id);
    if (cached.expiresAt > Date.now()) {
      return NextResponse.json(cached.data, {
        headers: {
          'X-Cache': 'HIT-MEMORY',
          'Cache-Control': 'public, max-age=60',
        },
      });
    }
  }
  
  // 2. データベースキャッシュ
  if (!forceRefresh) {
    const dbCached = await getCachedFromDatabase(id);
    if (dbCached && dbCached.expiresAt > new Date()) {
      // メモリキャッシュにも保存
      memoryCache.set(id, {
        data: dbCached.data,
        expiresAt: dbCached.expiresAt.getTime(),
      });
      
      return NextResponse.json(dbCached.data, {
        headers: {
          'X-Cache': 'HIT-DATABASE',
          'Cache-Control': 'public, max-age=300',
        },
      });
    }
  }
  
  // 3. 外部APIから取得
  const freshData = await fetchFromExternalAPI(id);
  
  // キャッシュに保存
  const expiresAt = new Date(Date.now() + 3600000); // 1時間後
  await saveToDatabaseCache(id, freshData, expiresAt);
  memoryCache.set(id, {
    data: freshData,
    expiresAt: expiresAt.getTime(),
  });
  
  return NextResponse.json(freshData, {
    headers: {
      'X-Cache': 'MISS',
      'Cache-Control': 'public, max-age=3600, stale-while-revalidate=86400',
    },
  });
}
```

#### 条件付きキャッシュ

```ts
export async function GET(request: NextRequest) {
  const ifNoneMatch = request.headers.get('if-none-match');
  const ifModifiedSince = request.headers.get('if-modified-since');
  
  const data = await fetchData();
  
  // ETag の生成
  const etag = `"${createHash('md5').update(JSON.stringify(data)).digest('hex')}"`;
  
  // 304 Not Modified の判定
  if (ifNoneMatch === etag) {
    return new Response(null, {
      status: 304,
      headers: {
        'ETag': etag,
        'Cache-Control': 'public, max-age=3600',
      },
    });
  }
  
  const lastModified = new Date(data.updatedAt).toUTCString();
  
  if (ifModifiedSince && new Date(ifModifiedSince) >= new Date(data.updatedAt)) {
    return new Response(null, {
      status: 304,
      headers: {
        'Last-Modified': lastModified,
        'Cache-Control': 'public, max-age=3600',
      },
    });
  }
  
  // 通常のレスポンス
  return NextResponse.json(data, {
    headers: {
      'ETag': etag,
      'Last-Modified': lastModified,
      'Cache-Control': 'public, max-age=3600',
      'Vary': 'Accept-Encoding',
    },
  });
}
```

## ✅ 確認ポイント

- [ ] `/api/refresh?id=123456` が正しく動作する
- [ ] レート制限が機能する
- [ ] エラーハンドリングが適切
- [ ] レスポンスヘッダーが正しく設定される

## 🏃 演習問題

1. **バリデーションの強化**: Zod を使ってリクエストボディの検証を実装
2. **キャッシュヘッダーの改善**: 条件に応じて異なるキャッシュ戦略を適用
3. **Webhook エンドポイント**: 更新通知を受け取る Webhook エンドポイントを作成

### 演習 1 の解答例

```ts
import { z } from 'zod';

const cacheUrlSchema = z.object({
  url: z.string().url(),
});

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const validated = cacheUrlSchema.parse(body);
    // ... 残りの処理
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Invalid input', details: error.errors },
        { status: 400 }
      );
    }
    // ... その他のエラー処理
  }
}
```

## 🔗 参考リンク

- [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Request/Response APIs](https://nextjs.org/docs/app/api-reference/functions/next-request)
- [Edge Runtime](https://nextjs.org/docs/app/api-reference/edge)

---

準備ができたら、[Step 4: データフェッチングとキャッシング](../step-04-data-fetching) に進みましょう！