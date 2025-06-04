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

### Edge Runtime の制限

- Node.js API は使用不可
- ファイルシステムアクセス不可
- 軽量ライブラリのみ使用可能

### 画像の最適化

- 適切なサイズ（1200x630）
- テキストの可読性
- コントラスト比の確保

### フォントの埋め込み

```tsx
const font = fetch(new URL('./font.ttf', import.meta.url))
  .then((res) => res.arrayBuffer());
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