---
title: "個人開発SaaSのアフィリエイト実装：A8.net Link ManagerをNext.jsに組み込む落とし穴"
emoji: "💰"
type: "tech"
topics: ["nextjs", "個人開発", "saas", "収益化"]
published: false
---

## A8.net Link Managerとは

A8.netが提供するA8 Link Managerは、サイト内のアフィリエイトプログラム参加企業のURLを**自動的にA8の計測URLに変換**するJavaScriptライブラリだ。

通常のアフィリエイトは：
1. A8の管理画面で計測URLを取得
2. そのURLをコードに直書き

Link Managerを使うと：
1. 広告主の直URLをhrefに書く
2. JS が自動的にA8の計測URLに変換

管理がシンプルになり、A8の承認状況が変わっても自動で追従する。

---

## Next.jsへの組み込み

`app/layout.tsx` の `<body>` 末尾に2つのScriptタグを追加する。

```tsx
// app/layout.tsx
import Script from "next/script";

export default function RootLayout({ children }) {
  return (
    <html lang="ja">
      <body>
        {children}

        {/* A8 Link Manager */}
        <Script
          src="//statics.a8.net/a8link/a8linkmgr.js"
          strategy="afterInteractive"
        />
        <Script id="a8-linkmgr" strategy="afterInteractive">
          {`a8linkmgr({ "config_id": "YOUR_CONFIG_ID" });`}
        </Script>
      </body>
    </html>
  );
}
```

`config_id` はA8の管理画面 → 「A8 Link Manager」から取得する。

---

## ⚠️ 落とし穴：プロキシAPIを経由するとリンクが変換されない

最初にやりがちなミス：クリック計測のために `/api/track?url=xxx` のようなプロキシAPIを作り、そこにリダイレクトさせる実装。

```tsx
// ❌ NG：プロキシAPI経由
<a href={`/api/track?url=${encodeURIComponent(program.directUrl)}`}>
  申し込む
</a>
```

これだとLink Managerは `/api/track` というURLしか見ておらず、広告主のドメインを検出できないため**A8の計測URLに変換されない**。

```tsx
// ✅ 正しい：広告主の直URLをhrefに書く
<a
  href={program.directUrl}
  target="_blank"
  rel="noopener noreferrer"
  onClick={() => trackClick(program)}
>
  申し込む
</a>
```

---

## クリック計測はGA4のカスタムイベントで行う

プロキシAPIを使わない代わりに、GA4のカスタムイベントでクリックを記録する。

```typescript
// lib/analytics.ts
export function trackAffiliateClick(affiliateId: string, name: string) {
  if (typeof window === "undefined" || !window.gtag) return;
  window.gtag("event", "affiliate_click", {
    affiliate_id: affiliateId,
    affiliate_name: name,
  });
}
```

```tsx
// components/LeadGen/AffiliateCard.tsx
<a
  href={program.directUrl}
  target="_blank"
  rel="noopener noreferrer"
  onClick={() => trackAffiliateClick(program.id, program.name)}
>
  {program.name}に申し込む
</a>
```

GA4のイベントエクスプローラーで `affiliate_click` イベントを確認できる。コンバージョン設定すれば、どのアフィリエイトが収益につながっているか把握できる。

---

## アフィリエイトプログラムのデータ設計

PropAnalyticsでは `src/lib/constants/affiliates.ts` にプログラムを定義している。

```typescript
export interface AffiliateProgram {
  id:         string;
  name:       string;
  directUrl:  string;  // 広告主のLP直URL（Link Managerが変換する）
  reward:     string;  // 報酬テキスト（表示用）
  category:   "mortgage" | "realestate" | "insurance" | "tax";
  description: string;
}

export const MORTGAGE_AFFILIATES: AffiliateProgram[] = [
  {
    id:          "sbi-mortgage",
    name:        "SBI新生銀行 住宅ローン",
    directUrl:   "https://www.sbishinseibank.co.jp/retail/housing/",
    reward:      "1件 最大5,000円",
    category:    "mortgage",
    description: "変動金利0.29%〜。ネット完結で最短1週間審査",
  },
  // ...
];
```

---

## SupabaseでアフィリエイトURLを上書き管理する

Link ManagerはA8参加プログラムのみ対応。参加していないプログラムや、A8以外のネットワーク（バリューコマース等）のURLは手動で管理する。

```sql
-- affiliate_overrides テーブル
create table affiliate_overrides (
  id           text primary key, -- affiliateId
  affiliate_url text not null,   -- 手動で設定した計測URL
  updated_at   timestamptz default now()
);
```

```typescript
// API Route でフロントへ返す
export async function GET() {
  const { data: overrides } = await supabase
    .from("affiliate_overrides")
    .select("id, affiliate_url");

  // 定数にoverridesをマージ
  const merged = MORTGAGE_AFFILIATES.map((p) => {
    const override = overrides?.find((o) => o.id === p.id);
    return override ? { ...p, directUrl: override.affiliate_url } : p;
  });

  return NextResponse.json(merged);
}
```

管理画面からURLを更新すれば、再デプロイなしで即時反映される。

---

## まとめ

| ポイント | 内容 |
|---------|------|
| Link Managerの組み込み | `layout.tsx` に `afterInteractive` でscriptを2本追加 |
| 最大の落とし穴 | プロキシAPIを経由するとリンク変換されない |
| クリック計測 | GA4のカスタムイベントで代替 |
| URL管理 | 定数 + Supabase overridesでnon-A8プログラムも管理 |

PropAnalyticsのアフィリエイト設計の全体像は「[不動産投資データサービスをNext.js×Supabaseで作った全記録](https://zenn.dev/keisuke58/books/how-to-build-propanalytics)」で解説している。
