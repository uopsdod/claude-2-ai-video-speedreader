---
name: m3-custom-domain
description: Course 2 Milestone 3 — point an `ai-video-speed-reader.<your-domain>.com` subdomain (Route 53) at the Vercel-deployed product site so users visit a real domain instead of `<repo-name>.vercel.app`. The apex (`<your-domain>.com`) is left untouched — students can use it for other future projects on the same domain. Vercel handles HTTPS automatically. No code changes — M3 is pure DNS + Vercel configuration. Use when the student says "啟動 M3", "start M3", "begin M3", "買 domain", "自訂網域", or any prompt mapping to a custom domain setup milestone.
---

# M3 — Custom Domain (自訂網域)

## What this skill does

Walks the student through Course 2 Milestone 3 end-to-end. By the end the student has:

1. A Route 53 **hosted zone** for their domain (already registered in prerequisites) with NS records propagated to the TLD registry.
2. A single CNAME record pointing the **`ai-video-speed-reader.<your-domain>.com`** subdomain at Vercel.
3. The subdomain added to the Vercel project, with Vercel's automatic HTTPS/TLS provisioning complete.
4. The product site accessible at `https://ai-video-speed-reader.<your-domain>.com` — same M2 app, new URL.

**What M3 is**: your SaaS gets a real domain name. Users bookmark `ai-video-speed-reader.<your-domain>.com`, not a `.vercel.app` URL. This is the "looks professional" milestone — one of the first things any paying customer notices.

**Why a subdomain, not the apex?** One Route 53 hosted zone can host many subdomains. By using `ai-video-speed-reader.<your-domain>.com` instead of the apex, the student keeps `<your-domain>.com` itself free for a future landing page, blog, or second product (e.g. `another-saas.<your-domain>.com`), without re-registering anything. This is the same multi-subdomain pattern most real SaaS founders use once they own a domain.

**Domain registration happens in prerequisites, not here.** The student buys `<your-domain>.com` (e.g. `christine003.com`) via the AWS Console (Route 53) in `m3-custom-domain-prerequisites` Section 4. This skill assumes the domain is already registered and a hosted zone exists.

**M3's scope (everything you're building in this skill):**
- Add the `ai-video-speed-reader.<your-domain>.com` subdomain to the Vercel project
- Add one CNAME record in Route 53 pointing the subdomain to Vercel (using the hosted zone created in prerequisites)
- Verify HTTPS works end-to-end
- Update Supabase auth redirect URLs

**What M3 does NOT change:**
- **No code changes.** Zero file edits in the repo. M3 is pure infrastructure configuration.
- **No apex / `www` records.** The apex `<your-domain>.com` and `www.<your-domain>.com` stay unconfigured — they'll return DNS-error for now. Students who later want a landing page on the apex can add records in a future milestone or on their own.
- **No Stripe webhook URL update.** The Stripe dashboard webhook endpoint stays pointed at `<repo-name>.vercel.app/api/stripe/webhook`. Vercel serves both the `.vercel.app` URL and the custom subdomain from the same deployment, so the webhook still lands at the same handler. Updating the webhook URL to the custom subdomain is a v2 cosmetic cleanup — functional either way.
- **No email DNS (MX/SPF/DKIM).** M3 only sets up the one CNAME for the product subdomain. Transactional email DNS is a separate exercise.
- **No Supabase custom domain.** Supabase auth callbacks (`<ref>.supabase.co`) stay as-is. Supabase custom domains are a paid Pro feature and not needed for the course.

## When to load this skill

Trigger phrases:
- 「啟動 M3」 / "start M3" / "begin M3"
- 「買 domain」 / 「自訂網域」 / 「設定自己的網域」
- 「我要做 M3」 / "set up custom domain"
- Any prompt the student gives that maps to buying a domain and pointing it at Vercel

Do NOT load this skill for M0/M1/M2/M4 — they have their own skills.

## Required external accounts

Same six as M2 — no new accounts:

| # | Service | Used for | Added in |
|---|---|---|---|
| 1 | GitHub | Source control | M0 |
| 2 | Supabase | Auth + Postgres + RLS | M0 |
| 3 | Vercel | Auto-deploy Next.js, **custom domain (M3)** | M0 |
| 4 | OpenAI | Whisper (`whisper-1`) | M1 |
| 5 | AWS | EC2 worker, **Route 53 domain + hosted zone (M3)** | M1 |
| 6 | Stripe (sandbox) | Checkout Sessions + webhook | M2 |

If the AWS connector / MCP is not authenticated, **stop and load `m3-custom-domain-prerequisites` first**.

## Required Cowork connectors / MCPs

All 4 from M0–M2, verified in `m3-custom-domain-prerequisites` Section 1:

| # | Connector / MCP | Used in M3 for |
|---|---|---|
| 1 | `mcp__supabase_remote__*` | Checklist verification |
| 2 | `mcp__vercel__*` | List deployments, verify deploy health after domain swap |
| 3 | `call_aws` (AWS API MCP) | Route 53: domain registration, hosted zone, DNS records |
| 4 | `mcp__stripe__*` | Not directly used, but M2 must remain green |

If any connector is expired or broken, **stop and run `m3-custom-domain-prerequisites` Section 1 re-auth**.

## Execution mode (single path — Claude Code driving MCPs)

M3 is entirely infrastructure configuration — no code edits, no `git push`. All operations go through MCPs (Cowork) or CLI equivalents.

| Component | Operation | How |
|---|---|---|
| **Vercel** | Add the `ai-video-speed-reader.<your-domain>.com` subdomain to the project (Step 1, dashboard) | Vercel dashboard (Project → Settings → Domains) or `vercel domains add` CLI. **Vercel MCP cannot *add* domains** as of 2026-05 — no `add_domain` tool. |
| **Vercel** | Verify the subdomain is attached (Step 1, post-add) | `mcp__vercel__get_project` returns the project's attached domains. Use to confirm the subdomain landed on the right project. |
| **AWS Route 53** | Add one CNAME record for the subdomain (Step 2) | `call_aws route53 change-resource-record-sets ...` (hosted zone already exists from prereqs Section 4) |
| **Vercel** | Verify HTTPS provisioning (Step 3) | `mcp__vercel__get_project` shows domain verification status; browser test confirms `https://ai-video-speed-reader.<your-domain>.com` loads with a valid TLS cert. |
| **Supabase** | Update auth redirect URLs (Step 4) | Supabase dashboard → Authentication → URL Configuration. The MCP can't modify auth config. |

## The shape of the final system

![AI Video Reader architecture](assets/ai_video_reader_structure.jpg)

*Same architecture as M2. The only change: users now reach the Vercel-hosted Product Site via `ai-video-speed-reader.<your-domain>.com` instead of `<repo-name>.vercel.app`. Route 53 resolves the subdomain to Vercel's edge network. Vercel auto-provisions the TLS cert. Everything behind the subdomain — Supabase, EC2 worker, Stripe webhook — stays unchanged.*

Text-tree version of M3's DNS flow:

```
Browser types: https://ai-video-speed-reader.<your-domain>.com
  └─ DNS resolver queries Route 53 hosted zone for <your-domain>.com
       └─ CNAME ai-video-speed-reader → cname.vercel-dns.com.
  └─ Vercel receives the request
       └─ matches ai-video-speed-reader.<your-domain>.com to the project
       └─ auto-provisioned TLS cert (Let's Encrypt) for this subdomain
       └─ serves the same Next.js app as before

(apex <your-domain>.com and www.<your-domain>.com → no DNS records, returns NXDOMAIN.
 Students can wire these up later for a landing page or a second product.)
```

## Conversational flow

The skill is conversational — you (Claude Code) drive the student through 4 steps. Don't dump all the steps at once. After each step, **wait for confirmation** before moving to the next.

> **Before Step 1:** confirm the student has done `m3-custom-domain-prerequisites` — including **registering the domain via the AWS Console** in prereqs Section 4 and **saving the hosted zone ID** for use in Step 2 below. If the domain isn't registered or the hosted zone ID isn't on hand, stop and load the prerequisites skill first.

---

### Step 1 — Add the `ai-video-speed-reader.<your-domain>.com` subdomain to Vercel

This step connects the product subdomain to the Vercel project so Vercel knows to serve your app when requests arrive at `ai-video-speed-reader.<your-domain>.com`. The apex (`<your-domain>.com`) and `www` stay untouched — students can use them later for other projects on the same domain.

> **Vercel MCP gap (as of 2026-05):** The Vercel MCP exposes `get_project` (which returns the project's attached domains) and `list_projects`, but **no tool to *add* a custom domain to a project** and no tool to retrieve the required DNS records programmatically. **Domain attachment goes through the Vercel dashboard.** Once the student has done that, we use `mcp__vercel__get_project` to verify.

Tell the student:

> 「現在要把 `ai-video-speed-reader.<your-domain>.com` 這個 subdomain 加到 Vercel project：
>
> **Dashboard 路徑：**
> 1. 到 https://vercel.com/dashboard
> 2. 點你的 project（就是 M0 建的那個）
> 3. **Settings** → **Domains**
> 4. 在輸入框打完整的 subdomain `ai-video-speed-reader.<your-domain>.com`（替換 `<your-domain>` 成你的 domain，例如 `ai-video-speed-reader.christine003.com`）→ 點 **Add**
> 5. Vercel 會自動偵測這是 subdomain，會跟你說需要加一個 **CNAME record**：
>    - **Type**: CNAME
>    - **Name**: `ai-video-speed-reader`
>    - **Value**: `cname.vercel-dns.com.`
> 6. **先不要關這個頁面** — 我們下一步要去 Route 53 加這個 CNAME
>
> 如果 Vercel 顯示的 CNAME target 跟 `cname.vercel-dns.com` 不一樣（很少見），跟我說。」

**Why only a CNAME (no A record)?** For a subdomain you only need a CNAME pointing at Vercel's edge — Vercel resolves the actual IP. The apex (`<your-domain>.com`) is the case that needs an A record (DNS doesn't allow CNAME on the apex). Since we're not configuring the apex, we don't need any A record.

**CLI fallback:**
```bash
vercel domains add ai-video-speed-reader.<your-domain>.com
# Follow the prompts — Vercel will print the required CNAME
```

**Verify via MCP before moving on.** Use `mcp__vercel__list_projects` to get the project ID (and team ID if not already known), then call `mcp__vercel__get_project`. The returned project object's `alias` / `domains` field should now contain `ai-video-speed-reader.<your-domain>.com`. It'll likely show a verification status indicating DNS isn't configured yet — that's expected until we add the CNAME in Step 2.

If the subdomain doesn't appear in `get_project`'s response: the student added it to the wrong project, or the Add didn't actually save. Re-check the Vercel dashboard URL — confirm they're on the Settings page of the M0 project (the one with the latest M2 deployment), not a different project.

---

### Step 2 — Add the CNAME record in Route 53

Now wire the subdomain to Vercel by adding the CNAME Vercel requested in Step 1.

Use the hosted zone ID saved from `m3-custom-domain-prerequisites` Section 4. One CNAME record is all we need:

```
call_aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "ai-video-speed-reader.<your-domain>.com",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [{"Value": "cname.vercel-dns.com"}]
        }
      }
    ]
  }'
```

> **Use the CNAME target Vercel gave you in Step 1**, not the hardcoded `cname.vercel-dns.com` above. The target is stable as of 2026-05 but always cross-check against what the Vercel Domains page shows.

The `change-resource-record-sets` call returns a `ChangeInfo` with `Status: PENDING`. DNS propagation is typically 30–60 seconds for Route 53 (since both registrar and DNS are Route 53, propagation is fast). Check status:

```
call_aws route53 get-change --id <change-id>
```

Wait until `Status` is `INSYNC`.

Tell the student:

> 「CNAME record 已經加好。通常 30 秒到 1 分鐘就會生效（因為你的 domain 和 DNS 都在 Route 53，propagation 很快）。接下來等 Vercel 偵測到 DNS 正確，它會自動幫你發 TLS 憑證。」

**Verify before moving on:**
1. `get-change` returns `INSYNC`.
2. `dig ai-video-speed-reader.<your-domain>.com CNAME +short` returns `cname.vercel-dns.com.` (or whatever target Vercel showed in Step 1).

If `dig` is not available (Cowork mode), skip to Step 3 — the Vercel dashboard will show the DNS status.

---

### Step 3 — Verify HTTPS and Vercel domain status

Vercel auto-provisions a TLS certificate via Let's Encrypt — this takes 1–5 minutes after DNS propagates. Verify both via MCP (project state) and via browser (real HTTPS check).

**MCP check first:** call `mcp__vercel__get_project` for the project ID. The returned `alias` / `domains` field should now show `ai-video-speed-reader.<your-domain>.com` as **verified** (no `verification` array indicating outstanding TXT challenges, no `Invalid Configuration` status). If it's still showing pending verification after 5 minutes, return to Step 2 and re-verify the CNAME record.

**Then ask the student to verify in browser:**

> 「Vercel MCP 顯示你的 subdomain 已經 verified。回到 Vercel 的 Domains 頁面確認：
> 1. `ai-video-speed-reader.<your-domain>.com` 應該顯示 ✅ Valid Configuration
> 2. 如果還顯示 ⚠️ 或 ❌，等 2–3 分鐘再刷新 — TLS 憑證發行需要一點時間
>
> 變成 ✅ 之後，開一個新的瀏覽器分頁：
> - 打開 `https://ai-video-speed-reader.<your-domain>.com`
> - 確認是你的 Video Speed Reader 頁面
> - 確認瀏覽器地址欄有 🔒（鎖頭圖示 = HTTPS 生效）
> - 試一次 sign in — 確認 Supabase auth 在新 subdomain 下還能用
>
> 都 OK 的話跟我說！」

**Troubleshooting:**

| Symptom | Cause | Fix |
|---|---|---|
| Vercel shows "Invalid Configuration" after 5+ minutes | CNAME wrong or not propagated yet | Re-check Step 2's record. `dig ai-video-speed-reader.<your-domain>.com CNAME +short` — should return `cname.vercel-dns.com.` If it returns nothing, the `change-resource-record-sets` call may have failed silently — re-run it. |
| HTTPS works but sign-in fails (redirect error) | Supabase auth redirect URL doesn't include the new subdomain | Go to Supabase dashboard → Authentication → URL Configuration → **Redirect URLs** → add `https://ai-video-speed-reader.<your-domain>.com/**`. See Step 4. |
| Subdomain shows Vercel 404 instead of the app | The subdomain isn't attached to the correct Vercel project | Verify via `mcp__vercel__get_project` that the subdomain is on the M0 project, not a different one. |
| Browser shows "NET::ERR_CERT_COMMON_NAME_INVALID" | TLS cert hasn't been provisioned yet | Wait 5 more minutes. Vercel retries automatically. If still broken after 10 minutes, remove + re-add the subdomain in Vercel. |
| Apex `<your-domain>.com` returns NXDOMAIN | **Expected** — M3 doesn't configure the apex. The product URL is the subdomain. | Not a problem. The apex is intentionally unconfigured so the student can use it later. |

---

### Step 4 — Update Supabase auth redirect URLs

Supabase auth only redirects to URLs listed in **Authentication → URL Configuration → Redirect URLs**. The current list has the `.vercel.app` URL from M0. Add the new subdomain.

Tell the student:

> 「最後一步 — 讓 Supabase auth 認識你的新 subdomain：
>
> 1. 到 Supabase dashboard → 你的 project
> 2. **Authentication** → **URL Configuration**
> 3. **Site URL**: 改成 `https://ai-video-speed-reader.<your-domain>.com`（這是 Supabase auth email 裡的連結會指向的 URL）
> 4. **Redirect URLs**: 加入 `https://ai-video-speed-reader.<your-domain>.com/**`（允許所有子路徑）
>    - **保留** 舊的 `https://<repo-name>.vercel.app/**` — 不要刪掉，因為 Stripe webhook 還指向它，而且開發時也可能用到
> 5. 儲存
>
> 測試：登出再登入一次 `https://ai-video-speed-reader.<your-domain>.com` — 應該正常。」

Note: Supabase auth config is managed via the dashboard UI, not SQL. The MCP can't modify it directly — the student must use the dashboard for this step.

**Verify before moving on:** sign out and sign back in on `https://ai-video-speed-reader.<your-domain>.com`. Auth flow completes without errors.

---

## Post-M3 state

After M3, the student's stack looks like this:

| Layer | Before M3 | After M3 |
|---|---|---|
| User-facing URL | `https://<repo-name>.vercel.app` | `https://ai-video-speed-reader.<your-domain>.com` |
| DNS | Vercel-managed `.vercel.app` subdomain | Route 53 hosted zone → CNAME → Vercel |
| TLS/HTTPS | Vercel auto-cert on `.vercel.app` | Vercel auto-cert on `ai-video-speed-reader.<your-domain>.com` |
| Apex `<your-domain>.com` | N/A | **Unconfigured** — returns NXDOMAIN. Reserved for the student's future projects. |
| Stripe webhook | `https://<repo-name>.vercel.app/api/stripe/webhook` | **Unchanged** — still `.vercel.app` (functional) |
| Supabase auth | Redirects to `.vercel.app` | Redirects to `ai-video-speed-reader.<your-domain>.com` (`.vercel.app` kept as fallback) |
| Everything else | Unchanged | Unchanged |

**The `.vercel.app` URL still works.** Vercel serves both URLs from the same deployment. Nothing breaks by adding the custom subdomain — it's purely additive.

## Future enhancements (v2 roadmap)

- **Apex landing page.** Add an A record (or ALIAS/ANAME) for `<your-domain>.com` pointing to a static landing page, a portfolio site, or a marketing site for the product. The hosted zone is already there — it's a one-record addition.
- **More products on the same domain.** Add `another-saas.<your-domain>.com`, `blog.<your-domain>.com`, `<future-product>.<your-domain>.com` — one CNAME each, all sharing the same `$0.50/month` hosted zone.
- **Update Stripe webhook endpoint URL** to `https://ai-video-speed-reader.<your-domain>.com/api/stripe/webhook`. Cosmetic — both URLs hit the same handler. Do it when you want a clean Stripe dashboard.
- **Email DNS (MX/SPF/DKIM)** for sending transactional emails from `@<your-domain>.com`. Needed if you add email notifications (e.g. "your transcript is ready").
- **Supabase custom domain** (`api.<your-domain>.com` proxying to `<ref>.supabase.co`). Requires Supabase Pro plan. Hides the Supabase ref from the user, but functionally identical.
- **DNSSEC** — Route 53 supports it for `.com` domains. Adds signing to DNS responses. Good security hygiene but not required for the course.

## Related skills

- [[m3-custom-domain-prerequisites]] — connector check + domain name selection
- [[m3-custom-domain-checklist]] — verifies M3 after this skill completes
- [[m2-stripe-credits]] — the stack M3 builds on (Stripe webhook stays on `.vercel.app`)
- [[aws-best-practice]] — IAM user rules apply to Route 53 calls
- [[supabase-best-practice]] — Supabase auth URL configuration
