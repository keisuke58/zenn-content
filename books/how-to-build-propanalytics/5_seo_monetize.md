---
title: "SEO戦略・収益化・アフィリエイト設計"
---

## SEO：データサイトで検索流入を取る戦略

PropAnalyticsのSEO戦略は「**ロングテールキーワード × 大量静的ページ**」だ。

### 生成しているページ群

| ページ種別 | 数 | 主なキーワード |
|-----------|------|-------------|
| 物件詳細 | 124件 | 「{物件名} 投資」「{物件名} 坪単価」 |
| 区別ガイド | 26件 | 「港区 不動産投資」「中央区 再開発」 |
| 路線別 | 6件 | 「山手線 再開発」「東横線 投資」 |
| 駅別 | 116件 | 「渋谷駅 再開発」「虎ノ門ヒルズ駅 投資」 |
| ガイド記事 | 8件 | 「東京 不動産投資 2025」「IRR 計算」 |

合計300ページ超をSSGで生成し、すべてに canonical・OGP・構造化データを付与している。

### `generateStaticParams` でSEOページを大量生成

```typescript
// app/guide/[ward]/page.tsx
export async function generateStaticParams() {
  const wards = [...new Set(REDEVELOPMENT_PROJECTS.map(p => p.cityCode))];
  return WARD_SLUGS.map(slug => ({ ward: slug }));
}
```

### サイトマップの自動生成

```typescript
// app/sitemap.ts
export default function sitemap(): MetadataRoute.Sitemap {
  const projectUrls = ALL_PROJECTS.map(p => ({
    url: `${BASE}/projects/${p.id}`,
    lastModified: new Date(),
    changeFrequency: "monthly" as const,
    priority: p.investmentScore >= 90 ? 0.9 : 0.8,
  }));
  return [...staticUrls, ...projectUrls, ...guideUrls];
}
```

investmentScoreが高い物件ほど priority を上げている。

## 収益化：アフィリエイト × サブスクリプションの二本柱

### アフィリエイト設計

不動産投資系の高単価アフィリエイト（住宅ローン・不動産会社）を各ページのCTAに組み込む。

**A8.net Link Managerを使った実装：**

```tsx
// app/layout.tsx
<Script src="//statics.a8.net/a8link/a8linkmgr.js" strategy="afterInteractive" />
<Script id="a8-linkmgr" strategy="afterInteractive">{`
  a8linkmgr({ "config_id": "ltL7E8hbCo72ZRzRFguO" });
`}</Script>
```

重要なのは**href に直接アフィリエイトのランディングURLを貼ること**。プロキシAPI（`/api/track?url=...`）を経由するとLink Managerがリンクを差し替えられないので注意。

```tsx
// ✅ 正しい：directUrlをhrefに直接使う
<a href={program.directUrl} onClick={() => trackClick(program)}>
  申し込む
</a>
```

クリック計測はGA4のカスタムイベントで行う：

```typescript
window.gtag("event", "affiliate_click", {
  affiliate_id: program.id,
  affiliate_name: program.name,
});
```

### Stripeサブスクリプション

```
無料プラン: 基本スコア・物件一覧・利回り計算（基本）
STARTERプラン（¥980/月）: 坪単価・AIレポート・IRR計算
PROプラン（¥2,980/月）: 全機能 + CSV + 比較4件
```

Checkout Sessionにメタデータとしてemailとplanを埋め込み、Webhook受信時にSupabaseを更新する設計。

## Vercel Cron：自動化の仕組み

```
毎週日曜 09:00（JST）  → ニュースレター配信（TOP5レポート）
毎週火曜 09:00（JST）  → スコアアラートメール
毎日 10:00（JST）      → X（Twitter）自動投稿
月次                   → サイトマップping（Googleに更新通知）
```

週次ニュースレターはResendで配信。無料ユーザーには3位までの表示 + PROへの誘導CTAを入れてコンバージョンを狙う。

## 振り返り：やって良かったこと・反省点

**良かった点：**
- 静的データ + 静的生成の組み合わせでSEO基盤が強い
- Supabase RLSでセキュリティをDB層で担保できた
- Vercel + Stripe + Supabaseの組み合わせは開発速度が速い

**反省点：**
- 初期は物件データをDBに入れようとして設計を迷った（結局静的に落ち着くまで1週間ロス）
- A8リンクマネージャーの挙動を理解せず、プロキシAPIを作って動かないと気付くまで数時間かかった
- `ssr: false` をServer Componentで使おうとしてビルドエラー→Client Componentラッパーが必要と学んだ

## まとめ

Next.js × Supabase × Vercel の組み合わせは、個人開発で「データ可視化SaaS + SEO集客 + 決済」を同時に実現するのに適している。インフラをほぼ気にせず機能開発に集中できる。

PropAnalyticsのコードは実装の詳細を公開しているので、参考にしてもらえれば。

---

> 東京再開発プロジェクト82件のデータを無料公開中: [PropAnalytics](https://real-estate-tracker-web-zeta.vercel.app)
