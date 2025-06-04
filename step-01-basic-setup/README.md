# Step 1: Next.js の基本セットアップ - 現代的な Web 開発の基盤構築

このステップでは、Next.js プロジェクトの基本的なセットアップを行いながら、現代的な Web 開発における React フレームワークの役割と設計思想を深く理解します。

## 🎯 このステップで学ぶこと

- **基礎知識**: Next.js 14 プロジェクトの作成と設定
- **アーキテクチャ理解**: App Router とファイルベースルーティングの設計思想
- **開発環境**: TypeScript、Tailwind CSS との統合
- **実践応用**: Azure UpdateSnap における基盤設計の実例
- **パフォーマンス**: 最適化機能の理解と活用

## 📚 第1章: Next.js の歴史的背景と設計思想

### 1.1 React エコシステムの進化

React が登場した 2013 年以降、フロントエンド開発は急速に進化しました。当初の React は「ライブラリ」として設計され、開発者は自らルーティング、状態管理、ビルドツールを選択する必要がありました。

```javascript
// 2013年頃の React アプリケーション
class TodoApp extends React.Component {
  constructor(props) {
    super(props);
    this.state = { items: [], text: '' };
  }
  
  render() {
    return (
      <div>
        <h3>TODO</h3>
        <TodoList items={this.state.items} />
        <form onSubmit={this.handleSubmit}>
          <input onChange={this.handleChange} value={this.state.text} />
          <button>{'Add #' + (this.state.items.length + 1)}</button>
        </form>
      </div>
    );
  }
}
```

この時代の課題：
- **設定の複雑さ**: Webpack、Babel、ESLint の個別設定
- **ルーティングの手動実装**: React Router の複雑な設定
- **SEO の困難さ**: CSR（Client-Side Rendering）による初期ロード問題
- **パフォーマンス**: バンドルサイズとロード時間の最適化

### 1.2 Next.js の登場と革新

2016年、Vercel（当時 Zeit）が Next.js を発表しました。これは「React のための生産環境フレームワーク」として設計され、以下の問題を解決しました：

#### 問題解決のアプローチ

```typescript
// Next.js の設計思想：設定より規約（Convention over Configuration）

// ❌ 従来の React アプリ - 複雑な設定
const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    loader: rootLoader,
    children: [
      {
        path: "team",
        element: <Team />,
        loader: teamLoader,
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}

// ✅ Next.js - ファイルベースルーティング
// pages/index.js → /
// pages/team/index.js → /team
// 設定ファイル不要、ディレクトリ構造がそのまま URL 構造に
```

### 1.3 Pages Router から App Router への進化

#### Pages Router の時代（2016-2022）

```typescript
// pages/api/users/[id].ts - Pages Router の API Routes
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  const { id } = req.query;
  
  if (req.method === 'GET') {
    const user = getUserById(id);
    res.status(200).json(user);
  }
}

// pages/users/[id].tsx - 動的ルート
export async function getServerSideProps(context: GetServerSidePropsContext) {
  const { id } = context.params!;
  const user = await fetchUser(id);
  
  return {
    props: { user }
  };
}

export default function UserPage({ user }: { user: User }) {
  return <div>{user.name}</div>;
}
```

**Pages Router の制限**：
- ページレベルのデータフェッチングのみ
- レイアウトの共有が複雑
- Streaming SSR の制限
- Server Components の未サポート

#### App Router の革新（2022-現在）

```typescript
// app/users/[id]/page.tsx - App Router
export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  // Server Component 内で直接データフェッチング
  const user = await fetchUser(id);
  
  return <div>{user.name}</div>;
}

// app/users/[id]/layout.tsx - ネストレイアウト
export default function UserLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="user-layout">
      <UserSidebar />
      {children}
    </div>
  );
}
```

**App Router の利点**：
- **Server Components**: サーバーサイドでのデータフェッチングとレンダリング
- **Streaming SSR**: 部分的なページ配信によるユーザー体験向上
- **Nested Layouts**: 階層的なレイアウト管理
- **Enhanced Caching**: より細かいキャッシュ制御

### 1.4 競合フレームワークとの比較

#### Next.js vs Nuxt.js (Vue.js エコシステム)

```vue
<!-- Nuxt.js 3 のアプローチ -->
<template>
  <div>
    <h1>{{ data.title }}</h1>
    <p>{{ data.description }}</p>
  </div>
</template>

<script setup>
// Nuxt.js の composables
const { data } = await $fetch('/api/content');
</script>
```

```typescript
// Next.js 14 のアプローチ
export default async function ContentPage() {
  // Server Component での直接フェッチ
  const data = await fetch('/api/content').then(res => res.json());
  
  return (
    <div>
      <h1>{data.title}</h1>
      <p>{data.description}</p>
    </div>
  );
}
```

#### Next.js vs SvelteKit

```svelte
<!-- SvelteKit のアプローチ -->
<script lang="ts">
  export let data;
</script>

<h1>{data.title}</h1>

<!-- +page.ts で load 関数を定義 -->
```

```typescript
// +page.ts
export async function load({ fetch }) {
  const response = await fetch('/api/content');
  const data = await response.json();
  return { data };
}
```

**選択の指針**：
- **Next.js**: React エコシステム、大規模アプリケーション、エンタープライズ用途
- **Nuxt.js**: Vue.js エコシステム、学習コストの低さ
- **SvelteKit**: バンドルサイズ最小化、新しい技術への挑戦

## 📖 第2章: App Router アーキテクチャの詳細解説

### 2.1 ファイルベースルーティングの設計思想

#### なぜファイルベースなのか？

従来のルーティング設定：
```typescript
// 従来の設定ベースルーティング（React Router 等）
const routes = [
  {
    path: '/',
    component: HomePage,
    exact: true
  },
  {
    path: '/blog/:slug',
    component: BlogPost,
    meta: { requiresAuth: false }
  },
  {
    path: '/admin',
    component: AdminLayout,
    children: [
      { path: 'dashboard', component: Dashboard },
      { path: 'users', component: UserList }
    ]
  }
];
```

Next.js のファイルベースアプローチ：
```
app/
├── page.tsx                 # / ルート
├── blog/
│   └── [slug]/
│       ├── page.tsx         # /blog/[slug]
│       ├── loading.tsx      # ローディング UI
│       └── error.tsx        # エラー UI
└── admin/
    ├── layout.tsx           # 共通レイアウト
    ├── dashboard/
    │   └── page.tsx         # /admin/dashboard
    └── users/
        └── page.tsx         # /admin/users
```

**利点の詳細分析**：

1. **Mental Model の一致**: ディレクトリ構造 = URL 構造
2. **自動コード分割**: 各ページが自動的に独立したチャンクに
3. **型安全性**: TypeScript との統合でパラメータの型推論
4. **IDE 支援**: ファイル移動によるリファクタリング

#### Azure UpdateSnap での実装例

```typescript
// Azure UpdateSnap のルート設計
app/
├── page.tsx                 # ホームページ
├── [slug]/                  # 動的ルート（Azure Update ID）
│   ├── page.tsx            # 更新詳細ページ
│   ├── loading.tsx         # ローディング UI
│   ├── error.tsx           # エラーハンドリング
│   ├── AzureUpdateView.tsx # 更新表示コンポーネント
│   └── RedirectingView.tsx # リダイレクト UI
├── api/                    # API Routes
│   ├── refresh/
│   │   └── route.ts        # キャッシュリフレッシュ
│   ├── cache-url/
│   │   └── route.ts        # URL キャッシング
│   └── og/
│       └── [id]/
│           └── route.tsx    # OG 画像生成
├── debug/                  # デバッグツール
│   └── og/
│       └── [id]/
│           └── page.tsx    # OG 画像デバッグ
├── layout.tsx              # ルートレイアウト
├── not-found.tsx           # 404 ページ
├── globals.css             # グローバルスタイル
└── providers.tsx           # コンテキストプロバイダー
```

### 2.2 Server Components の革新的アーキテクチャ

#### 従来の CSR vs SSR vs RSC（React Server Components）

```typescript
// 1. CSR (Client-Side Rendering) - 従来の React
function UserDashboard() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    Promise.all([
      fetch('/api/user').then(res => res.json()),
      fetch('/api/posts').then(res => res.json())
    ]).then(([userData, postsData]) => {
      setUser(userData);
      setPosts(postsData);
      setLoading(false);
    });
  }, []);
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <PostsList posts={posts} />
    </div>
  );
}
```

**CSR の問題点**：
- 初期ロード時間が長い（JavaScript ダウンロード → 実行 → データフェッチ）
- SEO に不利（初期 HTML が空）
- ネットワーク遅延の影響を受けやすい

```typescript
// 2. SSR (Server-Side Rendering) - Next.js Pages Router
export async function getServerSideProps() {
  const [userData, postsData] = await Promise.all([
    fetchUser(),
    fetchPosts()
  ]);
  
  return {
    props: {
      user: userData,
      posts: postsData
    }
  };
}

function UserDashboard({ user, posts }) {
  // プロップスとして受け取るため、ローディング状態不要
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <PostsList posts={posts} />
    </div>
  );
}
```

**SSR の改善点と残る課題**：
- ✅ 初期 HTML にコンテンツが含まれる（SEO 向上）
- ✅ 初期表示が高速
- ❌ ページ全体の再生成（部分更新不可）
- ❌ クライアントサイドでの重複フェッチ

```typescript
// 3. RSC (React Server Components) - Next.js App Router
export default async function UserDashboard() {
  // Server Component で直接データフェッチ
  const [user, posts] = await Promise.all([
    fetchUser(),
    fetchPosts()
  ]);
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      {/* Server Component 内でクライアントコンポーネントを使用 */}
      <InteractivePostsList initialPosts={posts} />
    </div>
  );
}

// Client Component（必要な部分のみ）
'use client';
function InteractivePostsList({ initialPosts }) {
  const [posts, setPosts] = useState(initialPosts);
  
  return (
    <div>
      {posts.map(post => (
        <PostCard 
          key={post.id} 
          post={post}
          onLike={() => handleLike(post.id)}
        />
      ))}
    </div>
  );
}
```

**RSC の革新的利点**：
- ✅ サーバーでのデータフェッチ（データベース直接アクセス可能）
- ✅ 部分的ストリーミング（Suspense との統合）
- ✅ 自動的なコード分割
- ✅ クライアントバンドルサイズの削減

#### Azure UpdateSnap での RSC 活用例

```typescript
// app/[slug]/page.tsx - Azure UpdateSnap のメインページ
export default async function UpdatePage({ params, searchParams }: Props) {
  const { slug } = await params;
  const { debug } = await searchParams;
  
  // Server Component 内で直接データベースアクセス
  const update = await getOrFetchAzureUpdate(slug);
  
  if (!update) {
    notFound(); // 適切な 404 レスポンス
  }
  
  // ヘッダー情報の取得（Server Component のみで可能）
  const { headers } = await import('next/headers');
  const headersList = await headers();
  const userAgent = headersList.get('user-agent');
  
  // Bot 検出ロジック
  const isBot = /bot|crawler|spider|crawling/i.test(userAgent || '');
  const isDebugMode = debug === 'true';
  
  // 条件に応じたコンポーネント選択
  if (isBot || isDebugMode) {
    // Bot や debug モードでは完全なコンテンツを表示
    return <AzureUpdateView update={update} />;
  }
  
  // 通常ユーザーはリダイレクト（Client Component）
  return <RedirectingView targetUrl={update.detailsUrl} />;
}

// メタデータの動的生成（Server Component のみ）
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const update = await getOrFetchAzureUpdate(slug);
  
  if (!update) {
    return { title: 'Not Found' };
  }
  
  const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';
  
  return {
    title: `${update.title} | Azure Update Viewer`,
    description: update.description,
    openGraph: {
      title: update.title,
      description: update.description,
      images: [`${baseUrl}/api/og/${slug}`],
    },
  };
}
```

### 2.3 Layout とネストレイアウトシステム

#### 階層的レイアウトの設計原理

従来の SPA における共通レイアウトの課題：
```typescript
// 従来のアプローチ - 手動でのレイアウト管理
function App() {
  const location = useLocation();
  const showSidebar = location.pathname.startsWith('/admin');
  
  return (
    <div>
      <Header />
      {showSidebar && <Sidebar />}
      <main>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/admin/*" element={<AdminRoutes />} />
        </Routes>
      </main>
      <Footer />
    </div>
  );
}
```

Next.js App Router のネストレイアウト：
```typescript
// app/layout.tsx - ルートレイアウト
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        {/* 全ページ共通のヘッダー */}
        <header className="bg-blue-600 text-white p-4">
          <h1 className="text-2xl font-bold">Azure Update Viewer</h1>
        </header>
        
        {/* ページコンテンツ */}
        {children}
        
        {/* 全ページ共通のフッター */}
        <footer className="bg-gray-800 text-white p-4">
          <p>&copy; 2024 Azure UpdateSnap</p>
        </footer>
      </body>
    </html>
  );
}

// app/admin/layout.tsx - 管理画面専用レイアウト
export default function AdminLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="admin-layout flex">
      {/* 管理画面専用サイドバー */}
      <aside className="w-64 bg-gray-100 p-4">
        <nav>
          <Link href="/admin/dashboard">Dashboard</Link>
          <Link href="/admin/users">Users</Link>
        </nav>
      </aside>
      
      {/* 管理画面のメインコンテンツ */}
      <main className="flex-1 p-8">
        {children}
      </main>
    </div>
  );
}

// レンダリング結果（/admin/dashboard の場合）:
// RootLayout(
//   AdminLayout(
//     DashboardPage()
//   )
// )
```

#### Azure UpdateSnap でのレイアウト戦略

```typescript
// app/layout.tsx - Azure UpdateSnap のルートレイアウト
import { Inter } from 'next/font/google';
import type { Metadata } from 'next';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: {
    template: '%s | Azure Update Viewer',
    default: 'Azure Update Viewer',
  },
  description: 'Microsoft Azure の更新情報を高速に表示するアプリケーション',
  metadataBase: new URL(process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000'),
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body className={inter.className}>
        {/* Azure のブランドカラーを使用したヘッダー */}
        <header className="bg-blue-600 text-white p-4 sticky top-0 z-50">
          <div className="max-w-6xl mx-auto flex items-center justify-between">
            <h1 className="text-2xl font-bold">Azure Update Viewer</h1>
            <nav>
              <Link href="/" className="hover:underline">
                Home
              </Link>
            </nav>
          </div>
        </header>
        
        {/* メインコンテンツエリア */}
        <main className="min-h-screen bg-gray-50">
          {children}
        </main>
        
        {/* フッター */}
        <footer className="bg-gray-800 text-white p-8">
          <div className="max-w-6xl mx-auto">
            <p>© 2024 Azure UpdateSnap. Microsoft Azure 更新情報の非公式ビューア</p>
            <p className="text-sm text-gray-400 mt-2">
              このサイトは Microsoft Corporation と関連はありません。
            </p>
          </div>
        </footer>
      </body>
    </html>
  );
}
```

### 2.4 型安全性と開発者体験

#### TypeScript との深い統合

```typescript
// 型安全なページパラメータの例
type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

// Next.js が自動生成する型
export default async function UpdatePage({ params, searchParams }: Props) {
  const { slug } = await params; // slug: string
  const { debug, theme } = await searchParams; // debug: string | string[] | undefined
  
  // 型安全なパラメータ検証
  if (typeof debug === 'string' && debug === 'true') {
    // debug モードの処理
  }
  
  // 型安全なメタデータ生成
  return <div>Update: {slug}</div>;
}

// generateMetadata も同じ型を使用
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params; // TypeScript が型を推論
  
  return {
    title: `Update ${slug}`,
  };
}
```

#### Azure UpdateSnap の型定義例

```typescript
// src/types/azure.ts - Azure Update の型定義
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

// パラメータの厳密な型定義
export type UpdatePageParams = {
  slug: string; // 数値 ID または GUID
};

export type UpdateSearchParams = {
  debug?: string;
  theme?: 'light' | 'dark';
  lang?: 'ja' | 'en';
};

// 型安全なページコンポーネント
type UpdatePageProps = {
  params: Promise<UpdatePageParams>;
  searchParams: Promise<UpdateSearchParams>;
};

export default async function UpdatePage({ params, searchParams }: UpdatePageProps) {
  // TypeScript が完全に型をチェック
  const { slug } = await params;
  const { debug, theme = 'light' } = await searchParams;
  
  // 型安全な分岐処理
  if (debug === 'true') {
    // ...
  }
}
```

## 🛠️ 第3章: 実践的セットアップと最適化

### 3.1 プロジェクト作成の詳細手順

#### create-next-app の内部動作

```bash
# Next.js 14 プロジェクトの作成
npx create-next-app@latest my-azure-viewer --typescript --tailwind --app
```

このコマンドの内部で実行される処理：

1. **テンプレートのダウンロード**: GitHub から最新テンプレートを取得
2. **依存関係の解決**: package.json の生成と npm install
3. **設定ファイルの生成**: TypeScript、Tailwind、ESLint の設定
4. **初期ファイルの作成**: layout.tsx、page.tsx、globals.css

生成されるファイル構造の詳細：

```
my-azure-viewer/
├── .eslintrc.json           # ESLint 設定
├── .gitignore              # Git 除外設定
├── next.config.js          # Next.js 設定
├── package.json            # 依存関係
├── postcss.config.js       # PostCSS 設定（Tailwind CSS 用）
├── tailwind.config.ts      # Tailwind CSS 設定
├── tsconfig.json           # TypeScript 設定
├── public/                 # 静的ファイル
│   ├── next.svg
│   └── vercel.svg
└── src/                    # ソースコード
    └── app/
        ├── favicon.ico
        ├── globals.css     # グローバルスタイル
        ├── layout.tsx      # ルートレイアウト
        └── page.tsx        # ホームページ
```

#### Azure UpdateSnap 向けのカスタマイズ

```typescript
// next.config.js の最適化設定
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 実験的機能の有効化
  experimental: {
    // App Router での PPR (Partial Prerendering)
    ppr: true,
  },
  
  // 画像最適化の設定
  images: {
    domains: [
      'admin.microsoft.com',
      'techcommunity.microsoft.com',
    ],
    formats: ['image/webp', 'image/avif'],
  },
  
  // ヘッダーの設定
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'Referrer-Policy',
            value: 'origin-when-cross-origin',
          },
        ],
      },
    ];
  },
  
  // リダイレクト設定
  async redirects() {
    return [
      {
        source: '/update/:id',
        destination: '/:id',
        permanent: true,
      },
    ];
  },
};

module.exports = nextConfig;
```

### 3.2 TypeScript 設定の詳細

#### tsconfig.json の最適化

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "es6"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/lib/*": ["./src/lib/*"],
      "@/types/*": ["./src/types/*"]
    }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts"
  ],
  "exclude": ["node_modules"]
}
```

#### 型安全性の向上設定

```typescript
// src/types/next.d.ts - Next.js 型拡張
import type { NextRequest } from 'next/server';

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NEXT_PUBLIC_BASE_URL: string;
      SUPABASE_URL: string;
      SUPABASE_ANON_KEY: string;
      TTL_HOURS: string;
    }
  }
}

// Request オブジェクトの拡張
declare module 'next/server' {
  interface NextRequest {
    nextUrl: URL;
    geo?: {
      country?: string;
      region?: string;
      city?: string;
    };
  }
}
```

### 3.3 Tailwind CSS の戦略的設定

#### tailwind.config.ts の詳細設定

```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      // Azure ブランドカラーの定義
      colors: {
        azure: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6', // Primary Azure Blue
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
        },
        microsoft: {
          blue: '#0078d4',
          green: '#107c10',
          orange: '#ff8c00',
          red: '#d13438',
        },
      },
      
      // タイポグラフィの拡張
      fontFamily: {
        sans: ['Inter', 'Segoe UI', 'system-ui', 'sans-serif'],
        mono: ['Fira Code', 'Monaco', 'Consolas', 'monospace'],
      },
      
      // アニメーションの定義
      animation: {
        'fade-in': 'fadeIn 0.5s ease-in-out',
        'slide-in-up': 'slideInUp 0.3s ease-out',
        'pulse-azure': 'pulseAzure 2s cubic-bezier(0.4, 0, 0.6, 1) infinite',
      },
      
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideInUp: {
          '0%': { transform: 'translateY(20px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        pulseAzure: {
          '0%, 100%': { boxShadow: '0 0 0 0 rgba(59, 130, 246, 0.7)' },
          '70%': { boxShadow: '0 0 0 10px rgba(59, 130, 246, 0)' },
        },
      },
      
      // レスポンシブブレークポイント
      screens: {
        'xs': '475px',
        '3xl': '1600px',
      },
    },
  },
  plugins: [
    require('@tailwindcss/typography'),
    require('@tailwindcss/forms'),
  ],
};

export default config;
```

#### CSS カスタムプロパティとの統合

```css
/* src/app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    /* Azure デザインシステムのカラートークン */
    --color-azure-primary: 59 130 246;
    --color-azure-secondary: 37 99 235;
    --color-text-primary: 17 24 39;
    --color-text-secondary: 107 114 128;
    
    /* スペーシングシステム */
    --spacing-xs: 0.25rem;
    --spacing-sm: 0.5rem;
    --spacing-md: 1rem;
    --spacing-lg: 1.5rem;
    --spacing-xl: 2rem;
    
    /* タイポグラフィスケール */
    --font-size-xs: 0.75rem;
    --font-size-sm: 0.875rem;
    --font-size-base: 1rem;
    --font-size-lg: 1.125rem;
    --font-size-xl: 1.25rem;
  }
  
  @media (prefers-color-scheme: dark) {
    :root {
      --color-text-primary: 243 244 246;
      --color-text-secondary: 156 163 175;
    }
  }
}

@layer components {
  /* Azure UpdateSnap 専用コンポーネント */
  .update-card {
    @apply bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow duration-300;
  }
  
  .update-title {
    @apply text-xl font-semibold text-gray-900 mb-2 line-clamp-2;
  }
  
  .update-description {
    @apply text-gray-600 text-sm line-clamp-3;
  }
  
  .tag-azure {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-azure-100 text-azure-800;
  }
  
  /* ローディング状態のスケルトン */
  .skeleton {
    @apply animate-pulse bg-gray-200 rounded;
  }
  
  .skeleton-text {
    @apply h-4 bg-gray-200 rounded;
  }
  
  .skeleton-title {
    @apply h-6 bg-gray-200 rounded w-3/4;
  }
}

@layer utilities {
  /* カスタムユーティリティ */
  .text-balance {
    text-wrap: balance;
  }
  
  .scrollbar-hidden {
    -ms-overflow-style: none;
    scrollbar-width: none;
  }
  
  .scrollbar-hidden::-webkit-scrollbar {
    display: none;
  }
  
  /* Azure ブランドグラデーション */
  .bg-azure-gradient {
    background: linear-gradient(135deg, rgb(var(--color-azure-primary)) 0%, rgb(var(--color-azure-secondary)) 100%);
  }
}
```

## 🚀 第4章: パフォーマンス最適化と本番環境対応

### 4.1 ビルド最適化戦略

#### Bundle Analyzer による分析

```bash
# Bundle 分析の設定
npm install --save-dev @next/bundle-analyzer
```

```javascript
// next.config.js に追加
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // 既存の設定...
});
```

```bash
# Bundle サイズの分析実行
ANALYZE=true npm run build
```

#### 動的インポートによるコード分割

```typescript
// 重いコンポーネントの遅延読み込み
import dynamic from 'next/dynamic';
import { Suspense } from 'react';

// Chart.js のような重いライブラリを遅延読み込み
const ChartComponent = dynamic(() => import('./ChartComponent'), {
  loading: () => <div className="skeleton h-64 w-full" />,
  ssr: false, // クライアントサイドのみで実行
});

// Azure Update の詳細分析コンポーネント（必要時のみ読み込み）
const DetailedAnalysis = dynamic(() => import('./DetailedAnalysis'), {
  loading: () => <DetailedAnalysisSkeleton />,
});

export default function UpdatePage({ update }: { update: AzureUpdate }) {
  const [showAnalysis, setShowAnalysis] = useState(false);
  
  return (
    <div>
      <AzureUpdateView update={update} />
      
      <button onClick={() => setShowAnalysis(true)}>
        詳細分析を表示
      </button>
      
      {showAnalysis && (
        <Suspense fallback={<DetailedAnalysisSkeleton />}>
          <DetailedAnalysis updateId={update.id} />
        </Suspense>
      )}
    </div>
  );
}
```

### 4.2 リソース最適化

#### 画像最適化戦略

```typescript
// next/image を使った最適化
import Image from 'next/image';

export default function AzureUpdateCard({ update }: { update: AzureUpdate }) {
  return (
    <div className="update-card">
      {/* Azure サービスのアイコン */}
      <Image
        src={`/icons/azure-services/${update.service}.svg`}
        alt={`${update.service} icon`}
        width={48}
        height={48}
        priority={false} // Above-the-fold でない場合は false
        placeholder="blur"
        blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD..." // 低品質プレースホルダー
      />
      
      <h3 className="update-title">{update.title}</h3>
      <p className="update-description">{update.description}</p>
    </div>
  );
}
```

#### フォント最適化

```typescript
// app/layout.tsx でのフォント最適化
import { Inter, Noto_Sans_JP } from 'next/font/google';

// 英語フォントの最適化
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
  preload: true,
});

// 日本語フォントの最適化
const notoSansJP = Noto_Sans_JP({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-noto-sans-jp',
  weight: ['400', '500', '700'],
  preload: false, // 必要時のみ読み込み
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja" className={`${inter.variable} ${notoSansJP.variable}`}>
      <body className="font-sans">
        {children}
      </body>
    </html>
  );
}
```

### 4.3 SEO とアクセシビリティ

#### 構造化データの実装

```typescript
// Azure Update 用の JSON-LD 構造化データ
function generateStructuredData(update: AzureUpdate) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: update.title,
    description: update.description,
    datePublished: update.publicDisclosureDate,
    author: {
      '@type': 'Organization',
      name: 'Microsoft Corporation',
    },
    publisher: {
      '@type': 'Organization',
      name: 'Azure UpdateSnap',
      logo: {
        '@type': 'ImageObject',
        url: `${process.env.NEXT_PUBLIC_BASE_URL}/logo.png`,
      },
    },
    keywords: update.tags.join(', '),
    articleSection: 'Azure Updates',
  };
}

export default function UpdatePage({ update }: { update: AzureUpdate }) {
  const structuredData = generateStructuredData(update);
  
  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
      />
      <AzureUpdateView update={update} />
    </>
  );
}
```

#### アクセシビリティの実装

```typescript
// ARIA ラベルとセマンティック HTML
export default function AzureUpdateView({ update }: { update: AzureUpdate }) {
  return (
    <article
      role="article"
      aria-labelledby="update-title"
      aria-describedby="update-description"
    >
      <header>
        <h1 id="update-title" className="update-title">
          {update.title}
        </h1>
        
        <div className="flex items-center gap-2 text-sm text-gray-600">
          <time
            dateTime={update.publicDisclosureDate}
            aria-label={`公開日: ${new Date(update.publicDisclosureDate).toLocaleDateString('ja-JP')}`}
          >
            {new Date(update.publicDisclosureDate).toLocaleDateString('ja-JP')}
          </time>
          
          <span aria-hidden="true">•</span>
          
          <span aria-label={`更新ID: ${update.id}`}>
            ID: {update.id}
          </span>
        </div>
      </header>
      
      <section>
        <h2 className="sr-only">更新の詳細</h2>
        <p id="update-description" className="update-description">
          {update.description}
        </p>
      </section>
      
      {update.tags.length > 0 && (
        <section aria-labelledby="tags-heading">
          <h3 id="tags-heading" className="sr-only">
            関連タグ
          </h3>
          <div className="flex flex-wrap gap-2">
            {update.tags.map((tag) => (
              <span
                key={tag}
                className="tag-azure"
                role="button"
                tabIndex={0}
                aria-label={`タグ: ${tag}`}
                onKeyDown={(e) => {
                  if (e.key === 'Enter' || e.key === ' ') {
                    // タグクリック処理
                  }
                }}
              >
                {tag}
              </span>
            ))}
          </div>
        </section>
      )}
    </article>
  );
}
```

## ✅ 確認ポイント

- [ ] **プロジェクト作成**: Next.js 14 プロジェクトが正常に作成される
- [ ] **開発サーバー**: `npm run dev` でローカルサーバーが起動する
- [ ] **TypeScript**: 型エラーがなく、型推論が正常に動作する
- [ ] **Tailwind CSS**: スタイルが正しく適用される
- [ ] **ビルド**: `npm run build` が成功する
- [ ] **アクセシビリティ**: スクリーンリーダーでの読み上げが適切
- [ ] **SEO**: メタデータが正しく設定される

## 🏃 演習問題

### 初級: 基本的なカスタマイズ

1. **ダークモード対応**: Tailwind CSS の dark mode を実装
2. **カスタムフォント**: Google Fonts から別のフォントを追加
3. **環境変数**: `.env.local` でアプリケーション名を設定

### 中級: コンポーネント設計

1. **再利用可能なカード**: 複数の場所で使えるカードコンポーネントを作成
2. **Loading コンポーネント**: スケルトン UI を含むローディング状態
3. **エラーバウンダリ**: React Error Boundary の実装

### 上級: パフォーマンス最適化

1. **Bundle 分析**: 不要な依存関係の特定と削除
2. **Core Web Vitals**: LCP、FID、CLS の測定と改善
3. **キャッシュ戦略**: HTTP キャッシュヘッダーの最適化

### 演習解答例

#### 1. ダークモード実装

```typescript
// tailwind.config.ts にダークモード設定追加
module.exports = {
  darkMode: 'class', // クラスベースでダークモード切り替え
  // ...既存の設定
};

// テーマ切り替えコンポーネント
'use client';

import { useState, useEffect } from 'react';

export default function ThemeToggle() {
  const [isDark, setIsDark] = useState(false);
  
  useEffect(() => {
    const saved = localStorage.getItem('theme');
    const isDarkMode = saved === 'dark' || 
      (!saved && window.matchMedia('(prefers-color-scheme: dark)').matches);
    
    setIsDark(isDarkMode);
    document.documentElement.classList.toggle('dark', isDarkMode);
  }, []);
  
  const toggleTheme = () => {
    const newTheme = !isDark;
    setIsDark(newTheme);
    localStorage.setItem('theme', newTheme ? 'dark' : 'light');
    document.documentElement.classList.toggle('dark', newTheme);
  };
  
  return (
    <button
      onClick={toggleTheme}
      className="p-2 rounded-lg bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-gray-200"
      aria-label={`${isDark ? 'ライト' : 'ダーク'}モードに切り替え`}
    >
      {isDark ? '☀️' : '🌙'}
    </button>
  );
}
```

## 🔗 参考リンク

### 公式ドキュメント
- [Next.js 14 Documentation](https://nextjs.org/docs)
- [App Router](https://nextjs.org/docs/app)
- [TypeScript with Next.js](https://nextjs.org/docs/app/building-your-application/configuring/typescript)
- [Tailwind CSS with Next.js](https://tailwindcss.com/docs/guides/nextjs)

### 学習リソース
- [React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)
- [Web.dev Performance](https://web.dev/performance/)
- [MDN Web Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility)

### ツールとライブラリ
- [Next.js Bundle Analyzer](https://www.npmjs.com/package/@next/bundle-analyzer)
- [Lighthouse](https://developers.google.com/web/tools/lighthouse)
- [axe DevTools](https://www.deque.com/axe/devtools/)

---

**次のステップ**: 基本的なセットアップが完了したら、[Step 2: ルーティングとページ作成](../step-02-routing) に進んで、Next.js App Router の動的ルーティングシステムを詳しく学びましょう！