---
name: m0-landing-and-signin-checklist
description: Video Speed Reader Milestone 0 verification — checks every artifact (GitHub repo, Lovable sync, Vercel deploy, landing page contents, Supabase auth) is real and correctly wired. Use when the student says "驗收 M0", "check M0", "M0 done?", or after the `m0-landing-and-signin` skill completes Step 9.
---

# M0 — Landing + Sign-in Checklist

## What this skill does

Verifies the student actually completed M0 — not just *thinks* they did. People (and LLMs) skip steps. This skill tests every artifact and reports pass/fail per item, then emits a `READY for M1` verdict.

**Run this AFTER `m0-landing-and-signin` Step 9, or any time the student claims M0 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — GitHub repo | `gh repo view` / `gh api` | GitHub MCP, or open repo URL in browser |
| B — Vercel deploy | `curl` | Vercel Connect connector, or open URL in browser |
| C — Landing page contents | `curl … \| grep` | Playwright MCP, or student inspects in browser |
| D — Supabase auth | `supabase` CLI / dashboard | Supabase Connector (preferred both modes) |

In Cowork mode every Bash block below is CLI-only — use the equivalent. Don't try to install `gh`/`curl` in Cowork.

## How to run

The student invokes this directly (e.g. types `驗收 M0`). You (Claude Code) **actively run** each check via Bash / the connectors and report results — don't just describe them.

### Step 1: Collect URLs (one message)

Ask the student for:
1. GitHub repo URL (`https://github.com/<user>/<repo>`)
2. Vercel deploy URL (`https://<app>.vercel.app`)
3. Supabase project URL (`https://<ref>.supabase.co`)

### Step 2: Run the checklist

#### Section A — GitHub repo
- **A1** Repo exists, is **public**, and has Lovable's files:
  ```bash
  gh repo view <owner>/<repo> --json name,visibility,defaultBranchRef
  gh api repos/<owner>/<repo>/contents | grep -oE '"name": "[^"]+"' | head
  ```
  *Recovery if `visibility` is PRIVATE:* M0 Step 4 — Settings → Danger Zone → Change repository visibility → Public.
- **A2** Recent commit from the Lovable→GitHub sync (or the Step 7 SPA push):
  ```bash
  gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message' | head -1
  ```
  *Recovery if missing:* re-connect GitHub in Lovable (M0 Step 3).
- **A3** The repo is a **plain Vite SPA**, not TanStack/SSR (the Step 7 conversion actually landed):
  ```bash
  gh api repos/<owner>/<repo>/contents/package.json --jq '.content' | base64 -d | grep -E '"build"|wrangler|tanstack'
  ```
  Expect a `vite build` script; **no** `wrangler` / `@lovable.dev/vite-tanstack-config` / `@cloudflare/*`.
  *Recovery:* M0 Step 7 — re-run the Vite SPA conversion and push to `main`.

#### Section B — Vercel deploy
- **B1** Live URL returns 200:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app
  ```
- **B2** It's auto-deploying from GitHub (push → redeploy). Confirm in the Vercel dashboard that the project's Git connection points at the repo from A1.
  *Recovery:* re-import the repo on vercel.com/new (M0 Step 8).
- **B3** **Deep link works (the SSR-vs-SPA trap):** `/app` does NOT 404:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/app
  ```
  Expect `200` (or a redirect to sign-in if unauthenticated — both are fine; a **404** means the app shipped as SSR without a SPA fallback). *Recovery:* M0 Step 7 — convert to a plain Vite SPA, re-deploy.

#### Section C — Landing page contents
- **C1** Hero + feature cards + Sign In/Up button present:
  ```bash
  curl -sS https://<app>.vercel.app | grep -oiE "登入|sign in|sign up|逐字稿|上傳影片|Video Speed Reader" | sort | uniq -c
  ```
  (Note: a Vite/React SPA may render client-side — if grep is empty, fall back to a browser/Playwright check.)
- **C2** Browser check (Cowork or if C1 empty): open the URL, confirm the hero headline 「上傳影片，三分鐘內拿到逐字稿。」, the three cards (高準確度逐字稿 / 三分鐘交付 / 可商用授權), the top-right Sign in / 登入 button, and the 「© 2026 Video Speed Reader」 footer are visible.

#### Section D — Supabase auth
- **D1** The student's OWN Supabase project has Email auth enabled (Authentication → Providers).
- **D2** **The decisive test:** sign up a brand-new test email on the live Vercel site, then check Supabase → Authentication → Users — the new user appears in the student's project (not a Lovable-default backend).
  ```bash
  # If supabase CLI is linked to the project:
  supabase projects list
  ```
  Cowork: use the Supabase Connector to query `auth.users`, e.g.
  ```sql
  SELECT email, created_at FROM auth.users ORDER BY created_at DESC LIMIT 3;
  ```
  The test email should be at the top with a `created_at` within the last few minutes.
- **D3** Sign-in AND sign-out both work on the live site (close the loop).
  *Recovery:* redo M0 Step 9 (swap auth to the student's Supabase) — common misses are leaving Lovable Cloud in place, or forgetting the `VITE_SUPABASE_*` env vars in Vercel + redeploy.
- **D4** **No custom tables yet** — M0 is `auth.users` only. Confirm the `public` schema has no `profiles` / `videos` / `jobs` tables:
  ```sql
  SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';
  ```
  Expect an empty result. If Lovable created tables, tell the student to drop them — the real schema arrives in M1 as tracked migration files (see `supabase-best-practice`). ⚠️ non-blocking if the student knowingly kept them, but flag it.

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 repo exists + public | ✅ / ❌ | |
| A2 Lovable sync commit | ✅ / ❌ | |
| A3 Vite SPA (no SSR/wrangler) | ✅ / ❌ | |
| B1 Vercel 200 | ✅ / ❌ | |
| B2 auto-deploy wired | ✅ / ❌ | |
| B3 /app deep-link not 404 | ✅ / ❌ | SSR-vs-SPA trap |
| C1/C2 landing contents | ✅ / ⚠️ / ❌ | |
| D1 email auth enabled | ✅ / ❌ | |
| D2 new user in student's Supabase | ✅ / ❌ | the key one |
| D3 sign-in + sign-out loop | ✅ / ❌ | |
| D4 no custom tables in `public` | ✅ / ⚠️ | M0 is auth-only |

**Verdict:**
- All ✅ → 「M0 驗收通過 ✅ READY for M1。跟我說『啟動 M1』，我們來把 AI 影片逐字稿 pipeline 接進來。」
- Any ❌ → list the failed items + the recovery step, and tell the student to fix then re-run `驗收 M0`.
