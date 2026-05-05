---
title: "Next.js App RouterでSSG+SEOページを100枚量産する設計パターン"
emoji: "📄"
type: "tech"
topics: ["nextjs", "seo", "typescript", "個人開発"]
published: false
---

## やったこと

不動産投資データサービス [PropAnalytics](https://real-estate-tracker-web-zeta.vercel.app) で、東京の再開発プロジェクト82件・区別ガイド26件・駅別116件など **300ページ超をSSGで自動生成**した。そのときの設計パターンをまとめる。

---

## 基本：`generateStaticParams` でページを量産する

```typescript
// app/projects/[id]/page.tsx
import { REDEVELOPMENT_PROJECTS } from "@/lib/constants/redevelopment";

export async function generateStaticParams() {
  return REDEVELOPMENT_PROJECTS.map((p) => ({ id: p.id }));
}

export default function ProjectPage({
  params,
}: {
  params: { id: string };
}) {
  const project = REDEVELOPMENT_PROJECTS.find((p) => p.id === params.id);
  if (!project) notFound();
  return <ProjectDetail project={project} />;
}
```

`generateStaticParams` が返す配列の要素ごとにビルド時にページが生成される。DBアクセスは不要で、TypeScriptの定数から直接生成できる。

---

## SEOに必要な3点セット

### 1. Metadata API

```typescript
export async function generateMetadata({
  params,
}: {
  params: { id: string };
}): Promise<Metadata> {
  const project = REDEVELOPMENT_PROJECTS.find((p) => p.id === params.id);
  if (!project) return {};

  return {
    title: `${project.projectName} 投資分析｜坪単価・利回り・スコア`,
    description: `${project.projectName}（${project.completionYear}年完成）の投資スコア${project.investmentScore}点。坪単価・地価・ハザードリスクをデータで分析。`,
    alternates: {
      canonical: `https://example.com/projects/${project.id}`,
    },
    openGraph: {
      title: `${project.projectName} | PropAnalytics`,
      description: project.description,
      url: `https://example.com/projects/${project.id}`,
    },
  };
}
```

### 2. sitemap.ts で全URLを登録

```typescript
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { REDEVELOPMENT_PROJECTS } from "@/lib/constants/redevelopment";

const BASE = "https://example.com";

export default function sitemap(): MetadataRoute.Sitemap {
  const projects = REDEVELOPMENT_PROJECTS.map((p) => ({
    url: `${BASE}/projects/${p.id}`,
    lastModified: new Date(),
    changeFrequency: "monthly" as const,
    // スコアが高いページを優先
    priority: p.investmentScore >= 90 ? 0.9 : 0.8,
  }));

  return [
    { url: BASE, changeFrequency: "daily", priority: 1.0 },
    { url: `${BASE}/ranking`, changeFrequency: "weekly", priority: 0.9 },
    ...projects,
  ];
}
```

### 3. robots.ts

```typescript
// app/robots.ts
export default function robots() {
  return {
    rules: { userAgent: "*", allow: "/" },
    sitemap: "https://example.com/sitemap.xml",
  };
}
```

---

## データをどこに持つか：静的 vs DB

| 観点 | TypeScript定数 | Supabase DB |
|------|--------------|------------|
| ビルド時SSG | ◎ | △（DBアクセスが必要） |
| 更新頻度 | 月1〜2回 | リアルタイム |
| SEOキャッシュ | ◎ エッジで配信 | △ |
| Git管理 | ◎ | ✕ |

**更新頻度が低くSEOが重要なデータは静的定数が正解。**

PropAnalyticsでは物件データを `src/lib/constants/redevelopment.ts` に置き、ビルド時にすべてのページを生成している。ユーザー固有のデータ（ウォッチリスト・課金状態）だけSupabaseに持つ。

---

## `"use client"` と `metadata` の共存問題

インタラクティブなツール系ページは `"use client"` が必要だが、`metadata` エクスポートと共存できない。

```
app/tools/yield-calculator/
  layout.tsx  ← Server Component。metadata だけここに書く
  page.tsx    ← "use client" で計算ロジック
```

```typescript
// layout.tsx（Server Component）
export const metadata: Metadata = {
  title: "表面利回り・実質利回り計算ツール（無料）",
  description: "...",
};
export default function Layout({ children }: { children: React.ReactNode }) {
  return <>{children}</>;
}
```

```typescript
// page.tsx
"use client";
export default function YieldCalculatorPage() {
  const [price, setPrice] = useState(50000000);
  // ...
}
```

---

## ビルド時間の最適化

300ページを生成するとビルドが遅くなる。対策：

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    // 並列ワーカー数を増やす
    workerThreads: true,
    cpus: 4,
  },
};
```

PropAnalyticsではVercel上で約52秒でビルドが完了している。

---

## まとめ

- `generateStaticParams` + TypeScript定数で大量SSGページを生成
- Metadata API + sitemap.ts + robots.ts の3点セットでSEO対応
- 計算ツール系は `layout.tsx` に metadata を分離して `"use client"` と共存
- 更新頻度が低いデータは静的定数に持つとSSGと相性が良い

この設計でPropAnalyticsは300ページ超を管理している。詳細は本「[不動産投資データサービスをNext.js×Supabaseで作った全記録](https://zenn.dev/keisuke58/books/how-to-build-propanalytics)」で解説している。
