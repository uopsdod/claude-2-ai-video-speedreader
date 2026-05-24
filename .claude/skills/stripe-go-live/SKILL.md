---
name: stripe-go-live
description: Walkthrough for flipping a Course 2 SaaS (subtitle-manager / ai-video-speed-reader) from Stripe **sandbox** to **live** mode after the course is done — Stripe account activation (phone + bank + business info), re-creating products + prices in live mode, swapping the three Stripe env vars on Vercel, re-pointing the webhook endpoint to the live-mode dashboard, updating `credit_products.stripe_price_id` in Supabase, and a small-amount real-card smoke test + refund. This skill is the **optional post-course path** — Course 2 itself (M0–M3) intentionally stays in sandbox per the course design. Use when the student says "開通正式環境", "go live", "switch from sandbox to live", or "start accepting real payments".
---

# Stripe Go-Live — sandbox → live mode

## Scope + when to use this skill

**This skill is the post-course go-live path.** The base course (M0–M3) runs entirely in Stripe **sandbox** by design — it lets students walk through the full payment loop (Checkout → webhook → credits ledger → worker metering) without phone/bank/business-info collection. Going live is a v2 step: necessary for receiving real payouts, **not** a prerequisite for finishing the course.

Use this skill when the student has:

1. ✅ Finished M2 (sandbox payments end-to-end green) — including the M2 checklist.
2. ✅ Finished M3 (custom domain attached + serving on `ai-video-speed-reader.<your-domain>.com`).
3. ✅ Decided they want to charge real customers — and has the **phone number, bank account, and business info** Stripe will demand during activation.

If any of those is missing, stop and finish that piece first — don't half-activate.

> **Why a separate skill, not an M4:** activation involves Stripe holding real identity + bank-account info that varies per country and per business structure. The course can't make assumptions about what students have. By breaking it out, students who never go live get a complete, working M2/M3 product; students who do go live get a focused walkthrough.

## What changes when you go live (the full surface area)

The single source of truth in this section: live mode is a **separate set of keys, products, prices, customers, webhook endpoints, and events** from sandbox. Nothing carries over. The two modes share only the Stripe **account login** and the activation status.

| Surface | In M2 sandbox | After go-live | Owner |
|---|---|---|---|
| Stripe account state | Sandbox (banner reads "Sandbox") | **Activated** (Settings → Account details → Activated) | Stripe dashboard, this skill §1 |
| Stripe Publishable key (`pk_*`) | `pk_test_...` | `pk_live_...` | Vercel env, this skill §3 |
| Stripe Secret key (`sk_*`) | `sk_test_...` | `sk_live_...` | Vercel env, this skill §3 |
| Stripe Webhook endpoint | Sandbox endpoint pointing at custom domain | **New** live-mode endpoint pointing at custom domain | Stripe dashboard, this skill §4 |
| Stripe Webhook secret (`whsec_*`) | Sandbox `whsec_...` | **New** live-mode `whsec_...` | Vercel env, this skill §4 |
| Stripe Products + Prices | 3 sandbox products + 3 sandbox `price_...` IDs | **3 new** live-mode products + 3 new live-mode `price_...` IDs | Stripe dashboard / MCP, this skill §2 |
| Supabase `credit_products.stripe_price_id` (3 rows) | sandbox `price_...` values | live-mode `price_...` values | Supabase migration, this skill §5 |
| Webhook URL it points at | `https://ai-video-speed-reader.<your-domain>.com/api/stripe/webhook` | **same URL** — the URL does not change, only which Stripe mode posts to it | n/a |
| Stripe MCP / CLI auth | Authenticated against **sandbox** account (`livemode: false`) | **Re-authenticate** against the **live** account so MCP calls hit live data | This skill §6 (optional) |
| Test card `4242 4242 4242 4242` | Works | **Stops working** — live mode rejects test cards | n/a |
| Real cards | Rejected by sandbox | Accepted | n/a |

**Six things change. The custom-domain URL does NOT change.** The webhook URL is identical; only the Stripe-side endpoint *definition* (and the `whsec_...` it generates) is new.

## How M3 connects to this skill

This skill assumes M3 already attached a custom domain — `ai-video-speed-reader.<your-domain>.com` — and Vercel is serving production there. If you go live without M3, your webhook would be pointing at `https://<project>.vercel.app/api/stripe/webhook`, which is still fine but advertises Vercel branding to customers and to Stripe's webhook UI. M3 first is the conventional order.

## Section 1 — Activate the Stripe account

> Plan ~30–60 minutes of focused dashboard time. Have your phone, a debit card or bank-account number, and your business info ready before you start.

1. Open the Stripe dashboard logged in as the same email you used for the M2 sandbox account.
2. Top-left switcher — switch from **Sandbox** to your **Live account** (it's the entry above "Sandboxes"). The whole dashboard turns from sandbox striping to live mode.
3. Top banner → **Activate your account**.
4. Stripe walks you through a form. The exact fields vary by country, but the universal asks are:
   - **Business type** — `Individual` is the fastest path for a solo developer; `Company` if you're operating under a registered entity.
   - **Legal name + DOB** (Individual) or **registered business name + tax ID** (Company).
   - **Phone number** — Stripe sends a verification SMS. This is non-skippable.
   - **Business address**.
   - **Industry / product description** — pick the closest match (e.g. "SaaS — software"). Write 1–2 sentences describing what `ai-video-speed-reader.<your-domain>.com` does. Stripe reads this for risk review.
   - **Statement descriptor** — what customers see on their card statement. Max 22 chars, must be recognizably the same product (e.g. `AI VIDEO SPEEDREADER`).
   - **Bank account** — the account that receives payouts. Stripe verifies via micro-deposits or instant-bank-link depending on country.
5. Submit. Stripe shows **"Pending verification"** or **"Activated"** immediately.
   - **Pending:** activation can take a few hours to a few days while Stripe's risk team reviews. You can continue this skill up through §2 and §3 (which only need `sk_live_*` / `pk_live_*` — these are issued on submission, not on full activation), but **do NOT run the smoke test in §7** until the dashboard banner says "Activated" and Settings → **Payouts** shows your bank account as verified. Triggering a real charge before activation completes can hold funds.
   - **Activated:** continue.

> **Where this falls down:** if Stripe rejects your activation (most common reasons: business address doesn't match phone, bank account in a different name than the legal entity, product description too vague), they email a remediation step. Resolve and re-submit before §7.

## Section 2 — Re-create the 3 credit-tier products + prices in live mode

The 3 sandbox products from M2 Step 3 don't exist in live mode. Live mode starts empty. You're creating 3 new products, getting 3 new `price_...` IDs, and treating them as the canonical IDs going forward.

> **Why not "copy from sandbox"?** Stripe deliberately keeps sandbox and live separate ledger-spaces. There is no API to copy a product from sandbox to live. The data is small (3 products, 3 prices); just re-create.

### 2.1 — Confirm you're targeting live mode

Critical foot-gun. Stripe MCP and Stripe CLI sessions inherit whichever account was authenticated last. If you authenticated against the M2 sandbox, every `create_product` / `create_price` call you're about to make would silently land in sandbox — and then §3's `sk_live_*` keys wouldn't see them.

**Option A — Re-authenticate Stripe MCP against live (Cowork):**

Disconnect the Stripe Connector in Cowork's panel → Re-Connect. When the consent page appears, in the account selector dropdown pick **Live account** (NOT the sandbox you used in M2 prereq). Authorize.

Verify:

`mcp__claude_ai_Stripe__get_stripe_account_info`

Expect `livemode: true` (or look for an absence of `livemode: false` plus the account being your activated live account).

**Option B — Stripe CLI (`stripe login --live`):**

```bash
stripe login --live
# opens browser; pick Live account; Allow access
stripe config --list
# expect live_mode_api_key set, test_mode_api_key may or may not be set
```

Either option is fine. **Just confirm `livemode: true` before proceeding to 2.2.**

### 2.2 — Create the 3 live-mode products + prices

Match the M2 sandbox tiers exactly so the Supabase rows stay 1:1. The M2 default tiers (from `m2-stripe-credits/SKILL.md` Step 3) are:

| name | credits | price (USD, one-time) |
|---|---|---|
| Starter | 60 | 5.00 |
| Standard | 200 | 15.00 |
| Pro | 500 | 35.00 |

> **If you changed tiers during M2 customization**, use *your* sandbox values from `m2-stripe-credits` Step 3, not the defaults above. Whatever's in your sandbox `credit_products` table is the source of truth for what to mirror here.

Ask Claude:

> 「請在 Stripe **live mode** 建這三個 one-time prices（USD）：
> - Starter — 60 credits — \$5.00
> - Standard — 200 credits — \$15.00
> - Pro — 500 credits — \$35.00
>
> 全部 `active=true`，`currency=usd`，`type=one_time`，`recurring=null`，**不要** 設 `metadata.credits`。建完把三個 live `price_id` 列給我，我要貼進 Supabase migration。」

Claude runs:

```
mcp__claude_ai_Stripe__create_product (name="Starter")
mcp__claude_ai_Stripe__create_price (product=<id>, unit_amount=500, currency=usd, ...)
# repeat for Standard ($1500) and Pro ($3500). Stripe amounts are in cents.
```

Or CLI equivalent: `stripe products create --name "Starter"` then `stripe prices create --product <id> --unit-amount 500 --currency usd`.

Capture the three new `price_...` IDs. They'll look like `price_1AbCdE...` but are **distinct from the sandbox ones**. Keep both lists (sandbox + live) side-by-side — you'll write the live ones into Supabase in §5.

> **Why `metadata.credits` stays off:** same reason as M2 — the credits-per-tier mapping lives in Supabase (`credit_products.credits`), not in Stripe. Keeping it Supabase-only means you can re-tier with a one-line SQL update without touching Stripe.

## Section 3 — Swap the 3 Stripe env vars on Vercel to live values

The Vercel project already has `STRIPE_SECRET_KEY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, and `STRIPE_WEBHOOK_SECRET` set in **Production** scope with sandbox (`sk_test_*` / `pk_test_*` / sandbox `whsec_*`) values from M2 Steps 2a and 9c. You're replacing the **values**, not the variable names.

> **About `STRIPE_WEBHOOK_SECRET`:** §4 below creates a new live-mode webhook endpoint and gives you a new `whsec_...`. Wait until then to update that one. This section covers only the two key envs.

### 3.1 — Grab the live keys

Stripe dashboard → switch to Live account if you aren't there → **Developers → API keys**:

- **Publishable key** → `pk_live_...`. Visible.
- **Secret key** → `sk_live_...`. Click **Reveal live key** (Stripe will ask you to re-auth). Copy once.

If "Reveal live key" is greyed out, you're not finished activating — finish §1 first.

### 3.2 — Update Vercel env vars

**Cowork path:** Vercel MCP does NOT expose env-var management (as of 2026-05). Open the Vercel dashboard:

`https://vercel.com/<your-team>/<your-project>/settings/environment-variables`

For each of the two:

1. Find `STRIPE_SECRET_KEY` (Production scope) → click ⋯ → Edit → paste `sk_live_...` → Save.
2. Find `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` (Production scope) → Edit → paste `pk_live_...` → Save.

**CLI path:**

```bash
vercel env rm STRIPE_SECRET_KEY production
vercel env add STRIPE_SECRET_KEY production
# paste sk_live_...

vercel env rm NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY production
vercel env add NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY production
# paste pk_live_...
```

### 3.3 — Trigger a re-deploy

Env-var changes don't take effect on the running deployment — Vercel needs a new build to pick them up. Trigger any deploy:

```bash
git commit --allow-empty -m "redeploy for stripe live keys"
git push origin main
```

Wait for the deploy to finish green. Verify in Vercel dashboard → Deployments → top row shows the new commit + "Ready".

> **Don't update `STRIPE_WEBHOOK_SECRET` yet.** The live-mode endpoint doesn't exist; the sandbox `whsec_...` currently in that env is stale but harmless until §4. Your app will fail signature verification on any real live-mode event until §4 + a second redeploy lands.

## Section 4 — Re-create the webhook endpoint in live mode

The sandbox webhook from M2 Step 9 lives in **sandbox**'s Webhooks list and only receives sandbox events. Live mode has its own Webhooks list, completely empty after activation. You're adding a new endpoint, getting a new `whsec_...`, and putting it on the same URL.

### 4.1 — Add the live-mode endpoint

Stripe dashboard → confirm top-left is **Live account** (NOT sandbox — most common foot-gun) → **Developers → Webhooks** → **Add endpoint**.

- **Endpoint URL:** `https://ai-video-speed-reader.<your-domain>.com/api/stripe/webhook` (same URL as M2 Step 9 — your custom domain from M3).
- **Listen to events on:** Your account.
- **Events to send:** at minimum `checkout.session.completed` (the only one M2 handles). Stripe also recommends `payment_intent.payment_failed` for surfacing card-decline UX, and `charge.refunded` if you plan to handle refunds in v2 — add them if you've built handlers, otherwise stick to `checkout.session.completed`.
- **API version:** match the version your app uses (`lib/stripe.ts` constructs `new Stripe(...)` without `apiVersion`, which means it uses the SDK default — let Stripe match it automatically).
- Click **Add endpoint**.

### 4.2 — Copy the live signing secret

After creation, Stripe shows the endpoint detail page. Click **Reveal** under "Signing secret". This is a fresh `whsec_...` — distinct from the sandbox one.

### 4.3 — Update Vercel env var + redeploy

**Cowork:** dashboard → Environment Variables → edit `STRIPE_WEBHOOK_SECRET` (Production) → paste new `whsec_...` → Save.

**CLI:**

```bash
vercel env rm STRIPE_WEBHOOK_SECRET production
vercel env add STRIPE_WEBHOOK_SECRET production
# paste whsec_...
```

Redeploy (env changes don't take effect on the running deployment):

```bash
git commit --allow-empty -m "redeploy for stripe live webhook secret"
git push origin main
```

Wait for green deploy.

> **Don't delete the sandbox webhook endpoint.** Keep it — you'll want sandbox available for any future test-card debugging. Stripe routes events by mode; the sandbox endpoint only receives sandbox events, the live endpoint only receives live events. They don't conflict.

## Section 5 — Update `credit_products.stripe_price_id` in Supabase

Currently the 3 rows in `credit_products` hold the **sandbox** `price_...` IDs. After §3+§4, your app is keyed to live mode, but Checkout Session creation in `/api/checkout/route.ts` reads `stripe_price_id` from this table and passes it to `stripe.checkout.sessions.create({ line_items: [{ price: <stripe_price_id>, ... }] })`. Pass a sandbox `price_...` to a live `sk_live_...` and Stripe returns `No such price: 'price_...'`.

### 5.1 — Build the migration

Per the project's auto-memory rule (`feedback_migration_files`): never `UPDATE` directly — write a migration file.

Create `supabase/migrations/<timestamp>_swap_credit_products_to_live_prices.sql`:

```sql
-- Swap credit_products.stripe_price_id from sandbox to live-mode price IDs.
-- Sandbox prices remain valid in sandbox mode but are no longer the active
-- payment surface for the deployed app (which now uses sk_live_*).

UPDATE public.credit_products
   SET stripe_price_id = '<live price_id for Starter>'
 WHERE name = 'Starter';

UPDATE public.credit_products
   SET stripe_price_id = '<live price_id for Standard>'
 WHERE name = 'Standard';

UPDATE public.credit_products
   SET stripe_price_id = '<live price_id for Pro>'
 WHERE name = 'Pro';
```

Replace each `<live price_id for X>` with the values you captured in §2.2.

### 5.2 — Apply the migration

Cowork (Supabase MCP):

`mcp__supabase_remote__apply_migration` with `name=swap_credit_products_to_live_prices` and `query=<the SQL above>`.

CLI:

```bash
supabase db push
```

### 5.3 — Verify

```sql
SELECT name, stripe_price_id FROM credit_products ORDER BY price_usd;
```

Expect 3 rows; every `stripe_price_id` should match your live `price_...` from §2.2.

> **No re-deploy needed for this step.** The app reads `credit_products` at request time from Supabase, not at build time.

## Section 6 — Optional: re-authenticate Stripe MCP / CLI against live (post-flip)

If §2.1 already had you authenticate against live, skip this. Otherwise: after going live you'll mostly want Stripe MCP / Stripe CLI pointed at live data (so `list_charges`, `list_disputes`, etc. show your real customers). Switch the auth:

- **Cowork:** Connectors → Stripe → Disconnect → Re-Connect → pick Live account on consent page → Authorize.
- **CLI:** `stripe login --live`.

To go *back* to sandbox temporarily (e.g. for a test-card debugging session): repeat the auth flow but pick the sandbox account instead. Stripe MCP / CLI hold one auth at a time.

> **Watch out:** any `create_product` / `create_price` / `refund` you ask Claude to run while MCP is authenticated against live mode affects real data. Re-read `livemode: true` in `get_stripe_account_info` before any destructive call.

## Section 7 — Smoke test against the live deploy (small amount + refund)

Final verification: a real card → real charge → real webhook → credits land → refund.

> **Pre-flight:** confirm §1 dashboard banner says "Activated" (not "Pending"). If still pending, stop. Skip ahead is fine for code/data prep (§2–§5), but the smoke test is the one thing you cannot do until Stripe approves activation.

### 7.1 — Small-amount real charge

1. Open the live site: `https://ai-video-speed-reader.<your-domain>.com/credits` (or wherever the credit-tier page lives — usually `/credits` per M2 Step 5).
2. Log in as **yourself** (a user account on your live Supabase that you can later credit-adjust). DO NOT log in as a test/customer account.
3. Click the cheapest tier (Starter = \$5.00 by default).
4. Stripe Checkout loads. **Pay with a real card you own** (your personal credit card is fine).
5. After redirect-back, you should land on your success URL.

### 7.2 — Confirm the webhook delivered + credits landed

Three places, all should be consistent within ~10 seconds:

1. **Stripe dashboard → Developers → Events** (live mode): top row should be `checkout.session.completed`, status `Succeeded`, your webhook endpoint listed as "delivered (200)".
2. **Vercel → Deployments → Runtime Logs**: search for `/api/stripe/webhook`. Should see a 200 response with no errors.
3. **Supabase**:
   ```sql
   SELECT * FROM credit_transactions
   WHERE user_id = '<your user id>'
   ORDER BY created_at DESC LIMIT 1;

   SELECT credits_balance FROM profiles WHERE id = '<your user id>';
   ```
   Expect: one new `credit_transactions` row of type `purchase` with `credits = 60` and `stripe_payment_intent_id` set; `profiles.credits_balance` increased by 60.

### 7.3 — Refund yourself

You don't want to keep a \$5 charge against your own card to validate this skill. Refund:

**Cowork:**

`mcp__claude_ai_Stripe__create_refund` with the `payment_intent` from the Events page (or `charge` ID).

**CLI:**

```bash
stripe refunds create --payment-intent <pi_...>
```

**Dashboard:** Payments → click the payment → Refund.

> **Important note about credits + refunds:** M2's webhook handler only credits on `checkout.session.completed`. It does NOT decrement on `charge.refunded` — that's a v2 roadmap item (see `m2-stripe-credits/SKILL.md` Step 5 notes). So your refund here will return the \$5 to your card but leave the 60 credits in your `profiles.credits_balance`. For the smoke test, manually balance the ledger:
>
> ```sql
> UPDATE profiles SET credits_balance = credits_balance - 60 WHERE id = '<your user id>';
> INSERT INTO credit_transactions (user_id, type, credits, note)
>   VALUES ('<your user id>', 'adjustment', -60, 'manual smoke-test refund offset');
> ```
>
> Once you've implemented `charge.refunded` handling, you can skip this manual offset.

### 7.4 — Test mode is now closed

Open `https://ai-video-speed-reader.<your-domain>.com/credits` in a private/incognito window and try the test card `4242 4242 4242 4242`. Expected: Stripe rejects with **"Your card was declined."** That's confirmation you're live — test cards do not work in live mode.

## Section 8 — Sanity checklist

After §1–§7, walk through:

- [ ] Stripe dashboard top banner reads "Activated" (not "Pending verification").
- [ ] Stripe **Developers → API keys** shows live keys revealable (`sk_live_*` available).
- [ ] Vercel Production env: `STRIPE_SECRET_KEY` starts with `sk_live_`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` starts with `pk_live_`, `STRIPE_WEBHOOK_SECRET` is the new `whsec_*` from §4.2 (not the old sandbox one).
- [ ] Latest Vercel deploy is **after** the env-var updates of §3.3 + §4.3.
- [ ] Stripe **Developers → Webhooks** (live mode): one endpoint exists, pointed at `https://ai-video-speed-reader.<your-domain>.com/api/stripe/webhook`, listening at minimum to `checkout.session.completed`, status "Enabled".
- [ ] Stripe **Products** (live mode): 3 products (Starter / Standard / Pro), 3 active prices.
- [ ] Supabase `credit_products`: 3 rows, every `stripe_price_id` is a live `price_*` (matches §2.2).
- [ ] §7 smoke test: real card → 200 webhook → credits ledger row → refund issued.
- [ ] Sandbox is still intact (top-left switcher still shows the sandbox; sandbox products + webhook still exist). Useful as a fallback debugging environment.

If any box is unchecked, fix that step before announcing the launch.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Vercel deploy is green but Checkout still loads with sandbox styling / test cards work | Env-var update saved to wrong scope (Preview instead of Production), OR the redeploy hasn't run yet | Re-check the scope in Vercel dashboard. Run an empty commit + push to force a Production redeploy. |
| Stripe Events page shows `checkout.session.completed` delivered, but no row in `credit_transactions` | Webhook signature verification failed silently | Vercel runtime logs → search for `/api/stripe/webhook`. If you see `webhook signature verification failed`, you didn't update `STRIPE_WEBHOOK_SECRET` (§4.3) or didn't redeploy. |
| "No such price: 'price_...'" from Checkout creation | `credit_products.stripe_price_id` still has sandbox IDs | Re-apply migration from §5. Verify with the SQL in §5.3. |
| `pk_live_*` works but Stripe Checkout returns "Your card was declined" on a real card | Activation is still "Pending" — Stripe lets you create sessions but rejects charges | Wait for activation; check Stripe dashboard email for remediation steps. |
| Webhook endpoint returns 401 from Stripe's "Send test event" button | The custom-domain URL is going through auth middleware | Confirm M2 Step 6c exception (the webhook route is excluded from `auth.getUser()`). The webhook MUST NOT pass through middleware auth. |
| "Reveal live key" greyed out in Developers → API keys | Activation incomplete | Finish §1. Without activation, only test keys are issued. |
| MCP creates products in sandbox even though I want live | MCP auth still bound to sandbox from M2 prereq | Re-auth per §2.1 / §6 and confirm `livemode: true` in `get_stripe_account_info`. |
| Tax / 3D Secure prompt blocks real-card checkout but worked in sandbox | Live mode enforces SCA + tax rules sandbox skips | This is expected for EU/UK/SG cards. Stripe Checkout handles SCA automatically — the customer just sees a 3DS challenge. For tax, decide whether to enable Stripe Tax (Settings → Tax) for live; sandbox didn't need it. |

## Why I'm NOT changing other parts of the codebase

These are deliberately untouched and stay sandbox-pattern after go-live:

- **App code** (`/api/checkout/route.ts`, `/api/stripe/webhook/route.ts`, `lib/stripe.ts`) — entirely env-var-driven. `sk_live_*` swapped in for `sk_test_*` is invisible to the code.
- **Worker code** (the EC2 / Fargate worker from M1) — never talks to Stripe. Worker reads `jobs` + writes transcripts; payment state is the Product Site's concern. No worker change needed.
- **AWS Secrets Manager** — secrets there are `openai-api-key` + Supabase URL/keys (from M1 prereq §2.3). Stripe keys live on Vercel, not on AWS, because only the Vercel-hosted Next.js needs them.
- **Supabase migrations except §5** — schema is identical between sandbox and live operation. Only `credit_products.stripe_price_id` values change.
- **`STRIPE_WEBHOOK_SECRET` variable name** — same name, new value. Renaming would force code changes for no benefit.
- **Custom-domain URL** — same `ai-video-speed-reader.<your-domain>.com/api/stripe/webhook`. The URL is stable across sandbox + live; only which Stripe mode posts to it changes.

## Cost reminder

Activation is free. Going live introduces these new ongoing costs you didn't have in sandbox:

- **Stripe per-transaction fee** — depends on country (US: 2.9% + \$0.30 per successful card charge as of 2026). Visible in dashboard → Settings → Pricing.
- **Stripe Tax** (if enabled) — additional 0.5% on EU/UK/SG charges. Optional, off by default.
- **3DS / SCA** — no extra fee, handled by Stripe Checkout.

No change to your Vercel + Supabase + AWS bill. Stripe is the only new line item.

## When done

> 「Stripe 正式環境開通完成 — `pk_live_*` / `sk_live_*` 已部署到 Vercel Production；live webhook endpoint 指向自有網域；Supabase `credit_products` 三筆價格已切到 live `price_*`；用真實卡實測了一筆 \$5 並退款，三邊（Stripe Events、Vercel logs、Supabase `credit_transactions`）對得起來。可以開始收真錢。」

## Course-position note

This skill is intentionally **outside** the M0–M3 course flow. If you're a course author updating the showcase / questionnaire, frame this as the **v2 roadmap entry** for "real payments" — students complete M0–M3 against sandbox; going live is the post-course exercise this skill walks them through. Don't ship M3-by-default with live keys — it forces every student to collect phone/bank/business info before they can finish the course, which gates the curriculum on personal-document collection.
