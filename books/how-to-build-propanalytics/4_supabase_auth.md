---
title: "Supabase Auth + RLS：認証と課金状態の管理"
---

## Supabaseクライアントをサーバー・クライアントで分離する

Next.js App Routerでは、Server ComponentとClient Componentで使うSupabaseクライアントを分ける必要がある。

```typescript
// lib/supabase/client.ts（"use client" 環境用）
import { createBrowserClient } from "@supabase/ssr";
export function getSupabaseClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}

// lib/supabase/server.ts（Server Component / API Route用）
import { createServerClient } from "@supabase/ssr";
export async function getSupabaseServerClient() {
  const cookieStore = await cookies();
  return createServerClient(url, key, { cookies: ... });
}
```

## premium_membersテーブルの設計

課金状態は `premium_members` テーブルで管理する：

```sql
create table premium_members (
  email        text primary key,
  plan         text default 'free',  -- 'free' | 'starter' | 'pro'
  valid_until  timestamptz,          -- サブスク有効期限
  trial_until  timestamptz,          -- 7日間トライアル期限
  created_at   timestamptz default now()
);
```

ポイントは **emailをPKにしたこと**。Supabaseの `auth.users` のUUIDを使わず、メールアドレスで紐付けることでStripeのWebhookと直接マッチングできる。

## useSubscription Hook

```typescript
export function useSubscription() {
  const { user } = useAuth();
  const [plan, setPlan] = useState<Plan>("free");
  const [isTrial, setIsTrial] = useState(false);

  useEffect(() => {
    supabase.from("premium_members")
      .select("plan, valid_until, trial_until")
      .eq("email", user.email)
      .single()
      .then(({ data }) => {
        const paidActive = (data.plan === "pro" || data.plan === "starter")
          && (!data.valid_until || new Date(data.valid_until) > new Date());

        const trialActive = !paidActive
          && data.trial_until
          && new Date(data.trial_until) > new Date();

        if (paidActive)       { setPlan(data.plan); setIsTrial(false); }
        else if (trialActive) { setPlan("starter");  setIsTrial(true); }
        else                  { setPlan("free");     setIsTrial(false); }
      });
  }, [user]);

  return { plan, isPro: plan !== "free", isTrial };
}
```

## 7日間トライアルの実装

新規ユーザーには自動でトライアルを付与する。発火タイミングは **メール認証後のコールバック**：

```typescript
// app/api/auth/callback/route.ts
export async function GET(request: Request) {
  const { data, error } = await supabase.auth.exchangeCodeForSession(code);
  if (!error && data.user?.email) {
    const isNew = await ensurePremiumRecord(data.user.email);
    if (isNew) {
      // 非同期でウェルカムメール送信
      fetch(`${origin}/api/email/welcome`, {
        method: "POST",
        body: JSON.stringify({ email: data.user.email }),
      }).catch(() => {});
    }
  }
  return NextResponse.redirect(`${origin}${next}`);
}
```

```typescript
// lib/auth.ts
export async function ensurePremiumRecord(email: string): Promise<boolean> {
  const { data: existing } = await admin
    .from("premium_members").select("email").eq("email", email).single();

  if (existing) return false; // 既存ユーザー

  // 新規ユーザー：7日間トライアル付きで挿入
  const trialUntil = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString();
  await admin.from("premium_members").insert({
    email, plan: "free", trial_until: trialUntil,
  });
  return true;
}
```

## StripeのWebhookでplan更新

```typescript
// app/api/stripe/webhook/route.ts
case "customer.subscription.updated": {
  const sub = event.data.object as Stripe.Subscription;
  const email = sub.metadata.email; // Checkout時にmetadataに入れておく
  const plan = sub.metadata.plan;   // "starter" | "pro"
  await admin.from("premium_members").upsert({
    email,
    plan,
    valid_until: new Date(sub.current_period_end * 1000).toISOString(),
  }, { onConflict: "email" });
  break;
}
```
