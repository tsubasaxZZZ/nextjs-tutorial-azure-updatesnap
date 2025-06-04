# Step 3: API Routes Implementation - Mastering Modern Server-Side Development

このステップでは、現代的なAPI設計の思想と Next.js の Route Handlers を使った実装を深く学びます。

## 🎯 このステップで学ぶこと

- **API発展の歴史**: REST → GraphQL → 現代的なアプローチ
- **Route Handlers**: Next.js 14 の革新的な API 設計
- **Microsoft API 統合**: エンタープライズ API との連携パターン
- **レート制限システム**: 本格的なプロダクション対応
- **セキュリティ**: エンタープライズアプリケーション向け設計
- **パフォーマンス**: Edge Computing と最適化戦略

## 📖 現代的 API 開発の完全ガイド

### API アーキテクチャの進化史

#### 第一世代：SOAP と XML-RPC (2000年代前半)
```xml
<!-- SOAP の例：冗長で複雑 -->
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetUserRequest>
      <UserId>12345</UserId>
    </GetUserRequest>
  </soap:Body>
</soap:Envelope>
```

SOAPは厳密な契約ベースの設計でしたが、オーバーヘッドが大きく、Web開発には適さないことが判明しました。

#### 第二世代：RESTful APIs (2000年代後半〜2010年代)
```ts
// REST の設計思想
GET    /api/users/123      // ユーザー取得
POST   /api/users          // ユーザー作成
PUT    /api/users/123      // ユーザー更新
DELETE /api/users/123      // ユーザー削除
```

RESTは **リソース志向** の設計で、HTTPの標準的な動詞を活用。シンプルで理解しやすい一方、複雑なデータ取得では N+1 問題や over-fetching が課題となりました。

#### 第三世代：GraphQL の台頭 (2010年代中期〜)
```graphql
# GraphQL の例：必要なデータだけを指定
query GetUserWithPosts($id: ID!) {
  user(id: $id) {
    name
    email
    posts(limit: 5) {
      title
      publishedAt
    }
  }
}
```

Facebookが開発したGraphQLは、**クエリ言語** として柔軟なデータ取得を可能にしましたが、学習コストとキャッシュの複雑さが問題となりました。

#### 第四世代：Next.js Route Handlers の革新 (2020年代)
```ts
// 現代的なアプローチ：型安全性 + エッジ最適化 + ストリーミング
export async function GET(request: NextRequest) {
  const data = await fetchData();
  return NextResponse.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=3600, stale-while-revalidate=86400',
    },
  });
}
```

Next.js Route Handlers は：
- **型安全性**：TypeScript ネイティブ
- **エッジ最適化**：世界中のCDNで実行
- **ストリーミング**：大量データの効率的配信
- **Web Standards**：標準的なAPIに準拠

### Route Handlers の革命的な設計思想

#### ファイルシステムベースルーティングの力

```
app/
├── api/
│   ├── users/
│   │   ├── route.ts                    # /api/users
│   │   ├── [id]/
│   │   │   ├── route.ts               # /api/users/[id]
│   │   │   └── posts/
│   │   │       └── route.ts           # /api/users/[id]/posts
│   │   └── bulk/
│   │       └── route.ts               # /api/users/bulk
│   ├── azure/
│   │   ├── updates/
│   │   │   ├── route.ts               # /api/azure/updates
│   │   │   └── [id]/
│   │   │       └── route.ts           # /api/azure/updates/[id]
│   │   └── health/
│   │       └── route.ts               # /api/azure/health
│   └── internal/
│       ├── cache-flush/
│       │   └── route.ts               # /api/internal/cache-flush
│       └── metrics/
│           └── route.ts               # /api/internal/metrics
```

この設計の利点：
- **直感的**: URLとファイル構造が一致
- **型安全**: 各ルートで適切な型定義
- **分散開発**: チームが独立してAPI開発可能
- **保守性**: 機能ごとに分離された責務

#### HTTP メソッドベースの関数エクスポート

```ts
// 各HTTPメソッドが独立した関数として定義される
export async function GET(request: NextRequest) {
  // データ取得ロジック
}

export async function POST(request: NextRequest) {
  // データ作成ロジック
}

export async function PUT(request: NextRequest) {
  // データ更新ロジック
}

export async function DELETE(request: NextRequest) {
  // データ削除ロジック
}

export async function PATCH(request: NextRequest) {
  // 部分的データ更新ロジック
}

export async function HEAD(request: NextRequest) {
  // メタデータのみ取得
}

export async function OPTIONS(request: NextRequest) {
  // CORS プリフライトレスポンス
}
```

従来のフレームワークとの比較：

**Express.js (従来型)**:
```ts
app.get('/api/users/:id', (req, res) => {
  // ルーティングとハンドラーが分離
});
app.post('/api/users/:id', (req, res) => {
  // 同じパスでもメソッドごとに分離
});
```

**Next.js Route Handlers (現代型)**:
```ts
// 1つのファイルで全メソッドを管理
// TypeScript の恩恵を最大限活用
export async function GET(request: NextRequest, 
  { params }: { params: { id: string } }
) {
  // 型安全性とエディタサポート
}
```

### Microsoft Azure 統合アーキテクチャ

#### エンタープライズ API との連携パターン

Azure UpdateSnap では、Microsoft の公式 API との統合が重要な要素です。エンタープライズ級のAPI統合には以下の考慮点があります：

**1. API契約の安定性**
```ts
// Microsoft API のエンドポイント設計
const API_ENDPOINTS = {
  // v2 API：安定したバージョン
  releaseCommunications: 'https://techcommunity.microsoft.com/plugins/custom/microsoft/o365/api-v2/releasecommunications',
  
  // フォールバック用 v1
  fallback: 'https://techcommunity.microsoft.com/plugins/custom/microsoft/o365/api/releasecommunications',
} as const;
```

**2. レスポンス形式の変化への対応**
```ts
// 型安全なレスポンス処理
interface MicrosoftApiResponse {
  id: number;
  guid: string;
  title: string;
  description: string;
  impactDescription?: string;
  publicDisclosureDate: string;
  tags?: string[];
  // Microsoft が将来追加する可能性のあるフィールド
  [key: string]: unknown;
}
```

**3. 認証とレート制限**
```ts
// Microsoft API の利用制限に対応
const API_CONFIG = {
  maxRetries: 3,
  baseDelay: 1000,
  maxDelay: 5000,
  timeout: 10000,
} as const;
```

#### レート制限システムの詳細設計

現代的なWebアプリケーションでは、レート制限は必須の機能です。特にエンタープライズ環境では、以下の観点が重要：

**1. 分散システムでのレート制限**
```ts
// Redis ベースの分散レート制限
class DistributedRateLimiter {
  private redis: Redis;
  
  constructor(redis: Redis) {
    this.redis = redis;
  }
  
  async checkLimit(key: string, limit: number, window: number): Promise<{
    allowed: boolean;
    remaining: number;
    resetTime: number;
  }> {
    const pipeline = this.redis.pipeline();
    const now = Date.now();
    const windowStart = now - window;
    
    // スライディングウィンドウアルゴリズム
    pipeline.zremrangebyscore(key, 0, windowStart);
    pipeline.zadd(key, now, now);
    pipeline.zcard(key);
    pipeline.expire(key, Math.ceil(window / 1000));
    
    const results = await pipeline.exec();
    const count = results?.[2]?.[1] as number;
    
    return {
      allowed: count <= limit,
      remaining: Math.max(0, limit - count),
      resetTime: now + window,
    };
  }
}
```

**2. トークンバケットアルゴリズム**
```ts
// より柔軟なレート制限
class TokenBucketRateLimiter {
  private buckets = new Map<string, TokenBucket>();
  
  check(identifier: string, capacity: number, refillRate: number): boolean {
    let bucket = this.buckets.get(identifier);
    
    if (!bucket) {
      bucket = new TokenBucket(capacity, refillRate);
      this.buckets.set(identifier, bucket);
    }
    
    return bucket.consume();
  }
}

class TokenBucket {
  private tokens: number;
  private lastRefill: number;
  
  constructor(
    private capacity: number,
    private refillRate: number
  ) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }
  
  consume(): boolean {
    this.refill();
    
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    
    return false;
  }
  
  private refill() {
    const now = Date.now();
    const elapsed = now - this.lastRefill;
    const tokensToAdd = (elapsed / 1000) * this.refillRate;
    
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}
```

## 🛠️ 実装：Azure UpdateSnap の API レイヤー

### 1. エンタープライズグレード Microsoft API クライアント

`src/lib/azureFetch.ts`:

```ts
// 企業級の設定管理
const API_CONFIG = {
  baseUrl: 'https://techcommunity.microsoft.com/plugins/custom/microsoft/o365/api-v2',
  timeout: 10000,
  maxRetries: 3,
  retryDelay: 1000,
} as const;

// 厳密な型定義
export interface AzureUpdate {
  readonly id: number;
  readonly guid: string;
  readonly title: string;
  readonly description: string;
  readonly impactDescription?: string;
  readonly publicDisclosureDate: string;
  readonly tags: readonly string[];
  readonly detailsUrl: string;
  readonly fetchedAt: string;
}

// エラー型の定義
export class AzureApiError extends Error {
  constructor(
    message: string,
    public readonly status?: number,
    public readonly endpoint?: string
  ) {
    super(message);
    this.name = 'AzureApiError';
  }
}

// 指数バックオフによるリトライ機能
async function fetchWithRetry(
  url: string,
  options: RequestInit,
  retries = API_CONFIG.maxRetries
): Promise<Response> {
  try {
    const response = await fetch(url, {
      ...options,
      signal: AbortSignal.timeout(API_CONFIG.timeout),
    });
    
    if (!response.ok && retries > 0) {
      // 5xx エラーの場合のみリトライ
      if (response.status >= 500) {
        await new Promise(resolve => 
          setTimeout(resolve, API_CONFIG.retryDelay * (API_CONFIG.maxRetries - retries + 1))
        );
        return fetchWithRetry(url, options, retries - 1);
      }
    }
    
    return response;
  } catch (error) {
    if (retries > 0 && (error as Error).name !== 'AbortError') {
      await new Promise(resolve => 
        setTimeout(resolve, API_CONFIG.retryDelay * (API_CONFIG.maxRetries - retries + 1))
      );
      return fetchWithRetry(url, options, retries - 1);
    }
    throw error;
  }
}

export async function fetchAzureUpdate(id: string): Promise<AzureUpdate | null> {
  try {
    let endpoint: string;
    
    if (/^\d+$/.test(id)) {
      endpoint = `${API_CONFIG.baseUrl}/releasecommunications/${id}`;
    } else if (/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(id)) {
      endpoint = `${API_CONFIG.baseUrl}/releasecommunications/guid/${id}`;
    } else {
      throw new AzureApiError(`Invalid ID format: ${id}`);
    }
    
    const response = await fetchWithRetry(endpoint, {
      headers: {
        'Accept': 'application/json',
        'User-Agent': 'Azure-UpdateSnap/1.0',
      },
      next: {
        revalidate: 3600,
      },
    });

    if (!response.ok) {
      if (response.status === 404) {
        return null;
      }
      throw new AzureApiError(
        `API request failed: ${response.statusText}`,
        response.status,
        endpoint
      );
    }

    const data = await response.json();
    
    return {
      id: data.id,
      guid: data.guid,
      title: data.title || 'Untitled Update',
      description: data.description || '',
      impactDescription: data.impactDescription,
      publicDisclosureDate: data.publicDisclosureDate,
      tags: Object.freeze(data.tags || []),
      detailsUrl: `https://admin.microsoft.com/adminportal/home#/MessageCenter/${data.guid}`,
      fetchedAt: new Date().toISOString(),
    };
    
  } catch (error) {
    if (error instanceof AzureApiError) {
      throw error;
    }
    
    console.error('Unexpected error fetching Azure update:', error);
    throw new AzureApiError('Failed to fetch Azure update');
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

## 📚 現代的 API 開発の核心概念

### エンタープライズレベルのセキュリティ考慮

#### 1. 入力検証とサニタイゼーション

```ts
import { z } from 'zod';

// 厳密なスキーマ定義
const AzureUpdateIdSchema = z.union([
  z.string().regex(/^\d+$/, 'Must be numeric ID'),
  z.string().uuid('Must be valid GUID'),
]);

export async function validateRequest<T>(
  schema: z.ZodSchema<T>,
  data: unknown
): Promise<T> {
  try {
    return schema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new ValidationError('Invalid input', error.errors);
    }
    throw error;
  }
}
```

#### 2. セキュリティヘッダーの実装

```ts
function setSecurityHeaders(response: NextResponse): NextResponse {
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-XSS-Protection', '1; mode=block');
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  response.headers.set(
    'Content-Security-Policy',
    \"default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'\"\n  );
  
  return response;
}
```

#### 3. CORS の適切な設定

```ts
export async function OPTIONS(request: NextRequest) {
  const origin = request.headers.get('origin');
  const allowedOrigins = [
    'https://admin.microsoft.com',
    'https://portal.azure.com',
    process.env.NEXT_PUBLIC_BASE_URL,
  ].filter(Boolean);
  
  const corsHeaders: Record<string, string> = {
    'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization, X-Requested-With',
    'Access-Control-Max-Age': '86400',
  };
  
  if (origin && allowedOrigins.includes(origin)) {
    corsHeaders['Access-Control-Allow-Origin'] = origin;\n  }
  
  return new Response(null, {\n    status: 200,\n    headers: corsHeaders,\n  });\n}
```

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

### Azure UpdateSnap での実装戦略

Azure UpdateSnap では、以下の戦略で API レイヤーを設計しています：

1. **多層防御**: クライアント側 → Edge Middleware → API Handler → External API
2. **段階的フォールバック**: API v2 → API v1 → HTML Scraping → Cache
3. **監視と可観測性**: メトリクス、ログ、アラート
4. **スケーラビリティ**: 水平スケール対応、ステートレス設計

```ts
// 包括的な API アーキテクチャ
export class AzureUpdateApiService {
  private rateLimiter: RateLimiter;
  private cache: CacheService;
  private metrics: MetricsService;
  
  constructor(options: ApiServiceOptions) {
    this.rateLimiter = new RateLimiter(options.rateLimit);
    this.cache = new CacheService(options.cache);
    this.metrics = new MetricsService(options.metrics);
  }
  
  async getUpdate(id: string, request: NextRequest): Promise<AzureUpdate | null> {
    const startTime = Date.now();
    
    try {
      // 1. レート制限チェック
      await this.rateLimiter.checkLimit(this.getClientId(request));
      
      // 2. キャッシュ確認
      const cached = await this.cache.get(id);
      if (cached) {
        this.metrics.recordCacheHit(id);
        return cached;
      }
      
      // 3. 外部API呼び出し
      const update = await this.fetchFromExternalApi(id);
      
      // 4. キャッシュに保存
      if (update) {
        await this.cache.set(id, update);
      }
      
      this.metrics.recordApiCall(id, Date.now() - startTime);
      return update;
      
    } catch (error) {
      this.metrics.recordError(id, error);
      throw error;
    }
  }
}
```

## ✅ 実装確認チェックリスト

### 基本機能
- [ ] Microsoft API との正常な通信が確立されている
- [ ] エラーハンドリングが適切に実装されている
- [ ] 型安全性が保たれている
- [ ] レスポンス時間が 2 秒以内

### セキュリティ
- [ ] 入力検証が実装されている
- [ ] セキュリティヘッダーが設定されている
- [ ] CORS が適切に設定されている
- [ ] レート制限が機能している

### パフォーマンス
- [ ] キャッシュヘッダーが適切に設定されている
- [ ] コネクションプールが使用されている
- [ ] タイムアウトが設定されている
- [ ] リトライロジックが実装されている

### 監視・運用
- [ ] ログが適切に出力されている
- [ ] メトリクスが収集されている
- [ ] ヘルスチェックエンドポイントがある
- [ ] エラー率の監視ができている

## 🏃 実践的演習問題

### 演習 1: 高度なバリデーション実装
Azure の更新データに特化したバリデーションシステムを実装してください。

```ts
import { z } from 'zod';

const AzureUpdateRequestSchema = z.object({
  id: z.union([
    z.string().regex(/^\d+$/, 'Must be numeric ID'),
    z.string().uuid('Must be valid GUID')
  ]),
  includeMetadata: z.boolean().optional().default(false),
  format: z.enum(['json', 'xml', 'text']).optional().default('json'),
  locale: z.string().regex(/^[a-z]{2}-[A-Z]{2}$/).optional().default('en-US'),
});

// 実装例
export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams;
    const validated = AzureUpdateRequestSchema.parse({
      id: searchParams.get('id'),
      includeMetadata: searchParams.get('metadata') === 'true',
      format: searchParams.get('format'),
      locale: searchParams.get('locale'),
    });
    
    // 検証されたデータで処理を続行
    const update = await fetchAzureUpdate(validated.id, validated);
    
    return NextResponse.json(update);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        {
          error: 'Validation failed',
          details: error.errors.map(err => ({
            field: err.path.join('.'),
            message: err.message,
            received: err.received,
          })),
        },
        { status: 400 }
      );
    }
    
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### 演習 2: 動的キャッシュ戦略
コンテンツの重要度や更新頻度に応じて異なるキャッシュ戦略を適用する実装。

```ts
function getCacheStrategy(update: AzureUpdate): {
  ttl: number;
  cacheControl: string;
} {
  const isHighPriority = update.tags.some(tag => 
    ['Security', 'Critical', 'Breaking Change'].includes(tag)
  );
  
  const isRecent = new Date(update.publicDisclosureDate) > 
    new Date(Date.now() - 7 * 24 * 60 * 60 * 1000); // 7日以内
  
  if (isHighPriority && isRecent) {
    return {
      ttl: 300, // 5分
      cacheControl: 'public, max-age=300, must-revalidate',
    };
  } else if (isRecent) {
    return {
      ttl: 1800, // 30分
      cacheControl: 'public, max-age=1800, stale-while-revalidate=3600',
    };
  } else {
    return {
      ttl: 86400, // 24時間
      cacheControl: 'public, max-age=86400, stale-while-revalidate=604800',
    };
  }
}
```

### 演習 3: Webhook 受信エンドポイント
Microsoft からのリアルタイム更新通知を受信する secure Webhook。

```ts
import crypto from 'crypto';

export async function POST(request: NextRequest) {
  try {
    // Webhook 署名の検証
    const signature = request.headers.get('x-ms-signature-256');
    const body = await request.text();
    
    if (!verifyWebhookSignature(signature, body)) {
      return NextResponse.json(
        { error: 'Invalid signature' },
        { status: 401 }
      );
    }
    
    const payload = JSON.parse(body);
    
    // 更新タイプに応じた処理
    switch (payload.eventType) {
      case 'MessageCenter.NewMessage':
        await handleNewMessage(payload.data);
        break;
      case 'MessageCenter.UpdateMessage':
        await handleUpdateMessage(payload.data);
        break;
      default:
        console.log('Unknown event type:', payload.eventType);
    }
    
    return NextResponse.json({ status: 'processed' });
    
  } catch (error) {
    console.error('Webhook processing error:', error);
    return NextResponse.json(
      { error: 'Processing failed' },
      { status: 500 }
    );
  }
}

function verifyWebhookSignature(signature: string | null, body: string): boolean {
  if (!signature || !process.env.WEBHOOK_SECRET) {
    return false;
  }
  
  const expectedSignature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(body)
    .digest('hex');
    
  return crypto.timingSafeEqual(
    Buffer.from(signature.replace('sha256=', '')),
    Buffer.from(expectedSignature)
  );
}
```

## 🔗 詳細リソース

### 公式ドキュメント
- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Edge Runtime API](https://nextjs.org/docs/app/api-reference/edge)
- [Microsoft Graph API](https://docs.microsoft.com/en-us/graph/)

### 学習リソース
- [RESTful API Design Best Practices](https://restfulapi.net/)
- [HTTP Status Codes Reference](https://httpstatuses.com/)
- [Web Security Fundamentals](https://web.dev/security/)

### Azure UpdateSnap 関連
- [Project Architecture](../../CLAUDE.md)
- [API Testing Guide](../testing/api-testing.md)
- [Deployment Checklist](../deployment/checklist.md)

---

**準備完了！** 強固な API 基盤が構築できました。次は [Step 4: データフェッチングとキャッシング](../step-04-data-fetching) でデータレイヤーを実装します。