---
title: "何を作ったか・なぜ作ったか"
---

## PropAnalyticsとは

[PropAnalytics](https://real-estate-tracker-web-zeta.vercel.app) は、東京の大規模再開発プロジェクト82件を追跡する不動産投資データプラットフォームだ。

主な機能：
- 投資スコアランキング（7指標の総合評価）
- 坪単価・地価データの可視化
- AIレポート自動生成（Claude API）
- 利回り・IRR計算ツール
- ウォッチリスト・スコアアラート
- Stripe決済によるPROプラン

## なぜ作ったのか

不動産投資の情報は「定性的な記事」か「高額な専門データベース」の二択が多い。机上の空論ではなく、**データに基づいた意思決定**を個人投資家でも使えるレベルで提供したかった。

また個人的に：
- Next.js App Routerの実践的な使い方を習得したかった
- Supabase + Stripe + AI APIを組み合わせたSaaSの構成を試したかった
- SEOを意識したデータサイトの作り方を学びたかった

この本ではプロダクトの設計から収益化まで、設計判断の理由も含めてすべて書く。

## 技術スタック

| レイヤー | 採用技術 | 理由 |
|---------|---------|------|
| フロントエンド | Next.js 15 App Router | SSG/SSR/ISRを柔軟に使い分けられる |
| スタイリング | Tailwind CSS | 速い・一貫性が出る |
| バックエンド | Next.js API Routes | 別サーバー不要でシンプルに |
| DB・認証 | Supabase | PostgreSQL + Auth + RLSが揃っている |
| 決済 | Stripe | サブスクリプション管理が楽 |
| AI | Anthropic Claude API | 日本語品質が高い |
| 地図 | Google Maps API | 再開発地点を可視化 |
| メール | Resend | 開発者体験が良い |
| デプロイ | Vercel | Next.jsとの相性が最良 |
| プッシュ通知 | Web Push (web-push) | スコアアラートに使用 |

## ディレクトリ構成（抜粋）

```
src/
  app/
    (marketing)/      # LP・ガイド等の静的ページ
    api/              # API Routes
      cron/           # Vercel Cron Jobs
      admin/          # 管理者専用API
    projects/[id]/    # 物件詳細（SSG）
    tools/            # 利回り・IRR計算ツール
  components/
    Map/              # Google Maps統合
    LeadGen/          # アフィリエイト・CTAコンポーネント
  hooks/              # useAuth, useSubscription
  lib/
    constants/        # 物件データ・ハザードデータ
    supabase/         # クライアント・サーバー分離
supabase/migrations/  # SQLマイグレーション（001〜008）
```

次のチャプターから実装の詳細に入っていく。
