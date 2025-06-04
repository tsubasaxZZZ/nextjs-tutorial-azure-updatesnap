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

### App Router

- `app/` ディレクトリ内のファイル構造がそのままルーティングに
- `page.tsx`: そのルートのページコンポーネント
- `layout.tsx`: 子ルートで共有されるレイアウト
- `loading.tsx`: ローディング UI
- `error.tsx`: エラー UI

### Server Components vs Client Components

- デフォルトは Server Components
- `"use client"` ディレクティブで Client Component に
- Server Components の利点：
  - データフェッチングが簡単
  - バンドルサイズの削減
  - SEO に有利

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