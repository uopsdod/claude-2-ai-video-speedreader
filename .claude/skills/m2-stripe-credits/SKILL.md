---
name: m2-stripe-credits
description: Course 2 Milestone 2 — add a Stripe-backed credits system to the M1 stack. Each signed-up user gets 30 free credits; one minute of video costs one credit; users buy more via three Stripe Checkout tiers ($10/10cr, $30/45cr, $60/90cr) with a clear discount story. Pre-check at submit blocks insufficient-balance jobs; worker deducts on `done`. Use when the student says "啟動 M2", "start M2", "begin M2", "加入 Stripe", "加入信用點數", or any prompt mapping to Course 2 module 2.3.
---

# M2 — Stripe 信用點數 (對應 2.3 加入金流與信用點數)

## What this skill does

Walks the student through Course 2 Milestone 2 end-to-end. By the end the student has:

1. A `credit_products`, `credit_transactions`, and `profiles.credits_balance` schema in Supabase, with an `on_auth_user_created` trigger that grants every new signup **30 free credits**.
2. A `/credits` page showing balance + 3 buy tiers ($10/10cr, $30/45cr, $60/90cr) with visible discount badges, plus a transaction history table.
3. A `/api/credits/checkout` route that creates a Stripe Checkout Session and redirects to `checkout.stripe.com`.
4. A `/api/stripe/webhook` route that verifies the signature, is idempotent on `payment_intent_id`, and credits the user on `checkout.session.completed`.
5. A pre-check inside `/api/jobs` that rejects submission when balance < 1 credit (the soft floor — full duration check happens on the worker).
6. A worker change: on the EC2, after `yt-dlp --print duration` resolves the video length, the worker compares to `profiles.credits_balance`. If insufficient, status → `insufficient_credits` (no Whisper call). If sufficient, run Whisper as before; on `done`, deduct credits and write a `deduction` ledger row.
7. A balance badge in the app header so the user always sees their current credits.

**What M2 is**: a complete, end-to-end payment system. Real Stripe Checkout, real webhook delivery, real ledger writes, real worker-side metering. A user could sign up today, pay with a (sandbox) card, and transcribe a video — every step of that journey is yours after M2. This is what real SaaS billing looks like on day one of a startup.

**M2's v1 scope (everything you're building):**
- 3-tier credit packs, signup bonus, balance display, transaction history
- Stripe Checkout Session + webhook handler with signature verification and DB-level idempotency
- Two-layer credit check (fast at submit, precise on worker)
- Sandbox keys against test card `4242 4242 4242 4242` — the same flow real customers would experience, minus real money

**Future enhancements (the v2 roadmap — pick these up when business signals demand them):**
- **Live Stripe mode** (`sk_live_*` keys + production endpoint webhook secret). Optional in this course — students who want to receive real payouts can go through Stripe activation (phone + bank + business info) as an exercise after the course. M3 keeps the sandbox flow as the green path.
- **Refund tracking** (`charge.refunded` handler). Add when you have customers asking for refunds. The production `milestone-payment-stripe` skill in this repo shows the production-tested implementation when you're ready.
- **Deduct-on-success-only** (M2's model) vs. **escrow-on-submit**. M2 deducts when the job reaches `done`; failed jobs cost the user nothing. Production apps often add escrow ($N held when job submits, released on completion, refunded on failure) once they hit scale and need stricter cost accounting. M2's simpler model is correct for v1 — fewer moving parts, identical end-state for the common case.
- **Rate limiting / multi-job race protection**. Skip in v1; add when usage logs show users actually racing submissions. Premature in M2.
- **Multi-currency** (TWD, EUR, etc.). The schema commits to USD. Adding currency is a one-column migration + a Stripe Price duplicate per currency — straightforward when business needs it.

The v2 list isn't homework you owe the codebase — it's the conventional path real companies travel as they grow. Many production SaaS apps live their whole life on M2-equivalent functionality and never need escrow.

## When to load this skill

Trigger phrases:
- 「啟動 M2」 / "start M2" / "begin M2"
- 「加入 Stripe」 / 「加入信用點數」 / 「加入金流」
- 「我要做 M2」 / "build the credits system"
- Any prompt the student gives that references Course 2 module 2.3

Do NOT load this skill for M0/M1/M3/M4 — they have their own skills.

## Required external accounts

Same five as M1, plus one:

| # | Service | Used for | Added in |
|---|---|---|---|
| 1 | GitHub | Source control | M0 |
| 2 | Supabase | Auth + Postgres + RLS | M0 |
| 3 | Vercel | Auto-deploy Next.js | M0 |
| 4 | OpenAI | Whisper (`whisper-1`) | M1 |
| 5 | AWS | EC2 worker | M1 |
| 6 | **Stripe (sandbox)** | Checkout Sessions + webhook | **M2** |

If 6 is missing, **stop and load `m2-stripe-credits-prerequisites` first**. Do not start this skill without a Stripe sandbox account + at least the three sandbox prices created.

## Execution mode (single path — Claude Code in the repo)

Same as M1 — all code edits go through Claude Code's `Edit`/`Write` directly on the cloned repo, then `git push` triggers Vercel re-deploy. M2 doesn't change the infra at all — just adds files.

The **Component** column names the box in the diagram (above) that the operation modifies, not the tool you wield. The tool is in **How**.

| Component | Operation | How |
|---|---|---|
| **GitHub** (repo) | Edit Next.js code on `main` (Steps 2, 5, 6, 8) | Claude Code `Edit`/`Write` in the cloned repo, then `git push origin main` — same flow as M1. Vercel auto-deploys to production on every push. |
| **GitHub** (repo) | Edit Python worker code (Step 7) | Same: Claude Code `Edit`/`Write` in `worker/worker.py`, push to `main`. **The push alone doesn't deploy the worker — EC2 only gets the new code when Step 7c / 10a run the SSM pull.** Vercel ignores `worker/`. |
| **Supabase** (Postgres) | Apply M2 migration (Step 4) | `mcp__supabase_remote__apply_migration` (Cowork) or `supabase db push` (CLI). Adds `credits_balance` column, `credit_transactions` + `credit_products` tables, signup-bonus trigger. |
| **Supabase** (Postgres) | Inspect tables / verify state | `mcp__supabase_remote__list_tables` / `execute_sql`, or `supabase db execute`, or the Supabase dashboard. |
| **Vercel** (Product Site) | Verify production deploy | `mcp__vercel__*` deployment-list tool, or browser to `<prod-vercel-url>`. |
| **Vercel** (Product Site) | Add env vars to Production (Steps 2a, 9c) | **Vercel MCP does NOT expose env-var management** as of 2026-05 — present-day Vercel MCP only does deploy/list/log. The primary path is the **Vercel dashboard** (Project → Settings → Environment Variables) or `vercel env add <var> production` in CLI mode. If a future MCP version exposes env tools, prefer those. All Stripe env vars go in Production scope, with `sk_test_*` / `pk_test_*` / `whsec_*` (sandbox) values. |
| **Stripe** (sandbox) | Create the three credit-tier prices (Step 3) | `mcp__stripe__*` `create_product` + `create_price` (Cowork) or `stripe products create` + `stripe prices create` (CLI). |
| **Stripe** (sandbox) | Create webhook dashboard endpoint (Step 9b) | **Stripe MCP does NOT expose webhook endpoint management** as of 2026-05 — only Products / Prices / Customers / etc. The primary path is the **Stripe dashboard** → Developers → Webhooks → Add endpoint, in both Cowork and CLI. One endpoint pointed at `<prod-vercel-url>/api/stripe/webhook`. |
| **Stripe** (sandbox) | Test webhooks end-to-end (Step 9) | Trigger via real test-card checkout against the production URL; verify in Stripe dashboard → Events + Vercel runtime logs + Supabase row. |
| **AWS EC2** (Worker) | Pull new worker code + restart service (Steps 7c, 10a) | `call_aws ssm send-command` — **use the shorthand form** `--parameters commands=["cmd1","cmd2"]` with simple commands. The `aws-best-practice` JSON-form (`--parameters '{"commands":["..."]}'`) is reliable with `aws-cli` directly, but the `call_aws` MCP's argv parser strips the JSON quoting; nested-quote and `&&` patterns fail. Keep each command simple — no `&&`, no nested quotes — and run separate commands instead. |
| **Cowork** (Claude Code) | Orchestrator only — never the *target* of a change | Drives the operations above via MCPs / Connectors (`mcp__supabase_*`, `mcp__vercel__*`, `mcp__stripe__*`, `call_aws`) and via `Edit`/`Write` on the cloned repo. If you're using the Claude Code CLI instead of Cowork, the same operations work — just through local `git` / `vercel` / `supabase` / `stripe` CLIs instead. |

**No new SSH paths.** No new AWS infra. M2 is mostly app-code + one webhook + one EC2 pull.

## The shape of the final system

![AI Video Reader architecture, M2 view](assets/ai_video_reader_structure.jpg)

*Left side: the M1 stack you already built (Cowork → GitHub → Vercel-hosted Product Site → Supabase ← AWS EC2 worker running OpenAI Whisper). The two yellow vertical bands are M2's two new gates: **"remaining credit check"** (the `/api/jobs` floor at submit plus the worker-side duration check before Whisper) and **"deduct credit"** (the worker writes a `deduction` row + decrements `profiles.credits_balance` when status flips to `done`). Right side, dashed box: M2's new payment surface — the `/credits` page on the Product Site calls Stripe's **Checkout API**, Stripe sends a **webhook** back, the Product Site writes to `credit_transactions` + `profiles` in Supabase. Nothing on the worker talks to Stripe; nothing on Stripe talks to the worker.*

Text-tree version of the same flows:

```
Sign up (Supabase auth)
  └─ public.handle_new_user trigger
       └─ INSERT INTO profiles (credits_balance = 30)
       └─ INSERT INTO credit_transactions (type='signup_bonus', amount=30)

User opens /credits
  └─ shows balance + 3 tier cards (read from credit_products)
       └─ click Buy
            └─ POST /api/credits/checkout
                 └─ stripe.checkout.sessions.create({ price, metadata: { user_id, product_id, credits }})
                      └─ redirect to checkout.stripe.com
                           └─ user pays with 4242 4242 4242 4242 (sandbox)
                                └─ Stripe sends checkout.session.completed
                                     └─ POST /api/stripe/webhook
                                          ├─ verify signature with STRIPE_WEBHOOK_SECRET
                                          ├─ idempotency on payment_intent_id
                                          ├─ INSERT credit_transactions (type='purchase')
                                          └─ UPDATE profiles.credits_balance += credits

User submits a video at /upload
  └─ POST /api/jobs
       ├─ require auth
       ├─ pre-check: profiles.credits_balance < 1 → reject 402 "insufficient credits"
       └─ INSERT jobs (status='pending')

EC2 worker picks up the job
  └─ yt-dlp --print duration → minutes = ceil(seconds / 60)
       ├─ if minutes > profiles.credits_balance:
       │    └─ UPDATE jobs SET status='insufficient_credits'
       │       INSERT credit_transactions (amount=0, description='video N min, balance M cr')
       │       (no Whisper call — costs the platform nothing)
       └─ else: run Whisper as before
            └─ on done:
                 ├─ INSERT credit_transactions (type='deduction', amount=-minutes, job_id=...)
                 └─ UPDATE profiles.credits_balance -= minutes
```

## Project conventions to match (canonical for this course)

Before writing any code in Steps 2–8, **inspect your M1 repo** and match what's already there. This skill assumes the M1 skill produced a project with these conventions; if yours diverges, use your project's actual values everywhere this skill names them.

| Convention | This course's canonical default | Where to confirm | If yours differs |
|---|---|---|---|
| TypeScript path alias for shared code | `@/*` → `./src/*` (Next.js 16 default with `src/` directory) — so `lib/stripe.ts` lives at **`src/lib/stripe.ts`** | `tsconfig.json` → `compilerOptions.paths` | If your alias maps to `./` (no `src/`), put files at `lib/stripe.ts`, `app/credits/page.tsx`, etc. Substitute throughout. |
| Supabase service-role env var name | **`SUPABASE_SECRET_KEY`** (what M1 prereq sets) | `.env.example` or Vercel Production env list | If M1 used `SUPABASE_SERVICE_ROLE_KEY` instead, use that name everywhere this skill says `SUPABASE_SECRET_KEY`. |
| Header / nav component | Per-page inline header in M1, or a shared component | `grep -r "<header" src/app/` and `src/app/layout.tsx` | If M1 left per-page inline headers, **create `src/components/AppHeader.tsx`** in Step 8 and migrate each page to use it (one refactor for the whole UI). Otherwise patch the existing shared component in place. |
| Middleware shape | Session-refresh only (`auth.getUser()` + matcher-based exclusion); no `isPublic` redirect helper | `src/middleware.ts` | If your M1 used an `isPublic` redirect pattern, add `/api/stripe/webhook` to the `isPublic` list (as in Step 6c). If matcher-based, add it to the `config.matcher` exclusion regex. Either way, the webhook must NOT pass through `auth.getUser()`. |
| Package manager + lockfile | **`bun`** with `bun.lock` (what M0 produces) | `ls bun.lock package-lock.json` | If `npm`/`package-lock.json`: the default Vercel build works. If `bun`/`bun.lock`: see Step 2b — you may need to add `"installCommand": "bun install --no-frozen-lockfile"` to `vercel.json` so Vercel can install the new `stripe` dep on first deploy. |
| `public.profiles` table exists with M0 | M0 only ships the Lovable landing page; **`profiles` is typically NOT yet present** — M1 added it for the user pipeline, but some M1 paths don't | `mcp__supabase_remote__list_tables` — look for `public.profiles` | If `profiles` is missing, Step 4's migration must CREATE it first (template includes `CREATE TABLE IF NOT EXISTS public.profiles` — verify it's there before applying). |

This block exists because real student projects diverge from the skill's reference implementation in lots of small ways after M0/M1. The skill's code snippets below use the canonical defaults; substitute your project's actual values without telling Claude every time.

## Conversational flow

The skill is conversational — drive the student through **9 steps**. Don't dump all steps at once. After each step, **wait for confirmation** before moving on.

> **Before Step 1:** confirm the student has done `m1-ai-video-transcript-checklist` (M1 fully green) and `m2-stripe-credits-prerequisites` (Stripe sandbox account exists, Stripe MCP authenticated against sandbox — OR `stripe login` is good against sandbox). If either is missing, switch to that skill and come back. Sanity checks:
>
> 1. `mcp__supabase_remote__list_tables` — note whether `public.profiles` exists. If it does (likely from M1), Step 4's migration will `ADD COLUMN IF NOT EXISTS credits_balance` to it. If it doesn't, Step 4's migration creates it from scratch. Either way works because the migration is idempotent.
> 2. `mcp__supabase_remote__execute_sql`: `SELECT id, name, credits, price_usd FROM credit_products` — should be empty or non-existent (we'll create it in Step 4).
> 3. M1 worker is `active`: `call_aws ssm send-command --instance-ids "$INSTANCE_ID" --document-name AWS-RunShellScript --parameters commands=["sudo systemctl is-active m1-distributor.service"]` returns `active`.
> 4. Three Stripe sandbox prices exist (Stripe MCP `list_prices` or `stripe prices list`).

---

### Step 1 — Plan the deduct-on-success credit model with the student

Before any code, make sure the student understands what they're building. Tell them verbatim:

> 「M2 加上 Stripe，但設計簡化了：
> 1. **新註冊就送 30 點**，每分鐘影片扣 1 點。
> 2. **兩層檢查，各司其職**：送出影片時先快速擋住「餘額 0 點」這類明顯不能跑的請求；真正比對影片時長 vs 餘額的精準檢查在 worker 上 — yt-dlp 抓到影片時長後比對你的點數，不夠就直接標 `insufficient_credits`，不打 Whisper、不扣錢。這個分工的好處：送出 API 不用下載影片（保持低延遲），worker 有完整資訊做正確判斷。
> 3. **扣款只在 status='done' 時發生**，失敗的工作不扣錢。
> 4. **不防刷單** — 同時送 100 個工作會繞過 pre-check，這是 v1 已知的取捨。
> 5. **三檔價格**：$10 / 10 點、$30 / 45 點（50% 加成）、$60 / 90 點（50% 加成，但一次買更多）。
> 6. **Stripe 用 sandbox** — 信用卡用 `4242 4242 4242 4242`，整套流程跟真實顧客體驗一模一樣，只差在沒走真實金流。整個課程（包括 M3 接自有網域）都會跑在 sandbox；想真的收錢是課後可選的延伸路徑（需要手機 + 銀行帳號 + 公司資料，超出本課程範圍）。」

If the student pushes back on any of these (especially "I want escrow"), point them at the production `milestone-payment-stripe` skill — it's a great reference for how escrow looks in production — but **stick to the M2 model in this skill**. M2's design choices match how most production SaaS apps run for years before they ever need escrow; building the simpler version first is the right engineering call, not a compromise.

### Step 2 — Add the Stripe SDK + env vars to Vercel Production

Same `git push origin main` flow as M1 — no feature branch, no preview deploys. Each push you make in this skill goes straight to your production Vercel URL. The reason this is fine: the entire M2 build runs against Stripe **sandbox** keys + test card `4242 4242 4242 4242`, so "production" is exercising real Stripe infrastructure without any real-money risk. This matches how most solo developers actually ship a v1.

#### 2a — Add two `STRIPE_*` env vars to Vercel **Production** scope

Two of the three Stripe env vars can go in now. The third (`STRIPE_WEBHOOK_SECRET`) waits until Step 9, when the Stripe dashboard endpoint exists and gives us a stable `whsec_...`.

> **⚠ Vercel MCP cannot manage env vars (as of 2026-05).** The MCP only exposes deploy/list/log. So the **dashboard is the primary path in both Cowork and CLI modes.** If a future MCP version exposes env tools, prefer those.

**Primary path — Vercel dashboard:** open Project → **Settings → Environment Variables**, scope = **Production**, add the two below.

**CLI alternative (if you have `vercel` installed locally):**
```bash
vercel env add STRIPE_SECRET_KEY production
# paste sk_test_...
vercel env add NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY production
# paste pk_test_...
```

Either way, the values you're adding are:
- `STRIPE_SECRET_KEY` = `sk_test_...` (from prereq §1)
- `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` = `pk_test_...`

Leave `STRIPE_WEBHOOK_SECRET` for Step 9 — the value doesn't exist yet.

> **Why Production scope (with sandbox `sk_test_*` keys)?** Vercel scopes (`Production` / `Preview` / `Development`) are about *which deploys* get the env var, not about which Stripe mode the keys belong to. The key value itself (`sk_test_*` vs `sk_live_*`) is what decides sandbox vs live. So putting `sk_test_*` in Production scope means "my main-branch deploy uses Stripe sandbox" — which is exactly what M2 wants. Live keys never enter this course (see prereq §1).

> **Why no `.env.local`?** This skill avoids local-shell setup entirely. The student never runs `npm run dev` or `stripe listen`; all tests happen against the deployed Vercel production URL. If you personally want a local dev loop later, you can add `.env.local` outside this skill — but the course path skips it.

#### 2b — Add the Stripe SDK to the repo

In Claude Code, add `stripe` to `package.json` (no shell required — Claude edits `package.json` directly and Vercel runs the install during the next deploy):

```json
{
  "dependencies": {
    "stripe": "^22.0.0"
  }
}
```

(If you prefer the exact version `^22.0.2` that the production app uses, pin to that — but `^22.x` is fine; the apiVersion is what matters.)

> **If your repo uses `bun` (M0 default):** the sandbox / Vercel build environment may not have `bun` installed by default, and `bun install --frozen-lockfile` will fail because `bun.lock` doesn't yet contain the `stripe` entry. Add this to `vercel.json` (create the file if it doesn't exist):
>
> ```json
> {
>   "installCommand": "bun install --no-frozen-lockfile"
> }
> ```
>
> This tells Vercel to install with bun and allow the lockfile to update during install. (Alternative: run `bun install` locally, commit the updated `bun.lock`, then keep Vercel's default — but the `vercel.json` override is more student-proof.) `npm`/`pnpm` projects need no change.

#### 2c — Create the shared Stripe client

File: **`src/lib/stripe.ts`** (or `lib/stripe.ts` if your `tsconfig` doesn't use a `src/` root — see [Project conventions](#project-conventions-to-match-canonical-for-this-course)).

```ts
import Stripe from 'stripe'

if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error('STRIPE_SECRET_KEY is required')
}

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: '2026-03-25.dahlia',
})
```

> **Why pin the apiVersion?** Stripe rolls API versions and breaking changes ship behind that pin. Two months from now your `package.json` will float a newer SDK, and without a pinned version you'd get a surprise breaking change. Pin and forget.

You can commit local progress as you go through Steps 2–8, but **don't push yet** — the first push lands in Step 9a once the schema, UI, API routes, and worker change are all in place, so Vercel's first M2 deploy boots with the full surface ready. If you prefer per-step commits to one giant commit, that's fine — just keep them all local until §9a.

### Step 3 — Create the three credit-tier prices in Stripe sandbox

M2 ships with three credit tiers. Default the rest of M2 assumes:

| Tier | Price | Credits | Effective $/credit | Discount vs baseline |
|---|---|---|---|---|
| Baseline | $10 | 10 | $1.00 | — |
| Mid | $30 | 45 | $0.667 | +50% |
| Big | $60 | 90 | $0.667 | +50% (more upfront) |

If you want sharper tiering, the `/credits` page (Step 5) reads tier info from `credit_products` in Supabase, not from Stripe directly — so as long as the price IDs are linked correctly, you can re-tier later by writing a one-line SQL migration without recreating Stripe prices.

#### 3a — Have Claude create them via the Stripe MCP (Cowork — primary path)

The student doesn't run any commands here. They tell Claude verbatim:

> 「幫我在 Stripe sandbox 建這三個 one-time 價格（USD）：
> - 一個叫 "10 Credits"，售價 \$10（unit_amount = 1000）
> - 一個叫 "45 Credits"，售價 \$30（unit_amount = 3000）
> - 一個叫 "90 Credits"，售價 \$60（unit_amount = 6000）
>
> 三個都要 active = true、currency = usd、type = one_time、`recurring = null`。建完把三個 `price_id` 列給我，我要貼進 Supabase migration。」

You (Claude) then run the MCP calls. The Stripe MCP typically exposes `create_product` + `create_price` separately, or a combined `create_price_with_product` — pick whichever the installed version provides. For each tier:

1. Create the product (`name = "10 Credits"`, etc.).
2. Create a price attached to that product: `unit_amount = <cents>`, `currency = "usd"`, `recurring = null` (i.e. one-time).
3. Capture the returned `price_id` (looks like `price_1XX...XX`).

Report back to the student in this format:

```
Created 3 prices in your Stripe sandbox:
- 10 Credits  ($10) → price_1XYZ...
- 45 Credits  ($30) → price_1ABC...
- 90 Credits  ($60) → price_1DEF...
```

The student saves those three `price_id` values — they go straight into the Supabase migration in Step 4.

> **Hard rules to follow when creating the prices:**
> 1. **All three must be `currency=usd` + `type=one_time`.** A subscription / recurring price won't work with the Checkout flow used in Step 6 (`mode: 'payment'` rejects recurring prices with `400`).
> 2. **`unit_amount` is in cents.** $10.00 = `1000`, $30.00 = `3000`, $60.00 = `6000`. Off-by-100 is the most common Stripe rookie error.
> 3. **Tag with `active=true`.** Inactive prices won't show up in the `/credits` page's product listing (Step 5).
> 4. **Do NOT include a `metadata.credits` value on the Stripe Price.** The credits-per-tier mapping lives in `credit_products` in Supabase, not on the Stripe side. Keeping the credit count out of Stripe metadata means you can re-tier later by writing a one-line SQL migration, without touching Stripe.

#### 3b — Create them via Stripe CLI (fallback path)

If the student is in pure CLI mode without the Stripe MCP authenticated (per `m2-stripe-credits-prerequisites` §2b):

```bash
# Tier 1: $10 / 10 credits
stripe products create --name="10 Credits"
# captures prod_XXX; copy the id
stripe prices create --product=prod_XXX --unit-amount=1000 --currency=usd
# captures price_XXX

# Tier 2: $30 / 45 credits
stripe products create --name="45 Credits"
stripe prices create --product=prod_YYY --unit-amount=3000 --currency=usd

# Tier 3: $60 / 90 credits
stripe products create --name="90 Credits"
stripe prices create --product=prod_ZZZ --unit-amount=6000 --currency=usd
```

Or use the dashboard: **Catalog → Add product** → fill name, set price `One-time`, USD, amount.

> Same four hard rules as §3a apply (one-time only, cents not dollars, `active=true`, no credits metadata on the Stripe side).

#### 3c — Verify

Confirm you see exactly three prices with the right amounts:

- Cowork: ask Claude to list prices via the Stripe MCP. Expect three `active=true`, one-time, USD prices at 1000 / 3000 / 6000 cents.
- CLI: `stripe prices list --active --limit 10`. Same expectation.

If you see more than three (e.g. you re-ran §3a or §3b and got duplicates), archive the extras: `stripe prices update price_XXX --active=false` (CLI), or ask Claude to deactivate the duplicates via MCP. Don't *delete* prices — Stripe doesn't allow deleting Prices that have ever been attached to a Checkout Session.

### Step 4 — Apply the M2 Supabase migration

This is one file under `supabase/migrations/`. The exact filename pattern is `YYYYMMDDHHMMSS_m2_credits_system.sql` (use today's date + a HHMMSS that's newer than every other migration in the directory — sort `ls supabase/migrations/` to confirm).

**Required content** (paraphrase for the student, then write the file):

```sql
-- 0. profiles table (idempotent — M0 ships Lovable-only; M1 may or may not have
-- created profiles; M2 needs it regardless). Safe to run even if M1 already made it.
CREATE TABLE IF NOT EXISTS public.profiles (
  id uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  role text NOT NULL DEFAULT 'user',
  email text,
  created_at timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

DO $$ BEGIN
  CREATE POLICY "Users can view own profile" ON public.profiles
    FOR SELECT USING (id = auth.uid());
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

-- 1. credits_balance on profiles
ALTER TABLE public.profiles
  ADD COLUMN IF NOT EXISTS credits_balance numeric NOT NULL DEFAULT 30;

-- Backfill: existing users get the 30-credit signup bonus retroactively
UPDATE public.profiles SET credits_balance = 30 WHERE credits_balance = 0;

-- 2. Ledger table
CREATE TABLE IF NOT EXISTS public.credit_transactions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  amount numeric NOT NULL,
  type text NOT NULL CHECK (type IN ('purchase', 'deduction', 'signup_bonus', 'admin_grant')),
  description text,
  job_id uuid REFERENCES public.jobs(id),
  stripe_payment_intent_id text,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_credit_transactions_user_id
  ON public.credit_transactions(user_id, created_at DESC);

-- Idempotency: at most one purchase row per payment_intent
CREATE UNIQUE INDEX IF NOT EXISTS uniq_credit_tx_payment_intent
  ON public.credit_transactions(stripe_payment_intent_id)
  WHERE stripe_payment_intent_id IS NOT NULL;

ALTER TABLE public.credit_transactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own transactions" ON public.credit_transactions
  FOR SELECT USING (user_id = auth.uid());

-- 3. Products catalog
CREATE TABLE IF NOT EXISTS public.credit_products (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  credits numeric NOT NULL,
  price_usd numeric NOT NULL,
  stripe_price_id text,
  active boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE public.credit_products ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Authenticated users can view active products" ON public.credit_products
  FOR SELECT TO authenticated USING (active = true);

INSERT INTO public.credit_products (name, credits, price_usd, stripe_price_id) VALUES
  ('10 Credits', 10, 10.00, 'price_REPLACE_ME_10'),
  ('45 Credits', 45, 30.00, 'price_REPLACE_ME_30'),
  ('90 Credits', 90, 60.00, 'price_REPLACE_ME_60');

-- 4. Signup trigger: 30-credit welcome bonus
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  INSERT INTO public.profiles (id, role, credits_balance)
    VALUES (NEW.id, 'user', 30)
    ON CONFLICT (id) DO NOTHING;
  INSERT INTO public.credit_transactions (user_id, amount, type, description)
    VALUES (NEW.id, 30, 'signup_bonus', 'Welcome bonus — 30 free credits');
  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- 5. New job status: insufficient_credits
-- If M1 created jobs.status as a free-text column with no CHECK constraint, this
-- block is a no-op. If M1 (or a Lovable-generated migration) added a CHECK
-- constraint, we must drop + re-add it to allow the new 'insufficient_credits'
-- value. Doing both unconditionally is safe and idempotent.
ALTER TABLE public.jobs DROP CONSTRAINT IF EXISTS jobs_status_check;
ALTER TABLE public.jobs ADD CONSTRAINT jobs_status_check
  CHECK (status IN ('pending', 'downloading', 'transcribing', 'done', 'error', 'insufficient_credits'));
-- ⚠ If your M1 used a different set of status values, list them ALL here, plus
-- 'insufficient_credits'. Check first: SELECT DISTINCT status FROM public.jobs;

-- 6. Backfill signup_bonus for users created before this migration
INSERT INTO public.credit_transactions (user_id, amount, type, description)
SELECT u.id, 30, 'signup_bonus', 'Welcome bonus — 30 free credits (backfilled)'
FROM auth.users u
LEFT JOIN public.credit_transactions ct
  ON ct.user_id = u.id AND ct.type = 'signup_bonus'
WHERE ct.id IS NULL;
```

**Replace the three `price_REPLACE_ME_*` placeholders** with the actual `price_id`s from `m2-stripe-credits-prerequisites` Section 3 BEFORE applying. If you forgot to capture them in the prereq, ask Claude to list prices via the Stripe MCP now, or run `stripe prices list --active` in CLI mode.

Apply via `mcp__supabase_remote__apply_migration` (Cowork) or `supabase db push` (CLI). Confirm with:

```sql
SELECT count(*) FROM credit_products WHERE active = true;
-- expect 3
SELECT count(*) FROM credit_transactions WHERE type = 'signup_bonus';
-- expect >= number of existing auth.users
```

> **Why a UNIQUE INDEX on `stripe_payment_intent_id` rather than a check in the webhook?** The DB-level constraint is the only thing that protects against a true race — two webhook retries landing within the same tick. A SELECT-then-INSERT pattern in the route handler has a TOCTOU window. The `INSERT ... ON CONFLICT` pattern in Step 6 leans on this unique index.

### Step 5 — Build the `/credits` page

File: **`src/app/credits/page.tsx`** (path-alias-resolved as `@/app/credits/page.tsx`). Read in the server component:

```ts
// Server component pattern
import { createServerClient } from '@/lib/supabase/server'   // your M1 helper

export default async function CreditsPage() {
  const supabase = await createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) redirect('/login')

  const { data: profile } = await supabase
    .from('profiles')
    .select('credits_balance')
    .eq('id', user.id)
    .single()

  const { data: products } = await supabase
    .from('credit_products')
    .select('id, name, credits, price_usd, stripe_price_id')
    .eq('active', true)
    .order('price_usd')

  const { data: transactions } = await supabase
    .from('credit_transactions')
    .select('id, amount, type, description, created_at')
    .eq('user_id', user.id)
    .order('created_at', { ascending: false })
    .limit(50)

  // ... render
}
```

**UI requirements** (a clean v1 of the production `app/credits/page.tsx` — balance, tiers, transaction history. The production page adds an escrow indicator, a per-job credit usage join, and refund display rows; those are v2 features that ship when business needs demand them, per the roadmap at the top of this skill):

1. **Balance card** at the top, showing `profile.credits_balance` and the phrase "1 credit = 1 minute of video".
2. **Three tier cards** — for each product, show:
   - `name` (e.g. "45 Credits")
   - `price_usd` formatted as `$30.00`
   - The cost-per-credit (`price_usd / credits`)
   - A **bonus badge** if `price_usd / credits` beats the cheapest tier's ratio. Compute against the smallest-`credits` product: `bonusPct = round((1 - tier.usdPerCredit / baseline.usdPerCredit) * 100)`. Show as a green `+50%` (or whatever the actual percent is) so the discount story is visible.
   - A **Buy** button — `<button onClick={() => buy(product.id)}>`.
3. **Transaction history** — table or list, color-coded:
   - `purchase` → green
   - `signup_bonus` → yellow
   - `deduction` → gray (amount shown as `-N`)
4. **No refund-display row.** Adding refund display lands with the `charge.refunded` webhook handler in v2 — see the roadmap at the top of this skill.

The Buy button POSTs to `/api/credits/checkout`:

```ts
'use client'
async function buy(productId: string) {
  const res = await fetch('/api/credits/checkout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ product_id: productId }),
  })
  if (!res.ok) {
    alert((await res.json()).error ?? 'Checkout failed')
    return
  }
  const { url } = await res.json()
  window.location.href = url
}
```

Track `purchasingId` in `useState` so the user can't double-click and open two checkout sessions. Disable all buttons while one is in flight.

### Step 6 — Build the API routes

#### 6a — `POST /api/credits/checkout`

File: `app/api/credits/checkout/route.ts`.

Responsibilities:

1. Require an authenticated user (`createServerClient()` + `supabase.auth.getUser()` — reject 401 if absent).
2. Read `product_id` from JSON body (reject 400 if missing).
3. Look up the row in `credit_products` — must be `active=true` and have a non-null `stripe_price_id` (reject 400 otherwise).
4. Build success/cancel URLs from the request `Origin` header (falls back to `process.env.NEXT_PUBLIC_SITE_URL`) so the same route works on the production Vercel URL today and (in M3) the custom domain — without code changes when the domain changes.
5. Call `stripe.checkout.sessions.create({ mode: 'payment', payment_method_types: ['card'], line_items: [{ price: stripe_price_id, quantity: 1 }], success_url, cancel_url, metadata: { user_id, product_id, credits: String(credits) }, client_reference_id: user.id })`.
6. Return `{ url: session.url }`.

**Important:** `credits` must be passed as a string in `metadata` (Stripe metadata values are all strings — passing a number works but Stripe coerces to string anyway; explicit `String()` is the honest pattern).

#### 6b — `POST /api/stripe/webhook`

File: `app/api/stripe/webhook/route.ts`. This is the most fragile route in M2 — read each rule before writing:

```ts
import { NextRequest, NextResponse } from 'next/server'
import { headers } from 'next/headers'
import { stripe } from '@/lib/stripe'
import { createServiceClient } from '@/lib/supabase/service'   // service-role client (NEW for M2)

export async function POST(req: NextRequest) {
  const body = await req.text()                  // RAW body — never .json() first
  const sig = (await headers()).get('stripe-signature')
  if (!sig) return new NextResponse('no signature', { status: 400 })

  let event
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch (err) {
    console.error('webhook signature verification failed', err)
    return new NextResponse('invalid signature', { status: 400 })
  }

  if (event.type !== 'checkout.session.completed') {
    return NextResponse.json({ received: true, ignored: event.type })
  }

  const session = event.data.object
  if (session.payment_status !== 'paid') {
    return NextResponse.json({ received: true, unpaid: true })
  }

  const userId = session.metadata?.user_id
  const productId = session.metadata?.product_id
  const credits = Number(session.metadata?.credits)
  const paymentIntentId = typeof session.payment_intent === 'string'
    ? session.payment_intent
    : session.payment_intent?.id
  if (!userId || !productId || !credits || !paymentIntentId) {
    console.error('missing required fields', { userId, productId, credits, paymentIntentId })
    return new NextResponse('missing metadata', { status: 400 })
  }

  const svc = createServiceClient()

  // 1. Insert ledger row — relies on the UNIQUE INDEX from Step 4 for idempotency
  const { error: insertErr } = await svc.from('credit_transactions').insert({
    user_id: userId,
    amount: credits,
    type: 'purchase',
    description: `Purchased ${credits} credits`,
    stripe_payment_intent_id: paymentIntentId,
  })
  if (insertErr) {
    if (insertErr.code === '23505') {  // unique_violation — duplicate webhook delivery
      return NextResponse.json({ received: true, duplicate: true })
    }
    console.error('insert failed', insertErr)
    return new NextResponse('db insert failed', { status: 500 })
  }

  // 2. Increment balance
  const { data: profile } = await svc.from('profiles').select('credits_balance').eq('id', userId).single()
  const newBalance = Number(profile?.credits_balance ?? 0) + credits
  const { error: updateErr } = await svc.from('profiles')
    .update({ credits_balance: newBalance })
    .eq('id', userId)
  if (updateErr) {
    console.error('balance update failed', updateErr)
    return new NextResponse('balance update failed', { status: 500 })
  }

  return NextResponse.json({ received: true, credited: credits })
}
```

> **Five things that bite every student in this route:**
> 1. **`request.text()` not `request.json()`.** Stripe's signature is computed over the exact byte stream. JSON-parsing first mutates whitespace → signature fails verification with a confusing 400.
> 2. **Service-role client, not the cookie-bound one.** Stripe is the caller, not the user — there's no auth cookie. The user-bound client will fail RLS on the `profiles` update. If you don't already have a `createServiceClient()` helper from M1, create one now reading **`SUPABASE_SECRET_KEY`** (this course's canonical name — if your M1 used `SUPABASE_SERVICE_ROLE_KEY`, use that name instead; same value either way).
> 3. **Read the unique-violation as success.** `23505` means "we already processed this payment_intent" — that's idempotency working, return 200 so Stripe stops retrying. Any other error: return 500 so Stripe retries.
> 4. **Two writes, not one transaction.** The insert lands first (the ledger is the source of truth); the balance update is derived. If the second write fails, you have a "missing credit" anomaly that's manually fixable by `UPDATE profiles SET credits_balance = (SELECT SUM(amount) FROM credit_transactions WHERE user_id = ...)`. Either accept this risk or rewrite as a Postgres function called via `rpc()`.
> 5. **No 200 before signature verification.** Stripe interprets non-200 as "retry"; that's what you want for transient errors. But returning 200 before verifying means a bad-signed request gets falsely acknowledged. Order matters.

#### 6c — Middleware exemption — CRITICAL

Without this step, Stripe webhooks may hit `auth.getUser()` and either redirect to `/login` (307) or fail signature verification (because middleware mutates the request). Symptom: Stripe dashboard shows the event fired, payment "succeeded", balance never moves, and there's either no `POST /api/stripe/webhook` line in Vercel logs (307) or a `400 invalid signature` line (auth refresh mangled the body).

**Find your middleware shape first** — there are two common patterns in this course's M1:

**Pattern A — `isPublic` redirect helper (the older M1 template).** Add `/api/stripe/webhook` to the public list:

```ts
const isPublic =
  request.nextUrl.pathname === '/' ||
  request.nextUrl.pathname.startsWith('/login') ||
  request.nextUrl.pathname.startsWith('/auth') ||
  request.nextUrl.pathname.startsWith('/api/stripe/webhook')   // NEW
```

**Pattern B — session-refresh-only with a `config.matcher` (Next.js 16 default; what most current M1 projects look like).** The middleware doesn't redirect — it just runs `supabase.auth.getUser()` on every matched request to refresh the cookie. Add `/api/stripe/webhook` to the matcher's exclusion regex so the middleware doesn't run for it at all:

```ts
export const config = {
  matcher: [
    // Skip Next internals, static files, AND stripe webhook
    '/((?!_next/static|_next/image|favicon.ico|api/stripe/webhook|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
}
```

(If your matcher is a different shape, the principle is the same: ensure `/api/stripe/webhook` is excluded.)

> Every future public webhook (GitHub, Slack, anything Stripe pushes that's NOT `/api/stripe/webhook`) needs the same exemption. The `auth.getUser()` middleware behavior is correct for everything except machine-to-machine callbacks.

### Step 7 — Worker change: duration check + deduct on `done`

This is the M2 worker patch. **One commit, three changes to `worker/worker.py`** (or the equivalent file from M1): (a) claim the job FIRST (flip status to `downloading`), (b) probe duration cheaply with a fallback, (c) deduct credits on `done`.

#### 7a — Claim the job BEFORE any external work

**Order matters.** Past students have crashed the worker on the duration probe (some video URLs return `NA` from `yt-dlp --print duration`, see 7b), which left the job stuck in `pending`. The distributor then re-spawned the same worker, which crashed again — infinite respawn loop, real Whisper-side cost zero but distributor-side cost real.

Fix: the **first DB write of every job iteration** is the status flip from `pending` → `downloading`. That claims the job; if anything below it crashes, the job is in `downloading` and won't be re-spawned by the distributor's pending-job poll. (Stuck-`downloading` recovery is a separate, slower path with a 1-hour timeout, which is the correct cadence for unattended crashes.)

```python
# At the top of the per-job loop body, BEFORE the duration probe / balance read:
supabase.table("jobs").update({"status": "downloading"}).eq("id", job["id"]).execute()
```

#### 7b — Probe duration cheaply, with a fallback for `NA`

`yt-dlp --print duration` returns the literal string `NA` for some sources (e.g. direct CloudFront `.mp4` URLs with no manifest). Doing `float("NA")` raises `ValueError` and crashes the worker — the #1 production bug past students have hit. Treat `NA` as "I don't know yet" and fall back to `ffprobe` after the video has been downloaded.

```python
import math, subprocess
from typing import Optional

def probe_duration_minutes_cheap(video_url: str) -> Optional[int]:
    """Probe duration WITHOUT downloading. Returns ceil(seconds/60), or None if
    the source doesn't expose duration in its manifest (CloudFront direct mp4s,
    some Internet Archive items, etc.). On None, fall back to ffprobe post-download."""
    try:
        out = subprocess.check_output(
            ["yt-dlp", "--print", "duration", "--no-warnings", video_url],
            text=True, timeout=30,
        ).strip()
    except subprocess.SubprocessError:
        return None
    if not out or out.upper() == "NA":
        return None
    try:
        seconds = float(out)
    except ValueError:
        return None
    return max(1, math.ceil(seconds / 60))   # minimum 1 credit even for <60s clips

def probe_duration_minutes_from_file(local_path: str) -> int:
    """After download, ffprobe always works. Use as fallback when 7a returned None."""
    out = subprocess.check_output(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "default=noprint_wrappers=1:nokey=1", local_path],
        text=True,
    ).strip()
    return max(1, math.ceil(float(out) / 60))

# In the job loop, AFTER the status flip in 7a, BEFORE Whisper:
balance = float(
    supabase.table("profiles")
    .select("credits_balance").eq("id", job["user_id"]).single()
    .execute().data["credits_balance"]
)

minutes = probe_duration_minutes_cheap(job["video_source_url"])

if minutes is not None and minutes > balance:
    # Cheap path: we know the duration without downloading. Block here.
    supabase.table("jobs").update({"status": "insufficient_credits"}).eq("id", job["id"]).execute()
    supabase.table("credit_transactions").insert({
        "user_id": job["user_id"],
        "amount": 0,
        "type": "deduction",
        "description": f"Insufficient credits: video is {minutes} min, you have {int(balance)}",
        "job_id": job["id"],
    }).execute()
    return   # skip Whisper, skip download

# Either minutes is known and balance is sufficient, OR minutes is None (probe
# couldn't read it without downloading). In both cases, proceed to download —
# then we'll have the file and can ffprobe for the precise duration.

local_path = download_video(job["video_source_url"])   # your existing M1 download step

if minutes is None:
    minutes = probe_duration_minutes_from_file(local_path)
    if minutes > balance:
        supabase.table("jobs").update({"status": "insufficient_credits"}).eq("id", job["id"]).execute()
        supabase.table("credit_transactions").insert({
            "user_id": job["user_id"],
            "amount": 0,
            "type": "deduction",
            "description": f"Insufficient credits: video is {minutes} min, you have {int(balance)}",
            "job_id": job["id"],
        }).execute()
        return
```

> **Why probe duration on the worker, not at submit?** `/api/jobs` runs on Vercel — invoking yt-dlp there means bundling a ~50 MB binary into a serverless function, plus dealing with cold-start latency. The worker already has yt-dlp installed (M1 prereq §2.6) and is the natural place to do anything that depends on the actual video URL.

> **Why `math.ceil`?** A 61-second clip is still 2 credits — never round down or users can game by submitting just-under-60s clips repeatedly. This is the one place the "no anti-gaming for v1" carve-out doesn't apply, because it's cheaper to write `ceil` than to argue about it.

> **Why the two-stage probe (yt-dlp first, ffprobe fallback)?** YouTube / Vimeo / archive.org with manifests give yt-dlp the duration for free with zero bytes downloaded — that's a free `insufficient_credits` gate that costs us nothing. Direct mp4 URLs (CloudFront, S3, …) don't have manifests; yt-dlp returns `NA`. For those we accept the cost of one download to get a precise ffprobe duration. Slightly less efficient, but: (a) we still don't call Whisper on insufficient credits, which is where the real money is, and (b) treating `NA` as a crash — as the original skill did — is a worse trade.

#### 7c — On successful Whisper completion, deduct credits

After `subtitle_txt_content` is written and `jobs.status` is about to become `done`, deduct:

```python
# Right before/with the final status update to 'done'
supabase.table("credit_transactions").insert({
    "user_id": job["user_id"],
    "amount": -minutes,
    "type": "deduction",
    "description": f"Transcribed {minutes} min video",
    "job_id": job["id"],
}).execute()

# Re-read balance and write back (not RPC — keep M2 simple, accept TOCTOU race)
profile = supabase.table("profiles").select("credits_balance").eq("id", job["user_id"]).single().execute().data
new_balance = max(0.0, float(profile["credits_balance"]) - minutes)
supabase.table("profiles").update({"credits_balance": new_balance}).eq("id", job["user_id"]).execute()

supabase.table("jobs").update({"status": "done"}).eq("id", job["id"]).execute()
```

> **Known race:** if the same user has two workers running simultaneously (only possible if you ran the M1 distributor with concurrency > 1, which the M1 skill doesn't), two `SELECT current; UPDATE current - N` flows can clobber each other. For v1 this is acceptable — the M1 distributor spawns one worker per job sequentially. Don't pull a Postgres `UPDATE ... SET credits_balance = credits_balance - $N RETURNING credits_balance` rewrite into M2; keep the simple read-then-write.

#### 7d — Don't deploy yet

The worker code is now correct on disk but **the EC2 doesn't have it yet** — `git push` happens in Step 9a, and the EC2's `git pull` runs in Step 10a. This is intentional: we want one push that contains everything (schema, UI, routes, worker) so the production cutover is atomic. Don't try to SSM-pull mid-build; the EC2 would pull old code that's missing the matching schema. Move on to Step 8.

### Step 8 — `/api/jobs` pre-check + header balance badge

#### 8a — Patch `app/api/jobs/route.ts`

Right before the INSERT into `jobs`, add the floor-1-credit check:

```ts
// existing: auth + body parse
const { data: profile } = await supabase
  .from('profiles')
  .select('credits_balance')
  .eq('id', user.id)
  .single()

if (!profile || Number(profile.credits_balance) < 1) {
  return NextResponse.json(
    { error: 'insufficient credits — please buy more at /credits' },
    { status: 402 }
  )
}
// then proceed with the existing INSERT
```

This is the **fast pre-check at submit** — it cheaply blocks the obvious "no credits at all" case so users get instant feedback at the form. The precise per-video credit math runs on the worker (Step 7a), where the video duration is actually known. Two-layer checks like this are how production billing systems are built: cheap-and-fast at the API edge, precise-and-authoritative in the worker.

Update the client-side error UX in `app/upload/page.tsx`: if the POST returns 402, show "You don't have enough credits — [Buy credits](/credits)".

#### 8b — Header balance badge

The M1 header may be one of:

- a shared `<AppHeader />` / `<NavBar />` component used by `layout.tsx` — patch it in place;
- a header block in `app/layout.tsx` directly — patch it in place;
- **per-page inline headers** in each page file (Lovable's default output, and what M0/M1 often leaves behind) — create a new shared `src/components/AppHeader.tsx` now and migrate each page to import + render it. This is a small refactor (4–5 pages typically), well worth doing once.

The shared header reads `profiles.credits_balance` for the signed-in user and renders a balance pill:

```
Credits: 30 [Buy more]
```

The `[Buy more]` link points at `/credits`. The badge re-renders on every server navigation, which is sufficient — the success page in Step 9g polls separately to make the post-purchase update feel instant.

### Step 9 — Push to main + test the webhook end-to-end

No localhost, no `stripe listen`, no `npm run dev`, no `.env.local`. We push everything Steps 2–8 created to `main` (Vercel auto-deploys to production), then plug Stripe's dashboard webhook endpoint into the production URL and run a real test-card purchase through it. The whole loop runs against real Stripe + real Vercel + real Supabase + sandbox keys.

#### 9a — Commit + push to main (triggers production deploy)

Same `git push origin main` flow as M1. Confirm `git status` shows the M2 file set (full list in the "Files created or changed" section at the bottom of this skill). Either one big commit or several per-step commits is fine — what matters is that everything Steps 2–8 created is pushed together so production gets a coherent deploy.

```bash
git add -A
git commit -m "m2: stripe credits system (Steps 2–8)"
git push origin main
```

(If you already committed per-step in Steps 2–8, just `git push origin main` now.)

In Cowork the same effect can be driven via Claude with the GitHub MCP — same shape, no shell required.

Wait for the production deploy to go ready. In Cowork: ask Claude to list deployments via the Vercel MCP and watch for `READY` state on the production deployment. CLI: `vercel ls` — the most recent production row should be `Ready`. Usually ~30–90 seconds.

Confirm your production Vercel URL is what you expect (looks like `https://<project>.vercel.app` or whatever your M0/M1 deploy ended up at). Open `<prod-vercel-url>/credits` signed in. The 3 tier cards from Step 5 should render and the balance badge from Step 8 should show `30` (or whatever the signup-bonus state is). If `/credits` 500s, look at Vercel runtime logs (`mcp__vercel__*` log tool or `vercel logs <prod-vercel-url>`) — most likely causes:

- Migration in Step 4 didn't apply to the remote Supabase — re-run `mcp__supabase_remote__apply_migration`.
- `STRIPE_SECRET_KEY` is missing from Vercel **Production** env (Step 2a) — add it via the dashboard, then trigger a redeploy with an empty commit.
- The Stripe SDK isn't installed — Step 2b added it to `package.json` but Vercel's build needs to see that file. Confirm `package.json` has `stripe` under `dependencies` and the diff was actually committed. For bun projects, also confirm `vercel.json` has the `installCommand` override from Step 2b — without it, `bun install --frozen-lockfile` will fail because `bun.lock` hasn't been updated to include `stripe`.

#### 9b — Create the Stripe dashboard webhook endpoint

This is the durable, non-rotating webhook secret — unlike `stripe listen`, which we're not using.

> **⚠ Stripe MCP cannot manage webhook endpoints (as of 2026-05).** It exposes Products / Prices / Customers / Payment Intents / etc., but NOT webhook endpoints. So **the dashboard is the primary path in both Cowork and CLI modes.** If a future Stripe MCP version exposes `create_webhook_endpoint`, prefer that.

**Primary path — Stripe dashboard:**

1. https://dashboard.stripe.com/test/webhooks → **Add endpoint**. (Confirm the top-right account selector still shows your sandbox, not "Live account".)
2. **Endpoint URL:** the `<prod-vercel-url>/api/stripe/webhook` you confirmed in §9a.
3. **Events to send:** `checkout.session.completed` (only — `charge.refunded` is a v2 feature; add it when you start handling refunds).
4. **Add endpoint** → on the resulting endpoint page, click **Reveal** under "Signing secret". Copy the `whsec_...`. That value goes into Vercel Production env in §9c.

**CLI alternative (if you have `stripe` installed and authenticated against sandbox):**

```bash
stripe webhook_endpoints create \
  --url "<prod-vercel-url>/api/stripe/webhook" \
  --enabled-events checkout.session.completed
# Capture the `secret` field from the response — that's your whsec_...
```

> **Stable, not rotating.** The dashboard-endpoint signing secret is bound to the endpoint URL and survives until you delete the endpoint. Unlike `stripe listen`'s ephemeral CLI secret (which rotates every restart and caused the "payment succeeded, balance didn't update" bug class for past students), this one you set once.

> **If your Vercel URL ever changes** (e.g. you switch the project or rename it), the bound endpoint stops receiving. Just create a new endpoint at the new URL and update `STRIPE_WEBHOOK_SECRET` in Vercel env. M3's custom domain will trigger exactly this — and the same drill works.

#### 9c — Add the secret to Vercel Production env

Same MCP gap as §2a — **Vercel MCP cannot manage env vars**, so the dashboard is the primary path.

**Primary path — Vercel dashboard:** Project → Settings → Environment Variables → Production scope → add `STRIPE_WEBHOOK_SECRET` = `whsec_...` (from §9b).

**CLI alternative:**
```bash
vercel env add STRIPE_WEBHOOK_SECRET production
# paste whsec_...
```

**Then redeploy** so the new env applies (Vercel reads env at build/start, not per-request). Use an empty commit + push:

```bash
git commit --allow-empty -m "trigger redeploy with webhook secret"
git push origin main
```

Wait for the new production deploy to go ready (Vercel MCP's deployment-list tool, or `vercel ls`, or the dashboard's Deployments tab — any of these works).

> **Why redeploy after adding env?** Vercel reads env at build/start time, not on every request. An env var added after a deploy doesn't apply until you redeploy. An empty `git commit --allow-empty` is the cheapest way to trigger one.

#### 9d — Make a real test-card purchase

1. Open `<prod-vercel-url>/credits` signed in.
2. Click **Buy** on the 10-credit tier.
3. Stripe Checkout opens. Fill in the form with **fake values** — your real card / name / email don't belong here. The canonical sandbox values are in the reference block immediately below.
4. Submit. You land on `<prod-vercel-url>/credits/success`.

##### Stripe sandbox test card reference (use these in §9d and any time you need to pay)

These are Stripe-published test values. They only work against `sk_test_*` keys (i.e. your sandbox account). They will be rejected if anyone ever flips this app to live mode.

| Field | Value | Notes |
|---|---|---|
| **Card number** | `4242 4242 4242 4242` | Visa, succeeds. The most-used sandbox card on the planet. |
| **Expiry** | any future MM/YY (`12/34` is conventional) | Stripe only validates the format; the actual date is ignored as long as it's future. |
| **CVC** | any 3 digits (`123`) | 4 digits for AmEx-style numbers, but `4242…` is Visa. |
| **ZIP / postal code** | any valid-looking code (`94103`, `10001`, `00000` all work) | Some Stripe accounts ask for ZIP, some don't, depending on Radar settings. |
| **Cardholder name** | anything (`Test User`) | Not validated. Do NOT use your real name — sandbox screenshots end up in lectures. |
| **Email** (if Checkout asks) | anything (`test@example.com`) | Stripe just passes this through; not used for verification. Avoid your real address for the same reason. |
| **Country** (if asked) | any valid country | Has minor effects on which payment methods show, but card-only flow works everywhere. |

If you need to test **edge-case responses**, Stripe publishes a small set of "magic" card numbers that force specific outcomes — useful for the checklist's negative-path checks:

| Card number | What it does | When to use |
|---|---|---|
| `4242 4242 4242 4242` | Succeeds | The default. M2 happy-path test. |
| `4000 0000 0000 0002` | Declined (`card_declined`) | If you want to manually verify the `/credits` page handles a failed checkout gracefully. M2's checklist doesn't require this, but it's two minutes to confirm. |
| `4000 0025 0000 3155` | Requires 3D Secure authentication | Optional. Tests the Strong-Customer-Authentication challenge flow. Stripe Checkout handles the UI; your app just needs to not crash. |
| `4000 0000 0000 9995` | Succeeds at Checkout, then refunded async | Save for v2 (when you add refund handling). Listed here so you don't accidentally pick it while it has no handler. |

Full reference: https://docs.stripe.com/testing — Stripe updates this page when they add cards.

> **Why these exact card numbers?** They share a structural Luhn-check pattern but Stripe's sandbox routes specific numbers to specific simulated responses. They're public knowledge, deliberately published, and zero-risk in sandbox. Don't try to "test with a different card" by inventing your own — you'll get `card_declined: invalid_card_number` because nothing else matches Stripe's test ranges.

#### 9e — Verify the loop fired (three independent checks)

1. **Stripe dashboard → Developers → Events** (sandbox mode): a `checkout.session.completed` event with delivery status "Succeeded" for the endpoint you just created.
2. **Vercel runtime logs** (Cowork: ask Claude for the latest function logs via `mcp__vercel__*`. CLI: `vercel logs <prod-vercel-url> --since=10m`): a `POST /api/stripe/webhook 200 in Xms` line within the last few minutes.
3. **Supabase**: `mcp__supabase_remote__execute_sql`: `SELECT amount, type, stripe_payment_intent_id FROM credit_transactions WHERE user_id = '<your-user-id>' ORDER BY created_at DESC LIMIT 3`. Expect a new `purchase` row (amount = 10) at the top, with a `pi_...` payment_intent_id.

Refresh `<prod-vercel-url>/credits` — balance should show `40` (30 signup + 10 purchase).

If steps 1–2 show "Succeeded" + 200 but step 3 has no new row, look at the Vercel function log body for an error after the 200 — almost always `SUPABASE_SERVICE_ROLE_KEY` missing from Vercel Production env (the webhook returns 200 because the *insert* succeeded but the *balance update* failed, and the route didn't fail-fast). Add the key in Vercel Production env, redeploy, click **Resend** on the dashboard event (§9f).

If step 1 shows "Succeeded" but step 2 has no `POST /api/stripe/webhook` log line, the request hit a 307 redirect — middleware exemption (Step 6c) wasn't merged into the branch.

If step 1 shows a non-2xx status, click the event in the dashboard for the response body and diagnose from there. Most common: signature verification failed (Vercel Production env's `STRIPE_WEBHOOK_SECRET` doesn't match the endpoint's actual secret — re-copy from §9b).

#### 9f — Resend a webhook (if needed)

If a flow ended in a weird state and you want to replay:

**Cowork:** ask Claude to resend the event via the Stripe MCP, given the event ID (`evt_1XX...` from §9e step 1).
**CLI:** `stripe events resend evt_1XX...`
**Dashboard:** open the event, click **Send test webhook** or **Resend** in the top right.

The idempotency check (UNIQUE INDEX on `stripe_payment_intent_id` from Step 4) means resending is safe — the second delivery is recognized as a duplicate and returns 200 without re-crediting.

#### 9g — Build the `/credits/success` page (if not already done in Step 5)

File: `app/credits/success/page.tsx`. Critically: **this page does NOT credit.** The webhook is the only thing that credits. The page exists for UX — to make the post-purchase wait feel instant. Either revalidate the balance query on mount (`router.refresh()` in a client component, or `revalidatePath('/credits')` from a server action), or poll `/api/credits` a few times, then show the new balance.

> The "webhook is source of truth, success page is UX" split matters. If you flip it — "let the success page POST to a credit-grant route" — a user closing the tab between paying and being redirected results in a paid purchase with no credits, and Stripe support tickets become your job. Webhook-only is the production-correct pattern.

### Step 10 — Deploy the worker change to EC2 + end-to-end smoke test

Step 9 already gave you a working purchase flow on production (real Stripe Checkout, real webhook, real `purchase` row in Supabase). Step 10 finishes the loop by getting the **worker M2 patch** onto the EC2 — your `git push` in §9a moved it to GitHub, but the EC2 only sees it after an explicit `git pull` — and then runs one end-to-end transcribe to confirm the deduction half of the system works.

#### 10a — SSM-pull the worker change onto EC2

Three `call_aws ssm send-command` calls. **Use the shorthand `commands=[...]` form**, not the JSON `--parameters '{"commands":[...]}'` form — the `call_aws` MCP's argv parser strips JSON quoting, so nested-quote and `&&` patterns fail. Keep each command simple (no `&&`, no nested quotes) and run them as separate calls.

```
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunShellScript \
  --parameters commands=["cd /home/ubuntu/app && sudo -u ubuntu git pull"]
```

Wait for that to complete (or check `aws ssm list-command-invocations`), then:

```
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunShellScript \
  --parameters commands=["sudo systemctl restart m1-distributor.service"]
```

Then read the log tail to confirm the new code is loaded:

```
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunShellScript \
  --parameters commands=["sudo tail -30 /var/log/m1-distributor.log"]
```

Expect a fresh poll loop with the new "duration probe" / "credit check" log lines you added in Step 7.

> **CLI-direct alternative:** if you're running `aws-cli` locally (not through the `call_aws` MCP), the JSON form `--parameters '{"commands":["..."]}'` works and is what [[aws-best-practice]] recommends. The shorthand here is specifically for the MCP wrapper.

#### 10b — End-to-end transcribe smoke test

Step 9 proved the **purchase** half. Now prove the **deduct** half:

1. Open `<prod-vercel-url>/upload` signed in. Your balance from Step 9 should be `40` (30 signup + 10 purchase).
2. Submit a 30-second test video (Internet Archive or a direct `.mp4` URL — not YouTube; see [[whisper-best-practice]] Rule 3). 1 minute, rounded up, = 1 credit.
3. Wait 1–3 minutes. The job walks `pending → downloading → transcribe → done`.
4. Confirm:
   - **Supabase** `credit_transactions` has a new `deduction` row, `amount = -1`, `job_id` matching the job.
   - **Supabase** `profiles.credits_balance` for your user is now `39` (40 - 1).
   - `<prod-vercel-url>/credits` shows balance `39` and a new "Transcribed 1 min video" entry in the transaction history.

That's the full loop. Real Stripe sandbox + real Vercel production + real Supabase + real EC2 worker — every box on the architecture diagram doing its job. M2 is shippable.

### Step 11 — Hand off to `m2-stripe-credits-checklist`

Tell the student verbatim:

> 「M2 主流程跑完了！現在請呼叫 `驗收 M2`（或 `check M2`）讓我幫你逐項驗證 — 包括三個 tier 是否正確顯示、webhook 是否真的扣對錢、扣 credit 邏輯有沒有跑。」

Then exit this skill — the checklist takes over.

## Files created or changed by this work

Paths below assume the canonical `@/*` → `./src/*` setup. If your project's path alias maps to root (`./`), drop the `src/` prefix.

- `package.json`, `bun.lock` / `package-lock.json` — added `stripe`
- `vercel.json` — added (if bun project; sets `installCommand` to `bun install --no-frozen-lockfile`)
- `src/lib/stripe.ts` — new shared Stripe client
- `src/lib/supabase/service.ts` — service-role client helper (reads `SUPABASE_SECRET_KEY` — or your project's equivalent name)
- `src/app/credits/page.tsx` — new (balance + tier cards + transaction history)
- `src/app/credits/success/page.tsx` — new (UX-only post-purchase landing)
- `src/components/buy-button.tsx` — new client component (POSTs to `/api/credits/checkout`); optional, can live inline in the page
- `src/components/AppHeader.tsx` — new shared header (if M1 used per-page inline headers); patches to existing header otherwise
- `src/app/api/credits/checkout/route.ts` — new (Checkout Session creator)
- `src/app/api/stripe/webhook/route.ts` — new (signature verify + idempotent credit)
- `src/app/api/jobs/route.ts` — patched (pre-check rejects 0-balance submits with 402)
- `src/app/upload/page.tsx` — patched (402 error UX with link to `/credits`)
- `src/app/layout.tsx` (or per-page headers) — patched to render `<AppHeader />`
- `src/middleware.ts` — patched (`/api/stripe/webhook` excluded; pattern depends on whether your M1 uses `isPublic` or `config.matcher`)
- `worker/worker.py` — patched (status-flip first, duration probe with NA-fallback, `insufficient_credits` short-circuit, deduct on `done`)
- `supabase/migrations/<timestamp>_m2_credits_system.sql` — new (everything in Step 4, including the `profiles` CREATE-IF-NOT-EXISTS guard and the `jobs.status` CHECK constraint refresh)
- Vercel env (Production scope) — `STRIPE_SECRET_KEY` (`sk_test_*`, Step 2a), `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` (`pk_test_*`, Step 2a), `STRIPE_WEBHOOK_SECRET` (`whsec_*` from §9b's endpoint, added in §9c). Added via Vercel **dashboard** because Vercel MCP doesn't expose env management.
- One Stripe dashboard webhook endpoint — points at `<prod-vercel-url>/api/stripe/webhook`, subscribed to `checkout.session.completed` (Step 9b). Created via Stripe **dashboard** because Stripe MCP doesn't expose webhook endpoint management.

## Debugging cheatsheet

- **Payment succeeds, no credits, no 200 in Vercel runtime log**: webhook isn't reaching the route. Check middleware exemption (Step 6c) and look at the Stripe dashboard → Events → click your event → "Response" tab to see exactly what your handler returned (status code + body). If it shows 307, middleware is redirecting; if 404, the endpoint URL was typo'd or pointed at the wrong Vercel deployment; if no delivery at all, the Stripe endpoint doesn't exist or is paused.
- **`400 webhook signature verification failed`**: `STRIPE_WEBHOOK_SECRET` in Vercel Production env doesn't match the secret of the dashboard endpoint that's actually sending the event. Re-copy the secret from `https://dashboard.stripe.com/test/webhooks/<we_...>` (Reveal button), update `STRIPE_WEBHOOK_SECRET` in Production env, redeploy to pick up the new env.
- **`Missing metadata`**: something changed the Checkout Session creation and dropped `user_id`/`product_id`/`credits`. Grep `app/api/credits/checkout/route.ts`.
- **Double-credit**: the unique index on `stripe_payment_intent_id` didn't apply (Step 4). `SELECT indexname FROM pg_indexes WHERE tablename = 'credit_transactions'` — must include `uniq_credit_tx_payment_intent`.
- **Worker doesn't deduct on `done`**: re-run the SSM deploy (Step 7c). Verify `/var/log/m1-distributor.log` shows the post-deduct INSERT. If it shows but the balance didn't move, look for an RLS-blocked update — the worker must use the service-role key, not the publishable key.
- **Worker stuck on `insufficient_credits`**: that's the new terminal state. The user needs to either buy more credits (and resubmit) or the operator can manually `UPDATE jobs SET status = 'pending'` after granting more credits. M2 deliberately doesn't auto-retry — keep it simple.

## Related skills

- [[m2-stripe-credits-prerequisites]] — Stripe account + CLI/MCP setup; must be done first
- [[m2-stripe-credits-checklist]] — verifies this milestone after completion
- [[m1-ai-video-transcript]] — the milestone this builds on (Vercel + Supabase + EC2 worker)
- [[milestone-payment-stripe]] — the production Stripe integration with escrow + refund tracking; reference this when you decide to ship the v2 enhancements listed at the top of this skill
- [[stripe-best-practices]] — generic Stripe API guidance (Checkout vs PaymentIntents, restricted keys, key handling)
- [[supabase-best-practice]] — migration discipline ([[feedback_migration_files]]) and RLS guidance
- [[aws-best-practice]] — the SSM JSON-form parameter rule that Step 7c relies on
