# Step 2: ルーティングとページ作成

このステップでは、Next.js App Router の動的ルーティングを実装します。

## 🎯 このステップで学ぶこと

- 動的ルーティング（`[slug]`）の実装
- パラメータの取得と型定義
- not-found ページの実装
- ミドルウェアの基本
- リダイレクトの実装

## 📖 解説

### 動的ルーティング

Azure UpdateSnap では、以下の URL パターンをサポート：
- `/123456` - 数値 ID
- `/a1b2c3d4-e5f6-7890-abcd-ef1234567890` - GUID

これを実現するために `[slug]` という動的セグメントを使用します。

### ファイル構造

```
app/
├── [slug]/
│   ├── page.tsx          # 動的ページ
│   ├── loading.tsx       # ローディング UI
│   └── error.tsx         # エラー UI
├── api/                  # API ルート（次のステップ）
├── layout.tsx
├── page.tsx
└── not-found.tsx         # 404 ページ
```

## 🛠️ ハンズオン

### 1. 動的ルートの作成

`src/app/[slug]/page.tsx` を作成：

```tsx
type Props = {
  params: Promise<{ slug: string }>;
};

export default async function UpdatePage({ params }: Props) {
  const { slug } = await params;
  
  // スラッグの検証
  const isValidId = /^\d+$/.test(slug) || 
    /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(slug);
  
  if (!isValidId) {
    return (
      <main className="min-h-screen p-8">
        <h1 className="text-2xl font-bold text-red-600">
          無効な ID です
        </h1>
        <p className="mt-4">
          指定された ID: {slug}
        </p>
      </main>
    );
  }

  return (
    <main className="min-h-screen p-8">
      <h1 className="text-3xl font-bold mb-4">
        Azure Update: {slug}
      </h1>
      <p className="text-gray-600">
        この ID の更新情報を表示します（Step 4 で実装）
      </p>
    </main>
  );
}
```

### 2. ローディング UI の追加

`src/app/[slug]/loading.tsx`:

```tsx
export default function Loading() {
  return (
    <div className="min-h-screen p-8">
      <div className="animate-pulse">
        <div className="h-8 bg-gray-300 rounded w-3/4 mb-4"></div>
        <div className="h-4 bg-gray-300 rounded w-1/2 mb-2"></div>
        <div className="h-4 bg-gray-300 rounded w-2/3"></div>
      </div>
    </div>
  );
}
```

### 3. Not Found ページ

`src/app/not-found.tsx`:

```tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <main className="min-h-screen p-8 flex flex-col items-center justify-center">
      <h1 className="text-4xl font-bold mb-4">404</h1>
      <p className="text-gray-600 mb-8">
        お探しのページが見つかりませんでした
      </p>
      <Link 
        href="/" 
        className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
      >
        ホームに戻る
      </Link>
    </main>
  );
}
```

### 4. リダイレクトコンポーネントの作成

`src/app/[slug]/RedirectingView.tsx`:

```tsx
'use client';

import { useEffect, useState } from 'react';

type Props = {
  targetUrl: string;
};

export default function RedirectingView({ targetUrl }: Props) {
  const [countdown, setCountdown] = useState(3);

  useEffect(() => {
    const timer = setInterval(() => {
      setCountdown((prev) => {
        if (prev <= 1) {
          window.location.href = targetUrl;
          return 0;
        }
        return prev - 1;
      });
    }, 1000);

    return () => clearInterval(timer);
  }, [targetUrl]);

  return (
    <div className="min-h-screen flex flex-col items-center justify-center p-8">
      <h1 className="text-2xl font-bold mb-4">
        Microsoft 公式ページにリダイレクトしています...
      </h1>
      <p className="text-gray-600 mb-8">
        {countdown} 秒後に自動的にリダイレクトされます
      </p>
      <a 
        href={targetUrl}
        className="text-blue-600 hover:underline"
      >
        今すぐリダイレクト →
      </a>
    </div>
  );
}
```

### 5. ミドルウェアの基本実装

`src/middleware.ts`:

```ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // パスの取得
  const pathname = request.nextUrl.pathname;
  
  // User-Agent の確認（Bot 検出の例）
  const userAgent = request.headers.get('user-agent') || '';
  const isBot = /bot|crawler|spider|crawling/i.test(userAgent);
  
  // リクエストヘッダーに情報を追加
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-is-bot', isBot ? '1' : '0');
  
  return NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  });
}

export const config = {
  matcher: '/:path*',
};
```

## 📚 重要な概念

### 動的ルーティングの詳細な仕組み

#### 動的セグメントの種類と使い分け

```tsx
// 1. [slug] - 単一の動的セグメント
// app/blog/[slug]/page.tsx
// マッチ: /blog/hello, /blog/123
// 非マッチ: /blog/hello/world

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  // slug = "hello" または "123"
}

// 2. [...slug] - キャッチオールセグメント
// app/docs/[...slug]/page.tsx
// マッチ: /docs/intro, /docs/guide/setup, /docs/api/v1/users
// 非マッチ: /docs（空のセグメント）

export default async function DocsPage({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params;
  // slug = ["intro"] または ["guide", "setup"] または ["api", "v1", "users"]
}

// 3. [[...slug]] - オプショナルキャッチオールセグメント
// app/shop/[[...slug]]/page.tsx
// マッチ: /shop, /shop/category, /shop/category/product
// すべてのセグメントパターンをキャッチ

export default async function ShopPage({ params }: { params: Promise<{ slug?: string[] }> }) {
  const { slug } = await params;
  // slug = undefined または ["category"] または ["category", "product"]
}
```

#### パラメータの型安全性とバリデーション

```tsx
import { z } from 'zod';

// Zod スキーマでバリデーション
const paramsSchema = z.object({
  slug: z.string().regex(/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i)
    .or(z.string().regex(/^\d+$/))
});

type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

export default async function UpdatePage({ params, searchParams }: Props) {
  const resolvedParams = await params;
  const resolvedSearchParams = await searchParams;
  
  // バリデーション
  const validationResult = paramsSchema.safeParse(resolvedParams);
  
  if (!validationResult.success) {
    throw new Error('Invalid slug format');
  }
  
  const { slug } = validationResult.data;
  const debug = resolvedSearchParams.debug === 'true';
  
  // 型安全な処理
  return <div>Valid slug: {slug}</div>;
}
```

### Client vs Server Components の境界設計

#### ハイドレーションとインタラクティビティ

```tsx
// Server Component (デフォルト)
export default async function UpdatePage({ params }: Props) {
  const { slug } = await params;
  
  // サーバーサイドでデータ取得
  const updateData = await getUpdateData(slug);
  
  return (
    <div>
      {/* Server Component のまま - 静的コンテンツ */}
      <h1>{updateData.title}</h1>
      <p>{updateData.description}</p>
      
      {/* Client Component - インタラクティブな部分のみ */}
      <InteractiveComments 
        initialComments={updateData.comments}
        updateId={slug}
      />
      
      {/* 条件付きで Client Component を使用 */}
      {!isBot && <RedirectingView targetUrl={updateData.officialUrl} />}
    </div>
  );
}

// Client Component - 必要最小限に限定
'use client';

function InteractiveComments({ initialComments, updateId }) {
  const [comments, setComments] = useState(initialComments);
  
  // クライアントサイドの状態管理とイベントハンドリング
  const addComment = (text) => {
    setComments(prev => [...prev, { id: Date.now(), text }]);
  };
  
  return (
    <div>
      {/* インタラクティブなUI */}
    </div>
  );
}
```

#### Suspense との統合

```tsx
import { Suspense } from 'react';

export default function UpdatePage({ params }) {
  return (
    <div>
      {/* 即座に表示される部分 */}
      <header>
        <h1>Azure Update</h1>
      </header>
      
      {/* データロードが必要な部分を Suspense でラップ */}
      <Suspense fallback={<UpdateSkeleton />}>
        <UpdateContent params={params} />
      </Suspense>
      
      {/* 別のデータソースも個別に Suspense */}
      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection params={params} />
      </Suspense>
    </div>
  );
}

// 重いデータフェッチを持つコンポーネント
async function UpdateContent({ params }) {
  const { slug } = await params;
  const data = await slowDataFetch(slug); // 3秒かかる処理
  
  return <div>{data.content}</div>;
}

// 別のデータソース
async function CommentsSection({ params }) {
  const { slug } = await params;
  const comments = await fetchComments(slug); // 1秒かかる処理
  
  return <div>{comments.map(c => <div key={c.id}>{c.text}</div>)}</div>;
}
```

### ミドルウェアの実行タイミングと制約

#### ミドルウェアの実行順序

```tsx
// src/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  console.log('1. ミドルウェア実行'); // Edge Runtime で実行
  
  // 実行順序：
  // 1. ミドルウェア（すべてのリクエストで最初）
  // 2. ルートハンドラー（API routes の場合）
  // 3. Server Components（ページの場合）
  // 4. Client Components のハイドレーション
  
  const pathname = request.nextUrl.pathname;
  
  // Bot 検出ロジック
  const userAgent = request.headers.get('user-agent') || '';
  const isBot = detectBot(userAgent);
  
  // リダイレクト条件
  if (pathname.startsWith('/old-path')) {
    return NextResponse.redirect(new URL('/new-path', request.url));
  }
  
  // ヘッダーの追加
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-is-bot', isBot ? '1' : '0');
  requestHeaders.set('x-pathname', pathname);
  
  // レスポンス ヘッダーの設定
  const response = NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  });
  
  response.headers.set('x-middleware-cache', 'MISS');
  
  return response;
}

function detectBot(userAgent: string): boolean {
  const botPatterns = [
    /googlebot/i,
    /bingbot/i,
    /slurp/i,
    /facebookexternalhit/i,
    /twitterbot/i,
    /linkedinbot/i,
    /whatsapp/i,
  ];
  
  return botPatterns.some(pattern => pattern.test(userAgent));
}

// マッチャーの詳細な設定
export const config = {
  matcher: [
    // API routes を除外
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
    // 特定のパスのみ
    '/updates/:path*',
    // 複数の条件
    {
      source: '/admin/:path*',
      has: [
        {
          type: 'header',
          key: 'authorization',
        }
      ]
    }
  ],
};
```

#### Edge Runtime の制約と利点

```tsx
// Edge Runtime で利用可能な API（制限あり）
export function middleware(request: NextRequest) {
  // ✅ 利用可能
  const url = new URL(request.url);
  const cookies = request.cookies;
  const headers = request.headers;
  
  // ✅ Web APIs
  const encoded = btoa('hello'); // Base64 エンコード
  const uuid = crypto.randomUUID(); // UUID 生成
  
  // ❌ Node.js APIs は利用不可
  // const fs = require('fs'); // エラー
  // const path = require('path'); // エラー
  
  // ✅ Fetch API
  const response = await fetch('https://api.example.com/data');
  
  // ✅ 高速な起動時間（コールドスタート数十ミリ秒）
  // ✅ 世界中のエッジロケーションで実行
  
  return NextResponse.next();
}
```

### Parallel Routes と Intercepting Routes の実用例

#### Parallel Routes - 複数コンテンツの同時レンダリング

```tsx
// app/dashboard/@analytics/page.tsx
export default async function Analytics() {
  const data = await getAnalytics();
  return <div>Analytics: {data}</div>;
}

// app/dashboard/@notifications/page.tsx  
export default async function Notifications() {
  const notifications = await getNotifications();
  return <div>Notifications: {notifications.length}</div>;
}

// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics, // @analytics スロット
  notifications, // @notifications スロット
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  notifications: React.ReactNode;
}) {
  return (
    <div className="dashboard-grid">
      <main>{children}</main>
      <aside className="analytics-panel">
        <Suspense fallback={<AnalyticsSkeleton />}>
          {analytics}
        </Suspense>
      </aside>
      <aside className="notifications-panel">
        <Suspense fallback={<NotificationsSkeleton />}>
          {notifications}
        </Suspense>
      </aside>
    </div>
  );
}
```

#### Intercepting Routes - モーダル表示

```tsx
// app/photos/[id]/page.tsx - 通常のページ
export default function PhotoPage({ params }) {
  return (
    <div className="full-page-photo">
      <img src={`/photos/${params.id}`} alt="Photo" />
    </div>
  );
}

// app/gallery/(..)photos/[id]/page.tsx - インターセプトされたルート
export default function PhotoModal({ params }) {
  return (
    <div className="modal-overlay">
      <div className="modal-content">
        <img src={`/photos/${params.id}`} alt="Photo" />
        <button onClick={() => router.back()}>Close</button>
      </div>
    </div>
  );
}

// gallery ページから写真をクリックした場合：
// - モーダルで表示（Intercepting Route が使用される）
// 直接 URL にアクセスした場合：
// - フルページで表示（通常のページが使用される）
```

## ✅ 確認ポイント

- [ ] `/123456` にアクセスすると動的ページが表示される
- [ ] `/invalid-id` にアクセスするとエラーが表示される
- [ ] ローディング UI が表示される
- [ ] 404 ページが正しく表示される

## 🏃 演習問題

1. **メタデータの動的生成**: `generateMetadata` 関数を実装して、ページタイトルを動的に設定
2. **パラメータ検証の改善**: より詳細なエラーメッセージを表示
3. **ローディングスケルトンの改善**: より実際のコンテンツに近いスケルトン UI を作成

### 演習 1 の解答例

```tsx
import { Metadata } from 'next';

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  
  return {
    title: `Azure Update ${slug} | Azure Update Viewer`,
    description: `Azure Update ${slug} の詳細情報`,
  };
}
```

## 🔗 参考リンク

- [Dynamic Routes](https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes)
- [Loading UI](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [Not Found](https://nextjs.org/docs/app/api-reference/file-conventions/not-found)
- [Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)

---

準備ができたら、[Step 3: API ルートの実装](../step-03-api-routes) に進みましょう！