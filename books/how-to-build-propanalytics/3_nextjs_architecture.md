---
title: "Next.js 15 App Router：SSG・SSR・Client Componentの使い分け"
---

## コンポーネント分類の唯一の基準

Next.js 15のApp Routerでは「どこに `"use client"` を書くか」がアーキテクチャの核心だ。基準はシンプルに1つだけ：

```text
インタラクション・状態・ブラウザAPIが要る → "use client"
それ以外はServer Componentのまま
```

PropAnalyticsの分類を具体的に示す：

| コンポーネント | 分類 | 理由 |
|-------------|------|------|
| `app/projects/[id]/page.tsx` | Server | SSGページ本体、SEO重視 |
| `app/projects/[id]/ProjectClient.tsx` | Client | タブ切替・地図表示 |
| `components/Map/RealEstateMap.tsx` | Client | Google Maps（ブラウザAPI） |
| `app/tools/yield-calculator/page.tsx` | Client | スライダー入力・計算 |
| `app/ranking/page.tsx` | Server | 静的ランキング表示 |
| `hooks/useSubscription.ts` | Client | Supabase購読・状態管理 |

**ServerとClientを混在させる正しいパターン**は「Server Componentの中でClient Componentをimportする」こと。逆（ClientからServer）はできない。

```typescript
// ✅ Server Component がページの骨格を担当
// app/projects/[id]/page.tsx
export default async function ProjectPage({ params }: { params: { id: string } }) {
  const project = ALL_PROJECTS.find((p) => p.id === params.id);
  if (!project) notFound();

  return (
    <main>
      {/* SEOコンテンツはサーバー側でレンダリング */}
      <h1>{project.projectName}</h1>
      <ScoreDisplay score={project.investmentScore} />
      
      {/* インタラクティブ部分だけClientに委譲 */}
      <ProjectClient project={project} />
    </main>
  );
}
```

## SSGで124ページを生成する

`generateStaticParams` が全物件ページのSSGを担う。ビルド時に一度だけ実行され、124件分のHTMLが生成される。

```typescript
// app/projects/[id]/page.tsx
export async function generateStaticParams() {
  return [...REDEVELOPMENT_PROJECTS, ...HISTORICAL_PROJECTS]
    .map((p) => ({ id: p.id }));
}

// generateMetadata で物件ごとのSEOを動的設定
export async function generateMetadata(
  { params }: { params: { id: string } }
): Promise<Metadata> {
  const project = ALL_PROJECTS.find((p) => p.id === params.id);
  if (!project) return {};

  return {
    title: `${project.projectName} 投資分析 | 投資スコア${project.investmentScore}点`,
    description: `${project.projectName}（${project.ward}）の投資スコア・坪単価・ハザードリスクを分析。`,
    alternates: {
      canonical: `https://real-estate-tracker-web-zeta.vercel.app/projects/${project.id}`,
    },
    openGraph: {
      title: `${project.projectName} — 投資スコア ${project.investmentScore}/100`,
      images: [`/api/og?id=${project.id}`],
    },
  };
}
```

**124件 × `generateMetadata` 分のHTMLがビルド時に生成される。** デプロイ後はエッジから静的ファイルが配信され、DBアクセスは一切発生しない。

## Google MapsのSSR問題と解決策

Google MapsはブラウザAPIに依存するため、サーバー側でレンダリングできない。しかし `dynamic(..., { ssr: false })` はServer Component内では使えない。

解決策は「`"use client"` ラッパーを一枚かます」こと：

```typescript
// app/projects/[id]/MapLoader.tsx  ← Client Component
"use client";

import dynamic from "next/dynamic";

// このファイルがClient ComponentなのでSSR無効化が使える
const RealEstateMap = dynamic(
  () => import("@/components/Map/RealEstateMap"),
  {
    ssr: false,
    loading: () => <div className="h-64 bg-slate-800 animate-pulse rounded-lg" />,
  }
);

export default function MapLoader({ lat, lng }: { lat: number; lng: number }) {
  return <RealEstateMap lat={lat} lng={lng} />;
}
```

```typescript
// app/projects/[id]/page.tsx (Server Component)
import MapLoader from "./MapLoader"; // ← Client Componentをimportするだけ

export default function ProjectPage(...) {
  return (
    <main>
      <MapLoader lat={project.lat} lng={project.lng} />
    </main>
  );
}
```

## メタデータとClient Componentの共存

`"use client"` を付けたページには `export const metadata` が書けない。計算ツールのように「SEOが必要だがインタラクティブ」なページは、`layout.tsx` にメタデータを分離する：

```text
app/tools/yield-calculator/
  layout.tsx    ← Server Component。metadataだけ書く
  page.tsx      ← "use client"。計算ロジックとUIだけ
```

```typescript
// layout.tsx
import type { Metadata } from "next";
export const metadata: Metadata = {
  title: "利回り・IRR計算ツール | PropAnalytics",
  description: "購入価格・自己資金・家賃から表面利回り・実質利回り・IRRを即時計算。",
};
export default function Layout({ children }: { children: React.ReactNode }) {
  return children;
}
```

## Vercel Cron Jobsの設定

定期実行処理は `vercel.json` でCronを定義する：

```json
{
  "crons": [
    { "path": "/api/cron/newsletter",  "schedule": "0 0 * * 0" },
    { "path": "/api/cron/score-alert", "schedule": "0 0 * * 2" },
    { "path": "/api/cron/tweet",       "schedule": "0 1 * * *" }
  ]
}
```

認証はBearerトークンで行う。Vercelは自動的に `Authorization` ヘッダーを付与してくれる：

```typescript
export async function POST(req: Request) {
  const auth = req.headers.get("authorization");
  if (auth !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "unauthorized" }, { status: 401 });
  }
  // ... 処理
}
```

`CRON_SECRET` はVercelのEnvironment Variablesに設定し、ローカル開発時は `.env.local` に置く。Vercelダッシュボードから手動実行もできる。

## Streaming RSCでAIレポートを高速表示

Claude APIのレスポンスはストリーミングで受け取ると初期表示が速くなる。App RouterのServer Componentは `<Suspense>` でストリーミングに対応している：

```typescript
// app/projects/[id]/page.tsx
import { Suspense } from "react";

export default function ProjectPage({ params }) {
  return (
    <main>
      <Suspense fallback={<ReportSkeleton />}>
        {/* この中のコンポーネントが非同期でも表示が始まる */}
        <AIReportSection projectId={params.id} />
      </Suspense>
    </main>
  );
}
```

```typescript
// AIReportSection.tsx (Server Component)
export async function AIReportSection({ projectId }: { projectId: string }) {
  const project = ALL_PROJECTS.find((p) => p.id === projectId);
  // Claudeへの問い合わせはPRO/Starter限定
  // ここはサーバーコンポーネントなのでAPIキーを安全に使える
  const report = await generateReport(project);
  return <ReportDisplay content={report} />;
}
```

PageコンポーネントはSuspenseまで即座に返し、レポートのみ後続でストリームされる。ユーザーが待つ時間の体感が大幅に改善する。
