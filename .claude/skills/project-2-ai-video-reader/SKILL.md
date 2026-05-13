---
name: project-2-ai-video-reader
description: Show the milestone architecture progression for the Claude Code Course 2 (Building a SaaS with Claude Code). Covers M0 (Lovable landing page) → M1 (Vercel + EC2 worker, Whisper-only TXT) → M2 (Stripe) → M3 (domain) → M4 (Lambda + Fargate serverless), mapped to course chapters 2.1–2.5, including required external accounts per milestone. Use when the user asks about the Course 2 architecture, milestone progression, which accounts to register for which chapter, or how the subtitle-manager stack evolved.
---

Show the milestone architecture progression for the Claude Code Course 2 (Building a SaaS with Claude Code).

## Milestone ↔ 課程章節對應

這份檔案是技術架構演進（M0 → M4），對應到課程章節（2.1 → 2.5）如下：

| 課程章節 | Milestone | 這節新增的 external accounts |
|---|---|---|
| **2.1** 快速搭建第一版可登入的 SaaS 入口網站 | **M0** Landing page | GitHub、Lovable、Supabase、Vercel |
| **2.2** 打造 AI 影片摘要核心功能 | **M1** Vercel + EC2 worker (Whisper-only TXT) | OpenAI、AWS（EC2） |
| **2.3** 串接點數制金流，讓使用者用多少付多少 | **M2** Add Stripe payment | Stripe |
| **2.4** 綁定自己的網域，讓產品變成真正「自己的」 | **M3** Domain | Domain registrar（Namecheap / Cloudflare / GoDaddy 任一）；AWS Route 53 用 M1 已開的帳號 |
| **2.5** 營運成本歸零、營收無上限的雲端擴展 | **M4 (Optional)** Fully Serverless Scaling | （沿用 M1 的 AWS 帳號，這節用到 Lambda + ECS Fargate + EventBridge） |

**累計帳號數：** M0 後 4 個 → M1 後 6 個 → M2 後 7 個 → M3 後 7 個（+ domain registrar）→ M4 不變。

**報名前最低要求（M0 開始之前）：** GitHub、Lovable、Supabase、Vercel 這 4 個。其他在課程中循序帶開。

**M1 範圍說明：** M1 學習目標是「跑通 AI 影片摘要 pipeline 的核心」。為了讓學員把注意力放在「web → 排隊 → worker → Whisper → 取回逐字稿」這條主軸，M1 故意把 pipeline 砍到只剩 `pending → downloading → transcribe → done`、輸出只有 TXT、worker 跑在一台 EC2 上（避開 Mac/Windows 的 ffmpeg 環境問題）。SRT/VTT、LLM 後處理（block-combining、typo check、proofread）、人工 review 這些留到後面當作進階主題。

**M1 為什麼直接上 EC2 而不是先在本機：** 一台便宜的 Ubuntu EC2（t3.small）上裝 ffmpeg + python 是兩行 apt — 但同樣的事情在 Mac 跟 Windows 上會卡住一半的學員（Homebrew 沒裝、PATH 沒設、Windows 完全沒有 native ffmpeg）。直接從 EC2 開始，第一次接 AWS 也順便建好給後面 M3（Route 53）跟 M4（Lambda + Fargate）用的帳號，少走一段。

---

## Architecture (Milestone 0) — Landing page
> 對應課程章節 **2.1 — 快速搭建第一版可登入的 SaaS 入口網站**

```
[Lovable] ──一句話 prompt──▶ [v1 landing page + 登入頁]
                                    │
                                    ▼
                            [GitHub repo]
                                    │ auto-import
                                    ▼
                            [Vercel preview URL]
```

**目的：** 用 Lovable 在 30 分鐘內生出一個**看起來像真產品**的入口網站骨架，連線到 Vercel preview URL，先讓自己（跟潛在使用者）「看到產品的樣子」。

**這一階段不寫後端、不接資料庫、不寫 auth 邏輯。** 純粹是把 UI 蓋出來，並建立後續所有開發要用的 GitHub repo + Vercel project。

### 為什麼 UI 在最前面

傳統工程師的順序是：先設計 DB → 寫 backend API → 最後才弄 UI。這順序對個人 SaaS 不適用，理由：

1. **看不到產品的樣子，很難堅持下去。** 寫完三天 backend 但畫面還是空白頁，會懷疑自己。
2. **UI 是行銷物料的源頭。** 你早點生出截圖，就能早點在 FB / 朋友圈先預熱、收 waitlist、驗證有沒有人想要。
3. **UI 設計決定資料模型。** 反過來說：先有 UI 你才知道資料庫該長什麼樣，避免「先設計一套華麗 schema，最後 UI 用不到一半」。

### 工具

- **Lovable** — 一句話 prompt 產出 v1 入口網站（含 hero、features、CTA、login modal 的視覺骨架）
- **GitHub** — Lovable 會自動推到一個 GitHub repo，後續所有 milestone 都接著這個 repo 開發
- **Vercel** — repo 一推上去就自動 deploy，每次 push 都有 preview URL，學員馬上看得到變化

### 學員產出（M0 結束時）

- 一個自己網域的 v1 入口網站（例如 `<your-saas>.vercel.app`）
- 一個可以登入（UI only，還沒接 Supabase auth）的 demo 頁
- 一個 GitHub repo，後面所有 milestone 都從這個 repo 接著開發

### Prerequisites（學員開始 M0 之前要先註冊）

| # | Service | Used for |
|---|---|---|
| 1 | **GitHub** | Source control，Lovable + Vercel 都要連這個 |
| 2 | **Lovable** | 一句話生 v1 UI + 一鍵接 Supabase auth |
| 3 | **Supabase** | 後端 — `auth.users`、之後 milestone 還會用 |
| 4 | **Vercel** | 自動 deploy，提供 preview URL |

> 註：M0 結束時你的網站就要能 sign up / sign in / sign out，所以 Supabase 帳號在 M0 就需要（不能等到 M1）。

### TODO（待補）

- [ ] Lovable 的實際 prompt 模板（你給 Lovable 的那句話長什麼樣）
- [ ] 從 Lovable 推到 GitHub → Vercel 自動 deploy 的步驟截圖
- [ ] M0 結束時學員應該交出來的成果驗收標準

---

## Architecture (Milestone 1) — Vercel + EC2 worker (Whisper-only TXT)
> 對應課程章節 **2.2 — 打造 AI 影片摘要核心功能**

```
[User Browser] ──video URL──▶ [Vercel: Next.js 16 app] ──POST /api/jobs──▶ [Supabase: jobs + job_sessions]
                                                                                         ▲ poll pending every 10s
                                                                                         │
                                                                              [AWS EC2 (t3.small Ubuntu):
                                                                                distributor.py + worker.py
                                                                                (plain Python, no Docker)]
                                                                                         │
                                                                                         ▼
                                                                                 [OpenAI Whisper API]
```

- **Remote Vercel** — Next.js 16 (App Router) web app: landing page, sign-in/up/out, `/upload` form, `/api/jobs` route handler.
- **Remote Supabase** — Postgres + auth + RLS. Two M1 tables: `jobs` (status FSM) + `job_sessions` (TXT content).
- **AWS EC2 (t3.small)** — runs `distributor.py` in tmux. Polls Supabase every 10 s, spawns one `worker.py` Python subprocess per pending job. Plain venv — no Docker.
- **Worker pipeline (M1, intentionally minimal):** `pending → downloading → transcribe → done`. Download via yt-dlp, ffmpeg → 64 kbps mono mp3, split into 10-min chunks, call OpenAI Whisper with `response_format='text'`, concatenate, write to `job_sessions.subtitle_txt_content`.

**Out of scope in M1 (deferred to later milestones):** SRT/VTT outputs, LLM cleanup (block-combining / typo / semantic / proofread), human review (`*_reviewed` columns), error recovery, Stripe credits, custom domain, serverless distributor.

### Prerequisites

#### External accounts (for students to register, ramped per chapter)

7 accounts total. Aligned with course modules so students only register what they need, when they need it.

| # | Service | First needed in | Used for | Notes |
|---|---|---|---|---|
| 1 | **GitHub** | 2.1 入口網站 | Source control + Vercel/Lovable repo linking | Free; required for Vercel + Lovable integrations |
| 2 | **Lovable** | 2.1 入口網站 | Generates the v1 homepage / landing scaffold; converts Vite→Next.js in M1 | Free tier; mind the monthly generation cap before regenerating |
| 3 | **Supabase** | 2.1 入口網站 | Postgres + auth | Free tier: 500MB DB / 1GB storage |
| 4 | **Vercel** | 2.1 入口網站 | Hosts the Next.js app + API routes | Free tier OK for personal projects |
| 5 | **OpenAI** | 2.2 影片摘要核心 (M1) | Whisper (`whisper-1`) transcription | Pay-as-you-go; preload ~\$5; set a soft monthly cap |
| 6 | **AWS** | 2.2 影片摘要核心 (M1) | One Ubuntu EC2 (t3.small) for the worker; Route 53 in M3; Lambda + ECS Fargate + EventBridge in M4 | Credit card required. Same account covers M1, M3, and M4 — no second AWS account needed. ~\$15/mo if EC2 left running; \$0 stopped. |
| 7 | **Stripe** | 2.3 點數制金流 (M2) | Checkout Sessions + webhook for credit purchases | Use Test Mode for the whole module. Taiwan accounts need company/business registration + bank account to leave Test Mode (3–7 business days). Submit application after M3 (domain) so the live URL is ready. |

> **Anthropic / Claude is NOT in this list.** The course's M1 deliberately stops at Whisper TXT output and skips the LLM cleanup pipeline (block-combining, typo check, semantic fix, contextual proofreading). If a future iteration adds those steps as an extension milestone, Anthropic gets added then.

**Module-by-module registration ramp:**

- **2.1 入口網站 (M0)** — register 1–4 (GitHub, Lovable, Supabase, Vercel).
- **2.2 影片摘要核心 (M1)** — add 5–6 (OpenAI, AWS). The AWS account covers M1's EC2 plus M3's Route 53 and M4's Lambda+Fargate, so you only sign up once.
- **2.3 點數制金流 (M2)** — add 7 (Stripe).
- **2.4 綁網域 (M3)** — no new SaaS account; just buy a domain at a registrar (Namecheap / Cloudflare / GoDaddy, ~\$10–15/year). Use the AWS account from M1 for Route 53.
- **2.5 雲端擴展 (M4, optional)** — no new accounts; reuse the AWS account from M1.

**Marketing one-liner:** "報名前只要註冊 4 個帳號（GitHub、Lovable、Supabase、Vercel），其他帳號我會在課程裡帶你一個一個開。"

### Running M1 (high-level)

1. **Web app on Vercel** — Lovable converts M0's Vite SPA to Next.js 16; push to GitHub; Vercel auto-deploys.
2. **Schema** — apply `supabase/migrations/<ts>_m1_jobs_and_sessions.sql` (or `mcp__supabase_remote__apply_migration` in Cowork mode).
3. **`/upload` page + `/api/jobs` route handler** — Lovable prompt + an inline route handler the student copies into `app/api/jobs/route.ts`.
4. **EC2 worker** — SSH in, `git pull`, `tmux new -s worker`, `./venv/bin/python distributor.py`. The distributor polls Supabase + spawns `worker.py` subprocesses.

The full step-by-step lives in the `m1-ai-video-transcript` skill (with prerequisites in `m1-ai-video-transcript-prerequisites` and end-to-end verification in `m1-ai-video-transcript-checklist`).

### Environment variables

**Vercel + `.env.local` (web app):**

| Variable | Description |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Browser-safe publishable key (RLS-gated) |
| `SUPABASE_SERVICE_KEY` | Server-only secret service-role key — never prefix with `NEXT_PUBLIC_` |

**EC2 `worker/.env`:**

| Variable | Description |
|---|---|
| `SUPABASE_URL` | Same as `NEXT_PUBLIC_SUPABASE_URL` |
| `SUPABASE_SERVICE_KEY` | Same as the web app's secret key |
| `OPENAI_API_KEY` | OpenAI API key for Whisper |

`worker/.env` must be in `.gitignore` before the first commit; both keys are catastrophic if leaked.

## Architecture (Milestone 2) — Add Stripe payment
> 對應課程章節 **2.3 — 串接點數制金流，讓使用者用多少付多少**

```
[User Browser] ──buy credits──▶ [Vercel: /api/credits/checkout]
                                         │
                                         ▼
                                [Stripe Checkout (hosted)]
                                         │ paid
                                         ▼
                                [Stripe] ──event──▶ [Vercel: /api/stripe/webhook]
                                                         │ verify signature
                                                         ▼
                                                [Supabase: credit_transactions
                                                          + profiles.credits_balance]
```

- **Checkout session** — `app/api/credits/checkout/route.ts` looks up `credit_products.stripe_price_id`, creates a Stripe Checkout Session in `mode: 'payment'`, and stashes `{user_id, product_id, credits}` in `metadata`.
- **Webhook** — `app/api/stripe/webhook/route.ts` verifies the signature, handles `checkout.session.completed` (credit the user) and `charge.refunded` (record refund, but do not deduct credits already spent).
- **Ledger** — `credit_transactions` is append-only. Each purchase inserts one row with `stripe_payment_intent_id`; refunds update the same row with `refunded_at`, `stripe_refund_id`, `amount_refunded_cents`.
- **Balance** — `profiles.credits_balance` is the running total. Workers deduct from it when a job completes (added in this milestone — M1 itself has no credit gating). Stripe only handles the *inflow* side.

### Step 1 — Model credits in Supabase

Migration: `supabase/migrations/20260325070000_credits_system.sql`.

- `profiles.credits_balance numeric NOT NULL DEFAULT 0` — the live balance.
- `credit_transactions` — append-only ledger (`purchase | deduction | refund | admin_grant | signup_bonus`), carries `stripe_payment_intent_id` and `job_id`.
- `credit_products` — purchasable tiers (`credits`, `price_usd`, `stripe_price_id`).
- Signup trigger grants 3.0 credits and writes a matching `signup_bonus` row.

Refund tracking is added later in `20260421010000_add_refund_tracking_to_credit_transactions.sql` (`refunded_at`, `stripe_refund_id`, `amount_refunded_cents`).

### Step 2 — Create products and prices in Stripe

For each row in `credit_products`, create a Product + one-time Price in the Stripe Dashboard (test mode first), then link them:

```sql
UPDATE public.credit_products
   SET stripe_price_id = 'price_...'
 WHERE credits = 10 AND price_usd = 10.00;
```

The repo tracks these mappings as migrations (`20260420010000_link_credit_products_to_stripe.sql`, `20260420020000_relink_credit_products_to_stripe_live.sql`) so test→live promotion is reproducible.

### Step 3 — Install the SDK and a shared client

```bash
npm install stripe
```

`lib/stripe.ts`:

```ts
import Stripe from 'stripe'

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2026-03-25.dahlia',
})
```

Pin `apiVersion` — it locks the event/object shapes the webhook handler parses.

### Step 4 — Checkout endpoint

`app/api/credits/checkout/route.ts`:

1. Require an authenticated Supabase user.
2. Read `product_id` from the request, load the matching `credit_products` row (must be `active` and have `stripe_price_id`).
3. `stripe.checkout.sessions.create({ mode: 'payment', line_items: [{ price, quantity: 1 }], success_url, cancel_url, client_reference_id: user.id, metadata: { user_id, product_id, credits } })`.
4. Return `{ url }`; the client does `window.location = url`.

Metadata is the contract the webhook will read — keep `user_id`, `product_id`, and `credits` in sync with the DB columns.

### Step 5 — Webhook endpoint

`app/api/stripe/webhook/route.ts`:

1. Read the raw body (`await request.text()` — do **not** `json()`, it breaks the signature).
2. `stripe.webhooks.constructEvent(body, sig, STRIPE_WEBHOOK_SECRET)` to verify.
3. `checkout.session.completed` branch:
   - Ignore unless `payment_status === 'paid'`.
   - **Idempotency:** `SELECT id FROM credit_transactions WHERE stripe_payment_intent_id = ?` — if a row exists, return 200 and stop.
   - Insert `credit_transactions { type: 'purchase', amount: credits, stripe_payment_intent_id }`.
   - `UPDATE profiles SET credits_balance = balance + credits`.
4. `charge.refunded` branch:
   - Find the original purchase by `stripe_payment_intent_id`.
   - Fetch the latest refund via `stripe.refunds.list({ charge, limit: 1 })` (refunds aren't on the default `Charge` payload).
   - If `(stripe_refund_id, amount_refunded_cents)` already matches, return 200 (idempotent).
   - Update the purchase row with `refunded_at`, `stripe_refund_id`, `amount_refunded_cents`.
   - Insert an `app_logs` warning — **do not deduct credits.** Users may have already spent them; refunds are recorded for accounting, not clawed back.
5. Return 200 for every other event type so Stripe doesn't retry.

Bypass the webhook path in `middleware.ts` so Supabase auth doesn't rewrite the request before the signature check.

### Step 6 — Environment variables

Add to `.env.local` (test mode) and to Vercel (live mode):

| Variable | Source |
|---|---|
| `STRIPE_SECRET_KEY` | Dashboard → Developers → API keys (`sk_test_...` / `sk_live_...`) |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Same page (`pk_test_...` / `pk_live_...`) |
| `STRIPE_WEBHOOK_SECRET` | Local: `stripe listen` prints `whsec_...`. Prod: the endpoint's signing secret in the Dashboard. |

### Step 7 — Test locally with Stripe CLI

```bash
stripe login
stripe listen --forward-to http://localhost:3005/api/stripe/webhook
# copy the whsec_... it prints into .env.local as STRIPE_WEBHOOK_SECRET
# in another shell:
stripe trigger checkout.session.completed
stripe trigger charge.refunded
```

Verify in Supabase that `credit_transactions` got a row and `profiles.credits_balance` moved by the expected amount. Replay the same event and confirm the second attempt is a no-op (the idempotency guard).

### Step 8 — Deploy and register the prod webhook

1. `vercel --prod`.
2. Stripe Dashboard → Developers → Webhooks → **Add endpoint** → `https://<domain>/api/stripe/webhook`, subscribe to `checkout.session.completed` and `charge.refunded`.
3. Copy the endpoint's signing secret into Vercel env as `STRIPE_WEBHOOK_SECRET`, then redeploy (env changes don't apply to existing deployments).
4. Switch live keys (`sk_live_...`, `pk_live_...`) and re-run the Step 2 relink migration against live `price_...` IDs.

### Key additions over Milestone 1

| | Milestone 1 | Milestone 2 |
|---|---|---|
| Credits inflow | None — every transcribe is "free" against the OpenAI bill | Self-serve Stripe Checkout |
| Payment verification | N/A | Signed webhook, idempotent on `payment_intent` |
| Refund handling | N/A | Recorded without deducting spent credits |
| Product catalog | N/A | `credit_products` table linked to Stripe prices |


## Architecture (Milestone 3) — Domain
> 對應課程章節 **2.4 — 綁定自己的網域，讓產品變成真正「自己的」**

```
[End user] → www.yourdomain.com → [AWS Route 53 (DNS)] → [Vercel]
                                                              │
                                                              ▼
                                                       已上線的 SaaS
```

**目的：** 把 M0–M2 已經能跑的產品，從 `<your-project>.vercel.app` 換成自己的網域（例如 `yourdomain.com`）。這是學員第一次把產品「正式對外」的時刻。

### 為什麼網域提前到 M3，而不是最後

1. **沒有自己的網域，就沒辦法做行銷。** FB 廣告、SEO、Google Analytics、email 寄件人地址，全部需要正式網域。
2. **Stripe 過審需要正式網域。** 台灣 Stripe 離開 Test Mode 要審核公司資訊，提交時要填產品網址 — `vercel.app` 子網域很容易被退件。
3. **網域好記、看起來像真產品，使用者才會真的回訪。**

### 步驟概要

1. 從 registrar（Namecheap、Cloudflare、GoDaddy 任一）買網域，~$10–15/年
2. AWS Route 53 建一個 hosted zone（或直接用 registrar 的 DNS，二擇一）
3. Vercel project → Settings → Domains → 加入你的網域
4. 跟著 Vercel 顯示的 DNS records，回到 Route 53 / registrar 那邊新增（A record、CNAME 或 nameservers）
5. 等 DNS propagation（通常 5 分鐘 ～ 幾小時），HTTPS 證書 Vercel 會自動處理

### Prerequisites（M3 之前要先有）

- AWS 帳號（這一節用到 Route 53；M1 已經開好了，M4 還會用到 Lambda + Fargate）
- 一個 domain registrar 帳號 + 買好的網域

### 學員產出（M3 結束時）

- 一個掛在 `yourdomain.com` 的正式產品
- HTTPS 自動 ready
- 可以拿這個網址直接去 Stripe 過審 / 跑 FB 廣告 / 投履歷給投資人

---

## Architecture (Milestone 4, Optional) — Fully Serverless Scaling
> 對應課程章節 **2.5 — 營運成本歸零、營收無上限的雲端擴展**

> **這一節是進階優化，不是必修。**
>
> M0–M3 結束時你的產品已經能上線、能收錢、有自己的網域 — 已經是一個會賺錢的 SaaS。M4 是當你流量真的長起來（例如同時跑 100+ 個影片）或想把伺服器成本壓到趨近於 0 的時候才做。
>
> **什麼時候該做 M4：** 月帳單超過 \$50 / 同時跑 3+ 個 job 開始卡頓 / 想睡覺時也賺錢但不想付 always-on EC2 的錢。
>
> **什麼時候不用做：** 還在驗證產品、月活躍使用者 < 100、覺得 AWS Lambda + Fargate 名詞看了就頭痛。M2 結束時的 EC2 / Vercel 部署模型完全可以撐你前 6 個月。

### Sub-step 1: EC2 Distributor + Fargate Workers (transitional)

```
[User Browser] → [Vercel: Next.js app] → [Supabase DB (remote)]
                                                ↑
                          [AWS EC2: distributor.py]   ← from M1
                                ↓ ecs:RunTask
                          [AWS Fargate containers]
                          (1 task per job, isolated compute)
```

- **EC2 distributor** — same `distributor.py` from M1, now spawning Fargate tasks via `ecs:RunTask` instead of local Python subprocesses. The worker code moves into a Docker image pushed to ECR.
- **Fargate workers** — each job runs in its own Fargate task with dedicated CPU/memory. No resource contention.
- **ECR** — Docker image pushed to Elastic Container Registry.

This sub-step solves the M1 scaling limit (single EC2 CPU shared by all concurrent jobs) without yet eliminating the EC2 itself. Useful as a stepping stone if you want to verify Fargate works before going fully serverless.

### Sub-step 2: Lambda Distributor + Fargate Workers (fully serverless)

```
[User Browser] → [Vercel: Next.js app] → [Supabase DB (remote)]
                                                ↑
                          [EventBridge: every 1 min]
                                ↓
                          [AWS Lambda: distributor]
                                ↓ ecs:RunTask
                          [AWS Fargate containers]
                          (1 task per job, isolated compute)
```

- **Lambda distributor** — Scheduled every 1 minute via EventBridge. Stateless — uses `fargate_task_arn` column on `job_sessions` table instead of in-memory tracking.
- **No EC2** — Fully serverless. No servers to maintain.
- **Human review** — Processed inside Lambda (fits within 5-minute timeout).

### Implementation summary (what actually got built)

Two Lambdas, not one — same handler code, different env vars, pointed at different Supabase projects and Fargate clusters:

| Lambda | Supabase polled | Fargate cluster |
|---|---|---|
| `subtitle-distributor-dev` | local dev Supabase (`szijyejqqfljsiuwpjkm`) | `subtitle-workers-dev` |
| `subtitle-distributor-prod` | remote prod Supabase (`zttwyahfmmwgulvoywrh`) | `subtitle-workers-prod` |

Both fire every 1 minute via EventBridge (`subtitle-distributor-{stage}-schedule`) and share a single deps layer `subtitle-distributor-deps`.

#### Steps taken

1. **Schema change** — added `job_sessions.fargate_task_arn TEXT` + partial index `WHERE fargate_task_arn IS NULL`. Applied to both local and remote Supabase. Replaces the in-memory `_spawned_jobs` set the laptop `distributor.py` uses.

2. **Handler** — `worker/lambda_distributor.py`, single file, ~300 lines:
   - **Spawn pass** — select pending + `review_by_human` jobs whose session has `fargate_task_arn IS NULL`, call `ecs.run_task`, write ARN. Fire-and-forget: no waiting for Fargate to finish, the worker self-reports status through the pipeline.
   - **Stuck-job recovery** — jobs whose status hasn't moved in 1h get a **new `job_sessions` row** (session N+1). The abandoned session gets its heavy content columns nulled (`subtitle_txt_content`, `subtitle_srt_content`, `subtitle_vtt_content`, `whisper_original_content`) but keeps `error_message` for audit. Different semantics from laptop `distributor.py`, which resets the same session in place.
   - **Daily storage cleanup** — gated by `settings.last_storage_cleanup_at` (DB-backed timestamp) instead of an in-memory counter, because Lambda is stateless.

3. **Lambda Layer** (`infra/lambda-layers/supabase-deps`) — `supabase-py` + `aws-lambda-powertools` bundled via Docker with `platform: 'linux/amd64'` so wheels match Lambda's x86_64 runtime (important when building on Apple Silicon). Handler itself is <5 KB and stays inline-editable in the Lambda console.

4. **CDK** (`infra/lib/infra-stack.ts`):
   - `makeDistributor(stage, cfg)` helper creates the Lambda + IAM + EventBridge rule per stage.
   - IAM scoped to `ecs:RunTask`, `ecs:DescribeTasks`, `ecs:StopTask` on `*`, plus `iam:PassRole` scoped to `TaskExecutionRole` + `WorkerTaskRole`.
   - `LoggingConfig` with `LogFormat=JSON`, `ApplicationLogLevel=INFO`, `SystemLogLevel=INFO` so our `log.info(...)` calls actually surface in CloudWatch (Lambda's default is WARNING).
   - Env vars wired from `.env.local` (dev) and `.env.remote` (prod).

5. **Logging** — migrated from stdlib `logging` to AWS Lambda Powertools. Every log line auto-includes `service`, `function_name`, `function_arn`, `cold_start`, `function_request_id`, `xray_trace_id`, plus a custom `stage` field. Matches the logging convention used in the `orange-insider-0608` reference project.

6. **EC2 retirement** — the M1 EC2 distributor instance is **stopped** (not terminated). CDK still creates it; removal is a follow-up cleanup commit.

7. **EC2 `distributor.py`** — kept as a manual fallback (the same setup the student built in M1). `launchctl bootout` (or stopping tmux) takes it offline; the `/restart_subtitle_webserver` skill was updated so it no longer starts the laptop/EC2 distributor by default (would race the Dev Lambda on the same Supabase). CLAUDE.md documents the "one distributor per Supabase at a time" rule and the switch-over commands.

#### Files touched

- `supabase/migrations/20260423010000_add_fargate_task_arn_to_job_sessions.sql` (new)
- `worker/lambda_distributor.py` (new)
- `infra/lambda-layers/supabase-deps/requirements.txt` (new)
- `infra/lib/infra-stack.ts` (Lambda + layer + EventBridge block added; EC2 UserData scoped down to keys the distributor actually needs so unrelated `.env.remote` additions stop triggering instance replacement)
- `.claude/commands/restart_subtitle_webserver.md` (renamed from `_and_worker`; defaults to Lambda-first)
- `~/.claude/commands/stop_subtitle_webserver.md` (renamed from `_and_worker`)
- `CLAUDE.md` — "Job Distributors" section

#### Verified end-to-end

Submitted a real job (#71) via `http://localhost:3005`. Dev Lambda tick at 21:04 found it, spawned Fargate task `571380c47b3642a8a68e1a158c157d13` in `subtitle-workers-dev`, wrote the ARN to `job_sessions.fargate_task_arn`. Next tick at 21:05 saw the ARN was set and correctly skipped. Job advanced through `downloading` → `transcribe_chatgpt` under the Fargate container.

### Key improvements over Milestone 1 (the EC2 baseline)

| | Milestone 1 | Milestone 4 |
|---|---|---|
| Worker compute | Single EC2 CPU shared by all jobs | Isolated Fargate tasks (one per job) |
| Scaling | Limited by EC2 instance size | Unlimited concurrent jobs |
| CPU contention | ~100% CPU with 3+ concurrent jobs | None (isolated per task) |
| Distributor | Always-on EC2 (~\$15/mo for t3.small, more if upgraded) | Lambda (~\$1.50/mo at typical course volume) |
| Cost when idle | EC2 running 24/7 unless manually stopped | \$0 (scales to zero) |
| Ops burden | SSH + tmux + manual restart | Fully serverless |

