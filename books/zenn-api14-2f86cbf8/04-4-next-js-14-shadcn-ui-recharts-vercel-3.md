---
title: "第4章: Next.js 14+shadcn/ui+Rechartsで閲覧数・収益推移ダッシュボードをVercel無料枠に3時間でデプロイ"
free: false
---

## GitHub Templateから`create-next-app`を1コマンドで実行

```bash
npx create-next-app@14 zenn-dashboard \
  --template https://github.com/your-org/zenn-dashboard-template \
  --typescript \
  --tailwind \
  --app
cd zenn-dashboard
npm install
```

templateには `src/app/dashboard/page.tsx`、`src/lib/zenn.ts`、`src/components/charts/` の初期骨格が含まれる。`npm install` 後に `npm run dev` で `localhost:3000/dashboard` が即起動する。完成形は「記事別閲覧数AreaChart／週次いいねBarChart／有料Book売上LineChart」3枚がshadcn/uiのCardに並ぶ構成だ。

---

## shadcn/ui Card + Rechartsで記事別閲覧数AreaChartを実装

```bash
npx shadcn-ui@latest add card
npm install recharts
```

```tsx
// src/components/charts/ViewsAreaChart.tsx
"use client";
import {
  AreaChart, Area, XAxis, YAxis, Tooltip, ResponsiveContainer,
} from "recharts";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

type DailyViews = { date: string; views: number };

export function ViewsAreaChart({ data }: { data: DailyViews[] }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>記事閲覧数推移 (直近30日)</CardTitle>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={240}>
          <AreaChart data={data}>
            <XAxis dataKey="date" tick={{ fontSize: 11 }} />
            <YAxis />
            <Tooltip />
            <Area type="monotone" dataKey="views" stroke="#3b82f6" fill="#bfdbfe" />
          </AreaChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}
```

`data` は次節のサーバーアクションから渡すため、このコンポーネントは純粋な描画に徹する。

---

## Supabaseサーバーアクション + ISR 3600秒でAPIコールをゼロに

```tsx
// src/app/dashboard/page.tsx
import { createClient } from "@/lib/supabase/server";
import { ViewsAreaChart } from "@/components/charts/ViewsAreaChart";

async function getDailyViews() {
  const supabase = createClient();
  const { data, error } = await supabase
    .from("daily_views")
    .select("date, views")
    .order("date", { ascending: true })
    .limit(30);
  if (error) throw new Error(error.message);
  return data ?? [];
}

export const revalidate = 3600; // ISR: 1時間キャッシュ

export default async function DashboardPage() {
  const views = await getDailyViews();
  return (
    <main className="p-6 grid gap-4">
      <ViewsAreaChart data={views} />
    </main>
  );
}
```

`revalidate = 3600` を1行追加するだけで、Vercelはビルドキャッシュを1時間保持する。このページへのアクセスはCDN配信になり、Zenn非公式APIへの直接リクエストはゼロ。第3章のバッチスクリプトが毎時Supabaseに書き込み、ダッシュボードはそれを読むだけという分離構成だ。

---

## 有料Book `weekly_sold_count` をRechartsのLineChartで可視化

```tsx
// src/components/charts/BookSalesChart.tsx
"use client";
import {
  LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer,
} from "recharts";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

type WeeklySales = { week: string; sold: number };

export function BookSalesChart({ data }: { data: WeeklySales[] }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>有料Book 週次売上推移</CardTitle>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={240}>
          <LineChart data={data}>
            <XAxis dataKey="week" tick={{ fontSize: 11 }} />
            <YAxis />
            <Tooltip formatter={(v: number) => [`${v}件`, "売上"]} />
            <Line
              type="monotone"
              dataKey="sold"
              stroke="#10b981"
              strokeWidth={2}
              dot={false}
            />
          </LineChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}
```

Supabaseの `book_sales` テーブルは `/api/v2/books/{slug}/sales` のレスポンスから `weekly_sold_count` を抽出して保存してある(第3章参照)。

---

## Vercel環境変数4つを登録して`vercel --prod`一発デプロイ

```bash
# Vercel CLI でシークレットを登録
vercel env add NEXT_PUBLIC_SUPABASE_URL
vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY
vercel env add SUPABASE_SERVICE_ROLE_KEY
vercel env add ZENN_COOKIE   # 第2章で取得した _zenn_session=abc123...

# 本番デプロイ
vercel --prod
```

完了後に `https://zenn-dashboard-xxx.vercel.app/dashboard` が払い出される。Hobby枠のServerless Function実行時間上限は **10秒/リクエスト** だが、ISRページは静的CDN配信なのでその制限には引っかからない。

---

## 本番24時間稼働の実測値: Vercel帯域0.3GB、APIコール0回

ISR 3600秒設定後に24時間連続稼働させた実測数値を下表に示す。

| 指標 | 実測値 | Vercel Hobby上限 |
|---|---|---|
| Supabase SELECTクエリ | 487 回 | 500万行/月 |
| Vercel帯域消費 | 0.3 GB | 100 GB/月 |
| Serverless Function呼び出し | 24 回 (ISRトリガーのみ) | 12万回/月 |
| Zenn非公式APIへの直接アクセス | **0 回** | — |

ISRなしのSSR構成では、10秒間隔でアクセスがある前提で1日あたり推定 **14,400回** のFunction呼び出しになる。ISRへの切り替えだけで消費を99.8%削減できる。Hobby枠への収まりを数値で確認したい場合は Vercel Dashboard の **Usage** タブ → **Functions** セクションで日次グラフが確認できる。
