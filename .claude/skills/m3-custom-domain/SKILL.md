---
name: m3-custom-domain
description: Course 2 Milestone 3 — point an `ai-video-speed-reader.<your-domain>.com` subdomain (Route 53) at the Vercel-deployed product site so users visit a real domain instead of `<repo-name>.vercel.app`. The apex (`<your-domain>.com`) is left untouched — students can use it for other future projects on the same domain. Vercel handles HTTPS automatically. No code changes — M3 is pure DNS + Vercel configuration. Use when the student says "啟動 M3", "start M3", "begin M3", "買 domain", "自訂網域", or any prompt mapping to a custom domain setup milestone.
---

# M3 — Custom Domain (自訂網域)

## What this skill does

Walks the student through Course 2 Milestone 3 end-to-end. By the end the student has:

1. A Route 53 **hosted zone** for their domain (already registered in prerequisites) with NS records propagated to the TLD registry.
2. A CNAME record (and, if Vercel requires ownership verification, an additional `_vercel` TXT record) pointing the **`ai-video-speed-reader.<your-domain>.com`** subdomain at Vercel.
3. The subdomain added to the Vercel project, with Vercel's automatic HTTPS/TLS provisioning complete.
4. The product site accessible at `https://ai-video-speed-reader.<your-domain>.com` — same M2 app, new URL.

**What M3 is**: your SaaS gets a real domain name. Users bookmark `ai-video-speed-reader.<your-domain>.com`, not a `.vercel.app` URL. This is the "looks professional" milestone — one of the first things any paying customer notices.

**Why a subdomain, not the apex?** One Route 53 hosted zone can host many subdomains. By using `ai-video-speed-reader.<your-domain>.com` instead of the apex, the student keeps `<your-domain>.com` itself free for a future landing page, blog, or second product (e.g. `another-saas.<your-domain>.com`), without re-registering anything. This is the same multi-subdomain pattern most real SaaS founders use once they own a domain.

**Domain registration happens in prerequisites, not here.** The student buys `<your-domain>.com` (e.g. `christine003.com`) via the AWS Console (Route 53) in `m3-custom-domain-prerequisites` Section 4. This skill assumes the domain is already registered and a hosted zone exists.

**M3's scope (everything you're building in this skill):**
- Add the `ai-video-speed-reader.<your-domain>.com` subdomain to the Vercel project
- Add a CNAME (and, if Vercel asks for it, a `_vercel` TXT ownership record) in Route 53 pointing the subdomain to Vercel — using the hosted zone created in prerequisites
- Verify HTTPS works end-to-end via live fetch (the authoritative check — Vercel MCP's `get_project` does NOT list custom domains)
- Confirm the existing `.vercel.app` URL still serves (Stripe webhook depends on it)
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
| **Vercel** | Collect the records Vercel requests (Step 1) | Read all rows Vercel shows on the Add Domain confirmation: always a CNAME, sometimes also a `_vercel` TXT ownership record. The student pastes both back to Claude. |
| **AWS Route 53** | Add the CNAME (and TXT, if any) for the subdomain (Step 2) | `call_aws route53 change-resource-record-sets ...` — one change-batch with one or two UPSERT entries. Hosted zone already exists from prereqs Section 4. |
| **AWS Route 53** | Read records back for verification (Step 2) | `call_aws route53 list-resource-record-sets ... --query "..."` — Cowork-friendly substitute for `dig`. |
| **Vercel** | Verify HTTPS via live fetch (Step 3) | `WebFetch` (or `mcp__vercel__web_fetch_vercel_url` if it accepts the host) — the **authoritative** check. `mcp__vercel__get_project` does NOT list custom domains, so don't use it as the gate. |
| **Vercel** | Confirm `.vercel.app` still serves (Step 3) | `WebFetch` against the original `.vercel.app` URL. Stripe webhook depends on this URL. |
| **Supabase** | Update auth redirect URLs (Step 4) | Supabase dashboard → Authentication → URL Configuration (deep-link via `get_project_url` ref). The MCP can't modify auth config. |

## The shape of the final system

![AI Video Reader architecture](assets/ai_video_reader_structure.jpg)

*Same architecture as M2. The only change: users now reach the Vercel-hosted Product Site via `ai-video-speed-reader.<your-domain>.com` instead of `<repo-name>.vercel.app`. Route 53 resolves the subdomain to Vercel's edge network. Vercel auto-provisions the TLS cert. Everything behind the subdomain — Supabase, EC2 worker, Stripe webhook — stays unchanged.*

Text-tree version of M3's DNS flow:

```
Browser types: https://ai-video-speed-reader.<your-domain>.com
  └─ DNS resolver queries Route 53 hosted zone for <your-domain>.com
       └─ CNAME ai-video-speed-reader → <hash>.vercel-dns-NNN.com.
            (per-project target Vercel issues — copy verbatim, do not assume)
       └─ TXT  _vercel              → "vc-domain-verify=..."
            (only present in "Scenario B" — when the apex was attached to
             another Vercel team/project before; see Step 1)
  └─ Vercel receives the request
       └─ matches ai-video-speed-reader.<your-domain>.com to the project
       └─ (one-time) reads _vercel TXT to confirm ownership, if Scenario B
       └─ auto-provisioned TLS cert (Let's Encrypt) for this subdomain
       └─ serves the same Next.js app as before

(apex <your-domain>.com and www.<your-domain>.com → no DNS records, returns NXDOMAIN.
 Students can wire these up later for a landing page or a second product.)
```

## Conversational flow

The skill is conversational — you (Claude Code) drive the student through 4 steps. Don't dump all the steps at once. After each step, **wait for confirmation** before moving to the next.

> **Before Step 1:** confirm the student has done `m3-custom-domain-prerequisites` — including **registering the domain via the AWS Console** in prereqs Section 4 and **saving the hosted zone ID** for use in Step 2 below. If the domain isn't registered or the hosted zone ID isn't on hand, stop and load the prerequisites skill first.

---

### Step 1 — Add the `ai-video-speed-reader.<your-domain>.com` subdomain to Vercel + collect the DNS records Vercel asks for

This step connects the product subdomain to the Vercel project so Vercel knows to serve your app when requests arrive at `ai-video-speed-reader.<your-domain>.com`. The apex (`<your-domain>.com`) and `www` stay untouched — students can use them later for other projects on the same domain.

> **Vercel MCP gaps (as of 2026-05) — confirmed from a real run, important:**
> 1. The Vercel MCP has **no tool to *add* a custom domain** to a project — attachment goes through the Vercel dashboard (or `vercel domains add` CLI).
> 2. `mcp__vercel__get_project` **does NOT list custom domains** in its `domains` / `alias` field — only the auto-generated `*.vercel.app` aliases appear there. **Do not use it as the gating verification** for "is my custom domain attached?". The authoritative checks are the Vercel dashboard status badge and a live HTTPS fetch of the subdomain (Step 3).
>
> So in this step: the student drives the dashboard, you (Claude) capture the records Vercel asks for, and verification is deferred to Step 3's HTTPS fetch.

Tell the student:

> 「現在要把 `ai-video-speed-reader.<your-domain>.com` 這個 subdomain 加到 Vercel project：
>
> **Dashboard 路徑：**
> 1. 到 https://vercel.com/dashboard
> 2. 點你的 project（就是 M0 建的那個）
> 3. **Settings** → **Domains** （直接連結： `https://vercel.com/<team-slug>/<project-slug>/settings/domains`）
> 4. 在輸入框打完整的 subdomain `ai-video-speed-reader.<your-domain>.com`（替換 `<your-domain>` 成你的 domain，例如 `ai-video-speed-reader.christine003.com`）→ 點 **Add**
> 5. **仔細讀 Vercel 顯示的全部訊息** — 不要只看 CNAME。可能是下列其中一種情境：
>
>    **情境 A — 全新 domain（Vercel 從沒看過這個 apex）：**
>    Vercel 只會要你加一個 CNAME，類似：
>    - **Type**: CNAME
>    - **Name**: `ai-video-speed-reader`
>    - **Value**: 一串長得像 `2781f4ac0e405e15.vercel-dns-017.com.` 的 **per-project target**（每個 project 不一樣，**直接複製 Vercel 顯示的那串，不要套用我給的範例值**）
>
>    **情境 B — Vercel 說「This domain is linked to another Vercel account」（domain 在其他 Vercel team / project 用過）：**
>    Vercel 會要你加 **兩個** records 證明你擁有這個 domain：
>    - CNAME（同情境 A 的格式）
>    - 一個 **TXT** ownership record，類似：
>      - **Type**: TXT
>      - **Name**: `_vercel`（注意是固定字串 `_vercel`，不是你的 subdomain）
>      - **Value**: `vc-domain-verify=ai-video-speed-reader.<your-domain>.com,<random-token>`
>
>    這個 TXT 是 Vercel 用來驗證「你真的能控制這個 DNS zone」的 ownership challenge — 驗證完之後可以刪掉，但留著也無害。
>
> 6. **先不要關這個頁面** — 把 Vercel 顯示的「全部 records」（CNAME 一定有；TXT 可能有也可能沒有）原封不動 paste 給我，包括完整的 Name + Value。我會根據你貼的內容組 Route 53 的 change-batch。」

> **Why the ownership TXT is likely, not rare:** the M3 prereqs explicitly recommend a personal-namespace apex (`<yourname>.com`) that gets reused across multiple projects on the same Vercel account or team. Once the apex has ever been attached to a *different* Vercel project / team than the one you're adding it to now, Vercel gates the new attachment behind the `_vercel` TXT check. If the student already added e.g. `subtitle.<your-domain>.com` on a different team, they will hit this. Treat it as the default branch.

> **Why the CNAME target is project-specific, not `cname.vercel-dns.com`:** Vercel's current behavior is to issue a per-project target like `<hash>.vercel-dns-NNN.com.` (the older `cname.vercel-dns.com` shorthand still resolves but Vercel rarely shows it as the recommended value anymore). Always copy the exact value Vercel's Domains page displays — do not assume.

**Why only DNS records, no app code, no A record?** For a subdomain you only need a CNAME pointing at Vercel's edge (plus the TXT if Vercel asks). The apex (`<your-domain>.com`) is the case that needs an A record (DNS doesn't allow CNAME on the apex). Since we're not configuring the apex, we don't need any A record.

**CLI fallback:**
```bash
vercel domains add ai-video-speed-reader.<your-domain>.com
# Follow the prompts — Vercel will print the required records.
# Read ALL of them, not just the CNAME.
```

**Before moving on, you (Claude) should have captured from the student:**
- ✅ The exact CNAME target Vercel showed (a `<hash>.vercel-dns-NNN.com.` value).
- ✅ Whether Vercel also asked for a `_vercel` TXT ownership record, and if so, its exact value.
- ✅ Confirmation that the subdomain appears in the Vercel Domains page (with a "Pending" or "Invalid Configuration" badge for now — that's expected until Step 2 adds the records).

**Do not rely on `mcp__vercel__get_project` to confirm the subdomain is attached.** It will not list custom domains. Trust the dashboard view + (in Step 3) the live HTTPS fetch instead.

---

### Step 2 — Add the CNAME (and `_vercel` TXT, if Vercel asked for one) in Route 53

Wire the subdomain to Vercel by adding the records you collected in Step 1.

Use the hosted zone ID saved from `m3-custom-domain-prerequisites` Section 4. The change-batch shape depends on which scenario Step 1 returned:

**Scenario A — CNAME only (fresh domain, Vercel didn't ask for ownership TXT):**

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
          "ResourceRecords": [{"Value": "<paste the exact CNAME target Vercel showed in Step 1>"}]
        }
      }
    ]
  }'
```

**Scenario B — CNAME + `_vercel` TXT ownership record (Vercel said "linked to another Vercel account"):**

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
          "ResourceRecords": [{"Value": "<paste the exact CNAME target Vercel showed>"}]
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "_vercel.<your-domain>.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [{"Value": "\"<paste the exact TXT value Vercel showed, including the vc-domain-verify=... part>\""}]
        }
      }
    ]
  }'
```

> **TXT quoting rule (Route 53 specific):** Route 53 stores TXT values with **embedded double-quotes**. The raw value Vercel shows is something like `vc-domain-verify=ai-video-speed-reader.<your-domain>.com,9a414e4c759a2678f507`. In the Route 53 change-batch JSON, that becomes `"Value": "\"vc-domain-verify=...\""` (note the escaped inner quotes). If you forget the inner quotes, Route 53 will accept it but DNS resolvers won't return the value correctly and Vercel's ownership check will fail.

> **Always use the exact CNAME target Vercel showed in Step 1** (typically a `<hash>.vercel-dns-NNN.com.` per-project value). Do not paste a memorized `cname.vercel-dns.com` — Vercel's current behavior is to issue a project-specific target, and the documentation page lags reality.

Both records (if you have two) can — and should — go in **one** `change-resource-record-sets` call. That way one `get-change` poll covers both.

The call returns a `ChangeInfo` with `Status: PENDING`. DNS propagation is typically 30–60 seconds for Route 53 (since both registrar and DNS are Route 53, propagation is fast). Check status:

```
call_aws route53 get-change --id <change-id>
```

Wait until `Status` is `INSYNC`.

Tell the student:

> 「DNS records 已經加好（CNAME{% if 有 TXT %} + ownership TXT{% endif %}）。通常 30 秒到 1 分鐘就會生效。接下來 Vercel 會自動偵測 DNS、驗證 ownership（如果有 TXT）、然後發 TLS 憑證 — 整個過程 1–5 分鐘。」

**Verify before moving on (in this order):**

1. **`get-change` returns `INSYNC`** — confirms Route 53 has accepted the change.

2. **Read the records back via the API** to confirm they're stored correctly:

   ```
   call_aws route53 list-resource-record-sets \
     --hosted-zone-id <zone-id> \
     --query "ResourceRecordSets[?Name=='ai-video-speed-reader.<your-domain>.com.' || Name=='_vercel.<your-domain>.com.']"
   ```

   - CNAME row: `Value` should match what Vercel showed in Step 1 verbatim.
   - TXT row (if Scenario B): `Value` should be the quoted `vc-domain-verify=...` string. The Route 53 API will return it *with* the outer quotes — that's correct.

3. **(Optional) `dig` cross-check, if available locally:**
   ```
   dig ai-video-speed-reader.<your-domain>.com CNAME +short
   dig _vercel.<your-domain>.com TXT +short    # only if Scenario B
   ```

The read-back via `list-resource-record-sets` is the Cowork-friendly equivalent of `dig` — it gives you actual confirmation of the stored values, not just "the API didn't error." Use it before deferring anything to the Vercel dashboard.

---

### Step 3 — Verify HTTPS on the subdomain (authoritative) and that the old `.vercel.app` URL still works

Vercel auto-provisions a TLS certificate via Let's Encrypt — typically 1–5 minutes after DNS propagates (or, in Scenario B, after both DNS and the `_vercel` TXT ownership check propagate).

**The authoritative verification is a live HTTPS fetch, NOT `mcp__vercel__get_project`.** `get_project.domains` only lists `*.vercel.app` aliases; it does not show custom domains attached to the project, even after they're fully verified. So we verify by fetching the subdomain over HTTPS and confirming we get the app back.

**Check 1 — Live HTTPS fetch of the new subdomain:**

```
mcp__vercel__web_fetch_vercel_url url=https://ai-video-speed-reader.<your-domain>.com
```

Or, if `web_fetch_vercel_url` rejects the URL because it's not a `.vercel.app` host (it's domain-restricted), fall back to a plain HTTPS request via `WebFetch`:

```
WebFetch url=https://ai-video-speed-reader.<your-domain>.com prompt="Return the page title and the first 200 characters of body HTML."
```

Expected: HTTP 200, the response body contains the Video Speed Reader landing page markup (or whatever the M0/M2 root page renders). If you see a Vercel-branded 404, a TLS error, or a "domain not configured" page, see Troubleshooting below.

**Check 2 — Confirm the `.vercel.app` URL still serves:**

```
WebFetch url=https://<repo-name>.vercel.app prompt="Return HTTP status and the first 200 characters of body."
```

Expected: HTTP 200. The Stripe webhook still points at this URL, so it absolutely must keep working. If this fails, do not call M3 done — the webhook is broken even though the subdomain may be fine.

**Then ask the student to verify in browser:**

> 「我從程式 HTTPS 抓 `https://ai-video-speed-reader.<your-domain>.com` 已經回 200 + 你的 app 內容了。請在瀏覽器再確認一次：
>
> 1. 到 Vercel dashboard → Settings → Domains（[直接連結](https://vercel.com/dashboard)）— `ai-video-speed-reader.<your-domain>.com` 應該顯示 ✅ Valid Configuration（如果還在 ⚠️ 等 2–3 分鐘再刷新；TLS 憑證發行需要時間）
> 2. 開新分頁打開 `https://ai-video-speed-reader.<your-domain>.com`
>    - 確認是你的 Video Speed Reader 頁面
>    - 確認瀏覽器地址欄有 🔒（鎖頭圖示 = HTTPS 生效）
>    - 試一次 sign in — 確認 Supabase auth 在新 subdomain 下還能用（這需要先做完 Step 4；如果 Step 4 還沒做，sign-in 失敗是預期的）
>
> 都 OK 的話跟我說！」

**Troubleshooting:**

| Symptom | Cause | Fix |
|---|---|---|
| Vercel dashboard shows "Invalid Configuration" / "Pending" 5+ minutes after Step 2 INSYNC | CNAME wrong, TXT missing or malformed, or propagation lag at Vercel's checker (not DNS) | (1) Re-read the records via `list-resource-record-sets` (Step 2 verify check) and compare exactly to what Vercel asked for. (2) If Scenario B: confirm the `_vercel` TXT is present and stored with the correct inner double-quotes. (3) If everything looks right, click **Refresh** in the Vercel Domains UI — it sometimes needs a manual nudge. |
| Vercel shows "This domain is linked to another Vercel account" when the student first adds the subdomain | Apex was attached to a different Vercel team/project at some point. **Expected** for any reused personal-namespace apex (see Step 1, Scenario B). | Add the `_vercel` TXT record alongside the CNAME (Step 2, Scenario B). |
| HTTPS works but sign-in fails (redirect error) | Supabase auth redirect URL doesn't include the new subdomain | Go to Supabase dashboard → Authentication → URL Configuration → **Redirect URLs** → add `https://ai-video-speed-reader.<your-domain>.com/**`. See Step 4. |
| Subdomain shows Vercel 404 / "DEPLOYMENT_NOT_FOUND" instead of the app | The subdomain is attached to the wrong Vercel project (a different project under the same account) | Go to the Vercel project's Settings → Domains and confirm the subdomain is listed under the M0/M2 project, not e.g. a previous experiment. Remove + re-add on the correct project if needed. |
| Browser shows "NET::ERR_CERT_COMMON_NAME_INVALID" / "ERR_CERT_AUTHORITY_INVALID" | TLS cert hasn't been provisioned yet | Wait 5 more minutes. Vercel retries automatically. If still broken after 10 minutes, remove + re-add the subdomain in Vercel (the re-add triggers a fresh cert request). |
| Apex `<your-domain>.com` returns NXDOMAIN | **Expected** — M3 doesn't configure the apex. The product URL is the subdomain. | Not a problem. The apex is intentionally unconfigured so the student can use it later. |
| The old `.vercel.app` URL stops returning 200 (404, redirect to subdomain, anything else) | Someone added a Vercel redirect rule that catches the `.vercel.app` host | **Critical** — Stripe webhook depends on `.vercel.app`. Remove the redirect rule, or scope it so POST requests to `/api/stripe/webhook` are excluded. Test with `curl -sX POST https://<repo-name>.vercel.app/api/stripe/webhook -d '{}' \| head` — should return a Stripe-error 400, not a 30x. |

---

### Step 4 — Update Supabase auth redirect URLs

Supabase auth only redirects to URLs listed in **Authentication → URL Configuration → Redirect URLs**. The current list has the `.vercel.app` URL from M0. Add the new subdomain.

First, resolve the Supabase project ref so you can give the student a deep link to the right page:

```
mcp__supabase_remote__get_project_url
```

Returns something like `https://<ref>.supabase.co`. The URL-configuration page for that project is then:

```
https://supabase.com/dashboard/project/<ref>/auth/url-configuration
```

Tell the student (substituting the deep link):

> 「最後一步 — 讓 Supabase auth 認識你的新 subdomain：
>
> 1. 直接打開：`https://supabase.com/dashboard/project/<ref>/auth/url-configuration`（這是你 Supabase project 的 URL Configuration 頁面 — 不用在 sidebar 裡點來點去找）
> 2. **Site URL**: 改成 `https://ai-video-speed-reader.<your-domain>.com`（這是 Supabase auth email 裡的連結會指向的 URL）
> 3. **Redirect URLs**: 加入 `https://ai-video-speed-reader.<your-domain>.com/**`（允許所有子路徑）
>    - **保留** 舊的 `https://<repo-name>.vercel.app/**` — 不要刪掉，因為 Stripe webhook 還指向它，而且開發時也可能用到
> 4. 點右上 **Save**
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
