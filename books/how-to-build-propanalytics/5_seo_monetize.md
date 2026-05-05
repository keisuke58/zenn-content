---
title: "SEO戦略・収益化：300ページのコンテンツ基盤をどう設計したか"
---

## SEO戦略の前提：データサイトに向いているアプローチ

PropAnalyticsのSEO戦略は「**ロングテールキーワード × 大量静的ページ**」の一点突破だ。

データサイトには特有の強みがある。物件・エリア・路線・駅という多次元の軸でページを生成できる。各ページが異なるロングテールキーワードを自然に捕捉する。

### 生成しているページ群

| 種別 | ページ数 | 狙うキーワード例 |
|------|--------|--------------|
| 物件詳細 | 124件 | 「TORCH TOWER 投資」「麻布台ヒルズ 坪単価」 |
| 区別ガイド | 26件 | 「港区 不動産投資 2026」「中央区 再開発」 |
| 路線別 | 6件 | 「山手線 再開発 投資」「東横線 沿線 物件」 |
| 駅別 | 116件 | 「虎ノ門ヒルズ駅 再開発」「渋谷駅 周辺 投資」 |
| ガイド記事 | 8件 | 「東京 不動産投資 2026」「IRR 計算 方法」 |
| **合計** | **280件超** | — |

すべてSSGで生成し、canonical・OGP・構造化データを付与している。

## `generateStaticParams` でSEOページを大量生成

区別ガイドページの生成例：

```typescript
// app/area/[ward]/page.tsx

export async function generateStaticParams() {
  // 物件データから区の一覧を抽出（重複排除）
  const wards = [...new Set(REDEVELOPMENT_PROJECTS.map((p) => p.ward))];
  return wards.map((ward) => ({ ward: encodeURIComponent(ward) }));
}

export async function generateMetadata(
  { params }: { params: { ward: string } }
): Promise<Metadata> {
  const ward = decodeURIComponent(params.ward);
  const projects = REDEVELOPMENT_PROJECTS.filter((p) => p.ward === ward);
  const avgScore = Math.round(
    projects.reduce((s, p) => s + p.investmentScore, 0) / projects.length
  );

  return {
    title: `${ward}の不動産投資完全ガイド2026 | 再開発${projects.length}件分析`,
    description: `${ward}の再開発プロジェクト${projects.length}件を分析。平均投資スコア${avgScore}点。坪単価・利回り・ハザードリスクをデータで解説。`,
    alternates: {
      canonical: `https://real-estate-tracker-web-zeta.vercel.app/area/${params.ward}`,
    },
  };
}
```

## サイトマップの自動生成

`app/sitemap.ts` でNext.jsに組み込まれたサイトマップAPIを使う：

```typescript
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { ALL_PROJECTS } from "@/lib/constants/redevelopment";

const BASE = "https://real-estate-tracker-web-zeta.vercel.app";

export default function sitemap(): MetadataRoute.Sitemap {
  const staticUrls = [
    { url: BASE, changeFrequency: "daily" as const, priority: 1.0 },
    { url: `${BASE}/ranking`, changeFrequency: "weekly" as const, priority: 0.9 },
    { url: `${BASE}/tools/yield-calculator`, changeFrequency: "monthly" as const, priority: 0.8 },
  ];

  const projectUrls = ALL_PROJECTS.map((p) => ({
    url: `${BASE}/projects/${p.id}`,
    lastModified: new Date(),
    changeFrequency: "monthly" as const,
    priority: p.investmentScore >= 90 ? 0.9 : 0.8,
  }));

  const wardUrls = WARD_SLUGS.map((ward) => ({
    url: `${BASE}/area/${encodeURIComponent(ward)}`,
    changeFrequency: "monthly" as const,
    priority: 0.7,
  }));

  return [...staticUrls, ...projectUrls, ...wardUrls];
}
```

`investmentScore >= 90` の物件（TORCH TOWER・麻布台ヒルズ等）はpriority 0.9に設定し、Googleに重要ページを示す。

## 収益化：アフィリエイト × サブスクリプションの二本柱

### サブスクリプション設計

```text
無料プラン      基本スコア・物件一覧・利回り計算（基本）
STARTER ¥980/月  坪単価・AIレポート・IRR計算・ウォッチリスト
PRO ¥2,980/月   全機能 + CSV出力 + 4件比較 + 優先サポート
```

価格設定の判断：競合の専門DB（年間数十万円）との比較で圧倒的に安い。STARTERを「月1回コーヒーを我慢する程度」に設定することで、意思決定コストを下げた。

### A8.netアフィリエイトの実装

`a8linkmgr.js` を使うと、リンクのURLをA8側で動的に管理できる。重要なのは**hrefには直接ランディングURLを書く**こと：

```tsx
// app/layout.tsx
import Script from "next/script";

// A8 Link Manager：href を動的に書き換えるスクリプト
<Script src="//statics.a8.net/a8link/a8linkmgr.js" strategy="afterInteractive" />
<Script id="a8-linkmgr" strategy="afterInteractive">{`
  a8linkmgr({ "config_id": "ltL7E8hbCo72ZRzRFguO" });
`}</Script>
```

```tsx
// components/LeadGen/MortgageCTA.tsx
export function MortgageCTA({ program }: { program: AffiliateProgram }) {
  const handleClick = () => {
    // GA4でクリック計測
    window.gtag?.("event", "affiliate_click", {
      affiliate_id: program.id,
      affiliate_name: program.name,
      page_location: window.location.href,
    });
  };

  return (
    // href に直接ランディングURLを書く（プロキシAPIを経由しない）
    <a
      href={program.directUrl}
      target="_blank"
      rel="noopener noreferrer"
      onClick={handleClick}
      className="block rounded-lg border border-blue-500/30 p-4 hover:border-blue-400"
    >
      {program.name} — {program.reward}
    </a>
  );
}
```

プロキシAPIを経由するとLink Managerがhrefを書き換えられなくなる。必ず直接URLをhrefに設定すること。

## Vercel Cron：自動化ループの設計

```text
毎週日曜 09:00 JST   ニュースレター（TOP5プロジェクトのAI分析）
毎週火曜 09:00 JST   スコアアラート（ウォッチリスト変動通知）
毎日    10:00 JST   X（Twitter）自動投稿（高スコア物件の紹介）
```

週次ニュースレターはResendで配信。無料ユーザーには3位まで表示し、PRO誘導CTAを入れる。

```typescript
// app/api/cron/newsletter/route.ts
export async function POST(req: Request) {
  if (req.headers.get("authorization") !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "unauthorized" }, { status: 401 });
  }

  // TOP5プロジェクトを投資スコアでソート
  const top5 = [...ALL_PROJECTS]
    .sort((a, b) => b.investmentScore - a.investmentScore)
    .slice(0, 5);

  // 全メール購読者を取得
  const { data: subscribers } = await adminSupabase
    .from("premium_members")
    .select("email, plan");

  // Resendで一括配信
  const { Resend } = await import("resend");
  const resend = new Resend(process.env.RESEND_API_KEY);

  for (const sub of subscribers ?? []) {
    const visibleProjects = sub.plan === "free" ? top5.slice(0, 3) : top5;
    await resend.emails.send({
      from: "PropAnalytics <newsletter@yourdomain.com>",
      to: sub.email,
      subject: `今週のTOP${visibleProjects.length}再開発プロジェクト`,
      html: buildNewsletterHTML(visibleProjects, sub.plan),
    });
  }

  return NextResponse.json({ sent: subscribers?.length ?? 0 });
}
```

## 振り返り：何が効いて何が不要だったか

**効果が高かった施策：**

- 静的データ + SSGによるページ大量生成（SEO基盤が速く作れた）
- Supabase RLSでセキュリティをDB層に集約（APIのアクセス制御コードが激減）
- A8リンクマネージャーの正しい使い方を把握してからCTRが改善

**やらなくて良かったこと：**

- 複雑なキャッシュ戦略（`revalidate` で十分だった）
- 管理画面の作り込み（物件データ更新はGitのPRで十分）
- 細かい分析ダッシュボード（GA4で代替できる範囲だった）

Next.js × Supabase × Vercel の組み合わせは「インフラを気にせず機能開発に集中できる」という意味で個人開発に最適だ。スタック全体で月1,000〜3,000円程度で動かせる。

---

> PropAnalyticsのデータは [こちらで無料公開中](https://real-estate-tracker-web-zeta.vercel.app/ranking)
