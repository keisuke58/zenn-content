---
title: "Vercel Cron × ResendでニュースレターとアラートメールをNext.jsだけで完結させる"
emoji: "📨"
type: "tech"
topics: ["nextjs", "vercel", "resend", "個人開発", "saas"]
published: false
---

## Vercel Cronとは

Vercel Cronは `vercel.json` にスケジュールを書くだけで、Next.jsのAPI Routeを定期実行できる機能だ。別途サーバーやCloud Functionsは不要。

```json
// vercel.json
{
  "crons": [
    { "path": "/api/cron/newsletter",  "schedule": "0 0 * * 0" },
    { "path": "/api/cron/score-alert", "schedule": "0 0 * * 2" },
    { "path": "/api/cron/tweet",       "schedule": "0 1 * * *" }
  ]
}
```

PropAnalyticsでは3本のCronを動かしている：

| Cron | スケジュール | 処理 |
|------|------------|------|
| newsletter | 毎週日曜 09:00 JST | ウォッチリストユーザーへ週次レポート |
| score-alert | 毎週火曜 09:00 JST | スコア変動アラートメール |
| tweet | 毎日 10:00 JST | X（Twitter）自動投稿 |

---

## セキュリティ：Cron APIは必ず認証する

Cron APIはURLに直接アクセスされると誰でも実行できてしまう。必ずBearerトークンで保護する。

```typescript
// app/api/cron/newsletter/route.ts
export const dynamic = "force-dynamic";
import { NextRequest, NextResponse } from "next/server";

export async function GET(req: NextRequest) {
  // Vercel CronはAuthorizationヘッダーを自動付与する
  if (
    req.headers.get("authorization") !==
    `Bearer ${process.env.CRON_SECRET}`
  ) {
    return NextResponse.json({ error: "unauthorized" }, { status: 401 });
  }

  // 処理...
  return NextResponse.json({ sent: count });
}
```

```bash
# .env.local
CRON_SECRET=ランダムな文字列（openssl rand -hex 32 で生成）
```

Vercel Cronは `vercel.json` に `CRON_SECRET` 環境変数が設定されていれば自動的に `Authorization: Bearer {CRON_SECRET}` ヘッダーを付けてリクエストしてくれる。

---

## Resendでメール送信する

Resendは開発者向けのメール送信サービス。無料プランで月3,000通まで送れる。

```bash
npm install resend
```

```typescript
// 初期化はリクエスト時にする（モジュールロード時はNG）
export async function GET(req: NextRequest) {
  if (!process.env.RESEND_API_KEY) return NextResponse.json({ skipped: true });

  // ✅ リクエスト時にインスタンス化（ビルドエラーを防ぐ）
  const { Resend } = await import("resend");
  const resend = new Resend(process.env.RESEND_API_KEY);

  const { error } = await resend.emails.send({
    from: "PropAnalytics <noreply@propanalytics.jp>",
    to: "user@example.com",
    subject: "今週の東京再開発レポート",
    html: buildNewsletterHtml(...),
  });
}
```

:::message alert
`const resend = new Resend(process.env.RESEND_API_KEY)` をモジュールのトップレベルに書くとビルド時にクラッシュする。env変数が未設定の状態でモジュールが評価されるため。必ずリクエストハンドラの中で初期化すること。
:::

---

## ニュースレターの実装パターン

SupabaseからウォッチリストユーザーのリストをService Roleで取得し、Resendで一斉送信する。

```typescript
function getAdmin() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );
}

export async function GET(req: NextRequest) {
  // 認証...
  const supabase = getAdmin();

  // ウォッチリスト保有ユーザーを取得
  const { data: users } = await supabase
    .from("watchlist")
    .select("user_id, profiles!inner(email)")
    .limit(500);

  const uniqueEmails = [...new Set(users?.map((u) => u.profiles.email))];

  let sent = 0;
  for (const email of uniqueEmails) {
    const { error } = await resend.emails.send({
      from: "PropAnalytics <noreply@propanalytics.jp>",
      to: email,
      subject: "今週の東京再開発レポート｜スコアTOP5",
      html: buildNewsletterHtml(email, top5Projects, isPro),
    });
    if (!error) sent++;
  }

  return NextResponse.json({ sent });
}
```

---

## 無料ユーザーへのアップグレード誘導

ニュースレターにPROアップグレードのCTAを組み込むと課金コンバージョンが上がる。

```typescript
function buildNewsletterHtml(email: string, projects: Project[], isPro: boolean) {
  return `
    ...
    ${projects.slice(0, 3).map(renderProjectCard).join("")}

    ${!isPro ? `
    <!-- 4位・5位はぼかして表示 -->
    <div style="filter:blur(4px);pointer-events:none">
      ${projects.slice(3, 5).map(renderProjectCard).join("")}
    </div>
    <div style="text-align:center;margin:20px 0">
      <a href="https://example.com/pricing"
         style="background:#f43f5e;color:#fff;padding:12px 24px;border-radius:999px">
        PROで全件見る（¥2,980/月〜）
      </a>
    </div>
    ` : ""}
    ...
  `;
}
```

「ぼかし表示 + CTA」はSaaSのフリーミアム戦略で効果的なパターンだ。

---

## ローカルでCronをテストする

Vercel Cronはローカルでは動かないので、curlで直接叩いてテストする。

```bash
# .env.local の CRON_SECRET を使う
curl -X GET http://localhost:3000/api/cron/newsletter \
  -H "Authorization: Bearer your_cron_secret"
```

---

## まとめ

- `vercel.json` にスケジュールを書くだけでCronが動く
- Cron APIは必ずBearerトークンで保護する
- Resendの初期化はモジュールトップではなくリクエスト時に行う
- ニュースレターにぼかし+CTAを入れると課金誘導になる

この実装の全体像は「[不動産投資データサービスをNext.js×Supabaseで作った全記録](https://zenn.dev/keisuke58/books/how-to-build-propanalytics)」で解説している。
