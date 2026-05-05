---
title: "データ設計：82件の物件データをどう持つか"
---

## 最初の判断：TypeScript定数 vs Supabase

PropAnalyticsを作るにあたって最初に決めたことがある。「物件データをどこに置くか」だ。

結論から言う。**TypeScriptの定数ファイルに置いた。Supabaseには置かなかった。**

判断の根拠は更新頻度だ。東京の再開発プロジェクトは一度発表されると、データが変わるのは「竣工」「計画変更」のタイミングだけで、頻度は月1〜2回程度。これはブログ記事の投稿頻度より低い。

| 観点 | TypeScript定数 | Supabase |
|------|-------------|---------|
| SSGビルド | DB接続なし・エッジ配信 | ビルド時にDB呼び出しが必要 |
| 型安全性 | TypeScriptで完全に保証 | 実行時まで不明 |
| データ変更管理 | GitのPRでレビュー可能 | 管理画面 or APIが必要 |
| SEOページ生成速度 | 静的生成・CDN配信 | SSRかISRが必要 |
| 更新反映 | デプロイ（〜1分） | 即時 |

「リアルタイム更新が不要で、SEOが重要なデータは静的が正解」という判断だった。

## 型定義：データ設計の核心

型定義から始めた。型を先に作ることで、82件のデータを入力していく過程でのミスが型チェックで検出できる。

```typescript
// src/lib/constants/types.ts

export type ProjectStatus = 
  | "completed"         // 竣工済み
  | "under_construction" // 建設中
  | "planned"           // 計画中
  | "historical";       // 歴史的建築

export type ProjectCategory = 
  | "large_scale"   // 大規模（5ha以上・1000億円以上）
  | "mid_scale"     // 中規模
  | "small_scale"   // 小規模
  | "historical";

export interface PriceRange {
  min: number;  // 万円/坪
  max: number;
}

export interface HazardInfo {
  floodRisk: 0 | 1 | 2 | 3;    // 0=低, 3=高
  liquefactionRisk: 0 | 1 | 2 | 3;
  earthquakeZone: "1" | "2" | "3";
}

export interface RedevelopmentProject {
  id: string;                    // URL slug ("torch-tower")
  projectName: string;           // 表示名
  lat: number;
  lng: number;
  completionYear: number;
  status: ProjectStatus;
  description: string;
  developer: string;
  investmentScore: number;       // 0-100（7指標の加重平均）
  nearestStation?: string;
  walkMinutes?: number;          // 最寄り駅徒歩分数
  pricePerTsubo?: PriceRange;    // 坪単価（万円）
  impactRadius: number;          // 周辺影響半径（m）
  totalCost?: number;            // 総事業費（億円）
  floorArea?: number;            // 延床面積（㎡）
  cityCode: string;              // 市区町村コード（"13103" = 港区）
  ward: string;                  // 区名（"港区"）
  hazard?: HazardInfo;
}
```

**`id` の設計が後のSEOを左右した。** 英語のケバブケース（"torch-tower"）にすることで、URLが `/projects/torch-tower` と読みやすくなり、構造化データとの対応も明確になる。日本語IDは文字化けリスクがあるのでNG。

## 投資スコアの計算ロジック

スコアは**リアルタイムで計算せず、事前に計算して定数に埋め込む**。82件×7指標の評価を毎リクエストで実行するのは過剰だし、SSGと相性が悪い。

重み付けの設計：

```typescript
// スコア計算（ビルド前の事前計算スクリプト）
const WEIGHTS = {
  location: 0.25,        // 立地：25%
  developer: 0.20,       // デベロッパー信頼度：20%
  landPriceTrend: 0.20,  // 地価トレンド：20%
  projectScale: 0.15,    // プロジェクト規模：15%
  completionPhase: 0.10, // 完成フェーズ：10%
  hazardRisk: 0.05,      // ハザードリスク：5%
  marketLiquidity: 0.05, // 市場流動性：5%
} as const;

function calcInvestmentScore(project: RawProjectData): number {
  const scores = {
    location: calcLocationScore(project),
    developer: DEVELOPER_SCORES[project.developer] ?? 50,
    landPriceTrend: calcLandPriceTrend(project.cityCode),
    projectScale: calcScaleScore(project.totalCost, project.floorArea),
    completionPhase: STATUS_SCORES[project.status],
    hazardRisk: calcHazardScore(project.hazard),
    marketLiquidity: calcLiquidityScore(project.ward),
  };

  return Math.round(
    Object.entries(WEIGHTS).reduce(
      (sum, [key, weight]) => sum + scores[key as keyof typeof scores] * weight,
      0
    )
  );
}
```

**立地・デベロッパー・地価トレンドの3指標で65%を占める。** 不動産投資の本質は「立地」と「誰が作るか」に尽きるという判断だ。

## Supabaseで管理するデータ：ユーザー状態のみ

物件データは静的にした一方、**ユーザーごとの状態変化するデータはすべてSupabaseに置く**。

```sql
-- 課金プラン（Stripeと同期）
create table premium_members (
  email        text primary key,
  plan         text default 'free',   -- 'free' | 'starter' | 'pro'
  valid_until  timestamptz,
  trial_until  timestamptz,
  created_at   timestamptz default now()
);

-- ウォッチリスト（RLSで自分のデータのみ参照可能）
create table watchlist (
  id         uuid default gen_random_uuid() primary key,
  user_id    uuid references auth.users not null,
  project_id text not null,           -- "torch-tower" など
  created_at timestamptz default now(),
  unique(user_id, project_id)
);

-- ウォッチリストのRLS
alter table watchlist enable row level security;
create policy "users can manage own watchlist"
  on watchlist for all
  using (auth.uid() = user_id);
```

RLSを設定すると、`user_id` のフィルタリングをAPI側に書く必要がなくなる。Supabaseクライアントが自動的にセッションのユーザーIDで絞り込む。

```typescript
// APIRoute不要。クライアントから直接呼ぶだけでOK
const { data } = await supabase
  .from("watchlist")
  .select("project_id");
// → 自動的に自分のwatchlistのみ返ってくる（他ユーザーのは見えない）
```

## データの境界線：判断基準のまとめ

| データの種類 | 格納先 | 判断理由 |
|------------|--------|--------|
| 物件基本情報（名前・座標・スコア） | TypeScript定数 | 更新頻度低、SEOに直結 |
| ユーザーウォッチリスト | Supabase | ユーザーごとに異なる |
| 課金プラン・有効期限 | Supabase | Stripeと同期が必要 |
| AIレポート生成結果 | 生成のたびにAPI呼び出し | キャッシュ不要（毎回新鮮） |
| ハザードリスクデータ | TypeScript定数 | 国交省データを事前加工 |
| アフィリエイトURL | Supabase | 随時URLを差し替えたい |

この境界線を最初に明確にしたことで、「どこに書けばいい？」という迷いが消えた。
