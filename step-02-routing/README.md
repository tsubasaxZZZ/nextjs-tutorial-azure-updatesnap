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

### 動的セグメント

- `[slug]`: 単一の動的セグメント
- `[...slug]`: キャッチオールセグメント
- `[[...slug]]`: オプショナルキャッチオールセグメント

### Parallel Routes と Intercepting Routes

Next.js 14 の高度な機能：
- Parallel Routes: 複数のページを同時にレンダリング
- Intercepting Routes: ルートをインターセプトしてモーダル表示など

### Server Components でのデータフェッチング

```tsx
// 非同期 Server Component
export default async function Page() {
  const data = await fetchData(); // 直接 await できる
  return <div>{data}</div>;
}
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