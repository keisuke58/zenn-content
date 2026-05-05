---
title: "何を作ったか・技術選定の理由・最終的な構成"
---

## PropAnalyticsとは

[PropAnalytics](https://real-estate-tracker-web-zeta.vercel.app) は、東京の大規模再開発プロジェクト82件を追跡する不動産投資データプラットフォームだ。

主な機能：

| 機能 | 概要 |
| ---- | ---- |
| 投資スコアランキング | 7指標の総合評価で82件を順位付け |
| 坪単価・地価データ | エリア別の価格水準を可視化 |
| AIレポート自動生成 | Claude APIによる投資分析サマリー |
| 利回り・IRR計算ツール | 購入価格・家賃から収益性を即算出 |
| ウォッチリスト | 気になる物件を保存してアラート受信 |
| Stripe決済 | starter/proプランのサブスクリプション |

月額課金のSaaSとして動いており、アフィリエイト（住宅ローン・不動産会社）も組み込んでいる。

---

## なぜ作ったのか

不動産投資の情報収集には構造的な問題がある。

- **ポータルサイト**（SUUMO・at home）→ 物件の表面情報のみ。投資判断に必要なハザードリスクや地価トレンドがない
- **専門データベース**（東洋経済・不動産経済研究所）→ 年間数十万円。個人投資家には高すぎる
- **不動産会社の資料** → バイアスがかかっている

「データに基づいた意思決定を、個人投資家が使えるコストで提供する」という明確なギャップを埋めるサービスを作ることにした。

技術的な動機としては以下の3点：

- Next.js 15 App Routerの実践的な使い方を習得したかった
- Supabase + Stripe + AI APIを組み合わせたSaaSの構成を試したかった
- SEOを意識したデータサイトの収益化パターンを学びたかった

---

## 技術スタック選定の理由

### Next.js 15（App Router）

Pages Routerから移行した理由は3つある。

**1. Server Componentによるパフォーマンス**
データフェッチをサーバー側で完結させられる。ユーザーのブラウザにJSバンドルを送らない。

**2. `generateStaticParams`**
82件の物件ページを完全にSSGできる。DBアクセスなしでエッジ配信されるため、初回ロードが極めて速い。

**3. Metadata API**
`generateMetadata` で動的なOGP・titleタグを型安全に生成できる。SEOの要件を型で担保できるのは大きなメリットだ。

Pages Routerの `getStaticPaths` + `getStaticProps` と比べると、App Routerは関連する処理が1ファイルに集まって可読性が上がった。

### Supabase

Firebase、PlanetScale、Neonと比較して選んだ決め手は **Row Level Security（RLS）**。

| サービス | 認証 | RLS | リアルタイム | 無料枠 |
| -------- | ---- | --- | ------------ | ------ |
| Firebase | ✓ | ✕ (Firestore独自) | ✓ | 小さめ |
| PlanetScale | ✕ | ✕ | ✕ | 中 |
| Supabase | ✓ | ✓ (PostgreSQL) | ✓ | 大きい |
| Neon | ✕ | ✕ | ✕ | 中 |

RLSを使うと「このユーザーは自分のウォッチリストしか見られない」という制御をDB層で完結させられる。APIルートにアクセス制御ロジックを書く量が激減した。

### Stripe

サブスクリプションAPIの実績と、Webhookの信頼性でStripeを選択。一度実装すれば、アップグレード・ダウングレード・解約・請求失敗時のリトライをすべて自動で処理してくれる。

### Anthropic Claude API

OpenAI GPT-4と比較した。不動産投資レポートを生成させると、Claudeの方が論理構造が明確で、数字に基づいた主張をする傾向があった。`claude-sonnet-4-6` はGPT-4oと同等コストで、日本語の品質が高い。

---

## 重要な設計判断：物件データをどこに置くか

最初はSupabaseに物件データを入れようとしたが、**SSGとの相性が悪い**ことに気付いて静的定数に変更した。

問題点：

- SSGビルド時にDB接続が必要 → Supabaseの接続プールを圧迫する可能性
- DBが変わるたびに再デプロイが必要で、結局静的と変わらない
- TypeScriptの型チェックが効かない

静的定数のメリット：

- `generateStaticParams` でビルド時に全ページ生成、エッジで配信
- TypeScriptの型でデータ不整合を防げる
- Gitでデータ変更が追跡できる、PRでレビューできる

**判断基準：更新頻度が「月1〜2回」程度なら静的定数が正解。** ユーザーごとの状態（ウォッチリスト・課金状態）だけSupabaseに置く設計にした。

---

## ディレクトリ構成

```text
src/
  app/
    api/
      cron/          # Vercel Cron（newsletter, score-alert, tweet）
      admin/         # 管理者専用API
      stripe/        # Webhook・Checkout
      email/         # ウェルカムメール等
    projects/[id]/   # 物件詳細 SSG（124ページ）
    area/[slug]/     # エリア別ガイド SSG（61ページ）
    guide/           # 投資ガイド記事
    tools/           # 利回り・IRR計算ツール
  components/
    Map/             # Google Maps統合（"use client"必須）
    LeadGen/         # アフィリエイトCTA
    Compare/         # 比較ツール
  hooks/
    useAuth.ts
    useSubscription.ts
  lib/
    constants/
      redevelopment.ts  # 物件データ本体（約3500行）
      hazard.ts
      affiliates.ts
    supabase/
      client.ts      # ブラウザ用
      server.ts      # Server Component / API Route用
    auth.ts          # getUserPlan / ensurePremiumRecord
supabase/
  migrations/        # 001〜008のSQLファイル
```

次チャプターからは実装の詳細に入る。
