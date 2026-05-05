---
title: "Next.js 15 App Router：SSG・SSR・Client Componentの使い分け"
---

## App Routerで最初に決めること

Next.js 15のApp Routerでは、すべてのコンポーネントはデフォルトでServer Componentだ。`"use client"` を付けるとClient Componentになる。

**判断基準はシンプル：**

```
インタラクション・状態・ブラウザAPIが必要 → "use client"
それ以外 → Server Component（デフォルト）
```

PropAnalyticsでの分類：

| コンポーネント | 種別 | 理由 |
|-------------|------|------|
| `app/projects/[id]/page.tsx` | Server | SSGで静的生成、SEO重視 |
| `app/projects/[id]/ProjectClient.tsx` | Client | 地図操作・タブ切替 |
| `components/Map/RealEstateMap.tsx` | Client | Google Maps API（ブラウザのみ） |
| `hooks/useSubscription.ts` | Client | Supabaseリアルタイム購読 |
| `app/tools/yield-calculator/page.tsx` | Client | スライダー・計算インタラクション |

## SSGで82件の物件ページを生成する

```typescript
// app/projects/[id]/page.tsx
export async function generateStaticParams() {
  return [...REDEVELOPMENT_PROJECTS, ...HISTORICAL_PROJECTS]
    .map((p) => ({ id: p.id }));
}

export default function ProjectPage({ params }: { params: { id: string } }) {
  const project = ALL_PROJECTS.find((p) => p.id === params.id);
  // ...
}
```

`generateStaticParams` でビルド時に全ページ生成。DBへのアクセスなし、エッジで配信されるため初回ロードが高速。

## ハマったポイント：`ssr: false` はServer Componentで使えない

```typescript
// ❌ これはエラーになる（Server Component内）
const CompareView = dynamic(() => import("./CompareView"), { ssr: false });

// ✅ "use client" のラッパーを作る
// app/compare/CompareClientLoader.tsx
"use client";
const CompareView = dynamic(() => import("@/components/Compare/CompareView"), { ssr: false });
```

Google MapsなどブラウザAPIに依存するコンポーネントは `ssr: false` が必要だが、これはClient Componentの中でしか使えない。

## Metadata APIとSEO

App Routerの`metadata`エクスポートでSEOを管理する：

```typescript
// app/guide/tokyo-investment-2025/page.tsx
export const metadata: Metadata = {
  title: "東京 不動産投資 2025年 完全ガイド｜再開発エリア × 利回り分析",
  description: "...",
  keywords: ["東京 不動産投資 2025", "東京 再開発 投資"],
  alternates: { canonical: `${BASE}/guide/tokyo-investment-2025` },
  openGraph: { ... },
};
```

注意点：`"use client"` と `metadata` の共存は不可。計算機など Client Component が必要なページは `layout.tsx` に `metadata` を分離する：

```
app/tools/yield-calculator/
  layout.tsx   ← metadata だけ書く（Server Component）
  page.tsx     ← "use client" で計算ロジック
```

## Vercel Cron Jobsの設定

`vercel.json` でCronを定義：

```json
{
  "crons": [
    { "path": "/api/cron/newsletter", "schedule": "0 0 * * 0" },
    { "path": "/api/cron/score-alert", "schedule": "0 0 * * 2" },
    { "path": "/api/cron/tweet",       "schedule": "0 1 * * *" }
  ]
}
```

認証はシンプルにBearerトークン：

```typescript
if (req.headers.get("authorization") !== `Bearer ${process.env.CRON_SECRET}`) {
  return NextResponse.json({ error: "unauthorized" }, { status: 401 });
}
```
