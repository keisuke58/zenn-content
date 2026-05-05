---
title: "Supabase × Next.jsで7日間トライアル付きSaaSを個人開発する"
emoji: "🚀"
type: "tech"
topics: ["supabase", "nextjs", "stripe", "saas", "個人開発"]
published: true
---

## 概要

PropAnalyticsで実装した「**新規登録 → 7日間PROトライアル自動付与 → Stripeで課金**」の仕組みを解説する。

全体のフロー：

```
ユーザー登録（メール / Google OAuth）
    ↓
/api/auth/callback
    ↓
ensurePremiumRecord() → 新規ならtrial_until = now() + 7日
    ↓
ウェルカムメール送信（Resend）
    ↓
trial_until 期間中はPRO機能が使える
    ↓
7日後 → 無料プランに戻る
    ↓
Stripeで決済 → webhookでplan更新
```

---

## Supabaseのテーブル設計

```sql
create table public.premium_members (
  email        text primary key,
  plan         text default 'free',   -- 'free' | 'starter' | 'pro'
  valid_until  timestamptz,           -- 有料プランの有効期限
  trial_until  timestamptz,           -- トライアル期限
  created_at   timestamptz default now()
);

-- RLS：本人のみ閲覧可、更新はservice_roleのみ
alter table premium_members enable row level security;
create policy "users can read own record"
  on premium_members for select
  using (auth.email() = email);
```

`email` をPKにした理由：Stripeのwebhookにはemail情報が含まれており、`auth.users` のUUIDを使うよりemailでマッチングする方がシンプルだ。

---

## 新規ユーザーにトライアルを付与する

Supabaseのメール認証はコールバックURLを経由する。ここでトライアルを設定する。

```typescript
// app/api/auth/callback/route.ts
import { NextResponse } from "next/server";
import { getSupabaseServerClient } from "@/lib/supabase/server";
import { ensurePremiumRecord } from "@/lib/auth";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");

  if (code) {
    const supabase = await getSupabaseServerClient();
    const { data, error } = await supabase.auth.exchangeCodeForSession(code);

    if (!error && data.user?.email) {
      const isNew = await ensurePremiumRecord(data.user.email);
      if (isNew) {
        // 非同期でウェルカムメール（リダイレクトをブロックしない）
        fetch(`${origin}/api/email/welcome`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ email: data.user.email }),
        }).catch(() => {});
      }
    }
  }

  return NextResponse.redirect(`${origin}/`);
}
```

```typescript
// lib/auth.ts
export async function ensurePremiumRecord(email: string): Promise<boolean> {
  const admin = await getSupabaseAdminClient();

  // 既存ユーザーかチェック
  const { data: existing } = await admin
    .from("premium_members")
    .select("email")
    .eq("email", email)
    .single();

  if (existing) return false; // 既存ユーザーは何もしない

  // 新規：7日間トライアル付きでinsert
  const trialUntil = new Date(
    Date.now() + 7 * 24 * 60 * 60 * 1000
  ).toISOString();

  await admin.from("premium_members").insert({
    email,
    plan: "free",
    trial_until: trialUntil,
  });

  return true; // 新規ユーザー
}
```

---

## useSubscription：課金状態をフックで管理する

```typescript
// hooks/useSubscription.ts
"use client";

export function useSubscription() {
  const { user } = useAuth();
  const [plan, setPlan]       = useState<Plan>("free");
  const [isTrial, setIsTrial] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!user?.email) { setLoading(false); return; }

    supabase
      .from("premium_members")
      .select("plan, valid_until, trial_until")
      .eq("email", user.email)
      .single()
      .then(({ data }) => {
        if (!data) { setLoading(false); return; }

        const paidActive =
          (data.plan === "pro" || data.plan === "starter") &&
          (!data.valid_until || new Date(data.valid_until) > new Date());

        const trialActive =
          !paidActive &&
          data.trial_until &&
          new Date(data.trial_until) > new Date();

        if (paidActive)       { setPlan(data.plan); setIsTrial(false); }
        else if (trialActive) { setPlan("starter");  setIsTrial(true); }
        else                  { setPlan("free");     setIsTrial(false); }

        setLoading(false);
      });
  }, [user]);

  return {
    plan,
    isPro: plan !== "free",
    isTrial,  // トライアル中かどうかをUIで表示できる
    loading,
  };
}
```

`isTrial` フラグを持つことで「トライアル中」バッジをUIに表示したり、期限をカウントダウン表示するなど、UX上の工夫ができる。

---

## Stripe Webhookでプラン更新する

```typescript
// app/api/stripe/webhook/route.ts
export async function POST(req: NextRequest) {
  const body = await req.text();
  const sig  = req.headers.get("stripe-signature")!;
  const event = stripe.webhooks.constructEvent(
    body, sig, process.env.STRIPE_WEBHOOK_SECRET!
  );

  switch (event.type) {
    case "customer.subscription.updated":
    case "customer.subscription.created": {
      const sub   = event.data.object as Stripe.Subscription;
      const email = sub.metadata.email;
      const plan  = sub.metadata.plan; // checkout時にmetadataにセット

      await admin.from("premium_members").upsert({
        email,
        plan,
        valid_until: new Date(sub.current_period_end * 1000).toISOString(),
      }, { onConflict: "email" });
      break;
    }

    case "customer.subscription.deleted": {
      const sub   = event.data.object as Stripe.Subscription;
      const email = sub.metadata.email;
      await admin.from("premium_members")
        .update({ plan: "free", valid_until: null })
        .eq("email", email);
      break;
    }
  }

  return NextResponse.json({ received: true });
}
```

Checkoutセッション作成時に `metadata: { email, plan }` を入れておくのがポイント。

---

## 環境変数まとめ

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # admin操作用

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Resend（メール）
RESEND_API_KEY=re_...
```

---

## まとめ

- `auth.callback` でのコールバックがトライアル付与の唯一のフック
- emailをPKにするとStripe webhookとのマッチングがシンプル
- `isTrial` フラグをhookに持つとUI表現が豊かになる
- Stripe webhookでのDB更新は `upsert + onConflict: "email"` が安全

詳細な実装は「[不動産投資データサービスをNext.js×Supabaseで作った全記録](https://zenn.dev/keisuke58/books/how-to-build-propanalytics)」を参照。
