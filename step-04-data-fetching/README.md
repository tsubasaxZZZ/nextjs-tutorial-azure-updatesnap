# Step 4: データフェッチングとキャッシング

このステップでは、Supabase を使ったデータ永続化とキャッシング戦略を実装します。

## 🎯 このステップで学ぶこと

- Supabase との連携
- Server Components でのデータフェッチング
- キャッシング戦略の実装
- エラー境界とサスペンス
- 環境変数の管理

## 📖 解説

### データフェッチングの戦略

Azure UpdateSnap では以下の戦略を採用：

1. **キャッシュファースト**: まず Supabase をチェック
2. **フォールバック**: キャッシュがない場合は Microsoft API から取得
3. **バックグラウンド更新**: 古いキャッシュは表示しつつ、バックグラウンドで更新

### Supabase のセットアップ

1. [Supabase](https://supabase.com) でプロジェクト作成
2. データベーススキーマの設定
3. 環境変数の設定

## 🛠️ ハンズオン

### 1. データベーススキーマの作成

`supabase/migrations/create_azure_updates.sql`:

```sql
-- Azure updates テーブル
CREATE TABLE IF NOT EXISTS azure_updates (
  id SERIAL PRIMARY KEY,
  update_id TEXT UNIQUE NOT NULL,
  guid TEXT,
  title TEXT NOT NULL,
  description TEXT,
  impact_description TEXT,
  public_disclosure_date TIMESTAMPTZ,
  tags TEXT[],
  details_url TEXT,
  raw_data JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  ttl_expires_at TIMESTAMPTZ
);

-- インデックス
CREATE INDEX idx_azure_updates_update_id ON azure_updates(update_id);
CREATE INDEX idx_azure_updates_guid ON azure_updates(guid);
CREATE INDEX idx_azure_updates_ttl ON azure_updates(ttl_expires_at);

-- 更新日時の自動更新
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_azure_updates_updated_at 
  BEFORE UPDATE ON azure_updates 
  FOR EACH ROW 
  EXECUTE FUNCTION update_updated_at_column();
```

### 2. Supabase クライアントの設定

`src/lib/db.ts`:

```ts
import { createClient } from '@supabase/supabase-js';

// 環境変数の型安全な取得
function getEnvVar(key: string): string {
  const value = process.env[key];
  if (!value) {
    throw new Error(`Missing environment variable: ${key}`);
  }
  return value;
}

// Supabase クライアントの作成
export const supabase = createClient(
  getEnvVar('SUPABASE_URL'),
  getEnvVar('SUPABASE_ANON_KEY'),
  {
    auth: {
      persistSession: false,
    },
  }
);

// データベースの型定義
export interface AzureUpdateDB {
  id: number;
  update_id: string;
  guid: string | null;
  title: string;
  description: string | null;
  impact_description: string | null;
  public_disclosure_date: string | null;
  tags: string[];
  details_url: string | null;
  raw_data: any;
  created_at: string;
  updated_at: string;
  ttl_expires_at: string | null;
}

// TTL の計算（デフォルト12時間）
export function calculateTTL(): Date {
  const ttlHours = parseInt(process.env.TTL_HOURS || '12', 10);
  return new Date(Date.now() + ttlHours * 60 * 60 * 1000);
}
```

### 3. 統合されたデータフェッチング関数

`src/lib/azure-updates.ts`:

```ts
import { supabase, AzureUpdateDB, calculateTTL } from './db';
import { fetchAzureUpdate, AzureUpdate } from './azureFetch';

export async function getOrFetchAzureUpdate(
  id: string
): Promise<AzureUpdate | null> {
  try {
    // 1. キャッシュをチェック
    const { data: cached, error: cacheError } = await supabase
      .from('azure_updates')
      .select('*')
      .eq('update_id', id)
      .single();

    if (cached && !cacheError) {
      // TTL チェック
      const now = new Date();
      const expiresAt = cached.ttl_expires_at 
        ? new Date(cached.ttl_expires_at) 
        : null;

      if (!expiresAt || now < expiresAt) {
        console.log(`Cache hit for ${id}`);
        return {
          id: parseInt(cached.update_id, 10),
          guid: cached.guid || '',
          title: cached.title,
          description: cached.description || '',
          impactDescription: cached.impact_description || undefined,
          publicDisclosureDate: cached.public_disclosure_date || '',
          tags: cached.tags,
          detailsUrl: cached.details_url || '',
        };
      }

      console.log(`Cache expired for ${id}`);
    }

    // 2. Microsoft API から取得
    console.log(`Fetching from API for ${id}`);
    const update = await fetchAzureUpdate(id);

    if (!update) {
      return null;
    }

    // 3. キャッシュに保存
    const { error: upsertError } = await supabase
      .from('azure_updates')
      .upsert({
        update_id: id,
        guid: update.guid,
        title: update.title,
        description: update.description,
        impact_description: update.impactDescription,
        public_disclosure_date: update.publicDisclosureDate,
        tags: update.tags,
        details_url: update.detailsUrl,
        raw_data: update,
        ttl_expires_at: calculateTTL().toISOString(),
      }, {
        onConflict: 'update_id',
      });

    if (upsertError) {
      console.error('Failed to cache update:', upsertError);
      // キャッシュ失敗してもデータは返す
    }

    return update;

  } catch (error) {
    console.error('Failed to get Azure update:', error);
    throw error;
  }
}

// バックグラウンド更新（オプション）
export async function refreshUpdateInBackground(id: string): Promise<void> {
  // 非ブロッキングで更新
  fetchAzureUpdate(id)
    .then(async (update) => {
      if (update) {
        await supabase
          .from('azure_updates')
          .upsert({
            update_id: id,
            guid: update.guid,
            title: update.title,
            description: update.description,
            impact_description: update.impactDescription,
            public_disclosure_date: update.publicDisclosureDate,
            tags: update.tags,
            details_url: update.detailsUrl,
            raw_data: update,
            ttl_expires_at: calculateTTL().toISOString(),
          }, {
            onConflict: 'update_id',
          });
      }
    })
    .catch(console.error);
}
```

### 4. Server Component での実装

`src/app/[slug]/page.tsx` を更新：

```tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { getOrFetchAzureUpdate } from '@/lib/azure-updates';
import AzureUpdateView from './AzureUpdateView';
import RedirectingView from './RedirectingView';

type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

// Bot 検出
function isBot(userAgent: string | null): boolean {
  if (!userAgent) return false;
  return /bot|crawler|spider|crawling/i.test(userAgent);
}

export default async function UpdatePage({ params, searchParams }: Props) {
  const { slug } = await params;
  const { debug } = await searchParams;
  
  // ヘッダーから User-Agent を取得
  const { headers } = await import('next/headers');
  const headersList = await headers();
  const userAgent = headersList.get('user-agent');
  
  // Bot チェック
  const isBotRequest = isBot(userAgent);
  const isDebugMode = debug === 'true';

  // データ取得
  const update = await getOrFetchAzureUpdate(slug);

  if (!update) {
    notFound();
  }

  // Bot またはデバッグモードの場合は完全表示
  if (isBotRequest || isDebugMode) {
    return <AzureUpdateView update={update} />;
  }

  // 通常ユーザーはリダイレクト
  return <RedirectingView targetUrl={update.detailsUrl} />;
}

// メタデータの生成
export async function generateMetadata({ params }: Props) {
  const { slug } = await params;
  const update = await getOrFetchAzureUpdate(slug);

  if (!update) {
    return {
      title: 'Not Found',
    };
  }

  return {
    title: update.title,
    description: update.description,
    openGraph: {
      title: update.title,
      description: update.description,
      type: 'article',
      publishedTime: update.publicDisclosureDate,
    },
  };
}
```

### 5. AzureUpdateView コンポーネント

`src/app/[slug]/AzureUpdateView.tsx`:

```tsx
import { AzureUpdate } from '@/lib/azureFetch';

type Props = {
  update: AzureUpdate;
};

export default function AzureUpdateView({ update }: Props) {
  return (
    <main className="max-w-4xl mx-auto p-8">
      <article className="bg-white rounded-lg shadow-lg p-8">
        <h1 className="text-3xl font-bold mb-4">
          {update.title}
        </h1>
        
        <div className="text-sm text-gray-600 mb-6">
          <span>ID: {update.id}</span>
          {update.publicDisclosureDate && (
            <>
              <span className="mx-2">•</span>
              <span>
                公開日: {new Date(update.publicDisclosureDate).toLocaleDateString('ja-JP')}
              </span>
            </>
          )}
        </div>

        {update.tags.length > 0 && (
          <div className="flex flex-wrap gap-2 mb-6">
            {update.tags.map((tag) => (
              <span
                key={tag}
                className="px-3 py-1 bg-blue-100 text-blue-800 text-sm rounded-full"
              >
                {tag}
              </span>
            ))}
          </div>
        )}

        <div className="prose max-w-none">
          <h2 className="text-xl font-semibold mb-3">概要</h2>
          <p className="text-gray-700 mb-6 whitespace-pre-wrap">
            {update.description}
          </p>

          {update.impactDescription && (
            <>
              <h2 className="text-xl font-semibold mb-3">影響</h2>
              <p className="text-gray-700 mb-6 whitespace-pre-wrap">
                {update.impactDescription}
              </p>
            </>
          )}
        </div>

        <div className="mt-8 pt-6 border-t">
          <a
            href={update.detailsUrl}
            target="_blank"
            rel="noopener noreferrer"
            className="inline-flex items-center text-blue-600 hover:text-blue-800"
          >
            Microsoft 公式ページで詳細を見る
            <svg className="w-4 h-4 ml-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14" />
            </svg>
          </a>
        </div>
      </article>
    </main>
  );
}
```

### 6. 環境変数の設定

`.env.local`:

```env
# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key

# キャッシュ設定
TTL_HOURS=12

# アプリケーション
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

## 📚 重要な概念

### Next.js データフェッチングアーキテクチャの深い理解

#### Server Components でのデータフェッチングの仕組み

```tsx
// Server Component での非同期データフェッチング
export default async function UpdatePage({ params }: Props) {
  // この関数はサーバーサイドでのみ実行される
  console.log('This runs on the server'); // ブラウザには表示されない
  
  // 1. 直接データベースアクセス
  const cachedData = await supabase
    .from('azure_updates')
    .select('*')
    .eq('update_id', id)
    .single();
  
  // 2. 複数の非同期処理を並列実行
  const [updateData, relatedData, userPreferences] = await Promise.all([
    fetchAzureUpdate(id),
    fetchRelatedUpdates(id),
    getUserPreferences(userId),
  ]);
  
  // 3. 計算集約的な処理もサーバーで実行
  const processedData = updateData.map(item => ({
    ...item,
    readingTime: calculateReadingTime(item.description),
    sentiment: analyzeSentiment(item.description), // 重い ML 処理
  }));
  
  // 4. 条件分岐もサーバーサイドで
  if (!updateData) {
    notFound(); // 適切な 404 レスポンス
  }
  
  // 5. レンダリング結果のみがクライアントに送信される
  return (
    <div>
      <h1>{updateData.title}</h1>
      <p>Reading time: {processedData.readingTime} minutes</p>
      {userPreferences.showDetails && (
        <DetailedView data={processedData} />
      )}
    </div>
  );
}
```

#### データフェッチングパターンの比較

```tsx
// ❌ Client Component での従来のフェッチング
'use client';
function ClientUpdatePage({ id }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    // クライアントサイドでの API 呼び出し
    fetch(`/api/updates/${id}`)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [id]);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!data) return <div>Not found</div>;
  
  return <div>{data.title}</div>;
}

// ✅ Server Component での最適化されたフェッチング
async function ServerUpdatePage({ params }) {
  // サーバーサイドで事前にデータを取得
  const data = await getOrFetchAzureUpdate(params.id);
  
  // データがない場合は適切に 404 を返す
  if (!data) notFound();
  
  // ローディング状態やエラー状態の管理が不要
  return <div>{data.title}</div>;
}
```

### キャッシング戦略の詳細設計

#### 多層キャッシュアーキテクチャ

```tsx
async function getOrFetchAzureUpdate(id: string): Promise<AzureUpdate | null> {
  // 1. Request Deduplication（同一リクエストの重複排除）
  const cacheKey = `azure-update-${id}`;
  
  // Next.js の fetch() での自動キャッシュ
  const response = await fetch(`${API_BASE}/updates/${id}`, {
    next: {
      revalidate: 3600, // 1時間キャッシュ
      tags: [`update-${id}`], // タグベースの無効化
    },
  });
  
  // 2. アプリケーションレベルキャッシュ
  const memoryCache = new Map();
  if (memoryCache.has(cacheKey)) {
    const cached = memoryCache.get(cacheKey);
    if (cached.expiresAt > Date.now()) {
      return cached.data; // メモリキャッシュヒット
    }
  }
  
  // 3. データベースキャッシュ
  const dbCached = await supabase
    .from('azure_updates')
    .select('*')
    .eq('update_id', id)
    .gte('ttl_expires_at', new Date().toISOString())
    .single();
  
  if (dbCached.data) {
    // メモリキャッシュにも保存
    memoryCache.set(cacheKey, {
      data: transformDbToApi(dbCached.data),
      expiresAt: Date.now() + 300000, // 5分間メモリキャッシュ
    });
    return transformDbToApi(dbCached.data);
  }
  
  // 4. 外部API + Stale-While-Revalidate
  const freshData = await fetchFromMicrosoftAPI(id);
  
  // 非同期でキャッシュ更新（レスポンスはブロックしない）
  updateCacheInBackground(id, freshData);
  
  return freshData;
}

async function updateCacheInBackground(id: string, data: AzureUpdate) {
  // データベース更新を非同期で実行
  Promise.resolve().then(async () => {
    await supabase
      .from('azure_updates')
      .upsert({
        update_id: id,
        ...transformApiToDb(data),
        ttl_expires_at: new Date(Date.now() + 12 * 60 * 60 * 1000).toISOString(),
      });
  }).catch(error => {
    console.error('Background cache update failed:', error);
  });
}
```

#### 動的キャッシュ戦略

```tsx
async function getUpdateWithStrategy(
  id: string, 
  options: {
    strategy?: 'cache-first' | 'network-first' | 'cache-only' | 'network-only';
    maxAge?: number;
    staleWhileRevalidate?: boolean;
  } = {}
): Promise<AzureUpdate | null> {
  
  const { strategy = 'cache-first', maxAge = 3600, staleWhileRevalidate = true } = options;
  
  switch (strategy) {
    case 'cache-only':
      // キャッシュのみ、ネットワークアクセスなし
      return await getCachedData(id);
      
    case 'network-only':
      // 常に最新データを取得
      const fresh = await fetchFromMicrosoftAPI(id);
      updateCacheInBackground(id, fresh);
      return fresh;
      
    case 'network-first':
      // ネットワーク優先、フォールバックでキャッシュ
      try {
        const networkData = await fetchFromMicrosoftAPI(id);
        updateCacheInBackground(id, networkData);
        return networkData;
      } catch (error) {
        console.warn('Network failed, falling back to cache:', error);
        return await getCachedData(id);
      }
      
    case 'cache-first':
    default:
      // キャッシュ優先戦略
      const cached = await getCachedData(id);
      
      if (cached) {
        const age = Date.now() - new Date(cached.updatedAt).getTime();
        
        if (age < maxAge * 1000) {
          // フレッシュなキャッシュ
          return cached;
        } else if (staleWhileRevalidate) {
          // 古いキャッシュを返しつつ、バックグラウンドで更新
          updateDataInBackground(id);
          return cached;
        }
      }
      
      // キャッシュがない、または期限切れ
      return await fetchFromMicrosoftAPI(id);
  }
}
```

### Suspense の高度な活用とストリーミング

#### 段階的データローディング

```tsx
export default function UpdatePage({ params }) {
  return (
    <div>
      {/* 即座に表示される部分 */}
      <Header />
      
      {/* 基本情報を先に読み込み */}
      <Suspense fallback={<BasicInfoSkeleton />}>
        <BasicInfo params={params} />
      </Suspense>
      
      {/* 詳細情報は遅延読み込み */}
      <Suspense fallback={<DetailsSkeleton />}>
        <DetailedContent params={params} />
      </Suspense>
      
      {/* 関連コンテンツも個別に読み込み */}
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedUpdates params={params} />
      </Suspense>
    </div>
  );
}

// 基本情報（高速なクエリ）
async function BasicInfo({ params }) {
  const basicData = await supabase
    .from('azure_updates')
    .select('title, description, tags')
    .eq('update_id', params.id)
    .single();
  
  return (
    <div>
      <h1>{basicData.data.title}</h1>
      <p>{basicData.data.description}</p>
    </div>
  );
}

// 詳細情報（重いクエリ）
async function DetailedContent({ params }) {
  // 時間のかかる処理
  const [details, analysis, history] = await Promise.all([
    fetchDetailedInfo(params.id),
    performContentAnalysis(params.id), // 3秒かかる AI 処理
    fetchUpdateHistory(params.id),
  ]);
  
  return (
    <div>
      <DetailedView details={details} />
      <AnalysisView analysis={analysis} />
      <HistoryView history={history} />
    </div>
  );
}
```

#### ストリーミング SSR の仕組み

```tsx
// app/layout.tsx - ストリーミング対応のルートレイアウト
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {/* 即座に送信される部分 */}
        <header>Azure Update Viewer</header>
        
        {/* Suspense 境界により段階的に送信 */}
        <main>
          {children}
        </main>
        
        {/* フッターも即座に表示 */}
        <footer>© 2024</footer>
      </body>
    </html>
  );
}

// ストリーミングの流れ：
// 1. HTML の shell（header, footer）が即座に送信
// 2. Suspense fallback が表示
// 3. Server Component の処理完了後、実際のコンテンツで置き換え
// 4. JavaScript によりハイドレーション

// 送信される HTML の例：
/*
<html>
  <body>
    <header>Azure Update Viewer</header>
    <main>
      <div>Loading basic info...</div> <!-- fallback -->
    </main>
    <footer>© 2024</footer>
  </body>
  
  <!-- 後から送信される部分 -->
  <script>
    // 実際のコンテンツでfallbackを置き換え
    document.getElementById('suspense-1').innerHTML = `
      <h1>Actual Title</h1>
      <p>Actual content...</p>
    `;
  </script>
</html>
*/
```

### データキャッシュの無効化とリバリデーション

#### タグベースの無効化

```tsx
import { revalidateTag } from 'next/cache';

// データフェッチング時にタグを付与
async function fetchUpdate(id: string) {
  const response = await fetch(`${API_BASE}/updates/${id}`, {
    next: {
      tags: [`update-${id}`, 'updates', `user-${userId}`],
    },
  });
  return response.json();
}

// API Route での無効化
export async function POST(request: NextRequest) {
  const { updateId } = await request.json();
  
  // 特定の更新のキャッシュを無効化
  revalidateTag(`update-${updateId}`);
  
  // 全ての更新のキャッシュを無効化
  revalidateTag('updates');
  
  return NextResponse.json({ revalidated: true });
}

// Webhook での自動無効化
export async function POST(request: NextRequest) {
  const signature = request.headers.get('x-signature');
  
  // Webhook の検証
  if (!verifySignature(signature, await request.text())) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  const { action, updateId } = await request.json();
  
  if (action === 'update.modified') {
    // Microsoft 側で更新があった場合、キャッシュを無効化
    revalidateTag(`update-${updateId}`);
    
    // データベースからも削除して強制再取得
    await supabase
      .from('azure_updates')
      .delete()
      .eq('update_id', updateId);
  }
  
  return NextResponse.json({ success: true });
}
```

#### 時間ベースのリバリデーション

```tsx
// 定期的なバックグラウンド更新
export async function GET() {
  // cron job や scheduled function で呼び出される
  
  // 期限切れのキャッシュを特定
  const { data: expiredEntries } = await supabase
    .from('azure_updates')
    .select('update_id')
    .lt('ttl_expires_at', new Date().toISOString());
  
  // バッチでリフレッシュ
  const refreshPromises = expiredEntries.map(async (entry) => {
    try {
      const freshData = await fetchFromMicrosoftAPI(entry.update_id);
      
      await supabase
        .from('azure_updates')
        .upsert({
          update_id: entry.update_id,
          ...transformApiToDb(freshData),
          ttl_expires_at: new Date(Date.now() + 12 * 60 * 60 * 1000).toISOString(),
        });
      
      // Next.js キャッシュも無効化
      revalidateTag(`update-${entry.update_id}`);
      
    } catch (error) {
      console.error(`Failed to refresh ${entry.update_id}:`, error);
    }
  });
  
  await Promise.allSettled(refreshPromises);
  
  return NextResponse.json({ 
    refreshed: expiredEntries.length,
    timestamp: new Date().toISOString(),
  });
}
```

## ✅ 確認ポイント

- [ ] Supabase との接続が成功する
- [ ] キャッシュが正しく動作する
- [ ] TTL が期待通りに機能する
- [ ] エラーが適切にハンドリングされる

## 🏃 演習問題

1. **キャッシュ統計**: キャッシュのヒット率を計測する機能を追加
2. **バッチ取得**: 複数の更新を一度に取得する関数を実装
3. **リアルタイム更新**: Supabase Realtime を使って更新を監視

### 演習 1 の解答例

```ts
// キャッシュ統計テーブル
CREATE TABLE cache_stats (
  id SERIAL PRIMARY KEY,
  update_id TEXT NOT NULL,
  hit BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

// 統計記録関数
export async function recordCacheStats(updateId: string, hit: boolean) {
  await supabase
    .from('cache_stats')
    .insert({ update_id: updateId, hit });
}
```

## 🔗 参考リンク

- [Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [Supabase JavaScript Client](https://supabase.com/docs/reference/javascript/introduction)
- [Caching](https://nextjs.org/docs/app/building-your-application/caching)

---

準備ができたら、[Step 5: 動的 OG 画像生成](../step-05-og-images) に進みましょう！