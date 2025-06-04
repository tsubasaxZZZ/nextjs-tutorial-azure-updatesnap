# Step 2: ルーティングとページ作成 - 現代的なルーティングアーキテクチャの深層理解

このステップでは、Next.js App Router の動的ルーティングシステムを実装しながら、現代的な Web アプリケーションにおけるルーティングアーキテクチャの設計原理と実装技術を体系的に学習します。

## 🎯 このステップで学ぶこと

- **ルーティング理論**: 動的ルーティングとパラメータ処理の設計原理
- **アーキテクチャ**: Server/Client Components の境界設計と最適化
- **実装技術**: 型安全なパラメータ処理とエラーハンドリング
- **パフォーマンス**: ローディング状態とストリーミング SSR の活用
- **プロダクション**: Azure UpdateSnap における実践的なルーティング戦略

## 📚 第1章: 動的ルーティングの設計思想と進化

### 1.1 ルーティングパラダイムの歴史

#### 従来の MPA（Multi-Page Application）アプローチ

```php
// 2000年代初期の PHP アプローチ
<?php
// update.php?id=123456
$updateId = $_GET['id'];
$update = fetchUpdateFromDatabase($updateId);

if (!$update) {
    header("HTTP/1.0 404 Not Found");
    include '404.php';
    exit;
}
?>

<html>
<head>
    <title><?= htmlspecialchars($update['title']) ?></title>
</head>
<body>
    <h1><?= htmlspecialchars($update['title']) ?></h1>
    <p><?= htmlspecialchars($update['description']) ?></p>
</body>
</html>
```

**MPA の特徴と制限**：
- ✅ SEO フレンドリー（各ページが独立した HTML）
- ✅ シンプルな実装
- ❌ ページ間遷移でのフル再読み込み
- ❌ 状態の保持が困難
- ❌ ユーザー体験の断絶

#### SPA（Single-Page Application）の登場

```javascript
// 2010年代の Angular/React Router アプローチ
// クライアントサイドルーティング
const router = new Router();

router.route('/update/:id', function(id) {
    // クライアントサイドで動的にコンテンツを更新
    const update = await fetchUpdate(id);
    renderUpdatePage(update);
});

// URL の変更をJavaScriptで管理
history.pushState({}, '', `/update/${updateId}`);
```

**SPA の革新と課題**：
- ✅ スムーズなページ遷移
- ✅ 状態の保持
- ✅ リッチなユーザー体験
- ❌ 初期ロード時間の増加
- ❌ SEO の困難さ
- ❌ JavaScript 無効時の問題

#### Next.js によるハイブリッドアプローチ

```typescript
// Next.js App Router - 最新のアプローチ
// app/update/[id]/page.tsx

export default async function UpdatePage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  
  // Server Component でのデータフェッチング
  const update = await fetchUpdate(id);
  
  if (!update) {
    notFound(); // 適切な 404 レスポンス
  }
  
  return (
    <div>
      <h1>{update.title}</h1>
      <p>{update.description}</p>
      {/* 必要な部分のみ Client Component */}
      <InteractiveComments updateId={id} />
    </div>
  );
}
```

**Next.js の統合的利点**：
- ✅ SSR による SEO 最適化
- ✅ クライアントサイドナビゲーション
- ✅ 自動コード分割
- ✅ プリフェッチング

### 1.2 Azure UpdateSnap のルーティング要件分析

#### ビジネス要件とURL設計

Azure UpdateSnap では以下の URL パターンをサポートする必要があります：

```typescript
// URL パターンの要件
interface RouteRequirements {
  // 数値 ID: Microsoft Message Center の更新ID
  numericId: '/123456';        // 例: /287654
  
  // GUID: Azure Update の一意識別子
  guidId: '/a1b2c3d4-e5f6-7890-abcd-ef1234567890';
  
  // クエリパラメータ
  debugMode: '/:id?debug=true';     // 開発者向けデバッグモード
  theme: '/:id?theme=dark';         // テーマ切り替え
  language: '/:id?lang=en';         // 多言語対応
}
```

#### ユーザージャーニーの設計

```typescript
// ユーザージャーニー別の動作仕様
interface UserJourney {
  // 一般ユーザー（Bot以外）
  regularUser: {
    flow: '閲覧 → リダイレクト → Microsoft公式ページ',
    purpose: 'SEO向上とトラフィック分析',
    redirectDelay: 3000, // 3秒後に自動リダイレクト
  };
  
  // Bot（検索エンジンクローラー等）
  searchEngineBots: {
    flow: '閲覧 → フルコンテンツ表示',
    purpose: 'SEO最適化とOG画像生成',
    contentDisplay: 'full',
  };
  
  // 開発者（デバッグモード）
  developers: {
    flow: '閲覧 → フルコンテンツ表示',
    purpose: '開発とテスト',
    features: ['データ確認', 'レンダリング検証', 'メタデータ確認'],
  };
}
```

## 📖 第2章: 動的ルーティングの詳細実装

### 2.1 ファイルベースルーティングの設計パターン

#### Dynamic Segments の種類と使い分け

```typescript
// 1. [slug] - 単一動的セグメント
// app/updates/[slug]/page.tsx
// 対応URL: /updates/123456, /updates/abc-def

type SingleSegmentProps = {
  params: Promise<{ slug: string }>;
};

export default async function SingleSegmentPage({ params }: SingleSegmentProps) {
  const { slug } = await params;
  // slug の値: "123456" または "abc-def"
  
  return <div>Update: {slug}</div>;
}

// 2. [...segments] - Catch-all Routes
// app/docs/[...segments]/page.tsx  
// 対応URL: /docs/api, /docs/api/v1, /docs/api/v1/users

type CatchAllProps = {
  params: Promise<{ segments: string[] }>;
};

export default async function CatchAllPage({ params }: CatchAllProps) {
  const { segments } = await params;
  // segments の値: ["api"], ["api", "v1"], ["api", "v1", "users"]
  
  const path = segments.join('/');
  return <div>Documentation: {path}</div>;
}

// 3. [[...segments]] - Optional Catch-all Routes
// app/shop/[[...segments]]/page.tsx
// 対応URL: /shop, /shop/category, /shop/category/product

type OptionalCatchAllProps = {
  params: Promise<{ segments?: string[] }>;
};

export default async function OptionalCatchAllPage({ params }: OptionalCatchAllProps) {
  const { segments } = await params;
  
  if (!segments) {
    return <div>Shop Home</div>; // /shop
  }
  
  if (segments.length === 1) {
    return <div>Category: {segments[0]}</div>; // /shop/electronics
  }
  
  return <div>Product: {segments.join(' > ')}</div>; // /shop/electronics/laptop
}
```

#### Azure UpdateSnap の Dynamic Routes 実装

```typescript
// app/[slug]/page.tsx - Azure UpdateSnap のメインルート
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { headers } from 'next/headers';
import type { Metadata } from 'next';

// 型定義: Azure Update ID の形式
type AzureUpdateId = string; // 数値ID または GUID

// ページプロパティの型定義
type Props = {
  params: Promise<{ slug: AzureUpdateId }>;
  searchParams: Promise<{ 
    debug?: string;
    theme?: 'light' | 'dark';
    lang?: 'ja' | 'en';
  }>;
};

// ID 形式の検証関数
function validateAzureUpdateId(id: string): { 
  isValid: boolean; 
  type: 'numeric' | 'guid' | 'invalid' 
} {
  // 数値ID の検証（Microsoft Message Center形式）
  const numericPattern = /^\d{6,8}$/;
  if (numericPattern.test(id)) {
    return { isValid: true, type: 'numeric' };
  }
  
  // GUID の検証（UUID v4 形式）
  const guidPattern = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  if (guidPattern.test(id)) {
    return { isValid: true, type: 'guid' };
  }
  
  return { isValid: false, type: 'invalid' };
}

// User-Agent ベースのBot検出
function detectBot(userAgent: string | null): {
  isBot: boolean;
  botType?: string;
} {
  if (!userAgent) {
    return { isBot: false };
  }
  
  const botPatterns = {
    googlebot: /googlebot/i,
    bingbot: /bingbot/i,
    slurp: /slurp/i, // Yahoo
    duckduckbot: /duckduckbot/i,
    baiduspider: /baiduspider/i,
    yandexbot: /yandexbot/i,
    facebookbot: /facebookexternalhit/i,
    twitterbot: /twitterbot/i,
    linkedinbot: /linkedinbot/i,
    whatsapp: /whatsapp/i,
    telegram: /telegrambot/i,
    discordbot: /discordbot/i,
    slackbot: /slackbot/i,
  };
  
  for (const [botType, pattern] of Object.entries(botPatterns)) {
    if (pattern.test(userAgent)) {
      return { isBot: true, botType };
    }
  }
  
  return { isBot: false };
}

// メインページコンポーネント
export default async function UpdatePage({ params, searchParams }: Props) {
  const { slug } = await params;
  const { debug, theme = 'light', lang = 'ja' } = await searchParams;
  
  // ID の検証
  const validation = validateAzureUpdateId(slug);
  if (!validation.isValid) {
    notFound();
  }
  
  // リクエストヘッダーの取得
  const headersList = await headers();
  const userAgent = headersList.get('user-agent');
  const botDetection = detectBot(userAgent);
  
  // デバッグモードの判定
  const isDebugMode = debug === 'true';
  
  console.log('Request Info:', {
    slug,
    idType: validation.type,
    isBot: botDetection.isBot,
    botType: botDetection.botType,
    isDebugMode,
    userAgent: userAgent?.substring(0, 100) + '...', // ログ用に短縮
  });
  
  // データ取得（詳細は Step 4 で実装）
  const update = await getOrFetchAzureUpdate(slug);
  
  if (!update) {
    notFound();
  }
  
  // レンダリング戦略の決定
  if (botDetection.isBot || isDebugMode) {
    // Bot または Debug モード: フルコンテンツを表示
    return (
      <Suspense fallback={<UpdateViewSkeleton />}>
        <AzureUpdateView 
          update={update}
          theme={theme}
          lang={lang}
          debugInfo={isDebugMode ? {
            botDetection,
            requestHeaders: Object.fromEntries(headersList.entries()),
          } : undefined}
        />
      </Suspense>
    );
  }
  
  // 一般ユーザー: リダイレクトページを表示
  return (
    <RedirectingView 
      targetUrl={update.detailsUrl}
      update={update}
      theme={theme}
    />
  );
}

// メタデータの動的生成
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  
  const validation = validateAzureUpdateId(slug);
  if (!validation.isValid) {
    return {
      title: 'Invalid ID | Azure Update Viewer',
      description: 'The specified update ID is not valid.',
    };
  }
  
  try {
    const update = await getOrFetchAzureUpdate(slug);
    
    if (!update) {
      return {
        title: 'Update Not Found | Azure Update Viewer',
        description: 'The specified Azure update could not be found.',
      };
    }
    
    const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';
    const ogImageUrl = `${baseUrl}/api/og/${slug}`;
    
    return {
      title: `${update.title} | Azure Update Viewer`,
      description: update.description || `Azure Update ${slug} の詳細情報`,
      
      // Open Graph メタデータ
      openGraph: {
        title: update.title,
        description: update.description || `Azure Update ${slug}`,
        type: 'article',
        publishedTime: update.publicDisclosureDate,
        tags: update.tags,
        images: [
          {
            url: ogImageUrl,
            width: 1200,
            height: 630,
            alt: update.title,
          },
        ],
      },
      
      // Twitter Card メタデータ
      twitter: {
        card: 'summary_large_image',
        title: update.title,
        description: update.description || `Azure Update ${slug}`,
        images: [ogImageUrl],
      },
      
      // その他のメタデータ
      keywords: update.tags.join(', '),
      authors: [{ name: 'Microsoft Corporation' }],
      robots: {
        index: true,
        follow: true,
        googleBot: {
          index: true,
          follow: true,
          'max-video-preview': -1,
          'max-image-preview': 'large',
          'max-snippet': -1,
        },
      },
    };
  } catch (error) {
    console.error('Error generating metadata:', error);
    
    return {
      title: 'Error | Azure Update Viewer',
      description: 'An error occurred while loading the update information.',
    };
  }
}
```

### 2.2 エラーハンドリングとUI状態管理

#### loading.tsx による段階的ローディング

```typescript
// app/[slug]/loading.tsx - ローディング UI
export default function UpdateLoading() {
  return (
    <div className="min-h-screen bg-gray-50">
      <div className="max-w-4xl mx-auto p-8">
        {/* ヘッダースケルトン */}
        <div className="mb-8">
          <div className="skeleton-title mb-4" />
          <div className="flex items-center gap-4 mb-6">
            <div className="skeleton h-4 w-20" />
            <div className="skeleton h-4 w-24" />
          </div>
        </div>
        
        {/* タグスケルトン */}
        <div className="flex gap-2 mb-6">
          {[...Array(3)].map((_, i) => (
            <div key={i} className="skeleton h-6 w-16 rounded-full" />
          ))}
        </div>
        
        {/* コンテンツスケルトン */}
        <div className="space-y-4">
          {[...Array(4)].map((_, i) => (
            <div key={i} className="skeleton-text" style={{ 
              width: `${Math.random() * 40 + 60}%` 
            }} />
          ))}
        </div>
        
        {/* 説明文スケルトン */}
        <div className="mt-8 space-y-3">
          {[...Array(6)].map((_, i) => (
            <div key={i} className="skeleton-text" />
          ))}
        </div>
      </div>
    </div>
  );
}
```

#### error.tsx による包括的エラーハンドリング

```typescript
// app/[slug]/error.tsx - エラー境界
'use client';

import { useEffect } from 'react';
import Link from 'next/link';

type ErrorProps = {
  error: Error & { digest?: string };
  reset: () => void;
};

export default function UpdateError({ error, reset }: ErrorProps) {
  useEffect(() => {
    // エラー報告（本番環境では外部サービスに送信）
    console.error('Update page error:', {
      message: error.message,
      digest: error.digest,
      stack: error.stack,
      timestamp: new Date().toISOString(),
    });
  }, [error]);
  
  // エラータイプの判定
  const getErrorType = (error: Error): {
    type: string;
    title: string;
    description: string;
    showRetry: boolean;
  } => {
    if (error.message.includes('NETWORK_ERROR')) {
      return {
        type: 'network',
        title: 'ネットワークエラー',
        description: 'ネットワーク接続を確認してください。',
        showRetry: true,
      };
    }
    
    if (error.message.includes('API_ERROR')) {
      return {
        type: 'api',
        title: 'データ取得エラー',
        description: 'Azure 更新情報の取得に失敗しました。',
        showRetry: true,
      };
    }
    
    if (error.message.includes('PARSING_ERROR')) {
      return {
        type: 'parsing',
        title: 'データ解析エラー',
        description: '取得したデータの形式が正しくありません。',
        showRetry: false,
      };
    }
    
    return {
      type: 'unknown',
      title: '予期しないエラー',
      description: 'アプリケーションでエラーが発生しました。',
      showRetry: true,
    };
  };
  
  const errorInfo = getErrorType(error);
  
  return (
    <div className="min-h-screen bg-gray-50 flex items-center justify-center">
      <div className="max-w-md mx-auto text-center p-8">
        {/* エラーアイコン */}
        <div className="w-16 h-16 mx-auto mb-6 bg-red-100 rounded-full flex items-center justify-center">
          <svg className="w-8 h-8 text-red-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L3.732 16.5c-.77.833.192 2.5 1.732 2.5z" />
          </svg>
        </div>
        
        {/* エラー情報 */}
        <h1 className="text-2xl font-bold text-gray-900 mb-2">
          {errorInfo.title}
        </h1>
        
        <p className="text-gray-600 mb-8">
          {errorInfo.description}
        </p>
        
        {/* アクション */}
        <div className="space-y-4">
          {errorInfo.showRetry && (
            <button
              onClick={reset}
              className="w-full bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700 transition-colors"
            >
              再試行
            </button>
          )}
          
          <Link
            href="/"
            className="block w-full bg-gray-200 text-gray-900 px-6 py-3 rounded-lg hover:bg-gray-300 transition-colors"
          >
            ホームに戻る
          </Link>
        </div>
        
        {/* 開発環境でのエラー詳細 */}
        {process.env.NODE_ENV === 'development' && (
          <details className="mt-8 text-left">
            <summary className="cursor-pointer text-sm text-gray-500 hover:text-gray-700">
              エラー詳細（開発環境のみ）
            </summary>
            <pre className="mt-4 p-4 bg-gray-100 rounded text-xs overflow-auto">
              {error.message}
              {error.stack && '\n\n' + error.stack}
            </pre>
          </details>
        )}
      </div>
    </div>
  );
}
```

#### not-found.tsx による 404 ページ

```typescript
// app/not-found.tsx - 404 ページ
import Link from 'next/link';
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'ページが見つかりません | Azure Update Viewer',
  description: 'お探しのページが見つかりませんでした。',
};

export default function NotFound() {
  return (
    <div className="min-h-screen bg-gray-50 flex items-center justify-center">
      <div className="max-w-md mx-auto text-center p-8">
        {/* 404 アイコン */}
        <div className="w-24 h-24 mx-auto mb-6 bg-blue-100 rounded-full flex items-center justify-center">
          <span className="text-4xl font-bold text-blue-600">404</span>
        </div>
        
        {/* メッセージ */}
        <h1 className="text-3xl font-bold text-gray-900 mb-4">
          ページが見つかりません
        </h1>
        
        <p className="text-gray-600 mb-8">
          お探しのAzure更新情報が見つかりませんでした。<br />
          IDが正しいかご確認ください。
        </p>
        
        {/* 候補アクション */}
        <div className="space-y-4">
          <Link
            href="/"
            className="block w-full bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700 transition-colors"
          >
            ホームに戻る
          </Link>
          
          <button
            onClick={() => window.history.back()}
            className="block w-full bg-gray-200 text-gray-900 px-6 py-3 rounded-lg hover:bg-gray-300 transition-colors"
          >
            前のページに戻る
          </button>
        </div>
        
        {/* 検索候補 */}
        <div className="mt-8 p-4 bg-blue-50 rounded-lg">
          <h3 className="font-semibold text-blue-900 mb-2">こんなことをお探しですか？</h3>
          <ul className="text-sm text-blue-700 space-y-1">
            <li>• Azure 更新情報の一覧を見る</li>
            <li>• 最新の更新情報を確認する</li>
            <li>• 特定のサービスの更新情報を検索する</li>
          </ul>
        </div>
      </div>
    </div>
  );
}
```

### 2.3 ミドルウェアによるリクエスト処理

#### middleware.ts の実装

```typescript
// src/middleware.ts - リクエスト前処理
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

// Bot 検出のための User-Agent パターン
const BOT_USER_AGENTS = [
  /googlebot/i,
  /bingbot/i,
  /slurp/i,
  /duckduckbot/i,
  /baiduspider/i,
  /yandexbot/i,
  /facebookexternalhit/i,
  /twitterbot/i,
  /linkedinbot/i,
  /whatsapp/i,
  /telegrambot/i,
  /discordbot/i,
  /slackbot/i,
];

// レート制限のための簡単な実装
class RateLimiter {
  private requests = new Map<string, number[]>();
  private readonly maxRequests = 100; // 1分間に100リクエスト
  private readonly windowMs = 60 * 1000; // 1分

  isAllowed(identifier: string): boolean {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    // 既存のリクエスト履歴を取得
    const requests = this.requests.get(identifier) || [];
    
    // ウィンドウ外のリクエストを除去
    const validRequests = requests.filter(time => time > windowStart);
    
    // 制限チェック
    if (validRequests.length >= this.maxRequests) {
      return false;
    }
    
    // 新しいリクエストを記録
    validRequests.push(now);
    this.requests.set(identifier, validRequests);
    
    return true;
  }
  
  // 定期的なクリーンアップ
  cleanup() {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    for (const [key, requests] of this.requests.entries()) {
      const validRequests = requests.filter(time => time > windowStart);
      if (validRequests.length === 0) {
        this.requests.delete(key);
      } else {
        this.requests.set(key, validRequests);
      }
    }
  }
}

const rateLimiter = new RateLimiter();

// 定期的なクリーンアップ（5分ごと）
if (typeof setInterval !== 'undefined') {
  setInterval(() => rateLimiter.cleanup(), 5 * 60 * 1000);
}

export function middleware(request: NextRequest) {
  const { pathname, search } = request.nextUrl;
  const userAgent = request.headers.get('user-agent') || '';
  
  // Bot 検出
  const isBot = BOT_USER_AGENTS.some(pattern => pattern.test(userAgent));
  
  // IP アドレスの取得（複数のヘッダーをチェック）
  const forwardedFor = request.headers.get('x-forwarded-for');
  const realIp = request.headers.get('x-real-ip');
  const ip = forwardedFor?.split(',')[0] || realIp || request.ip || 'unknown';
  
  // 地理的情報の取得（Vercel の場合）
  const country = request.geo?.country;
  const region = request.geo?.region;
  const city = request.geo?.city;
  
  console.log('Middleware - Request Info:', {
    pathname,
    ip: ip.substring(0, 10) + '...', // プライバシー保護のため部分的にログ
    isBot,
    country,
    userAgent: userAgent.substring(0, 100) + '...',
  });
  
  // レート制限のチェック（Bot は除外）
  if (!isBot && !rateLimiter.isAllowed(ip)) {
    console.warn('Rate limit exceeded:', { ip, pathname });
    
    return new NextResponse('Rate limit exceeded', {
      status: 429,
      headers: {
        'Retry-After': '60',
        'Content-Type': 'text/plain',
      },
    });
  }
  
  // セキュリティヘッダーの設定
  const response = NextResponse.next();
  
  // カスタムヘッダーの追加
  response.headers.set('x-is-bot', isBot ? '1' : '0');
  response.headers.set('x-user-ip', ip);
  response.headers.set('x-user-country', country || 'unknown');
  response.headers.set('x-timestamp', new Date().toISOString());
  
  // セキュリティヘッダー
  response.headers.set('x-frame-options', 'DENY');
  response.headers.set('x-content-type-options', 'nosniff');
  response.headers.set('referrer-policy', 'origin-when-cross-origin');
  response.headers.set('x-xss-protection', '1; mode=block');
  
  // CSP（Content Security Policy）
  const csp = [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval'", // Next.js requires unsafe-inline
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "font-src 'self' data:",
    "connect-src 'self' https://api.supabase.co https://techcommunity.microsoft.com",
    "frame-ancestors 'none'",
  ].join('; ');
  
  response.headers.set('content-security-policy', csp);
  
  // CORS ヘッダー（API ルートの場合）
  if (pathname.startsWith('/api/')) {
    response.headers.set('access-control-allow-origin', '*');
    response.headers.set('access-control-allow-methods', 'GET, POST, OPTIONS');
    response.headers.set('access-control-allow-headers', 'Content-Type');
  }
  
  // Analytics や監視のためのメタデータ
  response.headers.set('x-route-type', pathname.startsWith('/api/') ? 'api' : 'page');
  
  return response;
}

// ミドルウェアの適用範囲
export const config = {
  matcher: [
    // API routes を含む全てのパス
    '/((?!_next/static|_next/image|favicon.ico).*)',
    
    // より具体的な制御が必要な場合
    // {
    //   source: '/api/:path*',
    //   has: [
    //     {
    //       type: 'header',
    //       key: 'authorization',
    //     }
    //   ]
    // }
  ],
};
```

## 📖 第3章: Server Components と Client Components の戦略的設計

### 3.1 コンポーネント境界の設計原理

#### Server Components の活用戦略

```typescript
// Server Component: 데이터 フェッチングとSEO最適化
// app/[slug]/AzureUpdateView.tsx
import { Suspense } from 'react';
import type { AzureUpdate } from '@/types/azure';

type Props = {
  update: AzureUpdate;
  theme?: 'light' | 'dark';
  lang?: 'ja' | 'en';
  debugInfo?: any;
};

// Server Component - データを受け取り、静的コンテンツをレンダリング
export default function AzureUpdateView({ update, theme = 'light', lang = 'ja', debugInfo }: Props) {
  return (
    <main className={`max-w-4xl mx-auto p-8 ${theme === 'dark' ? 'dark' : ''}`}>
      <article className="bg-white dark:bg-gray-900 rounded-lg shadow-lg p-8">
        {/* ヘッダー情報 - Server Component で処理 */}
        <header className="mb-8">
          <h1 className="text-3xl font-bold text-gray-900 dark:text-white mb-4">
            {update.title}
          </h1>
          
          <div className="flex flex-wrap items-center gap-4 text-sm text-gray-600 dark:text-gray-400">
            <div className="flex items-center gap-2">
              <svg className="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                <path fillRule="evenodd" d="M10 2a8 8 0 100 16 8 8 0 000-16zM8.5 4a1.5 1.5 0 11-3 0 1.5 1.5 0 013 0zm-1.5 7a.5.5 0 01.5-.5h2a.5.5 0 010 1H7.5v2.5a.5.5 0 01-1 0V11z" clipRule="evenodd" />
              </svg>
              <span>ID: {update.id}</span>
            </div>
            
            {update.publicDisclosureDate && (
              <div className="flex items-center gap-2">
                <svg className="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                  <path fillRule="evenodd" d="M6 2a1 1 0 00-1 1v1H4a2 2 0 00-2 2v10a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2h-1V3a1 1 0 10-2 0v1H7V3a1 1 0 00-1-1zm0 5a1 1 0 000 2h8a1 1 0 100-2H6z" clipRule="evenodd" />
                </svg>
                <time dateTime={update.publicDisclosureDate}>
                  {new Date(update.publicDisclosureDate).toLocaleDateString(lang === 'ja' ? 'ja-JP' : 'en-US')}
                </time>
              </div>
            )}
            
            {update.guid && (
              <div className="flex items-center gap-2">
                <svg className="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                  <path fillRule="evenodd" d="M3 4a1 1 0 011-1h12a1 1 0 011 1v12a1 1 0 01-1 1H4a1 1 0 01-1-1V4zm1 0v12h12V4H4z" clipRule="evenodd" />
                </svg>
                <span className="font-mono text-xs">{update.guid}</span>
              </div>
            )}
          </div>
        </header>
        
        {/* タグ表示 - Server Component で処理 */}
        {update.tags.length > 0 && (
          <section className="mb-8">
            <h2 className="sr-only">関連タグ</h2>
            <div className="flex flex-wrap gap-2">
              {update.tags.map((tag) => (
                <span
                  key={tag}
                  className="inline-flex items-center px-3 py-1 bg-blue-100 dark:bg-blue-900 text-blue-800 dark:text-blue-200 text-sm rounded-full"
                >
                  {tag}
                </span>
              ))}
            </div>
          </section>
        )}
        
        {/* コンテンツ表示 - Server Component で処理 */}
        <section className="prose dark:prose-invert max-w-none">
          <h2 className="text-xl font-semibold mb-4">概要</h2>
          <p className="text-gray-700 dark:text-gray-300 leading-relaxed mb-6">
            {update.description}
          </p>
          
          {update.impactDescription && (
            <>
              <h2 className="text-xl font-semibold mb-4">影響</h2>
              <p className="text-gray-700 dark:text-gray-300 leading-relaxed mb-6">
                {update.impactDescription}
              </p>
            </>
          )}
        </section>
        
        {/* 外部リンク - Server Component で処理 */}
        <footer className="mt-8 pt-6 border-t border-gray-200 dark:border-gray-700">
          <a
            href={update.detailsUrl}
            target="_blank"
            rel="noopener noreferrer"
            className="inline-flex items-center text-blue-600 dark:text-blue-400 hover:text-blue-800 dark:hover:text-blue-300 font-medium"
          >
            Microsoft 公式ページで詳細を見る
            <svg className="w-4 h-4 ml-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14" />
            </svg>
          </a>
        </footer>
        
        {/* インタラクティブコンポーネント - Client Component */}
        <Suspense fallback={<div className="mt-8 animate-pulse h-32 bg-gray-200 dark:bg-gray-700 rounded" />}>
          <InteractiveSection 
            updateId={update.id.toString()}
            initialLikes={0}
            theme={theme}
          />
        </Suspense>
        
        {/* デバッグ情報 - Server Component で処理 */}
        {debugInfo && (
          <DebugInfoSection debugInfo={debugInfo} />
        )}
      </article>
    </main>
  );
}

// デバッグ情報コンポーネント（Server Component）
function DebugInfoSection({ debugInfo }: { debugInfo: any }) {
  return (
    <details className="mt-8 p-4 bg-yellow-50 dark:bg-yellow-900/20 rounded-lg">
      <summary className="cursor-pointer font-semibold text-yellow-800 dark:text-yellow-200">
        デバッグ情報
      </summary>
      <pre className="mt-4 text-xs overflow-auto bg-white dark:bg-gray-800 p-4 rounded">
        {JSON.stringify(debugInfo, null, 2)}
      </pre>
    </details>
  );
}
```

#### Client Components の戦略的実装

```typescript
// Client Component: インタラクティブ機能のみ
// app/[slug]/InteractiveSection.tsx
'use client';

import { useState, useEffect } from 'react';

type Props = {
  updateId: string;
  initialLikes: number;
  theme: 'light' | 'dark';
};

export default function InteractiveSection({ updateId, initialLikes, theme }: Props) {
  const [likes, setLikes] = useState(initialLikes);
  const [isLiked, setIsLiked] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  
  // ローカルストレージから like 状態を復元
  useEffect(() => {
    const likedUpdates = JSON.parse(localStorage.getItem('likedUpdates') || '[]');
    setIsLiked(likedUpdates.includes(updateId));
  }, [updateId]);
  
  const handleLike = async () => {
    if (isLoading) return;
    
    setIsLoading(true);
    
    try {
      // いいね状態の切り替え
      const newIsLiked = !isLiked;
      const newLikes = newIsLiked ? likes + 1 : likes - 1;
      
      // 楽観的更新
      setIsLiked(newIsLiked);
      setLikes(newLikes);
      
      // ローカルストレージの更新
      const likedUpdates = JSON.parse(localStorage.getItem('likedUpdates') || '[]');
      if (newIsLiked) {
        likedUpdates.push(updateId);
      } else {
        const index = likedUpdates.indexOf(updateId);
        if (index > -1) {
          likedUpdates.splice(index, 1);
        }
      }
      localStorage.setItem('likedUpdates', JSON.stringify(likedUpdates));
      
      // サーバーへの送信（実際のプロダクションでは実装）
      // await fetch(`/api/likes/${updateId}`, {
      //   method: 'POST',
      //   headers: { 'Content-Type': 'application/json' },
      //   body: JSON.stringify({ liked: newIsLiked })
      // });
      
    } catch (error) {
      console.error('Failed to update like status:', error);
      
      // エラー時は元の状態に戻す
      setIsLiked(!isLiked);
      setLikes(likes);
    } finally {
      setIsLoading(false);
    }
  };
  
  const handleShare = async () => {
    const url = window.location.href;
    const text = `Azure Update ${updateId} をチェック！`;
    
    if (navigator.share) {
      // Web Share API が利用可能な場合
      try {
        await navigator.share({
          title: text,
          url: url,
        });
      } catch (error) {
        console.log('Share cancelled or failed:', error);
      }
    } else {
      // フォールバック: クリップボードにコピー
      try {
        await navigator.clipboard.writeText(url);
        alert('URLをクリップボードにコピーしました！');
      } catch (error) {
        console.error('Failed to copy to clipboard:', error);
        // 最終フォールバック: プロンプトでURL表示
        prompt('URLをコピーしてください:', url);
      }
    }
  };
  
  return (
    <div className="mt-8 pt-6 border-t border-gray-200 dark:border-gray-700">
      <div className="flex items-center gap-4">
        {/* いいねボタン */}
        <button
          onClick={handleLike}
          disabled={isLoading}
          className={`
            flex items-center gap-2 px-4 py-2 rounded-lg transition-all duration-200
            ${isLiked 
              ? 'bg-red-100 text-red-700 hover:bg-red-200 dark:bg-red-900/20 dark:text-red-300' 
              : 'bg-gray-100 text-gray-700 hover:bg-gray-200 dark:bg-gray-700 dark:text-gray-300'
            }
            ${isLoading ? 'opacity-50 cursor-not-allowed' : 'hover:scale-105'}
          `}
          aria-label={isLiked ? 'いいねを取り消す' : 'いいねする'}
        >
          <svg 
            className={`w-5 h-5 transition-colors ${isLiked ? 'fill-current' : 'fill-none stroke-current'}`}
            viewBox="0 0 24 24"
          >
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4.318 6.318a4.5 4.5 0 000 6.364L12 20.364l7.682-7.682a4.5 4.5 0 00-6.364-6.364L12 7.636l-1.318-1.318a4.5 4.5 0 00-6.364 0z" />
          </svg>
          <span className="font-medium">{likes}</span>
        </button>
        
        {/* シェアボタン */}
        <button
          onClick={handleShare}
          className="flex items-center gap-2 px-4 py-2 bg-blue-100 text-blue-700 rounded-lg hover:bg-blue-200 dark:bg-blue-900/20 dark:text-blue-300 transition-all duration-200 hover:scale-105"
          aria-label="この更新情報をシェア"
        >
          <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M8.684 13.342C8.886 12.938 9 12.482 9 12c0-.482-.114-.938-.316-1.342m0 2.684a3 3 0 110-2.684m0 2.684l6.632 3.316m-6.632-6l6.632-3.316m0 0a3 3 0 105.367-2.684 3 3 0 00-5.367 2.684zm0 9.316a3 3 0 105.367 2.684 3 3 0 00-5.367-2.684z" />
          </svg>
          <span className="font-medium">シェア</span>
        </button>
        
        {/* ブックマークボタン */}
        <BookmarkButton updateId={updateId} theme={theme} />
      </div>
    </div>
  );
}

// ブックマーク機能のコンポーネント
function BookmarkButton({ updateId, theme }: { updateId: string; theme: 'light' | 'dark' }) {
  const [isBookmarked, setIsBookmarked] = useState(false);
  
  useEffect(() => {
    const bookmarks = JSON.parse(localStorage.getItem('bookmarkedUpdates') || '[]');
    setIsBookmarked(bookmarks.includes(updateId));
  }, [updateId]);
  
  const toggleBookmark = () => {
    const bookmarks = JSON.parse(localStorage.getItem('bookmarkedUpdates') || '[]');
    
    if (isBookmarked) {
      const index = bookmarks.indexOf(updateId);
      if (index > -1) {
        bookmarks.splice(index, 1);
      }
    } else {
      bookmarks.push(updateId);
    }
    
    localStorage.setItem('bookmarkedUpdates', JSON.stringify(bookmarks));
    setIsBookmarked(!isBookmarked);
  };
  
  return (
    <button
      onClick={toggleBookmark}
      className={`
        flex items-center gap-2 px-4 py-2 rounded-lg transition-all duration-200
        ${isBookmarked 
          ? 'bg-yellow-100 text-yellow-700 hover:bg-yellow-200 dark:bg-yellow-900/20 dark:text-yellow-300' 
          : 'bg-gray-100 text-gray-700 hover:bg-gray-200 dark:bg-gray-700 dark:text-gray-300'
        }
        hover:scale-105
      `}
      aria-label={isBookmarked ? 'ブックマークを削除' : 'ブックマークに追加'}
    >
      <svg 
        className={`w-5 h-5 ${isBookmarked ? 'fill-current' : 'fill-none stroke-current'}`}
        viewBox="0 0 24 24"
      >
        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 5a2 2 0 012-2h10a2 2 0 012 2v16l-7-3.5L5 21V5z" />
      </svg>
      <span className="font-medium">
        {isBookmarked ? 'ブックマーク済み' : 'ブックマーク'}
      </span>
    </button>
  );
}
```

#### RedirectingView - ユーザーエクスペリエンス最適化

```typescript
// app/[slug]/RedirectingView.tsx - リダイレクト処理
'use client';

import { useEffect, useState, useCallback } from 'react';
import { AzureUpdate } from '@/types/azure';

type Props = {
  targetUrl: string;
  update: AzureUpdate;
  theme?: 'light' | 'dark';
};

export default function RedirectingView({ targetUrl, update, theme = 'light' }: Props) {
  const [countdown, setCountdown] = useState(3);
  const [isRedirecting, setIsRedirecting] = useState(false);
  const [progress, setProgress] = useState(0);
  
  // リダイレクト実行
  const executeRedirect = useCallback(() => {
    setIsRedirecting(true);
    window.location.href = targetUrl;
  }, [targetUrl]);
  
  // カウントダウンとプログレスバー
  useEffect(() => {
    const startTime = Date.now();
    const duration = 3000; // 3秒
    
    const timer = setInterval(() => {
      const elapsed = Date.now() - startTime;
      const remaining = Math.max(0, duration - elapsed);
      const newCountdown = Math.ceil(remaining / 1000);
      const newProgress = Math.min(100, (elapsed / duration) * 100);
      
      setCountdown(newCountdown);
      setProgress(newProgress);
      
      if (remaining <= 0) {
        clearInterval(timer);
        executeRedirect();
      }
    }, 50); // 滑らかな更新のため50ms間隔
    
    return () => clearInterval(timer);
  }, [executeRedirect]);
  
  // キーボードショートカット
  useEffect(() => {
    const handleKeyPress = (event: KeyboardEvent) => {
      if (event.key === 'Enter' || event.key === ' ') {
        event.preventDefault();
        executeRedirect();
      }
      
      if (event.key === 'Escape') {
        event.preventDefault();
        window.history.back();
      }
    };
    
    window.addEventListener('keydown', handleKeyPress);
    return () => window.removeEventListener('keydown', handleKeyPress);
  }, [executeRedirect]);
  
  return (
    <div className={`min-h-screen flex items-center justify-center p-8 ${theme === 'dark' ? 'dark bg-gray-900' : 'bg-gray-50'}`}>
      <div className="max-w-2xl mx-auto text-center">
        {/* メインメッセージ */}
        <div className="mb-8">
          <div className="w-20 h-20 mx-auto mb-6 bg-blue-100 dark:bg-blue-900/20 rounded-full flex items-center justify-center">
            {isRedirecting ? (
              <svg className="w-10 h-10 text-blue-600 dark:text-blue-400 animate-spin" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
              </svg>
            ) : (
              <span className="text-3xl font-bold text-blue-600 dark:text-blue-400">
                {countdown}
              </span>
            )}
          </div>
          
          <h1 className="text-2xl font-bold text-gray-900 dark:text-white mb-4">
            Microsoft 公式ページにリダイレクトしています
          </h1>
          
          <p className="text-gray-600 dark:text-gray-400 mb-6">
            {isRedirecting ? 
              'リダイレクト中...' : 
              `${countdown} 秒後に自動的にリダイレクトされます`
            }
          </p>
        </div>
        
        {/* プログレスバー */}
        <div className="mb-8">
          <div className="w-full bg-gray-200 dark:bg-gray-700 rounded-full h-2 mb-2">
            <div 
              className="bg-blue-600 dark:bg-blue-500 h-2 rounded-full transition-all duration-100 ease-out"
              style={{ width: `${progress}%` }}
            />
          </div>
          <p className="text-sm text-gray-500 dark:text-gray-400">
            {Math.round(progress)}% 完了
          </p>
        </div>
        
        {/* 更新情報プレビュー */}
        <div className="mb-8 p-6 bg-white dark:bg-gray-800 rounded-lg shadow-md text-left">
          <h2 className="text-lg font-semibold text-gray-900 dark:text-white mb-3">
            {update.title}
          </h2>
          
          <p className="text-gray-600 dark:text-gray-400 text-sm line-clamp-3">
            {update.description}
          </p>
          
          {update.tags.length > 0 && (
            <div className="flex flex-wrap gap-2 mt-4">
              {update.tags.slice(0, 3).map((tag) => (
                <span
                  key={tag}
                  className="inline-flex items-center px-2 py-1 bg-blue-100 dark:bg-blue-900/20 text-blue-800 dark:text-blue-200 text-xs rounded"
                >
                  {tag}
                </span>
              ))}
              {update.tags.length > 3 && (
                <span className="inline-flex items-center px-2 py-1 bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-400 text-xs rounded">
                  +{update.tags.length - 3}
                </span>
              )}
            </div>
          )}
        </div>
        
        {/* アクションボタン */}
        <div className="space-y-4">
          <button
            onClick={executeRedirect}
            disabled={isRedirecting}
            className="w-full bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors font-medium"
          >
            今すぐリダイレクト
          </button>
          
          <button
            onClick={() => window.history.back()}
            disabled={isRedirecting}
            className="w-full bg-gray-200 dark:bg-gray-700 text-gray-900 dark:text-gray-300 px-6 py-3 rounded-lg hover:bg-gray-300 dark:hover:bg-gray-600 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
          >
            前のページに戻る
          </button>
        </div>
        
        {/* キーボードショートカット説明 */}
        <div className="mt-8 text-sm text-gray-500 dark:text-gray-400">
          <p>キーボードショートカット:</p>
          <p><kbd className="px-2 py-1 bg-gray-200 dark:bg-gray-700 rounded text-xs">Enter</kbd> または <kbd className="px-2 py-1 bg-gray-200 dark:bg-gray-700 rounded text-xs">Space</kbd>: 今すぐリダイレクト</p>
          <p><kbd className="px-2 py-1 bg-gray-200 dark:bg-gray-700 rounded text-xs">Esc</kbd>: 前のページに戻る</p>
        </div>
      </div>
    </div>
  );
}
```

## 🚀 第4章: パフォーマンス最適化とストリーミング

### 4.1 Suspense による段階的ローディング

#### 段階的データ読み込みの実装

```typescript
// 段階的ローディング戦略
export default function UpdatePage({ params, searchParams }: Props) {
  return (
    <div>
      {/* 即座に表示される静的な部分 */}
      <header className="bg-blue-600 text-white p-4">
        <h1>Azure Update Viewer</h1>
      </header>
      
      {/* 基本情報を先に読み込み */}
      <Suspense fallback={<BasicInfoSkeleton />}>
        <BasicUpdateInfo params={params} />
      </Suspense>
      
      {/* 詳細情報は遅延読み込み */}
      <Suspense fallback={<DetailsSkeleton />}>
        <DetailedUpdateInfo params={params} />
      </Suspense>
      
      {/* 関連情報も個別に読み込み */}
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedUpdates params={params} />
      </Suspense>
    </div>
  );
}

// 基本情報（高速なクエリ）
async function BasicUpdateInfo({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  
  // 軽量なデータのみ取得
  const basicInfo = await getBasicUpdateInfo(slug);
  
  return (
    <div className="p-6">
      <h2 className="text-2xl font-bold">{basicInfo.title}</h2>
      <p className="text-gray-600">{basicInfo.summary}</p>
    </div>
  );
}

// 詳細情報（重いクエリ）
async function DetailedUpdateInfo({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  
  // 時間のかかる処理
  const [details, analysis] = await Promise.all([
    getDetailedInfo(slug),
    performContentAnalysis(slug), // 重い処理
  ]);
  
  return (
    <div className="p-6">
      <div className="prose max-w-none">
        <p>{details.description}</p>
        <div className="mt-6">
          <h3>影響分析</h3>
          <p>{analysis.impact}</p>
        </div>
      </div>
    </div>
  );
}
```

### 4.2 キャッシュ戦略の実装

#### Next.js キャッシュレイヤーの活用

```typescript
// キャッシュ戦略の実装
export default async function UpdatePage({ params }: Props) {
  const { slug } = await params;
  
  // fetch() での自動キャッシュ
  const update = await fetch(`${API_BASE}/updates/${slug}`, {
    next: {
      revalidate: 3600, // 1時間キャッシュ
      tags: [`update-${slug}`], // タグベースの無効化
    },
  }).then(res => res.json());
  
  // unstable_cache での関数キャッシュ
  const analysis = await getCachedAnalysis(slug);
  
  return <UpdateView update={update} analysis={analysis} />;
}

// 関数レベルのキャッシュ
import { unstable_cache } from 'next/cache';

const getCachedAnalysis = unstable_cache(
  async (updateId: string) => {
    // 重い分析処理
    return await performAnalysis(updateId);
  },
  ['update-analysis'], // キャッシュキー
  {
    revalidate: 7200, // 2時間キャッシュ
    tags: ['analysis'], // タグ
  }
);
```

## ✅ 確認ポイント

- [ ] **動的ルート**: `/123456` および GUID 形式の URL が正常に動作する
- [ ] **パラメータ検証**: 無効な ID に対して適切なエラーが表示される
- [ ] **Bot 検出**: User-Agent に基づいて Bot が正しく判定される
- [ ] **リダイレクト**: 一般ユーザーに対してリダイレクトが動作する
- [ ] **ローディング UI**: Suspense による段階的ローディングが動作する
- [ ] **エラーハンドリング**: エラー境界が適切に機能する
- [ ] **メタデータ**: 動的メタデータが正しく生成される
- [ ] **型安全性**: TypeScript による型チェックが完全に機能する

## 🏃 演習問題

### 初級: 基本的なルーティング拡張

1. **言語切り替え**: `?lang=en` パラメータで英語表示に切り替える機能
2. **テーマ切り替え**: `?theme=dark` パラメータでダークモードに切り替える機能
3. **検索パラメータのバリデーション**: Zod を使ったクエリパラメータの検証

### 中級: ユーザー体験の向上

1. **プリフェッチング**: 関連する更新情報のプリフェッチ実装
2. **オフライン対応**: Service Worker による基本的なオフライン機能
3. **パフォーマンス監視**: Core Web Vitals の測定と表示

### 上級: 高度な最適化

1. **増分静的再生成**: ISR（Incremental Static Regeneration）の実装
2. **Edge-side Includes**: 部分的な SSG と SSR の組み合わせ
3. **A/B テスト**: 異なるレイアウトのランダム表示

### 演習解答例

#### 1. 言語切り替え機能

```typescript
// i18n 対応の実装
type Translations = {
  ja: {
    title: string;
    description: string;
    publishedAt: string;
    tags: string;
    readMore: string;
  };
  en: {
    title: string;
    description: string;
    publishedAt: string;
    tags: string;
    readMore: string;
  };
};

const translations: Translations = {
  ja: {
    title: 'Azure 更新情報',
    description: '説明',
    publishedAt: '公開日',
    tags: 'タグ',
    readMore: '詳細を見る',
  },
  en: {
    title: 'Azure Update',
    description: 'Description',
    publishedAt: 'Published at',
    tags: 'Tags',
    readMore: 'Read more',
  },
};

export default async function UpdatePage({ params, searchParams }: Props) {
  const { slug } = await params;
  const { lang = 'ja' } = await searchParams;
  
  const t = translations[lang as keyof Translations] || translations.ja;
  const update = await getOrFetchAzureUpdate(slug);
  
  if (!update) notFound();
  
  return (
    <div>
      <h1>{t.title}: {update.title}</h1>
      <p>{t.description}: {update.description}</p>
      <time>{t.publishedAt}: {update.publicDisclosureDate}</time>
    </div>
  );
}
```

#### 2. パフォーマンス監視の実装

```typescript
// Core Web Vitals の測定
'use client';

import { useEffect } from 'react';

export function WebVitals() {
  useEffect(() => {
    import('web-vitals').then(({ getCLS, getFID, getFCP, getLCP, getTTFB }) => {
      getCLS(console.log);
      getFID(console.log);
      getFCP(console.log);
      getLCP(console.log);
      getTTFB(console.log);
    });
  }, []);
  
  return null;
}

// app/layout.tsx に追加
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body>
        {children}
        {process.env.NODE_ENV === 'development' && <WebVitals />}
      </body>
    </html>
  );
}
```

## 🔗 参考リンク

### 公式ドキュメント
- [Dynamic Routes](https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes)
- [Loading UI and Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [Error Handling](https://nextjs.org/docs/app/building-your-application/routing/error-handling)
- [Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [Server and Client Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)

### パフォーマンス関連
- [React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)
- [Web Vitals](https://web.dev/vitals/)
- [Next.js Caching](https://nextjs.org/docs/app/building-your-application/caching)

### セキュリティ
- [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [OWASP Security Headers](https://owasp.org/www-project-secure-headers/)

---

**次のステップ**: ルーティングシステムの理解が深まったら、[Step 3: API ルートの実装](../step-03-api-routes) に進んで、Next.js の Route Handlers と外部 API 連携について学びましょう！