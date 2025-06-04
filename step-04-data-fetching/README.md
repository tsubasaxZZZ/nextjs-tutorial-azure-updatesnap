# Step 4: Data Fetching and Caching - Mastering Modern Data Architecture

このステップでは、現代的なデータフェッチングアーキテクチャと高度なキャッシング戦略を学び、エンタープライズレベルのパフォーマンスを実現します。

## 🎯 このステップで学ぶこと

- **データフェッチング進化史**: CSR → SSR → ISR → Hybrid の変遷
- **Supabase統合**: PostgreSQL ベースのモダンBaaS活用
- **多層キャッシング**: メモリ → データベース → CDN の戦略
- **Azure UpdateSnap**: 実践的なキャッシング実装
- **データベース設計**: スケーラブルなスキーマ設計
- **パフォーマンス最適化**: 大規模データ処理のベストプラクティス

## 📖 現代的データアーキテクチャの完全ガイド

### データフェッチングパターンの進化史

#### 第一世代：Client-Side Rendering (CSR) - 2010年代前半

```js
// jQuery 時代のデータフェッチング
$(document).ready(function() {
    // ページ読み込み後にデータを取得
    $.ajax({
        url: '/api/updates',
        method: 'GET',
        success: function(data) {
            $('#content').html(renderTemplate(data));
        },
        error: function() {
            $('#content').html('<p>Error loading data</p>');
        }
    });
});
```

**課題**:
- 初期ページ表示が白紙状態
- SEO対応が困難
- ネットワーク遅延の影響が大きい
- JavaScript無効環境で動作しない

#### 第二世代：Server-Side Rendering (SSR) - 2010年代後半

```php
// PHP での従来的 SSR
<?php
$updates = fetchUpdatesFromDatabase($id);
if (!$updates) {
    http_response_code(404);
    include '404.php';
    exit;
}
?>
<html>
<body>
    <h1><?= htmlspecialchars($updates['title']) ?></h1>
    <p><?= htmlspecialchars($updates['description']) ?></p>
</body>
</html>
```

**利点**:
- 即座にコンテンツが表示
- SEO フレンドリー
- JavaScript なしでも動作

**課題**:
- サーバーリソース消費が大きい
- 動的なインタラクションが制限
- ページ遷移でのフル再読み込み

#### 第三世代：Static Site Generation (SSG) - 2010年代末

```js
// Gatsby での Static Generation
export const query = graphql`
  query UpdatePage($id: String!) {
    update(id: { eq: $id }) {
      title
      description
      publishedDate
    }
  }
`;

// ビルド時にページを生成
export const getStaticPaths = async () => {
    const updates = await fetchAllUpdates();
    const paths = updates.map(update => ({
        params: { id: update.id.toString() }
    }));
    
    return { paths, fallback: false };
};
```

**利点**:
- 極めて高速な表示
- CDN での効率的配信
- 高いセキュリティ

**課題**:
- データ更新のリアルタイム性
- 大量ページのビルド時間
- 動的コンテンツへの対応

#### 第四世代：Incremental Static Regeneration (ISR) - 2020年代前半

```js
// Next.js での ISR
export async function getStaticProps({ params }) {
    const update = await fetchUpdate(params.id);
    
    return {
        props: { update },
        revalidate: 3600, // 1時間ごとに再生成
    };
}
```

**革新**:
- 静的生成の高速性
- 動的更新の対応
- オンデマンド再生成

#### 第五世代：Hybrid Rendering (Next.js App Router) - 2020年代後半

```tsx
// 現代的な Hybrid アプローチ
export default async function UpdatePage({ params }: { params: { id: string } }) {
    // Server Component: サーバーで事前実行
    const update = await getOrFetchAzureUpdate(params.id);
    
    if (!update) notFound();
    
    return (
        <div>
            {/* 静的部分：即座に表示 */}
            <StaticHeader />
            
            {/* 動的部分：サーバーレンダリング */}
            <UpdateContent update={update} />
            
            {/* インタラクティブ部分：Client Component */}
            <InteractiveComments updateId={params.id} />
        </div>
    );
}
```

**最新の特徴**:
- **Component 単位での選択**: Server vs Client Components
- **Streaming**: 段階的なページ表示
- **Suspense**: 宣言的ローディング状態
- **Edge Rendering**: CDN エッジでの実行

### Supabase: 次世代 Backend-as-a-Service

#### 従来の BaaS との比較

**Firebase (Google)**:
```js
// NoSQL ベース
const snapshot = await db.collection('updates').doc(id).get();
const data = snapshot.data();
```

**Supabase (PostgreSQL)**:
```ts
// SQL ベース（型安全）
const { data, error } = await supabase
    .from('azure_updates')
    .select('*')
    .eq('update_id', id)
    .single();
```

#### Supabase の技術的優位性

1. **PostgreSQL ベース**: 成熟したRDBMSの信頼性
2. **リアルタイム機能**: WebSocket によるライブ更新
3. **Row Level Security**: 細粒度のセキュリティ制御
4. **Edge Functions**: Deno ベースのサーバーレス
5. **TypeScript ネイティブ**: 完全な型安全性

```ts
// 高度な Supabase クエリの例
const { data } = await supabase
    .from('azure_updates')
    .select(`
        *,
        tags,
        related_updates!inner(
            id,
            title
        )
    `)
    .eq('status', 'active')
    .gte('published_date', startDate)
    .order('priority', { ascending: false })
    .range(0, 9); // ページネーション
```

### Azure UpdateSnap の多層キャッシングアーキテクチャ

#### 1. CDN Layer (Vercel Edge Network)
```
ユーザー → エッジロケーション → オリジンサーバー
         ↑ キャッシュヒット時はここで完結
```

#### 2. Next.js Data Cache
```ts
// 自動的なリクエストメモ化
const response = await fetch('/api/updates/123', {
    next: {
        revalidate: 3600, // 1時間キャッシュ
        tags: ['update-123'] // タグベース無効化
    }
});
```

#### 3. Application Memory Cache
```ts
// アプリケーションレベルの高速キャッシュ
const memoryCache = new Map<string, CacheEntry>();

interface CacheEntry {
    data: AzureUpdate;
    expiresAt: number;
    accessCount: number;
    lastAccessed: number;
}
```

#### 4. Database Cache (Supabase)
```sql
-- 高度なインデックス戦略
CREATE INDEX CONCURRENTLY idx_azure_updates_composite 
ON azure_updates (update_id, ttl_expires_at) 
WHERE ttl_expires_at > NOW();

-- パーティショニング（大規模データ対応）
CREATE TABLE azure_updates_2024 PARTITION OF azure_updates 
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

## 🛠️ 実装：エンタープライズグレードのデータレイヤー

### 1. スケーラブルなデータベーススキーマ設計

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

### Azure UpdateSnap の高度なデータ戦略

```ts
// 統合的なデータサービスクラス
export class AzureUpdateDataService {
  private cache: MultiLevelCache;
  private database: SupabaseClient;
  private metrics: MetricsCollector;
  
  constructor() {
    this.cache = new MultiLevelCache({
      memory: { maxSize: 100, ttl: 300000 }, // 5分
      database: { ttl: 43200000 }, // 12時間
    });
    this.database = createSupabaseClient();
    this.metrics = new MetricsCollector();
  }
  
  async getUpdate(id: string, options: FetchOptions = {}): Promise<AzureUpdate | null> {
    const startTime = performance.now();
    
    try {
      // 1. メモリキャッシュチェック
      const memoryCached = await this.cache.memory.get(id);
      if (memoryCached) {
        this.metrics.recordCacheHit('memory', id, performance.now() - startTime);
        return memoryCached;
      }
      
      // 2. データベースキャッシュチェック
      const dbCached = await this.cache.database.get(id);
      if (dbCached && !this.isCacheExpired(dbCached, options.maxAge)) {
        // メモリキャッシュにも保存
        await this.cache.memory.set(id, dbCached.data);
        this.metrics.recordCacheHit('database', id, performance.now() - startTime);
        return dbCached.data;
      }
      
      // 3. 外部API取得
      const freshData = await this.fetchFromMicrosoftAPI(id);
      if (!freshData) {
        this.metrics.recordCacheMiss(id, performance.now() - startTime);
        return null;
      }
      
      // 4. 全レイヤーにキャッシュ
      await Promise.all([
        this.cache.memory.set(id, freshData),
        this.cache.database.set(id, freshData, calculateTTL(freshData)),
      ]);
      
      this.metrics.recordFreshFetch(id, performance.now() - startTime);
      return freshData;
      
    } catch (error) {
      this.metrics.recordError(id, error, performance.now() - startTime);
      
      // エラー時のフォールバック戦略
      const staleData = await this.cache.database.getStale(id);
      if (staleData && options.allowStale) {
        return staleData;
      }
      
      throw error;
    }
  }
  
  // バックグラウンド更新
  async refreshInBackground(id: string): Promise<void> {
    try {
      const freshData = await this.fetchFromMicrosoftAPI(id);
      if (freshData) {
        await Promise.all([
          this.cache.memory.set(id, freshData),
          this.cache.database.set(id, freshData, calculateTTL(freshData)),
          this.invalidateRelatedCaches(id),
        ]);
      }
    } catch (error) {
      console.error(`Background refresh failed for ${id}:`, error);
    }
  }
}
```

## ✅ 実装確認チェックリスト

### データベース設計
- [ ] スキーマが正規化されている
- [ ] 適切なインデックスが設定されている
- [ ] パーティショニング戦略が計画されている
- [ ] トランザクション整合性が保たれている

### キャッシング戦略
- [ ] 多層キャッシュが実装されている
- [ ] TTL管理が適切に動作する
- [ ] キャッシュ無効化が機能する
- [ ] ヒット率が 80% 以上を維持

### パフォーマンス
- [ ] データ取得時間が 200ms 以内
- [ ] メモリ使用量が適切に管理されている
- [ ] データベース接続プールが最適化されている
- [ ] N+1 問題が発生していない

### 可用性・信頼性
- [ ] エラーハンドリングが適切
- [ ] フォールバック機能が動作する
- [ ] データ整合性が保たれている
- [ ] 監視・アラートが設定されている

## 🏃 実践的演習問題

### 演習 1: インテリジェントキャッシュシステム
AI を活用したキャッシュ戦略を実装してください。

```ts
interface CacheAnalytics {
  accessPattern: {
    frequency: number;
    lastAccess: Date;
    timeOfDayDistribution: number[];
    userTypeDistribution: Record<string, number>;
  };
  contentAnalytics: {
    importance: 'low' | 'medium' | 'high' | 'critical';
    updateFrequency: number;
    impactScope: string[];
  };
}

class IntelligentCacheManager {
  async decideCacheStrategy(
    updateId: string,
    analytics: CacheAnalytics
  ): Promise<CacheStrategy> {
    // アクセスパターンとコンテンツ重要度を基に最適化
    const { accessPattern, contentAnalytics } = analytics;
    
    if (contentAnalytics.importance === 'critical') {
      return {
        memoryTTL: 3600000, // 1時間
        databaseTTL: 86400000, // 24時間
        prefetchRelated: true,
        backgroundRefresh: true,
      };
    }
    
    if (accessPattern.frequency > 100) {
      return {
        memoryTTL: 1800000, // 30分
        databaseTTL: 43200000, // 12時間
        prefetchRelated: false,
        backgroundRefresh: true,
      };
    }
    
    // 低頻度アクセスのコンテンツ
    return {
      memoryTTL: 300000, // 5分
      databaseTTL: 21600000, // 6時間
      prefetchRelated: false,
      backgroundRefresh: false,
    };
  }
}
```

### 演習 2: リアルタイムデータ同期
Supabase Realtime を使った双方向データ同期を実装。

```ts
export class RealtimeUpdateService {
  private supabase: SupabaseClient;
  private subscribers = new Map<string, Set<(update: AzureUpdate) => void>>();
  
  constructor() {
    this.supabase = createSupabaseClient();
    this.setupRealtimeSubscription();
  }
  
  private setupRealtimeSubscription() {
    this.supabase
      .channel('azure_updates_changes')
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'azure_updates',
        },
        (payload) => this.handleRealtimeUpdate(payload)
      )
      .subscribe();
  }
  
  private async handleRealtimeUpdate(payload: any) {
    const { eventType, new: newRecord, old: oldRecord } = payload;
    
    switch (eventType) {
      case 'INSERT':
        await this.broadcastNewUpdate(newRecord);
        break;
      case 'UPDATE':
        await this.broadcastUpdateChange(newRecord, oldRecord);
        break;
      case 'DELETE':
        await this.broadcastUpdateDeletion(oldRecord);
        break;
    }
  }
  
  subscribeToUpdate(updateId: string, callback: (update: AzureUpdate) => void) {
    if (!this.subscribers.has(updateId)) {
      this.subscribers.set(updateId, new Set());
    }
    this.subscribers.get(updateId)!.add(callback);
  }
  
  unsubscribeFromUpdate(updateId: string, callback: (update: AzureUpdate) => void) {
    this.subscribers.get(updateId)?.delete(callback);
  }
}
```

### 演習 3: 分散キャッシュ無効化
複数のサーバーインスタンス間でのキャッシュ無効化を実装。

```ts
import Redis from 'ioredis';

export class DistributedCacheInvalidator {
  private redis: Redis;
  private instanceId: string;
  
  constructor() {
    this.redis = new Redis(process.env.REDIS_URL);
    this.instanceId = `instance-${Math.random().toString(36).substr(2, 9)}`;
    this.setupInvalidationListener();
  }
  
  async invalidateUpdate(updateId: string, reason: string = 'manual') {
    const invalidationMessage = {
      updateId,
      reason,
      timestamp: Date.now(),
      sourceInstance: this.instanceId,
    };
    
    // Redis pub/sub で他のインスタンスに通知
    await this.redis.publish('cache_invalidation', JSON.stringify(invalidationMessage));
    
    // ローカルキャッシュも無効化
    await this.invalidateLocalCache(updateId);
  }
  
  private setupInvalidationListener() {
    this.redis.subscribe('cache_invalidation');
    
    this.redis.on('message', async (channel, message) => {
      if (channel === 'cache_invalidation') {
        const invalidation = JSON.parse(message);
        
        // 自分が送信したメッセージは無視
        if (invalidation.sourceInstance === this.instanceId) {
          return;
        }
        
        await this.invalidateLocalCache(invalidation.updateId);
        console.log(`Cache invalidated for ${invalidation.updateId} due to ${invalidation.reason}`);
      }
    });
  }
  
  private async invalidateLocalCache(updateId: string) {
    // メモリキャッシュから削除
    memoryCache.delete(updateId);
    
    // Next.js データキャッシュも無効化
    revalidateTag(`update-${updateId}`);
  }
}
```

## 🔗 詳細リソース

### 公式ドキュメント
- [Next.js Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [Supabase JavaScript Client](https://supabase.com/docs/reference/javascript/introduction)
- [PostgreSQL Performance Tuning](https://www.postgresql.org/docs/current/performance-tips.html)

### 学習リソース
- [Database Caching Strategies](https://aws.amazon.com/caching/database-caching/)
- [Data Architecture Patterns](https://martinfowler.com/articles/data-monolith-to-mesh.html)
- [Realtime Web Applications](https://web.dev/websockets/)

### Azure UpdateSnap 関連
- [Caching Architecture](../../docs/caching-strategy.md)
- [Database Schema](../../supabase/migrations/)
- [Performance Monitoring](../../docs/monitoring.md)

---

**データレイヤー完成！** 高性能で拡張性のあるデータアーキテクチャが構築できました。次は [Step 5: 動的 OG 画像生成](../step-05-og-images) でソーシャルメディア最適化を実装します。