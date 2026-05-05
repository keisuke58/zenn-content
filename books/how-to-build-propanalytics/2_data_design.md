---
title: "データ設計：82件の物件データをどう持つか"
---

## 静的データ vs DB：設計の判断

最初に直面した問題は「物件データをどこに置くか」だ。

候補は2つ：
1. **TypeScriptの定数ファイル** (`src/lib/constants/redevelopment.ts`)
2. **Supabase（PostgreSQL）**

PropAnalyticsでは **定数ファイルを選んだ**。理由：

| 観点 | 定数ファイル | Supabase |
|------|-----------|---------|
| 更新頻度 | 月1〜2回 | — |
| SEO（SSG） | ◎ ビルド時に全ページ生成 | △ 毎回API呼び出しが必要 |
| ページロード速度 | ◎ データフェッチなし | △ DBレイテンシあり |
| 管理のしやすさ | ◎ GitでPR管理 | △ 管理画面が必要 |
| リアルタイム更新 | ✕ 要再デプロイ | ◎ 即時反映 |

更新頻度が低くSEOが重要なデータは **静的が正解**。

## RedevelopmentProjectの型定義

```typescript
export interface RedevelopmentProject {
  id: string;
  projectName: string;
  lat: number;
  lng: number;
  completionYear: number;
  status: "completed" | "under_construction" | "planned" | "historical";
  description: string;
  developer: string;
  investmentScore: number;       // 0-100
  nearestStation?: string;
  pricePerM2?: { min: number; max: number };
  impactRadius: number;          // 周辺影響半径(m)
  listings?: Listing[];
  category: "large_scale" | "mid_scale" | "small_scale" | "historical";
  cityCode: string;              // 市区町村コード
}
```

## 投資スコアの計算ロジック

スコアは事前計算して定数に埋め込んでいる（リアルタイム計算しない）。

7指標の重み付け：

```
投資スコア = 
  立地スコア       × 0.25 +
  デベロッパー信頼度 × 0.20 +
  地価トレンド      × 0.20 +
  プロジェクト規模   × 0.15 +
  完成フェーズ      × 0.10 +
  ハザードリスク    × 0.05 +
  市場流動性       × 0.05
```

立地・デベロッパー・地価トレンドの3指標で65%を占める。これは「不動産投資の本質は立地とデベロッパーの信頼性」という考えを反映している。

## Supabaseで管理するデータ

一方でリアルタイム性が必要なデータはSupabaseに置く：

```sql
-- ユーザー管理
premium_members (email, plan, valid_until, trial_until)

-- ウォッチリスト
watchlist (user_id, project_id, created_at)

-- スコアアラート設定
score_alerts (user_id, project_id, threshold)

-- アフィリエイトURL管理
affiliate_overrides (id, affiliate_url, updated_at)

-- プッシュ通知購読
push_subscriptions (user_id, endpoint, p256dh, auth)
```

ユーザーごとの状態管理だけDBに任せ、コアデータは静的という分離が重要だった。
