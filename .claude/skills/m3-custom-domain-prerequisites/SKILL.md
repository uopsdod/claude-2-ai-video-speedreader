---
name: m3-custom-domain-prerequisites
description: One-time setup the student needs BEFORE starting M3 (Course 2 — custom domain via AWS Route 53). Confirms all 4 Cowork connectors (Supabase, Vercel, AWS, Stripe) are still authenticated, verifies Route 53 access, confirms M2 is green, and walks the student through registering a domain via the AWS Console. No new external accounts — M3 reuses the same AWS account from M1. Use when the student says "M3 環境準備", "setup for M3", or when `m3-custom-domain` / `m3-custom-domain-checklist` detects a missing hosted zone or broken M2.
---

# M3 Prerequisites — Route 53 readiness + domain registration

## What this skill does

Confirms the student is ready to start M3 and walks them through registering the domain via the AWS Console. Unlike M1 and M2 prerequisites, **M3 adds zero new external accounts and zero new Cowork connectors** — Route 53 is part of the same AWS account the student set up in M1 prerequisites. The domain purchase happens in the browser (AWS Console), not via MCP, because it involves entering personal contact info one time and is easier + safer to do directly in the AWS UI.

By the end of this skill the student has:

1. Confirmed all **4 Cowork connectors** (Supabase, Vercel, AWS, Stripe) are still authenticated and returning data.
2. Confirmed M2 is fully green per `m2-stripe-credits-checklist` — M3 builds on the deployed Vercel URL and working Stripe webhook; if M2 is broken, M3 is impossible to verify.
3. **Registered a domain via the AWS Console** (Route 53) and confirmed the ICANN verification email.
4. A Route 53 hosted zone automatically created for the domain, ready for the main M3 skill to add DNS records.

**No new accounts. No new connectors. No new infra.** Just connector verification + domain purchase.

## Required external accounts (cumulative — unchanged from M2)

| # | Service | Used for | Added in |
|---|---|---|---|
| 1 | GitHub | Source control | M0 |
| 2 | Supabase | Auth + Postgres + RLS | M0 |
| 3 | Vercel | Auto-deploy Next.js, **custom domain (M3)** | M0 |
| 4 | OpenAI | Whisper (`whisper-1`) | M1 |
| 5 | AWS | EC2 worker, **Route 53 domain + hosted zone (M3)** | M1 |
| 6 | Stripe (sandbox) | Checkout Sessions + webhook | M2 |

M3 uses AWS Route 53 (same AWS account) and Vercel's custom domain feature (same Vercel account). No new row in this table.

## Required Cowork connectors / MCPs (read this first)

All 4 connectors from M0–M2 must still be authenticated. **M3 uses three of them actively** (Supabase for checklist queries, Vercel for custom domain setup, AWS for Route 53). Stripe is not directly used in M3, but M2 must be green — and the Stripe connector must stay authenticated so the M3 checklist can verify the webhook still fires against the new domain.

| # | Connector / MCP | Added in | Used in M3 for | How to verify |
|---|---|---|---|---|
| 1 | `mcp__supabase_remote__*` | M0 | Checklist queries (schema, job status) | `mcp__supabase_remote__execute_sql`: `SELECT 1` — must return `1` |
| 2 | `mcp__vercel__*` | M0 | `list_projects` + `get_project` to verify the custom domain landed on the right project (Vercel MCP can read project domains but cannot *add* a domain — that step is dashboard-only) | `mcp__vercel__list_projects` — must return the M0 project |
| 3 | `call_aws` / `suggest_aws_commands` (AWS API MCP) | M1 | Route 53: register domain, create hosted zone, manage DNS records | `call_aws route53 list-hosted-zones --max-items 1` — must not error |
| 4 | `mcp__stripe__*` | M2 | Not directly used in M3, but required for M2-green verification | `mcp__stripe__list_prices` — must return `livemode: false` rows or empty list |

**CLI fallback:** if not using Cowork connectors, the equivalent CLI tools are:
- `supabase db execute` or dashboard
- `vercel domains ls` / `vercel domains add`
- `aws route53 ...`
- `stripe prices list`

### What if a connector has expired?

Connectors can expire between milestones (OAuth token rotation, access key deletion, session timeout). If any of the 4 checks above fails:

| Connector | How to re-authenticate |
|---|---|
| Supabase MCP | Cowork: reconnect via Connectors panel. CLI: `supabase login`. |
| Vercel MCP | Cowork: reconnect via Connectors panel. CLI: `vercel login`. |
| AWS API MCP | Check IAM → Users → Security credentials. If the access key was deleted, create a new one and paste into the MCP settings. See [[m1-ai-video-transcript-prerequisites]] §2.0. |
| Stripe MCP | Cowork: reconnect via Connectors panel (re-authorize OAuth — **pick sandbox, not live**). CLI: `stripe login`. See [[m2-stripe-credits-prerequisites]] §2a. |

Fix all 4 before proceeding. Don't start M3 with a broken connector — you'll waste time debugging "is this a DNS issue or is my MCP just dead?"

## Section 1 — Verify all 4 connectors

Run these 4 checks in sequence. Each one should take under 10 seconds. Report pass/fail per connector:

**Check 1 — Supabase:**
```
mcp__supabase_remote__execute_sql: SELECT current_timestamp;
```
Must return a timestamp. If it errors, re-authenticate.

**Check 2 — Vercel:**
Call `mcp__vercel__list_projects` (passing the team ID from `mcp__vercel__list_teams` if needed). Must return the M0 project. If it errors, re-authenticate.

**Check 3 — AWS (Route 53 specifically):**
```
call_aws route53 list-hosted-zones --max-items 1
```
Must return (possibly empty) without an auth error. If `AccessDeniedException`: re-check IAM permissions. If `InvalidClientTokenId`: access key was rotated — create a new one.

**Check 4 — Stripe:**
```
mcp__stripe__list_prices
```
Must return prices with `livemode: false` (or empty list). If auth error, re-connect. If `livemode: true`, the student is on the live account — revoke and re-authorize against sandbox.

**Result matrix:**

| All 4 pass | → proceed to Section 2 |
|---|---|
| Any fail | → fix per the re-auth table above, then re-run the failed check |

## Section 2 — Confirm M2 is green

M3 modifies the Vercel project's domain configuration. If the Vercel deploy is broken, M3 changes can't be verified.

Ask the student: 「M2 checklist 全都通過了嗎？如果不確定，先跑一次 `m2-stripe-credits-checklist`。」

## Section 3 — Route 53 pricing awareness

Before buying, the student should know what they're signing up for:

> 「Route 53 的費用：
> - **Domain 註冊**：`.com` 通常是 $13/year（第一年），自動續約
> - **Hosted zone**：$0.50/month（每年 $6）
> - **DNS 查詢**：$0.40 per million queries — 學習用量基本是 $0
> - 總計：第一年大概 **$19** 上下
>
> Route 53 會在你的 AWS 帳號裡每月扣款。你可以隨時在 Route 53 console 關掉自動續約。」

## Section 4 — Register a domain via AWS Console

The student registers the domain through the AWS Console (not via MCP/CLI). This is a one-time purchase that involves entering personal contact info — easier and safer to do directly in the browser.

> **Pick a personal-namespace apex, not a product name.** M3 will host the AI Video Speed Reader at `ai-video-speed-reader.<your-domain>.com` (a subdomain), so the apex itself can be reused for future projects on the same domain — landing page, blog, second product, etc. Recommended pattern: pick something short and personal like `christine003.com` / `ted007.com` / `<yourname><number>.com`, NOT a product-specific name like `videospeedreader.com` (you'd box yourself in).

Tell the student:

> 「現在去 AWS Console 買你的 domain：
>
> 1. 到 https://console.aws.amazon.com/route53/home#DomainRegistration
> 2. 點 **Register domains**
> 3. 輸入你想要的 domain name（建議像 `christine003.com`、`ted007.com` 這種「個人 namespace」的名字 — M3 會幫你開 `ai-video-speed-reader.<your-domain>.com` 這個 subdomain 給產品用，apex 留著以後做別的）→ 點 **Check**
>    - 如果顯示 **Available** → 點 **Add to cart** → **Continue**
>    - 如果 **Unavailable** → 換一個名字再試
> 4. 填寫聯絡資訊（ICANN 規定）：
>    - **名字、姓氏、Email、電話、地址**
>    - 格式注意：電話用 `+886.912345678`（國碼 + 號碼，無空格）
>    - 勾選 **Privacy Protection**（隱藏 WHOIS 個人資料 — Route 53 預設就會勾）
> 5. 確認 **Auto-Renew: Enabled**（預設）
> 6. 勾 "I have read and agree to the terms" → 點 **Submit**
> 7. 等 1–15 分鐘，Route 53 會處理註冊
> 8. **ICANN 驗證信**：你的 Email 會收到一封驗證信 — **請去收信點確認連結**（15 天內不點的話 domain 會被暫停）
>
> 註冊完成後，把你的 domain name 貼回來給我。」

**ICANN email verification is critical.** If the student doesn't confirm within 15 days, the domain gets suspended. Remind them.

After the student confirms registration, verify via MCP that the domain and hosted zone are live:

```
call_aws route53domains get-domain-detail --domain-name <your-domain>.com
```

Must return without error. Check that `StatusList` does not contain `clientHold` (which means ICANN verification was not completed).

Also confirm Route 53 automatically created a hosted zone:

```
call_aws route53 list-hosted-zones-by-name --dns-name <your-domain>.com --max-items 1
```

Must return a hosted zone with:
- `Name`: `<your-domain>.com.` (note the trailing dot)
- `Type`: `public`
- A hosted zone ID (format `/hostedzone/Z...`)

**Save the hosted zone ID** — the main M3 skill needs it to add the CNAME record for the product subdomain.

Also verify the NS records are in place:

```
call_aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Type=='NS']"
```

Must return 4 NS records (Route 53's name servers). These are auto-configured when Route 53 registers the domain — NS delegation from the TLD registry is already wired up.

> **What if the hosted zone doesn't exist?** Either the registration didn't complete (see `clientHold` check above — student needs to click the ICANN email), or the student registered the domain at another registrar (GoDaddy / Namecheap) and is just moving DNS. For the second case, create one manually:
>
> ```
> call_aws route53 create-hosted-zone --name <your-domain>.com --caller-reference "m3-$(date +%s)"
> ```
>
> Then copy the 4 NS records from the new hosted zone and update them at the registrar. **For Route 53-registered domains, this is automatic — skip the manual step.**

**Verify before moving on:**
1. `get-domain-detail` returns successfully, no `clientHold`.
2. A hosted zone exists for the domain with 4 NS records.
3. The student has clicked the ICANN verification email.
4. The hosted zone ID is saved (for use in the main M3 skill).

## Sanity check before starting `m3-custom-domain`

Before invoking the main M3 skill, confirm all of these:

1. ✅ All 4 Cowork connectors are authenticated and returning data (Section 1).
2. ✅ M2 is green (Section 2).
3. ✅ The student understands the ~$19/year cost (Section 3).
4. ✅ The domain is registered in Route 53 and a hosted zone exists (Section 4).
5. ✅ The ICANN verification email has been confirmed (Section 4).

If any fail, fix here. Don't proceed to the main skill with unresolved issues.

## Related skills

- [[m1-ai-video-transcript-prerequisites]] — where the AWS account + IAM user + MCP were first set up (§2.0–§2.1)
- [[m2-stripe-credits-prerequisites]] — where the Stripe connector was added (§2a)
- [[m2-stripe-credits-checklist]] — must be green before M3 starts
- [[m3-custom-domain]] — the main M3 walkthrough that this skill prepares for
- [[m3-custom-domain-checklist]] — verifies M3 after the main skill is done
- [[aws-best-practice]] — IAM user and SSM rules still apply
