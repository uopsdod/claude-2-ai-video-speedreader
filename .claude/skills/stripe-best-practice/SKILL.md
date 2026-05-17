---
name: stripe-best-practice
description: Hard rules and operational SOP for integrating Stripe Checkout + webhooks in this course (Course 2 M2 — credits system). Use whenever a student is wiring `/api/credits/checkout`, the `/api/stripe/webhook` handler, debugging "payment succeeded but credits didn't land", switching sandbox→live, or hitting signature-verification / middleware / idempotency issues. Sourced from real M2 try-outs.
---

# Stripe Best Practice

Battle-tested rules for integrating Stripe Checkout Sessions + webhooks into the M2 credits system. Each rule maps to an incident from a real try-out session — students hit these in the *first hour* of M2, usually before they've internalized that "the webhook is the real payment flow, not the redirect."

When you (Claude Code) are guiding a student through any Stripe-touching code, **apply these rules proactively**. Don't wait for the student to ask. If you see them about to break one, stop and explain why.

This skill is the *application-layer* sibling of [[supabase-best-practice]] (DB schema + migration discipline) and [[aws-best-practice]] (infrastructure secrets). For higher-level API-selection questions (Checkout Sessions vs Payment Element vs Connect, Treasury, Connect Accounts v2), see the separate [[stripe-best-practices]] (plural) skill — this one is M2-specific operational rules.

---

## Execution mode: Cowork vs CLI

Stripe is a hosted API; the calling code runs identically in both modes. The deltas are around webhook delivery, env-var management, and CLI auth.

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Listen for webhooks during local dev | `stripe listen --forward-to localhost:3005/api/stripe/webhook` | **Not available.** Use a deployed preview URL + a dashboard-created endpoint instead. Or use Stripe MCP `mcp__stripe__authenticate` to read events post-hoc. |
| Create / list / inspect prices and products | `stripe prices list` / `stripe products list` | `mcp__stripe__*` tools (if available) **or** Stripe dashboard |
| Create a webhook endpoint | `stripe webhook_endpoints create --url ... --enabled-events ...` | **Dashboard primary** — Stripe MCP doesn't manage webhook endpoints as of 2026-05. See [[m2-stripe-credits-prerequisites]]. |
| Set Stripe env vars in Vercel | `vercel env add STRIPE_SECRET_KEY production` | **Dashboard primary** — Vercel MCP doesn't manage env vars as of 2026-05. |
| Resend a specific event | `stripe events resend <evt_id>` | Stripe dashboard → Developers → Events → Resend |
| Verify webhook reached your endpoint | `stripe listen` terminal shows `<-- [200]` | Dashboard → Webhooks → endpoint → recent deliveries |

**Stripe keys never sit in committed files.** `.env.local` (gitignored) for local; Vercel project env vars for prod. The webhook secret is *different* between `stripe listen` (rotates per restart) and a dashboard-created endpoint (stable) — see Rule 4.

---

## Hard rules

### Rule 1 — The webhook is the source of truth for crediting — never credit in the success/redirect page

> **The rule:** `POST /api/stripe/webhook` is the only place that writes to `credit_transactions` and increments `profiles.credits_balance`. The `/credits/success` page is UX-only — it polls `/api/credits` to refresh the displayed balance, but it must never call any "grant credits" RPC.

**Why:** Three failure modes the webhook handles and the redirect can't:
1. **User closes the browser after paying.** Stripe still fires `checkout.session.completed`; the user's balance still goes up. If you credit on `/credits/success`, the user got charged but got no credits.
2. **Network blip on the redirect.** Same shape — charge succeeded, redirect failed, no credits.
3. **Crediting from the client side is forgeable.** Any user can hit `/credits/success?session_id=cs_test_xxx` directly with no actual payment. Trusting the redirect = free credits for anyone who reads your network tab.

**How to apply:**
- `/credits/success/page.tsx` reads balance from `/api/credits`, does NOT mutate.
- All `INSERT INTO credit_transactions` and all `UPDATE profiles SET credits_balance = ...` lives in `app/api/stripe/webhook/route.ts`.
- The webhook uses `createServiceClient()` (service-role) because Stripe is not an authenticated Supabase user.

If a student says "but the webhook hasn't arrived yet when the user lands on /success" — that's expected; the success page polls. Typical webhook latency is sub-second.

---

### Rule 2 — Always read the raw body with `request.text()` before signature verification — never `request.json()` first

> **The rule:** In the webhook route, the very first thing you do with the request is `const body = await request.text()`. Then `stripe.webhooks.constructEvent(body, sig, STRIPE_WEBHOOK_SECRET)`. Never call `request.json()` first, never re-stringify.

**Why:** Stripe signs the **exact byte stream** of the request body. If Next.js parses JSON and you re-serialize, key ordering, whitespace, and Unicode escaping can all change — the recomputed HMAC won't match the `Stripe-Signature` header, and you'll get `400 Webhook signature verification failed` for every event. The symptom looks like "my secret is wrong" but the secret is fine; the bytes are different.

**How to apply:**
```ts
export async function POST(request: Request) {
  const body = await request.text()
  const sig = request.headers.get('stripe-signature')!
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch (err) {
    return new Response(`Webhook Error: ${(err as Error).message}`, { status: 400 })
  }
  // ... handle event.type
}
```

- App Router routes do NOT have the old Pages-Router `bodyParser` config — `request.text()` just works.
- If you see `JSON.parse` or `await request.json()` anywhere above `constructEvent`, that's the bug.

---

### Rule 3 — Always enforce idempotency on `stripe_payment_intent_id` — Stripe WILL retry

> **The rule:** Before inserting a `credit_transactions` row for a `checkout.session.completed` event, check whether a row with the same `stripe_payment_intent_id` already exists. If yes, return 200 with `{ received: true, duplicate: true }` and do nothing else. Enforce this at the DB level with a `UNIQUE INDEX` too — defense in depth.

**Why:** Stripe retries any webhook delivery that doesn't return a 2xx within ~10 seconds, and re-fires every event after dashboard "Resend" clicks. Without an idempotency check, every retry double-credits — the user buys 10 credits once and gets 30 because Stripe retried twice. The retries also happen invisibly on Stripe's side; you won't see them in your logs unless you check the dashboard.

**How to apply:**
- DB level — migration:
  ```sql
  CREATE UNIQUE INDEX IF NOT EXISTS credit_transactions_stripe_payment_intent_id_unique
    ON public.credit_transactions (stripe_payment_intent_id)
    WHERE stripe_payment_intent_id IS NOT NULL;
  ```
  Partial index so legacy non-Stripe rows (signup bonus, manual deductions) aren't constrained.
- Application level — in the webhook:
  ```ts
  const { data: existing } = await supabase
    .from('credit_transactions')
    .select('id')
    .eq('stripe_payment_intent_id', paymentIntentId)
    .maybeSingle()
  if (existing) {
    return Response.json({ received: true, duplicate: true })
  }
  ```
- Return 200 (not 4xx) for duplicates. Stripe treats anything non-2xx as a retry signal, and you don't want it retrying a deliberate no-op.
- The webhook handler must always finish in under 10 seconds. If you add slow work (e.g. sending an email), do it after the DB writes and consider moving it to a background queue.

---

### Rule 4 — The `STRIPE_WEBHOOK_SECRET` from `stripe listen` is NOT the same as the dashboard-endpoint secret — and `stripe listen` rotates it every restart

> **The rule:** `stripe listen --forward-to ...` prints a fresh `whsec_...` on its first line every time it starts. That value is what goes in your local `.env.local`. A webhook endpoint created in the Stripe dashboard has a *different*, stable secret. Don't mix them.

**Why:** Students restart `stripe listen` (because their laptop slept, or they killed the terminal) and don't update `.env.local`. Every event after that fails signature verification because the secret used to sign is the *new* CLI session's secret, but the env still has the *old* one. The symptom — `400 Webhook signature verification failed` — looks like a code bug. It's a stale env var.

Conversely, students sometimes copy the *dashboard endpoint's* secret into `.env.local` for local development. That endpoint signs events going to the deployed URL (e.g. `subtitle.svuncle.com`), not to `localhost`. Same `400` symptom, opposite root cause.

**How to apply:**

| Environment | Secret source | Stability |
|---|---|---|
| Local dev (`localhost:3005`) | `stripe listen` startup banner | **Rotates every restart** — re-copy + restart Next.js |
| Vercel preview / production | Stripe dashboard → Webhooks → endpoint → Signing secret | Stable until you click "Roll" |

- Document the local-dev secret rotation in your run-book. Common workflow:
  1. Start `stripe listen --forward-to localhost:3005/api/stripe/webhook`.
  2. Copy the `whsec_...` from its first line.
  3. Paste into `.env.local` as `STRIPE_WEBHOOK_SECRET`.
  4. Restart `npm run dev` (Next.js only re-reads `.env.local` on boot).
- When you go to prod (M2 step 9), the value in Vercel env is the *endpoint's* secret, not the CLI's. They are two unrelated `whsec_*` strings.

---

### Rule 5 — Expose the webhook path in `middleware.ts` — auth middleware will silently 307 every Stripe event otherwise

> **The rule:** The webhook endpoint `/api/stripe/webhook` must be matched as a public path in `middleware.ts`. If your middleware redirects unauthenticated requests to `/login` (HTTP 307), Stripe webhooks will hit that redirect, return 307, and never reach your handler.

**Why:** Stripe's webhook delivery follows the HTTP spec: non-2xx is a failure. 307 is non-2xx. Stripe sees a delivery failure, retries with backoff, eventually marks the event as failed. The symptom is unique and confusing: **Stripe dashboard shows the event fired, payment succeeded, the dashboard endpoint shows delivery attempts, but your Next.js logs show ZERO hits on `/api/stripe/webhook`** — because the middleware short-circuited the request before the route ran.

Auth middleware tends to be added or refactored *after* the webhook is working locally with `stripe listen` (which talks directly to the route via the CLI tunnel, bypassing middleware). So this bug lands in production, weeks later, the first time a real customer pays.

**How to apply:**

Pattern A — `isPublic` allowlist in middleware body:
```ts
const isPublic =
  request.nextUrl.pathname === '/' ||
  request.nextUrl.pathname.startsWith('/login') ||
  request.nextUrl.pathname.startsWith('/auth') ||
  request.nextUrl.pathname.startsWith('/api/stripe/webhook')
if (!user && !isPublic) {
  return NextResponse.redirect(new URL('/login', request.url))
}
```

Pattern B — `config.matcher` exclusion regex (the middleware never runs at all for matching paths):
```ts
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api/stripe/webhook).*)'],
}
```

Either pattern is fine; pick the one that matches your existing middleware shape. If you later add another public webhook endpoint (GitHub, SendGrid, Twilio), it needs the same treatment — same incident shape, different vendor.

**Verification:** after deploying the middleware change, click "Resend" on a recent event in the Stripe dashboard. You should see a 200 in both the dashboard delivery list and the Vercel function logs.

---

### Rule 6 — Stash `user_id`, `product_id`, and `credits` in BOTH `metadata` AND `client_reference_id` when creating the Checkout Session

> **The rule:** When calling `stripe.checkout.sessions.create`, set `metadata: { user_id, product_id, credits }` AND `client_reference_id: user_id`. The webhook reads from `session.metadata`; `client_reference_id` is a redundant safety net that also shows in the dashboard's session list.

**Why:** The webhook has no other way to know *which user* gets credited. Stripe doesn't know about your `auth.users.id`. If `metadata.user_id` is missing or empty, the webhook can't proceed safely — it would have to refuse the event (returning 400 means Stripe retries forever) or fail silently (the user pays and no one gets credited). Both outcomes are bad. Set the metadata at session-creation time, validate its presence in the webhook, and return 400 only when it's structurally missing (which means your own checkout route has a bug, not a Stripe issue).

`client_reference_id` is a top-level Stripe field surfaced in the dashboard session UI, which makes manual debugging ("which user paid for cs_test_abc?") a one-click lookup instead of clicking into metadata.

**How to apply:**
```ts
const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  payment_method_types: ['card'],
  line_items: [{ price: product.stripe_price_id, quantity: 1 }],
  metadata: {
    user_id: user.id,
    product_id: product.id,
    credits: String(product.credits), // metadata values are strings
  },
  client_reference_id: user.id,
  success_url: `${origin}/credits/success?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url: `${origin}/credits`,
})
```

In the webhook:
```ts
const { user_id, product_id, credits } = session.metadata ?? {}
if (!user_id || !product_id || !credits) {
  console.error('Missing metadata on session', session.id)
  return new Response('Missing metadata', { status: 400 })
}
const creditsToAdd = parseInt(credits, 10)
```

**Stripe metadata values are always strings.** `JSON.stringify({ credits: 10 })` becomes `"10"` on the wire. Parse with `parseInt` in the webhook before using.

---

### Rule 7 — Test cards only ever work in test mode — and the same card numbers behave differently in live

> **The rule:** `4242 4242 4242 4242` (and the rest of Stripe's [test card list](https://docs.stripe.com/testing#cards)) work *only* when your keys are `sk_test_...` / `pk_test_...`. The same numbers in live mode will be declined as fraudulent. Conversely, **never** paste a real card number into a sandbox checkout to "see what happens" — Stripe's sandbox can charge real cards in some configurations and you've just transferred real money.

**Why:** This is the most common "I thought I was testing" incident. Students switch their Vercel env to `sk_live_...` to test the prod deploy, forget to switch back, then run their local M2 test with `4242 4242 4242 4242` — get a decline they don't understand. Or worse: they test live mode with a real card "just to see if the redirect works" and accidentally take their own money.

Stripe's sandbox = Stripe-built fake environment that issues fake `pi_test_*` / `cs_test_*` IDs and never moves money. Live = real charges. The mode is determined entirely by which key you use — there is no separate "test endpoint."

**How to apply:**
- M2 runs entirely in **sandbox** ([[project_course2_sandbox_only]]). Keys are `sk_test_...` and `pk_test_...`. Live activation needs phone + bank account + business info and is deferred to M3 at earliest.
- The Stripe dashboard has a sandbox/live toggle in the top-left corner. Always check before creating products, prices, or webhook endpoints — a price created in sandbox does not exist in live and vice versa.
- For the M2 smoke test, use exactly:
  - Card: `4242 4242 4242 4242`
  - Expiry: any future date (e.g. `12/34`)
  - CVC: any 3 digits (`123`)
  - ZIP: any (`12345`)
- For live testing later, Stripe has dashboard-issued "Test mode in live" trick cards — but if you're not sure, **don't**. Use a small real charge ($1) and immediately refund.

---

### Rule 8 — A Stripe Price object is immutable — to re-tier, create a new Price or only rename the Product

> **The rule:** You cannot edit `unit_amount` (the dollar amount) on an existing Stripe Price. Once `price_1XYZ` is $10, it is $10 forever. To change the price, create a new Price and update your `credit_products.stripe_price_id` to point at it. To change the *credits granted* without changing the dollar amount, keep the Price and update only the `credits` column in your DB.

**Why:** Students think of Stripe Prices as editable rows; they're actually immutable financial primitives, more like a Git commit than a SQL row. Trying to mutate one via the API returns a 400. The Stripe dashboard hides this — it lets you "archive" an old Price and "create" a new one, but they're separate objects with different IDs.

This matters during a re-pricing: you write a migration that updates `credit_products`, the migration succeeds, the next purchase still creates a Checkout Session against the OLD price because Stripe doesn't care about your migration — it only cares about which `price_id` you pass to `checkout.sessions.create`.

**How to apply:**

Re-pricing with a new dollar amount → new Price:
1. Stripe dashboard / `stripe prices create` — create the new Price under the existing Product.
2. Migration:
   ```sql
   UPDATE public.credit_products
     SET stripe_price_id = 'price_NEW_id', price_usd = 20.00, credits = 25
     WHERE id = '<product-row-id>';
   ```
3. Apply locally + remote via [[supabase-best-practice]] Rule 1.

Re-tiering with the same dollar amount, different credits → keep the Price:
1. Rename the Product in the Stripe dashboard (cosmetic; helps grep later — e.g. `30_to_45_credits`).
2. Migration updates only `credits` and `name`, NOT `stripe_price_id`:
   ```sql
   UPDATE public.credit_products SET credits = 45, name = '45 Credits'
     WHERE stripe_price_id = 'price_1XYZ';
   ```

The `/credits` page reads tier info from `credit_products`, not Stripe directly — so once the migration is applied, the new tier shows up everywhere.

---

### Rule 9 — Worker deducts credits ONLY when a job transitions to `done` — never on `pending`/`downloading`/etc.

> **The rule:** The worker (M1+) reads `profiles.credits_balance` and writes `new_balance = max(0.0, balance - minutes)` exactly once per job, at the moment the job's `status` becomes `done`. Pre-check at job submission (`/api/jobs`) refuses to enqueue jobs that already have insufficient balance. Mid-pipeline transitions don't touch credits.

**Why:** Three concrete failure modes if you deduct earlier:
1. **Deduct on `pending`/insert:** the user is charged before the worker has even confirmed the video downloads. If the URL is bad, the user lost credits for a job that produced nothing.
2. **Deduct on `downloading`:** Whisper might 5xx out, the job retries on session N+1, you deduct twice for the same final transcript.
3. **Deduct in two places (pre-check AND on done):** classic double-charge if you don't perfectly separate "estimate to refuse" from "actual deduction."

The canonical flow: pre-check uses the *estimate* (cheap probe of duration before download — see [[whisper-best-practice]] Rule 3 + M1 worker Step 7b). Actual deduction uses the *real* minutes from the completed transcript (ffprobe on the downloaded file). The pre-check protects against obvious-insufficient cases; the post-deduction is the ledger truth.

**How to apply:**
```python
# in worker.py, after the transcript is successfully written to job_sessions
profile = supabase.table("profiles").select("credits_balance").eq("id", job["user_id"]).single().execute().data
new_balance = max(0.0, float(profile["credits_balance"]) - minutes)
supabase.table("profiles").update({"credits_balance": new_balance}).eq("id", job["user_id"]).execute()
supabase.table("credit_transactions").insert({
    "user_id": job["user_id"],
    "job_id": job["id"],
    "type": "deduction",
    "amount": -minutes,
    "description": f"Transcribed {minutes} min",
}).execute()
supabase.table("jobs").update({"status": "done"}).eq("id", job["id"]).execute()
```

- `float()` not `Number()` — Python, not JS. (Common typo when students copy from the Node webhook code.)
- The `max(0.0, ...)` floor protects against rounding weirdness; a job should never push the balance negative — that's what the pre-check is for.
- If the pre-check passes but the actual duration exceeds the estimate by more than a few minutes, log to `app_logs` so admins can review; don't silently overcharge.

---

### Rule 10 — Send the user_id via metadata, NOT by inferring from the Stripe customer / email

> **The rule:** The webhook reads `user_id` from `session.metadata.user_id` (set in Rule 6). Never look up the Supabase user by `session.customer_email`, `session.customer_details.email`, or any other identity field on the Stripe side.

**Why:** Three failure modes:
1. **Email mismatch.** Users sign up to Supabase with one email and pay through Checkout with another (corporate card with billing email, Apple Pay using a different masked iCloud address). The webhook's email-based lookup finds no user → credits are stranded.
2. **Account takeover via email collision.** If `user_id` is derived from `customer_email`, an attacker who can pay with a target's email gets credits added to the target's account — or worse, to their own account that happens to share an email.
3. **Stripe customer objects are per-session by default.** Until you explicitly create a `Customer` and reuse its ID, every checkout session creates a new ephemeral customer record with no link back to your Supabase user.

The `metadata.user_id` is set by *your* server, at session-creation time, when the user is authenticated. It cannot be forged by the buyer. Trust it.

**How to apply:**
- In `/api/credits/checkout`, the route must be auth-gated; `supabase.auth.getUser()` must succeed before creating the session. The session's `metadata.user_id` comes from `user.id`, not from a request body parameter (which the client could spoof).
- In the webhook, never call `supabase.auth.admin.getUserByEmail(...)` for credit-grant logic. Email-based lookups are fine for *display* ("Send receipt to..." workflows), never for *write authorization*.

---

## What Stripe is NOT (in the M2 context) — set student expectations early

| What Stripe IS (in M2) | What Stripe is NOT (in M2) |
|---|---|
| One-time card payments via Checkout Sessions | Subscriptions / recurring billing (deferred — different mode, different webhook events) |
| Sandbox-only — test cards, fake `pi_test_*` IDs | Live charges — needs phone + bank + business info (M3+) |
| Hosted Checkout (redirect to `checkout.stripe.com`) | Embedded Payment Element / custom form (Stripe supports it, M2 doesn't use it) |
| Card payments (USD) | Apple Pay / Google Pay / local methods (Stripe supports them, M2 sticks to card-only) |
| Sender-side: you, charging your users | Connect / marketplace platform that splits payments |
| Test card `4242 4242 4242 4242` | Real cards |

The production stack in this repo also handles `charge.refunded` events (see [[milestone-payment-stripe]] step 13) — but M2 stops at one-time purchases deliberately, so the student understands the core flow before adding refund logic.

---

## Things to actively watch out for

These show up often enough to recognize without being hard rules.

1. **`request.json()` accidentally added to the webhook route.** Anywhere above `constructEvent` kills signature verification. Re-read Rule 2 if a student sees `400 signature verification failed` and the secret is correct.

2. **`stripe listen` not running.** Local-dev webhooks silently never fire. Symptom: Checkout completes, redirect works, no `POST /api/stripe/webhook` in the Next.js log. Confirm `stripe listen` is in a foreground terminal with green "ready" status.

3. **Wrong port in `--forward-to`.** Next.js defaults to `:3000`, this project uses `:3005`. A mismatched port = `stripe listen` forwards to nothing. Symptom looks identical to #2.

4. **Vercel function logs for production webhooks** are at `vercel logs <deployment-url>` (CLI) or Vercel dashboard → Logs → filter by function `/api/stripe/webhook`. Don't try to find webhook errors in browser DevTools — webhooks never touch the browser.

5. **Stripe MCP auth state is per-account.** If you authenticate the MCP against sandbox (`acct_1TOUcRFqDfsTD8K9`) and later want to query live (`acct_1TOUcEFmU1cozSnr`), you need a separate `mcp__stripe__authenticate` flow. Trying to call live endpoints with sandbox auth returns 401.

6. **Sandbox prices are NOT visible in live and vice versa.** A common confusion when switching: students create prices in sandbox, switch the dashboard toggle to Live, and panic that "all my products are gone." They're not gone; you're looking at a different dataset. Toggle back.

7. **`session.payment_status === 'paid'` check is important.** A `checkout.session.completed` event can fire with `payment_status: 'unpaid'` for delayed payment methods (ACH, SEPA, BNPL). For card-only M2, treat anything other than `'paid'` as "do nothing yet." Stripe will fire a follow-up event when the payment actually clears.

8. **Metadata size limit: 500 characters per value, 50 keys per object.** For M2 we only stash three small fields, so this never matters — but if a future milestone adds long JSON to metadata, you'll silently truncate.

9. **Don't `await` long-running work in the webhook before returning 200.** Stripe times out delivery at ~10s. Do the DB writes, return 200, then do any slow side-effect (email, Slack ping) in the background. For M2 the only work *is* the DB writes, so this is fine — but worth flagging for later.

10. **Checkout redirect URLs need an absolute origin.** `success_url: '/credits/success'` doesn't work; Stripe needs `https://...`. Build it from the request `Origin` header, with `NEXT_PUBLIC_SITE_URL` as a fallback for routes that don't have a request context (server-side cron, etc.).

11. **`stripe-signature` header is `Stripe-Signature` with a capital S in Stripe's docs but lowercase in Next.js's `request.headers.get`.** Use lowercase — Next.js normalizes headers to lowercase per the Fetch API spec.

---

## Out of scope for M2

Real Stripe-in-production concerns we deliberately don't enforce in M2:

- **Live mode activation.** Needs phone + bank + business identity — deferred to M3 ([[project_course2_sandbox_only]]).
- **Subscriptions / recurring billing.** M2 is one-time top-ups only. Adding subscriptions means switching to `mode: 'subscription'`, handling `customer.subscription.*` events, and managing recurring price IDs. Future milestone.
- **`charge.refunded` handling.** Production stack handles it ([[milestone-payment-stripe]] step 13); M2 doesn't. If a student refunds in sandbox, the credits stay — that's correct for the M2 scope.
- **Tax (Stripe Tax) / VAT.** Not enabled in M2. If you eventually sell to EU/UK/JP, you'll need to enable Tax in the dashboard and set `automatic_tax: { enabled: true }` on Checkout Sessions.
- **Connect / marketplace flow.** Different API surface (Accounts v2, controller properties). See [[stripe-best-practices]] (plural) for higher-level routing.
- **Embedded Payment Element / custom forms.** M2 uses hosted Checkout for simplicity. Custom forms need a client-side `@stripe/stripe-js` integration and Setup Intents — a bigger lift, no clear M2 benefit.
- **Webhooks for `payment_intent.*` events.** M2 only listens to `checkout.session.completed`. Lower-level events fire too but we don't act on them.

---

## Cross-references

- [[m2-stripe-credits]] — where the Checkout + webhook + worker-deduct code is actually written (Steps 6 + 7)
- [[m2-stripe-credits-prerequisites]] — Stripe MCP auth flow + sandbox account setup
- [[m2-stripe-credits-checklist]] — verification of webhook idempotency, middleware exemption, worker deduct ordering
- [[milestone-payment-stripe]] — production-stack record (includes refund handling, tier re-pricing, live rollout)
- [[supabase-best-practice]] — migration discipline for `credit_products` / `credit_transactions` schema
- [[aws-best-practice]] — where `STRIPE_SECRET_KEY` lives on the EC2 worker (Secrets Manager, not `.env`)
- [[project_course2_sandbox_only]] — sandbox-only constraint for Course 2 M2
- [[stripe-best-practices]] (plural) — higher-level API selection (Checkout vs PaymentIntents vs Connect vs Treasury)
- Stripe webhook docs: https://docs.stripe.com/webhooks
- Stripe test cards: https://docs.stripe.com/testing#cards
- Stripe Go-Live checklist: https://docs.stripe.com/get-started/checklist/go-live
