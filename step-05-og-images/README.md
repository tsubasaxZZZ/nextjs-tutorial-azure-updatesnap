# Step 5: 動的 OG 画像生成

このステップでは、@vercel/og を使って動的な OG 画像を生成し、ソーシャルメディアでの表示を最適化します。

## 🎯 このステップで学ぶこと

- @vercel/og による動的画像生成
- Edge Runtime での画像生成
- メタデータの最適化
- フォントの埋め込み
- パフォーマンス最適化

## 📖 解説

### OG 画像とは

Open Graph 画像は、SNS でリンクを共有した際に表示されるプレビュー画像です。

### @vercel/og の利点

- Edge Runtime で高速動作
- React コンポーネントで画像を定義
- 動的なコンテンツに対応
- 自動的な最適化

## 🛠️ ハンズオン

### 1. @vercel/og のインストール

```bash
npm install @vercel/og
```

### 2. OG 画像生成 API の実装

`src/app/api/og/[id]/route.tsx`:

```tsx
import { ImageResponse } from '@vercel/og';
import { getOrFetchAzureUpdate } from '@/lib/azure-updates';

export const runtime = 'edge';

// フォントの読み込み
const font = fetch(
  new URL('../../../../assets/NotoSansJP-Bold.ttf', import.meta.url)
).then((res) => res.arrayBuffer());

export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params;
    const fontData = await font;

    // Azure Update データを取得
    const update = await getOrFetchAzureUpdate(id);

    if (!update) {
      return new Response('Not found', { status: 404 });
    }

    // タイトルを適切な長さに調整
    const title = update.title.length > 60 
      ? update.title.substring(0, 57) + '...' 
      : update.title;

    // タグを3つまでに制限
    const displayTags = update.tags.slice(0, 3);

    return new ImageResponse(
      (
        <div
          style={{
            height: '100%',
            width: '100%',
            display: 'flex',
            flexDirection: 'column',
            backgroundColor: '#ffffff',
            backgroundImage: 'linear-gradient(to bottom right, #f0f9ff, #e0f2fe)',
          }}
        >
          {/* ヘッダー */}
          <div
            style={{
              display: 'flex',
              alignItems: 'center',
              justifyContent: 'space-between',
              padding: '40px 60px',
              borderBottom: '2px solid #e5e7eb',
            }}
          >
            <div style={{ display: 'flex', alignItems: 'center' }}>
              <div
                style={{
                  width: 60,
                  height: 60,
                  backgroundColor: '#2563eb',
                  borderRadius: '12px',
                  display: 'flex',
                  alignItems: 'center',
                  justifyContent: 'center',
                  marginRight: '20px',
                }}
              >
                <span style={{ color: 'white', fontSize: 32, fontWeight: 'bold' }}>
                  AU
                </span>
              </div>
              <div style={{ display: 'flex', flexDirection: 'column' }}>
                <span style={{ fontSize: 28, fontWeight: 'bold', color: '#1f2937' }}>
                  Azure Update
                </span>
                <span style={{ fontSize: 18, color: '#6b7280' }}>
                  ID: {update.id}
                </span>
              </div>
            </div>
            {update.publicDisclosureDate && (
              <span style={{ fontSize: 20, color: '#6b7280' }}>
                {new Date(update.publicDisclosureDate).toLocaleDateString('ja-JP')}
              </span>
            )}
          </div>

          {/* メインコンテンツ */}
          <div
            style={{
              flex: 1,
              display: 'flex',
              flexDirection: 'column',
              padding: '60px',
            }}
          >
            {/* タイトル */}
            <h1
              style={{
                fontSize: 48,
                fontWeight: 'bold',
                color: '#1f2937',
                marginBottom: '30px',
                lineHeight: 1.3,
              }}
            >
              {title}
            </h1>

            {/* タグ */}
            {displayTags.length > 0 && (
              <div style={{ display: 'flex', gap: '12px', marginBottom: '40px' }}>
                {displayTags.map((tag) => (
                  <span
                    key={tag}
                    style={{
                      backgroundColor: '#dbeafe',
                      color: '#1e40af',
                      padding: '8px 20px',
                      borderRadius: '9999px',
                      fontSize: 18,
                    }}
                  >
                    {tag}
                  </span>
                ))}
                {update.tags.length > 3 && (
                  <span
                    style={{
                      backgroundColor: '#f3f4f6',
                      color: '#6b7280',
                      padding: '8px 20px',
                      borderRadius: '9999px',
                      fontSize: 18,
                    }}
                  >
                    +{update.tags.length - 3}
                  </span>
                )}
              </div>
            )}

            {/* 説明文 */}
            <p
              style={{
                fontSize: 24,
                color: '#4b5563',
                lineHeight: 1.5,
                overflow: 'hidden',
                display: '-webkit-box',
                WebkitLineClamp: 3,
                WebkitBoxOrient: 'vertical',
              }}
            >
              {update.description}
            </p>
          </div>

          {/* フッター */}
          <div
            style={{
              display: 'flex',
              alignItems: 'center',
              justifyContent: 'space-between',
              padding: '30px 60px',
              borderTop: '2px solid #e5e7eb',
              backgroundColor: '#f9fafb',
            }}
          >
            <span style={{ fontSize: 20, color: '#6b7280' }}>
              azure-update.vercel.app
            </span>
            <span style={{ fontSize: 18, color: '#9ca3af' }}>
              Powered by Azure UpdateSnap
            </span>
          </div>
        </div>
      ),
      {
        width: 1200,
        height: 630,
        fonts: [
          {
            name: 'NotoSansJP',
            data: fontData,
            style: 'normal',
            weight: 700,
          },
        ],
      }
    );
  } catch (error) {
    console.error('OG image generation error:', error);
    
    // エラー時のフォールバック画像
    return new ImageResponse(
      (
        <div
          style={{
            height: '100%',
            width: '100%',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            backgroundColor: '#ef4444',
            color: 'white',
            fontSize: 48,
          }}
        >
          Error generating image
        </div>
      ),
      {
        width: 1200,
        height: 630,
      }
    );
  }
}
```

### 3. フォントファイルの配置

`src/assets/NotoSansJP-Bold.ttf` をダウンロードして配置。

### 4. メタデータの更新

`src/app/[slug]/page.tsx` のメタデータ生成を更新：

```tsx
export async function generateMetadata({ params }: Props) {
  const { slug } = await params;
  const update = await getOrFetchAzureUpdate(slug);

  if (!update) {
    return {
      title: 'Not Found | Azure Update Viewer',
    };
  }

  const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';
  const ogImageUrl = `${baseUrl}/api/og/${slug}`;

  return {
    title: `${update.title} | Azure Update Viewer`,
    description: update.description,
    openGraph: {
      title: update.title,
      description: update.description,
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
    twitter: {
      card: 'summary_large_image',
      title: update.title,
      description: update.description,
      images: [ogImageUrl],
    },
  };
}
```

### 5. OG 画像のデバッグツール

`src/app/debug/og/[id]/page.tsx`:

```tsx
type Props = {
  params: Promise<{ id: string }>;
};

export default async function OGDebugPage({ params }: Props) {
  const { id } = await params;
  const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';

  return (
    <main className="min-h-screen p-8">
      <h1 className="text-2xl font-bold mb-4">
        OG 画像デバッグ: {id}
      </h1>
      
      <div className="space-y-4">
        <div>
          <h2 className="text-lg font-semibold mb-2">生成された画像:</h2>
          <img 
            src={`${baseUrl}/api/og/${id}`}
            alt="OG Image"
            className="border rounded shadow-lg"
            width={1200}
            height={630}
          />
        </div>

        <div className="bg-gray-100 p-4 rounded">
          <h3 className="font-semibold mb-2">画像 URL:</h3>
          <code className="text-sm">{`${baseUrl}/api/og/${id}`}</code>
        </div>

        <div>
          <h3 className="font-semibold mb-2">テストツール:</h3>
          <ul className="space-y-2">
            <li>
              <a 
                href={`https://developers.facebook.com/tools/debug/?q=${encodeURIComponent(`${baseUrl}/${id}`)}`}
                target="_blank"
                rel="noopener noreferrer"
                className="text-blue-600 hover:underline"
              >
                Facebook Sharing Debugger →
              </a>
            </li>
            <li>
              <a 
                href={`https://cards-dev.twitter.com/validator`}
                target="_blank"
                rel="noopener noreferrer"
                className="text-blue-600 hover:underline"
              >
                Twitter Card Validator →
              </a>
            </li>
          </ul>
        </div>
      </div>
    </main>
  );
}
```

### 6. パフォーマンス最適化

`src/app/api/og/[id]/route.tsx` に最適化を追加：

```tsx
// キャッシュヘッダーの追加
export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  // ... 既存のコード ...

  const response = new ImageResponse(/* ... */);

  // キャッシュヘッダーを設定
  response.headers.set(
    'Cache-Control',
    'public, immutable, no-transform, s-maxage=31536000, max-age=31536000'
  );

  return response;
}

// 静的パラメータの生成（オプション）
export async function generateStaticParams() {
  // よくアクセスされる ID を事前生成
  return [
    { id: '123456' },
    { id: '789012' },
    // ... 他の ID
  ];
}
```

## 📚 重要な概念

### Edge Runtime での画像生成アーキテクチャ

#### @vercel/og の内部実装と最適化

```tsx
import { ImageResponse } from '@vercel/og';

// Edge Runtime での動作原理
export const runtime = 'edge';

export async function GET(request: Request) {
  // 1. Edge Runtime は V8 Isolate で実行
  // - コールドスタート: 数十ミリ秒
  // - メモリ制限: 128MB
  // - 実行時間制限: 30秒
  
  // 2. @vercel/og は内部的に以下を使用:
  // - Satori: React 要素を SVG に変換
  // - resvg-wasm: SVG を PNG に変換
  // - WebAssembly: ブラウザレベルの高速処理
  
  console.log('Edge function execution start');
  
  const startTime = Date.now();
  
  // 3. フォントの事前読み込み最適化
  const fontPromise = getFontData(); // 並行処理
  const dataPromise = getImageData(request); // 並行処理
  
  const [fontData, imageData] = await Promise.all([
    fontPromise,
    dataPromise,
  ]);
  
  console.log(`Data fetch took: ${Date.now() - startTime}ms`);
  
  // 4. React 要素の構築（仮想DOM）
  const element = createImageElement(imageData);
  
  // 5. 画像生成パイプライン
  const response = new ImageResponse(element, {
    width: 1200,
    height: 630,
    fonts: [fontData],
    debug: process.env.NODE_ENV === 'development', // デバッグ情報
  });
  
  console.log(`Total generation time: ${Date.now() - startTime}ms`);
  
  return response;
}
```

#### ImageResponse の詳細オプション

```tsx
return new ImageResponse(
  element,
  {
    // 基本設定
    width: 1200,
    height: 630,
    
    // フォント設定
    fonts: [
      {
        name: 'Inter',
        data: await interFont,
        style: 'normal',
        weight: 400,
      },
      {
        name: 'Inter',
        data: await interBoldFont,
        style: 'normal',
        weight: 700,
      },
      {
        name: 'NotoSansJP',
        data: await notoSansJPFont,
        style: 'normal',
        weight: 400,
      },
    ],
    
    // デバッグ設定
    debug: false, // SVG 出力でデバッグ
    
    // 画質設定
    emoji: 'twemoji', // 絵文字レンダリング
    
    // HTTP ヘッダー設定
    headers: {
      'Cache-Control': 'public, immutable, no-transform, max-age=31536000',
      'Content-Type': 'image/png',
    },
  }
);
```

### CSS-in-JS から SVG への変換プロセス

#### Satori による CSS 解釈の制限と回避策

```tsx
// ❌ サポートされていない CSS プロパティ
const unsupportedStyles = {
  // ボックスシャドウ
  boxShadow: '0 4px 6px rgba(0, 0, 0, 0.1)', // ❌
  
  // グラデーション（一部制限あり）
  background: 'conic-gradient(from 180deg, red, blue)', // ❌
  
  // トランスフォーム（一部のみサポート）
  transform: 'perspective(1000px) rotateX(45deg)', // ❌
  
  // フィルター
  filter: 'blur(5px)', // ❌
  
  // アニメーション
  animation: 'spin 1s linear infinite', // ❌
};

// ✅ サポートされている CSS プロパティ
const supportedStyles = {
  // レイアウト
  display: 'flex',
  flexDirection: 'column',
  justifyContent: 'center',
  alignItems: 'center',
  
  // ボックスモデル
  width: '100%',
  height: '100%',
  padding: '20px',
  margin: '10px',
  
  // 背景
  backgroundColor: '#ffffff',
  backgroundImage: 'linear-gradient(to right, #ff0000, #0000ff)',
  
  // テキスト
  fontSize: '24px',
  fontWeight: 'bold',
  color: '#333333',
  textAlign: 'center',
  lineHeight: 1.5,
  
  // ボーダー
  border: '2px solid #000000',
  borderRadius: '8px',
  
  // 位置
  position: 'absolute',
  top: '10px',
  left: '20px',
};

// 回避策: SVG エフェクトの直接実装
function ShadowEffect({ children }) {
  return (
    <div style={{ position: 'relative' }}>
      {/* 疑似シャドウ */}
      <div
        style={{
          position: 'absolute',
          top: '4px',
          left: '4px',
          backgroundColor: 'rgba(0, 0, 0, 0.1)',
          borderRadius: '8px',
          width: '100%',
          height: '100%',
          zIndex: -1,
        }}
      />
      <div
        style={{
          backgroundColor: 'white',
          borderRadius: '8px',
          padding: '20px',
          position: 'relative',
        }}
      >
        {children}
      </div>
    </div>
  );
}
```

#### 複雑なレイアウトの実装パターン

```tsx
// パターン1: グリッドレイアウトの実装
function GridLayout({ items }) {
  const itemsPerRow = 3;
  const rows = Math.ceil(items.length / itemsPerRow);
  
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '10px' }}>
      {Array.from({ length: rows }).map((_, rowIndex) => (
        <div key={rowIndex} style={{ display: 'flex', gap: '10px' }}>
          {items
            .slice(rowIndex * itemsPerRow, (rowIndex + 1) * itemsPerRow)
            .map((item, colIndex) => (
              <div
                key={colIndex}
                style={{
                  flex: 1,
                  padding: '10px',
                  backgroundColor: '#f0f0f0',
                  borderRadius: '4px',
                }}
              >
                {item}
              </div>
            ))}
        </div>
      ))}
    </div>
  );
}

// パターン2: レスポンシブテキスト
function ResponsiveText({ text, maxWidth, maxLines = 3 }) {
  // 文字数に基づくフォントサイズ調整
  const fontSize = text.length > 100 ? 24 : text.length > 50 ? 32 : 40;
  
  // 改行の処理
  const lines = text.split('\n').slice(0, maxLines);
  const truncated = lines.length === maxLines && text.split('\n').length > maxLines;
  
  return (
    <div
      style={{
        maxWidth,
        fontSize,
        lineHeight: 1.3,
        overflow: 'hidden',
      }}
    >
      {lines.map((line, index) => (
        <div key={index}>
          {index === maxLines - 1 && truncated ? line + '...' : line}
        </div>
      ))}
    </div>
  );
}

// パターン3: アイコンとテキストの組み合わせ
function IconWithText({ icon, text, iconColor = '#666' }) {
  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: '8px' }}>
      {/* SVG アイコンの埋め込み */}
      <div
        style={{
          width: '24px',
          height: '24px',
          backgroundColor: iconColor,
          borderRadius: '50%',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
        }}
      >
        <span style={{ color: 'white', fontSize: '12px', fontWeight: 'bold' }}>
          {icon}
        </span>
      </div>
      <span style={{ fontSize: '16px', color: '#333' }}>{text}</span>
    </div>
  );
}
```

### フォント最適化とエンベッディング

#### 日本語フォントの最適化戦略

```tsx
// フォントサブセット化による最適化
class FontOptimizer {
  private static cache = new Map<string, ArrayBuffer>();
  
  // 必要な文字のみを含むフォントサブセットを生成
  static async getOptimizedFont(
    fontUrl: string,
    text: string,
    language: 'ja' | 'en' = 'ja'
  ): Promise<ArrayBuffer> {
    const cacheKey = `${fontUrl}-${this.getUniqueChars(text)}`;
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }
    
    // よく使われる文字セット
    const baseChars = language === 'ja' 
      ? 'あいうえおかきくけこさしすせそたちつてとなにぬねのはひふへほまみむめもやゆよらりるれろわをん'
      : 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    
    const uniqueChars = this.getUniqueChars(text + baseChars);
    
    // 実際の実装では fonttools-py などでサブセット化
    const optimizedFont = await this.subsetFont(fontUrl, uniqueChars);
    
    this.cache.set(cacheKey, optimizedFont);
    return optimizedFont;
  }
  
  private static getUniqueChars(text: string): string {
    return [...new Set(text)].join('');
  }
  
  private static async subsetFont(fontUrl: string, chars: string): Promise<ArrayBuffer> {
    // フォントサブセット化のロジック
    // 実際の実装では外部ツールやサービスを使用
    const response = await fetch(fontUrl);
    return response.arrayBuffer();
  }
}

// 使用例
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const text = searchParams.get('text') || '';
  
  // テキストに基づいてフォントを最適化
  const optimizedFont = await FontOptimizer.getOptimizedFont(
    'https://fonts.google.com/download?family=Noto%20Sans%20JP',
    text,
    'ja'
  );
  
  return new ImageResponse(
    <div style={{ fontSize: '32px' }}>{text}</div>,
    {
      fonts: [
        {
          name: 'NotoSansJP',
          data: optimizedFont,
          style: 'normal',
          weight: 400,
        },
      ],
    }
  );
}
```

#### 複数フォントのフォールバック戦略

```tsx
// フォントフォールバックの実装
async function getFontStack(): Promise<Array<{ name: string; data: ArrayBuffer; weight: number }>> {
  const fonts = [
    {
      name: 'Inter',
      url: new URL('../assets/Inter-Regular.woff2', import.meta.url),
      weight: 400,
    },
    {
      name: 'Inter',
      url: new URL('../assets/Inter-Bold.woff2', import.meta.url),
      weight: 700,
    },
    {
      name: 'NotoSansJP',
      url: new URL('../assets/NotoSansJP-Regular.woff2', import.meta.url),
      weight: 400,
    },
  ];
  
  const fontPromises = fonts.map(async (font) => {
    try {
      const response = await fetch(font.url);
      const data = await response.arrayBuffer();
      return { name: font.name, data, weight: font.weight };
    } catch (error) {
      console.error(`Failed to load font ${font.name}:`, error);
      return null;
    }
  });
  
  const results = await Promise.allSettled(fontPromises);
  return results
    .filter((result): result is PromiseFulfilledResult<{ name: string; data: ArrayBuffer; weight: number }> => 
      result.status === 'fulfilled' && result.value !== null
    )
    .map(result => result.value);
}

// スタイルでのフォントスタック指定
const textStyle = {
  fontFamily: 'Inter, "Noto Sans JP", sans-serif',
  fontWeight: 400,
  fontSize: '24px',
};
```

### パフォーマンス最適化とキャッシング戦略

#### 多層キャッシング実装

```tsx
// 1. CDN レベルキャッシング
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const id = searchParams.get('id');
  const theme = searchParams.get('theme') || 'light';
  const lang = searchParams.get('lang') || 'ja';
  
  // Cache-Control ヘッダーによる階層化キャッシュ
  const cacheHeaders = {
    // CDN: 1年間キャッシュ
    'Cache-Control': 'public, immutable, no-transform, s-maxage=31536000, max-age=31536000',
    // Vercel Edge Cache: より短期間
    'Vercel-CDN-Cache-Control': 'max-age=86400',
    // ブラウザキャッシュ: 最も短期間
    'CDN-Cache-Control': 'max-age=3600',
  };
  
  // 2. メモリキャッシング
  const memoryCache = new Map();
  const cacheKey = `og-${id}-${theme}-${lang}`;
  
  if (memoryCache.has(cacheKey)) {
    const cached = memoryCache.get(cacheKey);
    return new Response(cached.buffer, {
      headers: {
        'Content-Type': 'image/png',
        'X-Cache': 'HIT-MEMORY',
        ...cacheHeaders,
      },
    });
  }
  
  // 3. 画像生成
  const imageResponse = await generateImage(id, theme, lang);
  
  // メモリキャッシュに保存
  const buffer = await imageResponse.arrayBuffer();
  memoryCache.set(cacheKey, { buffer, timestamp: Date.now() });
  
  return new Response(buffer, {
    headers: {
      'Content-Type': 'image/png',
      'X-Cache': 'MISS',
      ...cacheHeaders,
    },
  });
}

// 4. 条件付きキャッシング
export async function GET(request: Request) {
  const ifNoneMatch = request.headers.get('if-none-match');
  const ifModifiedSince = request.headers.get('if-modified-since');
  
  const lastModified = await getLastModified(id);
  const etag = await generateETag(id, theme, lang);
  
  // 304 Not Modified レスポンス
  if (ifNoneMatch === etag || 
      (ifModifiedSince && new Date(ifModifiedSince) >= lastModified)) {
    return new Response(null, {
      status: 304,
      headers: {
        'ETag': etag,
        'Last-Modified': lastModified.toUTCString(),
        'Cache-Control': 'public, max-age=3600',
      },
    });
  }
  
  // 通常のレスポンス
  const imageResponse = await generateImage(id, theme, lang);
  
  return new Response(await imageResponse.arrayBuffer(), {
    headers: {
      'Content-Type': 'image/png',
      'ETag': etag,
      'Last-Modified': lastModified.toUTCString(),
      'Cache-Control': 'public, max-age=3600',
    },
  });
}
```

#### バッチ処理と事前生成

```tsx
// 静的生成による事前処理
export async function generateStaticParams() {
  // よくアクセスされる更新の OG 画像を事前生成
  const popularUpdates = await getPopularUpdates();
  
  return popularUpdates.flatMap(update => [
    { id: update.id },
    { id: update.id, theme: 'dark' },
    { id: update.id, lang: 'en' },
  ]);
}

// バックグラウンドでの画像事前生成
export async function POST(request: Request) {
  const { updateIds } = await request.json();
  
  // 非同期で複数の画像を生成
  const generationPromises = updateIds.map(async (id: string) => {
    try {
      const themes = ['light', 'dark'];
      const languages = ['ja', 'en'];
      
      for (const theme of themes) {
        for (const lang of languages) {
          await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/og/${id}?theme=${theme}&lang=${lang}`);
        }
      }
      
      return { id, status: 'success' };
    } catch (error) {
      return { id, status: 'error', error: error.message };
    }
  });
  
  const results = await Promise.allSettled(generationPromises);
  
  return NextResponse.json({
    total: updateIds.length,
    successful: results.filter(r => r.status === 'fulfilled').length,
    failed: results.filter(r => r.status === 'rejected').length,
    results,
  });
}
```

## ✅ 確認ポイント

- [ ] OG 画像が正しく生成される
- [ ] 日本語フォントが表示される
- [ ] エラー時のフォールバックが機能する
- [ ] キャッシュが効いている

## 🏃 演習問題

1. **テーマ切り替え**: ダークモード版の OG 画像を生成
2. **多言語対応**: Accept-Language ヘッダーに基づいて言語を切り替え
3. **A/B テスト**: 異なるデザインの OG 画像をランダムに表示

### 演習 1 の解答例

```tsx
// クエリパラメータでテーマを指定
const searchParams = new URL(request.url).searchParams;
const theme = searchParams.get('theme') || 'light';

const isDark = theme === 'dark';

return new ImageResponse(
  (
    <div
      style={{
        backgroundColor: isDark ? '#1f2937' : '#ffffff',
        color: isDark ? '#ffffff' : '#1f2937',
        // ... 他のスタイル
      }}
    >
      {/* コンテンツ */}
    </div>
  )
);
```

## 🔗 参考リンク

- [@vercel/og Documentation](https://vercel.com/docs/concepts/functions/edge-functions/og-image-generation)
- [Open Graph Protocol](https://ogp.me/)
- [Twitter Cards](https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/abouts-cards)

---

## 🎉 完了！

おめでとうございます！すべてのステップを完了しました。

### 次のステップ

1. **デプロイ**: Vercel にデプロイして本番環境で動作確認
2. **カスタマイズ**: 独自の機能を追加
3. **最適化**: パフォーマンスの改善

### 学んだこと

- Next.js 14 App Router の基本から応用まで
- 外部 API との連携とキャッシング
- 動的コンテンツの生成と最適化
- モダンな Web アプリケーションの構築方法

Happy coding! 🚀