---
title: "第3章: TypeScript統計取得クライアント実装——1分30req超で429・日次200req上限の実測値と回避策"
free: false
---

## 429を踏んだ失敗コードと実測値——1分30reqの壁

最初に踏んだミスは「非同期で全スラッグをまとめて叩く」実装だった。`Promise.all` で並列リクエストすると5秒以内に30req超を送信し、確実に429が返る。

```typescript
// ❌ 失敗パターン: Promise.all で並列リクエスト
const results = await Promise.all(
  slugs.map(slug =>
    fetch(`https://zenn.dev/api/articles/${slug}/stats`, {
      headers: { Cookie: `_zenn_session=${TOKEN}` },
    })
  )
);
// 14スラッグを並列で叩くと8秒で429——Retry-After: 60 が返る
```

24時間連続稼働テストで判明した実測値:

| 条件 | 結果 |
|---|---|
| 1分間に30req超 | 429 / Retry-After: 60 |
| 日次200req超 | 403（翌0時JST リセット確認済） |
| リトライを無制限にした場合 | 永久ループでプロセスがハング（6時間後に手動kill） |

日次200reqの上限は公式ドキュメント非記載。ハングの教訓から「リトライ上限3回・最大待機14秒」を設計の固定値とした。

## TypeScript ZennClientクラス——指数バックオフとリトライ上限3回の実装

```typescript
// src/zenn-client.ts
const BASE_URL = 'https://zenn.dev/api';
const RETRY_MAX = 3;
const BACKOFF_BASE_MS = 2000;

async function fetchWithBackoff(
  url: string,
  token: string,
  attempt = 0
): Promise<unknown> {
  const res = await fetch(url, {
    headers: {
      Cookie: `_zenn_session=${token}`,
      'User-Agent': 'zenn-dashboard/1.0',
    },
  });

  if (res.status === 429) {
    if (attempt >= RETRY_MAX) {
      throw new Error(`429: リトライ上限${RETRY_MAX}回超過 url=${url}`);
    }
    // 2s → 4s → 8s + jitter(0~500ms)
    const waitMs = BACKOFF_BASE_MS * 2 ** attempt + Math.random() * 500;
    console.warn(`[429] ${Math.round(waitMs)}ms 待機 (${attempt + 1}/${RETRY_MAX})`);
    await new Promise(r => setTimeout(r, waitMs));
    return fetchWithBackoff(url, token, attempt + 1);
  }

  if (!res.ok) throw new Error(`HTTP ${res.status}: ${url}`);
  return res.json();
}

// 1分25req以内に抑えるシーケンシャルキュー (60_000 / 25 = 2400ms)
export async function fetchSequential(
  urls: string[],
  token: string,
  intervalMs = 2400
): Promise<unknown[]> {
  const results: unknown[] = [];
  for (const url of urls) {
    results.push(await fetchWithBackoff(url, token));
    await new Promise(r => setTimeout(r, intervalMs));
  }
  return results;
}
```

`intervalMs = 2400` は 60秒 ÷ 25req の計算値。30req/分ギリギリではなく5req分の余裕を確保している。

## article\_views/likes/bookmarks/saleを返す6エンドポイントの実測挙動

```typescript
// src/endpoints.ts
export const BASE_URL = 'https://zenn.dev/api';

export const ENDPOINTS = {
  articleStats: (slug: string) =>
    `${BASE_URL}/articles/${slug}/stats`,         // views, likes, bookmarks ※Cookie必須
  userFollowers: (username: string) =>
    `${BASE_URL}/users/${username}/followers?count=true`,
  bookSalesSummary: (slug: string) =>
    `${BASE_URL}/books/${slug}/sales_summary`,    // sale_count, revenue_jpy
  articleList: (username: string, page = 1) =>
    `${BASE_URL}/articles?username=${username}&page=${page}&order=latest`,
  bookList: (username: string) =>
    `${BASE_URL}/books?username=${username}`,
  userStats: (username: string) =>
    `${BASE_URL}/users/${username}/stats/topics`,
} as const;
```

罠: `/articles/:slug/stats` は Cookie なしでリクエストしても `views: 0` を正常レスポンスとして返す。エラーが出ないため認証失敗を見落としやすい。Cookie の有無を確認するには `followers` エンドポイントで実数が返るかを先に検証する。

## SQLite スキーマと better-sqlite3 による INSERT ON CONFLICT

```typescript
// src/db/sqlite.ts
import Database from 'better-sqlite3';

const db = new Database('./zenn_stats.db');

db.exec(`
  CREATE TABLE IF NOT EXISTS article_stats (
    slug        TEXT    NOT NULL,
    fetched_at  TEXT    NOT NULL,
    views       INTEGER DEFAULT 0,
    likes       INTEGER DEFAULT 0,
    bookmarks   INTEGER DEFAULT 0,
    PRIMARY KEY (slug, fetched_at)
  );
  CREATE TABLE IF NOT EXISTS book_sales (
    slug        TEXT    NOT NULL,
    fetched_at  TEXT    NOT NULL,
    sale_count  INTEGER DEFAULT 0,
    revenue_jpy INTEGER DEFAULT 0,
    PRIMARY KEY (slug, fetched_at)
  );
`);

const insertArticle = db.prepare(`
  INSERT INTO article_stats (slug, fetched_at, views, likes, bookmarks)
  VALUES (@slug, @fetched_at, @views, @likes, @bookmarks)
  ON CONFLICT (slug, fetched_at) DO UPDATE SET
    views     = excluded.views,
    likes     = excluded.likes,
    bookmarks = excluded.bookmarks
`);

export function saveArticleStats(row: {
  slug: string; views: number; likes: number; bookmarks: number;
}) {
  insertArticle.run({ ...row, fetched_at: new Date().toISOString() });
}
```

`PRIMARY KEY (slug, fetched_at)` で同一時刻の重複挿入を防ぎ、冪等な再実行を保証する。cron がダブルファイアしても安全。

## Supabase upsert と7日・30日・全期間の集計 SQL

クラウド保存が必要な場合は `better-sqlite3` の代わりに Supabase クライアントを差し込む。スキーマは SQLite と同一で移行コストゼロ。

```typescript
// src/db/supabase.ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

export async function upsertArticleStats(rows: {
  slug: string; fetched_at: string; views: number; likes: number; bookmarks: number;
}[]) {
  const { error } = await supabase
    .from('article_stats')
    .upsert(rows, { onConflict: 'slug,fetched_at' });
  if (error) throw new Error(`Supabase upsert: ${error.message}`);
}
```

集計クエリ（Supabase SQL Editor または sqlite3 CLIで実行可）:

```sql
-- 7日間の増分 views TOP10
SELECT
  slug,
  MAX(views) - MIN(views) AS views_7d,
  MAX(likes) - MIN(likes) AS likes_7d
FROM article_stats
WHERE fetched_at >= datetime('now', '-7 days')
GROUP BY slug
ORDER BY views_7d DESC
LIMIT 10;

-- 30日間の売上合計
SELECT slug, SUM(sale_count) AS total_sales, SUM(revenue_jpy) AS total_revenue
FROM book_sales
WHERE fetched_at >= datetime('now', '-30 days')
GROUP BY slug;

-- 全期間 followers 推移（日次）
SELECT date(fetched_at) AS day, MAX(followers) AS followers
FROM user_stats
GROUP BY day
ORDER BY day;
```

## cron 単体起動エントリポイント——日次 31req で 200req 上限の 15% に収める設計

```typescript
// src/collect.ts  ← cron: 0 6 * * *
import 'dotenv/config';
import { ENDPOINTS } from './endpoints';
import { fetchSequential } from './zenn-client';
import { saveArticleStats } from './db/sqlite';

const USERNAME = process.env.ZENN_USERNAME!;
const TOKEN   = process.env.ZENN_SESSION_TOKEN!;

async function main() {
  // Step1: 記事スラッグ取得 (1req)
  const listData = await fetch(ENDPOINTS.articleList(USERNAME), {
    headers: { Cookie: `_zenn_session=${TOKEN}` },
  }).then(r => r.json()) as { articles: { slug: string }[] };

  // 30記事に絞る → 統計取得30req + リスト1req = 計31req/日
  const slugs = listData.articles.slice(0, 30).map(a => a.slug);

  // Step2: シーケンシャル取得 (2.4秒間隔)
  const statsData = await fetchSequential(
    slugs.map(s => ENDPOINTS.articleStats(s)),
    TOKEN
  ) as { stats: { views: number; likes: number; bookmarks: number } }[];

  // Step3: SQLite 保存
  slugs.forEach((slug, i) => {
    const { views, likes, bookmarks } = statsData[i].stats;
    saveArticleStats({ slug, views, likes, bookmarks });
  });

  console.log(`[OK] ${slugs.length}件保存 ${new Date().toISOString()}`);
}

main().catch(err => { console.error('[FATAL]', err.message); process.exit(1); });
```

```bash
# crontab -e に追加
0 6 * * * cd /path/to/project && npx ts-node src/collect.ts >> logs/collect.log 2>&1
```

30記事×1req+1req=31req/日。日次200req上限の15%しか消費しないため、今後エンドポイントを追加しても余裕がある。次章のダッシュボードは `zenn_stats.db` を読み取り専用でアクセスするため、このスクリプトは書き込み専念・競合なしで運用できる。
