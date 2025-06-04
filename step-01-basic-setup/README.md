# Step 1: Next.js の基本セットアップ

このステップでは、Next.js プロジェクトの基本的なセットアップを行います。

## 🎯 このステップで学ぶこと

- Next.js 14 プロジェクトの作成
- TypeScript の設定
- Tailwind CSS の設定
- 基本的なプロジェクト構造の理解
- 開発サーバーの起動

## 📖 解説

### Next.js とは？

Next.js は React ベースのフルスタックフレームワークです。主な特徴：

- **App Router**: ファイルベースのルーティング
- **Server Components**: デフォルトでサーバーサイドレンダリング
- **API Routes**: バックエンド API の実装も可能
- **最適化**: 画像、フォント、スクリプトの自動最適化

### プロジェクト構造

```
nextjs-app/
├── src/
│   ├── app/              # App Router のルート
│   │   ├── layout.tsx    # ルートレイアウト
│   │   ├── page.tsx      # ホームページ
│   │   └── globals.css   # グローバルスタイル
│   └── lib/              # ユーティリティ関数
├── public/               # 静的ファイル
├── package.json
├── tsconfig.json         # TypeScript 設定
├── tailwind.config.ts    # Tailwind CSS 設定
└── next.config.js        # Next.js 設定
```

## 🛠️ ハンズオン

### 1. プロジェクトの作成

```bash
npx create-next-app@latest my-azure-viewer --typescript --tailwind --app
cd my-azure-viewer
```

プロンプトが表示されたら、以下を選択：
- TypeScript: Yes
- ESLint: Yes
- Tailwind CSS: Yes
- `src/` directory: Yes
- App Router: Yes
- Customize import alias: No

### 2. 開発サーバーの起動

```bash
npm run dev
```

ブラウザで http://localhost:3000 を開いて確認。

### 3. 基本的なページの作成

`src/app/page.tsx` を編集：

```tsx
export default function Home() {
  return (
    <main className="min-h-screen p-8">
      <h1 className="text-4xl font-bold mb-4">
        Azure Update Viewer
      </h1>
      <p className="text-gray-600">
        Microsoft Azure の更新情報を高速に表示するアプリケーション
      </p>
    </main>
  );
}
```

### 4. レイアウトの設定

`src/app/layout.tsx` を編集：

```tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Azure Update Viewer",
  description: "Microsoft Azure の更新情報ビューア",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body className={inter.className}>
        <header className="bg-blue-600 text-white p-4">
          <h1 className="text-2xl font-bold">Azure Update Viewer</h1>
        </header>
        {children}
      </body>
    </html>
  );
}
```

## 📚 重要な概念

### App Router の詳細な仕組み

App Router は Next.js 13 で導入された新しいルーティングシステムです。

#### ディレクトリ構造とルーティングの関係

```
app/
├── page.tsx                 # / (ルート)
├── about/
│   └── page.tsx            # /about
├── blog/
│   ├── page.tsx            # /blog
│   └── [slug]/
│       └── page.tsx        # /blog/[slug] (動的ルート)
└── api/
    └── hello/
        └── route.ts        # /api/hello (API ルート)
```

**なぜファイルベースルーティングなのか？**
- 明示的なルート定義が不要
- ディレクトリ構造を見れば URL 構造がわかる
- 自動的なコード分割とプリフェッチ

#### 特殊ファイルの役割と実行順序

```tsx
// 1. layout.tsx - 最初に実行され、子要素をラップ
export default function Layout({ children }: { children: React.ReactNode }) {
  // このコンポーネントは一度だけレンダリングされ、
  // ページ遷移時も再レンダリングされない
  return (
    <div>
      <nav>共通ナビゲーション</nav>
      {children}
    </div>
  );
}

// 2. loading.tsx - Suspense のフォールバック
export default function Loading() {
  // page.tsx のレンダリング中に表示される
  return <div>Loading...</div>;
}

// 3. error.tsx - エラーバウンダリ
'use client'; // エラーバウンダリは必ず Client Component
export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>エラーが発生しました</h2>
      <button onClick={reset}>再試行</button>
    </div>
  );
}

// 4. page.tsx - 実際のページコンテンツ
export default async function Page() {
  // Server Component として実行される
  const data = await fetch('...'); // サーバーサイドでのデータ取得
  return <div>{/* コンテンツ */}</div>;
}
```

### Server Components の深い理解

#### Server Components の実行環境

```tsx
// これは Server Component（デフォルト）
export default async function ServerComponent() {
  // サーバーでのみ実行される
  console.log('This runs on the server');
  
  // 以下のことが可能：
  // 1. 直接データベースにアクセス
  const db = await import('./db');
  const data = await db.query('SELECT * FROM users');
  
  // 2. ファイルシステムにアクセス
  const fs = await import('fs/promises');
  const file = await fs.readFile('./data.json', 'utf-8');
  
  // 3. 環境変数に直接アクセス（NEXT_PUBLIC_ プレフィックス不要）
  const apiKey = process.env.SECRET_API_KEY;
  
  // 4. 大きな依存関係をインポート（クライアントバンドルに含まれない）
  const heavyLib = await import('heavy-computation-library');
  
  return <div>{/* レンダリング結果のみがクライアントに送信される */}</div>;
}
```

#### Client Components との境界

```tsx
// ClientComponent.tsx
'use client'; // この宣言により Client Component になる

import { useState, useEffect } from 'react';

export default function ClientComponent({ initialData }) {
  // Client Component でのみ可能なこと：
  // 1. React Hooks の使用
  const [count, setCount] = useState(0);
  
  // 2. ブラウザ API の使用
  useEffect(() => {
    window.addEventListener('resize', handleResize);
    localStorage.setItem('key', 'value');
  }, []);
  
  // 3. イベントハンドラの使用
  const handleClick = () => {
    setCount(count + 1);
  };
  
  return <button onClick={handleClick}>Count: {count}</button>;
}

// ServerComponent.tsx (Server Component)
import ClientComponent from './ClientComponent';

export default async function ServerComponent() {
  // Server Component で データを取得
  const data = await fetchData();
  
  return (
    <div>
      {/* Client Component に props として渡す */}
      {/* 注意: 渡せるのはシリアライズ可能な値のみ */}
      <ClientComponent initialData={data} />
    </div>
  );
}
```

### レイアウトの継承とネスティング

```tsx
// app/layout.tsx - ルートレイアウト
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Header /> {/* 全ページ共通 */}
        {children}
        <Footer /> {/* 全ページ共通 */}
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx - ネストされたレイアウト
export default function DashboardLayout({ children }) {
  return (
    <div className="dashboard">
      <Sidebar /> {/* ダッシュボード配下のページのみ */}
      <main>{children}</main>
    </div>
  );
}

// app/dashboard/analytics/page.tsx
export default function AnalyticsPage() {
  // このページは両方のレイアウトに包まれる：
  // RootLayout > DashboardLayout > AnalyticsPage
  return <div>Analytics Content</div>;
}
```

### Metadata API の仕組み

```tsx
// 静的メタデータ
export const metadata: Metadata = {
  title: 'Azure Update Viewer',
  description: 'Microsoft Azure の更新情報ビューア',
  openGraph: {
    title: 'Azure Update Viewer',
    images: ['/og-image.png'],
  },
};

// 動的メタデータ
export async function generateMetadata({ params }): Promise<Metadata> {
  // ページのパラメータに基づいて動的に生成
  const post = await getPost(params.id);
  
  return {
    title: post.title,
    description: post.summary,
    openGraph: {
      images: [post.image],
    },
  };
}
```

### Next.js の最適化の仕組み

#### 自動的なコード分割

```tsx
// 各 page.tsx は自動的に別のチャンクに分割される
// app/home/page.tsx → home.js
// app/about/page.tsx → about.js

// 動的インポートでさらに細かく分割
const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <p>Loading...</p>,
  // このコンポーネントは必要になるまでロードされない
});
```

#### プリフェッチング

```tsx
import Link from 'next/link';

export default function Navigation() {
  return (
    <nav>
      {/* Link コンポーネントは viewport に入ると自動的にプリフェッチ */}
      <Link href="/about" prefetch={true}>
        About
      </Link>
      
      {/* prefetch={false} で無効化も可能 */}
      <Link href="/heavy-page" prefetch={false}>
        Heavy Page
      </Link>
    </nav>
  );
}
```

## ✅ 確認ポイント

- [ ] 開発サーバーが起動する
- [ ] ページにタイトルとヘッダーが表示される
- [ ] Tailwind CSS のスタイルが適用される
- [ ] TypeScript のエラーがない

## 🏃 演習問題

1. **フッターの追加**: `layout.tsx` にフッターを追加してみましょう
2. **About ページの作成**: `app/about/page.tsx` を作成して、アプリケーションの説明ページを作りましょう
3. **環境変数の設定**: `.env.local` ファイルを作成し、`NEXT_PUBLIC_APP_NAME` を設定して表示してみましょう

## 🔗 参考リンク

- [Next.js Getting Started](https://nextjs.org/docs/getting-started)
- [App Router](https://nextjs.org/docs/app)
- [TypeScript](https://nextjs.org/docs/app/building-your-application/configuring/typescript)
- [Tailwind CSS with Next.js](https://tailwindcss.com/docs/guides/nextjs)

---

準備ができたら、[Step 2: ルーティングとページ作成](../step-02-routing) に進みましょう！