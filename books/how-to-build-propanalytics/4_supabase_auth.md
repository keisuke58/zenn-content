---
title: "Supabase Auth + RLS：認証と課金状態の管理"
---

## クライアント分離：最初に理解すべき設計

Next.js App RouterでSupabaseを使うとき、**クライアントはサーバー用とブラウザ用を分ける必要がある**。同じ `createClient` を使い回すと認証状態がサーバー側に漏洩するリスクがある。

```typescript
// src/lib/supabase/client.ts（ブラウザ用）
import { createBrowserClient } from "@supabase/ssr";

export function getSupabaseClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

```typescript
// src/lib/supabase/server.ts（Server Component / API Route用）
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function getSupabaseServerClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (newCookies) => {
          newCookies.forEach(({ name, value, options }) => {
            cookieStore.set(name, value, options);
          });
        },
      },
    }
  );
}
```

```typescript
// src/lib/supabase/admin.ts（RLS無視が必要な管理操作用）
import { createClient } from "@supabase/supabase-js";

// SUPABASE_SERVICE_ROLE_KEY はサーバー専用。絶対にクライアントに渡さない
export const adminSupabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);
```

AdminクライアントはRLSをバイパスするため、StripeのWebhookやCron処理など「ユーザーの代わりではなく、システムとして操作する」ときのみ使う。

## premium_membersテーブルの設計

課金状態管理の核心はこのテーブルだ：

```sql
create table premium_members (
  email        text primary key,
  plan         text not null default 'free',  -- 'free' | 'starter' | 'pro'
  valid_until  timestamptz,                    -- 有料プランの有効期限
  trial_until  timestamptz,                    -- 7日間トライアル期限
  created_at   timestamptz default now()
);

-- anon/authenticated ユーザーは自分のレコードのみ参照可能
alter table premium_members enable row level security;
create policy "users can read own plan"
  on premium_members for select
  using (email = auth.jwt() ->> 'email');
```

**PKをUUIDでなくemailにした理由**が重要だ。StripeのWebhookに含まれるのはメールアドレスだけで、Supabaseの `auth.users.id` は入っていない。emailをPKにすることで、Webhookから直接 `upsert` できる。

## 7日間トライアルの実装フロー

新規ユーザーへのトライアル付与は、OAuth認証後のコールバックで行う：

```typescript
// src/app/api/auth/callback/route.ts
export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  const next = searchParams.get("next") ?? "/";

  if (code) {
    const supabase = await getSupabaseServerClient();
    const { data, error } = await supabase.auth.exchangeCodeForSession(code);

    if (!error && data.user?.email) {
      const isNew = await ensurePremiumRecord(data.user.email);

      if (isNew) {
        // fire-and-forgetでウェルカムメール送信（失敗しても認証には影響しない）
        fetch(`${origin}/api/email/welcome`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ email: data.user.email }),
        }).catch(() => {});
      }
    }
  }
  return NextResponse.redirect(`${origin}${next}`);
}
```

```typescript
// src/lib/auth.ts
export async function ensurePremiumRecord(email: string): Promise<boolean> {
  // 既存ユーザーチェック
  const { data: existing } = await adminSupabase
    .from("premium_members")
    .select("email")
    .eq("email", email)
    .single();

  if (existing) return false;

  // 新規ユーザー：7日間トライアルを付与
  const trialUntil = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString();
  await adminSupabase.from("premium_members").insert({
    email,
    plan: "free",
    trial_until: trialUntil,
  });
  return true;
}
```

## useSubscription Hook：プランとトライアルを一元管理

```typescript
// src/hooks/useSubscription.ts
"use client";

export type Plan = "free" | "starter" | "pro";

export function useSubscription() {
  const { user } = useAuth();
  const [plan, setPlan] = useState<Plan>("free");
  const [isTrial, setIsTrial] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!user?.email) return;

    getSupabaseClient()
      .from("premium_members")
      .select("plan, valid_until, trial_until")
      .eq("email", user.email)
      .single()
      .then(({ data }) => {
        if (!data) { setLoading(false); return; }

        const now = new Date();
        const paidActive =
          (data.plan === "pro" || data.plan === "starter") &&
          (!data.valid_until || new Date(data.valid_until) > now);

        const trialActive =
          !paidActive &&
          data.trial_until != null &&
          new Date(data.trial_until) > now;

        if (paidActive)       { setPlan(data.plan); setIsTrial(false); }
        else if (trialActive) { setPlan("starter");  setIsTrial(true); }
        else                  { setPlan("free");      setIsTrial(false); }

        setLoading(false);
      });
  }, [user?.email]);

  return { plan, isPro: plan !== "free", isTrial, loading };
}
```

## StripeのWebhookでプランを同期

Checkoutセッション作成時に `metadata` にemailとplanを埋め込み、Webhookで受け取ってDB更新する：

```typescript
// Checkoutセッション作成（app/api/stripe/checkout/route.ts）
const session = await stripe.checkout.sessions.create({
  mode: "subscription",
  line_items: [{ price: PRICE_IDS[plan], quantity: 1 }],
  metadata: { email, plan },                // ← ここが重要
  success_url: `${origin}/dashboard?upgraded=true`,
  cancel_url: `${origin}/pricing`,
});
```

```typescript
// Webhook処理（app/api/stripe/webhook/route.ts）
const event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);

switch (event.type) {
  case "customer.subscription.updated":
  case "customer.subscription.created": {
    const sub = event.data.object as Stripe.Subscription;
    const { email, plan } = sub.metadata;

    await adminSupabase.from("premium_members").upsert(
      {
        email,
        plan,
        valid_until: new Date(sub.current_period_end * 1000).toISOString(),
      },
      { onConflict: "email" }
    );
    break;
  }
  case "customer.subscription.deleted": {
    const sub = event.data.object as Stripe.Subscription;
    await adminSupabase
      .from("premium_members")
      .update({ plan: "free", valid_until: null })
      .eq("email", sub.metadata.email);
    break;
  }
}
```

`upsert` の `onConflict: "email"` で、新規登録と既存ユーザーのプラン変更を同じコードで処理できる。

## getUserPlan：Server ComponentでのプランチェックAPI

ページのアクセス制御は Server Component 側で行う：

```typescript
// src/lib/auth.ts
export async function getUserPlan(email: string): Promise<Plan> {
  const { data } = await adminSupabase
    .from("premium_members")
    .select("plan, valid_until, trial_until")
    .eq("email", email)
    .single();

  if (!data) return "free";

  const now = new Date();

  const paidActive =
    (data.plan === "pro" || data.plan === "starter") &&
    (!data.valid_until || new Date(data.valid_until) > now);

  const trialActive =
    !paidActive &&
    data.trial_until != null &&
    new Date(data.trial_until) > now;

  if (paidActive)   return data.plan as Plan;
  if (trialActive)  return "starter";
  return "free";
}
```

```typescript
// PRO限定ページでの使用例
export default async function ProOnlyPage() {
  const supabase = await getSupabaseServerClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user?.email) redirect("/login");

  const plan = await getUserPlan(user.email);
  if (plan === "free") redirect("/pricing");

  return <ProContent />;
}
```
