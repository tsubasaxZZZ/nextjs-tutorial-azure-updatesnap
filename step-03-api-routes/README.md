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

### Edge Runtime vs Node.js Runtime

```ts
// Edge Runtime（高速、制限あり）
export const runtime = 'edge';

// Node.js Runtime（フル機能）
export const runtime = 'nodejs'; // デフォルト
```

### レスポンスヘッダーの制御

```ts
return NextResponse.json(data, {
  headers: {
    'Cache-Control': 'public, s-maxage=60, stale-while-revalidate',
    'X-Custom-Header': 'value',
  },
});
```

### ストリーミングレスポンス

```ts
export async function GET() {
  const stream = new ReadableStream({
    async start(controller) {
      controller.enqueue('data: Hello\n\n');
      await new Promise(resolve => setTimeout(resolve, 1000));
      controller.enqueue('data: World\n\n');
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
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