# 演習問題集

各ステップの演習問題と解答をまとめています。

## 📝 演習問題の進め方

1. 各ステップの学習を完了する
2. 演習問題に挑戦
3. 自分で実装してみる
4. 解答を確認
5. さらに発展的な課題にも挑戦

## 🏃 演習問題一覧

### Step 1: 基本セットアップ

#### 演習 1-1: フッターの追加
レイアウトにフッターを追加し、著作権表示とリンクを含めてください。

<details>
<summary>💡 ヒント</summary>

- `layout.tsx` でヘッダーと同様にフッターを追加
- `<footer>` タグを使用
- Tailwind CSS でスタイリング

</details>

<details>
<summary>📖 解答</summary>

```tsx
// src/app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body className={inter.className}>
        <div className="min-h-screen flex flex-col">
          <header className="bg-blue-600 text-white p-4">
            <h1 className="text-2xl font-bold">Azure Update Viewer</h1>
          </header>
          <main className="flex-1">
            {children}
          </main>
          <footer className="bg-gray-800 text-white p-4 mt-auto">
            <div className="text-center">
              <p>&copy; 2024 Azure Update Viewer</p>
              <a 
                href="https://github.com/your-repo" 
                className="text-blue-400 hover:underline"
              >
                GitHub
              </a>
            </div>
          </footer>
        </div>
      </body>
    </html>
  );
}
```

</details>

#### 演習 1-2: About ページの作成
アプリケーションの説明ページを作成してください。

<details>
<summary>📖 解答</summary>

```tsx
// src/app/about/page.tsx
export default function AboutPage() {
  return (
    <div className="max-w-4xl mx-auto p-8">
      <h1 className="text-3xl font-bold mb-6">About Azure Update Viewer</h1>
      
      <div className="prose prose-lg">
        <p className="mb-4">
          Azure Update Viewer は、Microsoft Azure の更新情報を
          高速かつ効率的に表示するためのアプリケーションです。
        </p>
        
        <h2 className="text-2xl font-semibold mt-8 mb-4">特徴</h2>
        <ul className="list-disc pl-6 space-y-2">
          <li>高速なキャッシング</li>
          <li>SEO 最適化</li>
          <li>動的 OG 画像生成</li>
        </ul>
      </div>
    </div>
  );
}
```

</details>

### Step 2: ルーティング

#### 演習 2-1: カスタム 404 ページ
より詳細な 404 エラーページを作成してください。

<details>
<summary>📖 解答</summary>

```tsx
// src/app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <main className="min-h-screen p-8 flex items-center justify-center">
      <div className="text-center max-w-md">
        <div className="text-9xl font-bold text-gray-300 mb-4">404</div>
        <h1 className="text-3xl font-bold mb-4">ページが見つかりません</h1>
        <p className="text-gray-600 mb-8">
          お探しのページは存在しないか、移動した可能性があります。
        </p>
        <div className="space-x-4">
          <Link 
            href="/" 
            className="inline-block bg-blue-600 text-white px-6 py-3 rounded hover:bg-blue-700 transition"
          >
            ホームに戻る
          </Link>
          <Link 
            href="/help" 
            className="inline-block border border-gray-300 px-6 py-3 rounded hover:bg-gray-50 transition"
          >
            ヘルプ
          </Link>
        </div>
      </div>
    </main>
  );
}
```

</details>

### Step 3: API ルート

#### 演習 3-1: ヘルスチェック API
アプリケーションの状態を確認する API エンドポイントを作成してください。

<details>
<summary>📖 解答</summary>

```ts
// src/app/api/health/route.ts
import { NextResponse } from 'next/server';
import { supabase } from '@/lib/db';

export async function GET() {
  const status = {
    app: 'ok',
    database: 'unknown',
    timestamp: new Date().toISOString(),
  };

  try {
    // データベース接続チェック
    const { error } = await supabase
      .from('azure_updates')
      .select('count')
      .limit(1);

    status.database = error ? 'error' : 'ok';

    const httpStatus = status.database === 'ok' ? 200 : 503;

    return NextResponse.json(status, { status: httpStatus });
  } catch (error) {
    status.database = 'error';
    return NextResponse.json(status, { status: 503 });
  }
}
```

</details>

### Step 4: データフェッチング

#### 演習 4-1: バッチ取得 API
複数の更新を一度に取得する API を実装してください。

<details>
<summary>📖 解答</summary>

```ts
// src/app/api/batch/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getOrFetchAzureUpdate } from '@/lib/azure-updates';

export async function POST(request: NextRequest) {
  try {
    const { ids } = await request.json();

    if (!Array.isArray(ids) || ids.length === 0) {
      return NextResponse.json(
        { error: 'IDs array is required' },
        { status: 400 }
      );
    }

    if (ids.length > 10) {
      return NextResponse.json(
        { error: 'Maximum 10 IDs allowed' },
        { status: 400 }
      );
    }

    // 並列で取得
    const promises = ids.map(id => 
      getOrFetchAzureUpdate(id)
        .then(data => ({ id, data, error: null }))
        .catch(error => ({ id, data: null, error: error.message }))
    );

    const results = await Promise.all(promises);

    return NextResponse.json({
      results,
      success: results.filter(r => r.data).length,
      failed: results.filter(r => r.error).length,
    });

  } catch (error) {
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

</details>

### Step 5: OG 画像

#### 演習 5-1: OG 画像のカスタマイズ
クエリパラメータでスタイルを変更できる OG 画像を実装してください。

<details>
<summary>📖 解答</summary>

```tsx
// src/app/api/og/[id]/route.tsx の一部
export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const { searchParams } = new URL(request.url);
  
  // スタイルパラメータ
  const style = searchParams.get('style') || 'default';
  const color = searchParams.get('color') || 'blue';

  // ... データ取得 ...

  const styles = {
    default: {
      bg: 'linear-gradient(to bottom right, #f0f9ff, #e0f2fe)',
      primary: '#2563eb',
    },
    dark: {
      bg: 'linear-gradient(to bottom right, #1f2937, #111827)',
      primary: '#3b82f6',
    },
    minimal: {
      bg: '#ffffff',
      primary: '#000000',
    },
  };

  const currentStyle = styles[style] || styles.default;

  return new ImageResponse(
    (
      <div
        style={{
          backgroundImage: currentStyle.bg,
          // ... 他のスタイル
        }}
      >
        {/* コンテンツ */}
      </div>
    ),
    {
      width: 1200,
      height: 630,
    }
  );
}
```

</details>

## 🚀 発展課題

### 1. 検索機能の実装
- 全文検索 API の作成
- 検索 UI の実装
- 検索結果のハイライト

### 2. お気に入り機能
- LocalStorage でのお気に入り管理
- お気に入り一覧ページ
- エクスポート機能

### 3. RSS フィードの生成
- 最新の更新情報を RSS で配信
- カテゴリ別フィード
- Webhook 通知

### 4. 管理画面の作成
- キャッシュの管理
- 統計情報の表示
- 手動更新機能

## 📚 追加リソース

- [Next.js Learn Course](https://nextjs.org/learn)
- [Vercel Templates](https://vercel.com/templates)
- [Supabase Examples](https://supabase.com/docs/guides/examples)

---

すべての演習を完了したら、独自の機能を追加してアプリケーションをカスタマイズしてみましょう！