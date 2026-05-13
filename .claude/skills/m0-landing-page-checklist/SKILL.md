---
name: m0-landing-page-checklist
description: Course 2 Milestone 0 verification — checks every artifact (GitHub repo, Lovable sync, Vercel deploy, landing page contents) is real and correctly wired. Use when the student says "驗收 M0", "check M0", "M0 done?", or after the `m0-landing-page` skill completes Step 6.
---

# M0 — Landing Page Checklist

## What this skill does

Verifies the student has actually completed M0 — not just *thinks* they have. LLMs (and humans) skip steps. This skill goes through every artifact and tests it, then reports pass/fail per item.

**Run this AFTER `m0-landing-page` Step 6, or any time the student claims M0 is done.**

## Execution mode: Cowork vs CLI (read this first)

This checklist supports both execution modes — confirm which one the student is in before running anything. See `m0-landing-page-prerequisites` for the full mode table.

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — GitHub repo | `gh repo view` / `gh api` | GitHub MCP if installed; otherwise the GitHub web UI (student opens repo URL and confirms) |
| B — Vercel deploy (HTTP checks) | `curl` | `mcp__vercel__*` deployment tools, OR ask student to open URL in browser and paste status |
| C — Landing page contents | `curl ... | grep` | `mcp__playwright__browser_navigate` + `browser_snapshot` if available; otherwise student does the browser inspection and pastes back |
| D — Two-way sync | `gh api repos/.../commits` | GitHub MCP / GitHub web UI commit timestamp check |
| E — Supabase | `supabase` CLI fallback | `mcp__supabase_remote__*` (preferred in both modes) |

In Cowork mode, every Bash block below is **CLI-only**. Use the equivalent above. Do not try to install `gh` / `curl` / `vercel` in Cowork — they aren't there.

## How to run

**This skill is meant to be invoked directly by the student** (e.g. they type `驗收 M0` and the skill kicks in). You (Claude Code) **actively execute** each check — don't just describe how to verify; actually run the commands via the `Bash` tool, then report results.

### Step 1: Collect URLs from the student (one message)

Ask the student for these four URLs upfront:

1. GitHub repo URL (e.g. `https://github.com/<username>/<product-name>`)
2. Lovable project URL (e.g. `https://lovable.dev/projects/<id>`)
3. Vercel deploy URL (e.g. `https://<repo-name>-<random>.vercel.app`)
4. Supabase project URL (e.g. `https://<ref>.supabase.co`)

If any URL is missing, **stop**. The student hasn't actually finished M0.

### Step 2: Preflight — verify tools are available

#### CLI mode

Before running any checks, run this in one Bash call to inventory what's available:

```bash
echo "--- gh ---"   && gh --version && gh auth status 2>&1 | head -3
echo "--- vercel ---" && vercel --version && vercel whoami 2>&1
echo "--- supabase ---" && supabase --version 2>&1 || echo "supabase CLI not installed (OK if Supabase MCP is available)"
```

#### Cowork mode

Skip the Bash inventory — there's no shell. Instead, scan the available tool list for:

- `mcp__vercel__*` — required for Section B
- `mcp__supabase_remote__*` (or `_local__*`) — required for Section E
- GitHub MCP tools (if installed) — used for Section A / D; otherwise fall back to the GitHub web UI
- `mcp__playwright__browser_*` — optional, lets you do Section B/C/E4 without bothering the student

If `mcp__vercel__*` or `mcp__supabase_*__*` is missing, stop and refer the student back to `m0-landing-page-prerequisites`.

**Interpret the output:**

- ✅ `gh` installed + `Logged in to github.com as <user>` → ready for Section A
- ❌ `gh: command not found` → guide the student through install, then `gh auth login`
- ❌ `gh` installed but `not logged in` → tell the student to run `gh auth login` in their own terminal (cannot be done via Bash tool from this session — it's an interactive browser flow)

Same shape for `vercel` and `supabase`.

**If any tool is missing or not logged in, refer the student to the `m0-landing-page-prerequisites` skill** which has the install + login commands per OS. Don't try to install in this session — that's the prerequisites skill's job. But if the student says "I'm on macOS / Windows / Linux", you can paste the right one-liner from `m0-landing-page-prerequisites` to save a trip.

**Tool priority for this skill (prefer MCP > CLI):**

| Service | First try (MCP) | Fallback (CLI) |
|---|---|---|
| Supabase | `mcp__supabase_remote__*` (list_tables, execute_sql, get_project_url, etc.) | `supabase` CLI after `supabase login` |
| Vercel | `mcp__vercel__*` if available in this session (varies by setup) | `vercel` CLI after `vercel login` |
| GitHub | `gh` CLI is the de-facto standard; no GitHub MCP needed | (CLI only) |

**Always check at the start of each section whether the relevant MCP is available** (look at the list of `mcp__*` tools you've been given). If yes, prefer MCP — it's already authenticated and won't trigger an OS-specific install path. If no, drop to CLI.

### Step 3: Run the checklist below

For each item: **don't take the student's word for it — verify with a Bash command (CLI mode) or MCP tool (Cowork mode)**. Report results as a table at the end. Each row is one check, with: ✅ pass, ❌ fail, ⚠️ couldn't verify.

The checklist tables below show **CLI-mode commands** as the default. For each row, the Cowork-mode equivalent is:

- `gh` / `gh api` → GitHub MCP tool, or open the URL in the browser and have the student paste back what they see
- `curl` → `mcp__playwright__browser_navigate` + `browser_snapshot`, or student paste-back
- `vercel env ls` → `mcp__vercel__*` env-listing tool, or student paste-back from Vercel dashboard
- `supabase` CLI → `mcp__supabase_remote__*` (already preferred in both modes)

---

## Checklist

### Section A — GitHub repo (3 checks)

| # | Check | How to verify |
|---|---|---|
| A1 | Repo exists at the URL the student gave | `gh repo view <owner>/<repo>` returns repo info (not 404) |
| A2 | Repo has commits (not empty) | `gh api repos/<owner>/<repo>/commits --jq 'length > 0'` — must be `true` |
| A3 | Repo has a `package.json` at root | `gh api repos/<owner>/<repo>/contents/package.json --jq '.name'` returns a string (not error) |

If A1 fails: the student typo'd the URL or never connected Lovable to GitHub. Send them back to `m0-landing-page` Step 3.

If A2 fails: Lovable connected but didn't push the initial code. Have the student go to Lovable and make any tiny edit + save — that should trigger a commit.

If A3 fails: Lovable scaffolded something unusual. Look at what's actually at the repo root with `gh api repos/<owner>/<repo>/contents/ --jq '.[].name'` and decide if it's recoverable.

### Section B — Vercel deploy (4 checks)

| # | Check | How to verify |
|---|---|---|
| B1 | Vercel URL returns HTTP 200 | `curl -sI <vercel-url> \| head -1` shows `HTTP/2 200` |
| B2 | Page is HTML (not Vercel error page) | `curl -s <vercel-url> \| grep -c "<html"` ≥ 1 AND the response does NOT contain "DEPLOYMENT_NOT_FOUND" or "404: NOT_FOUND" |
| B3 | Page contains the product name "Video Speed Reader" | `curl -s <vercel-url> \| grep -ic "video speed reader"` ≥ 1 |
| B4 | Vercel is auto-deploying from the GitHub repo | Check the deploy URL pattern: should be `<repo-name>-*.vercel.app` or a custom one the student set. If the repo name doesn't appear in the deploy URL, ask the student to confirm in Vercel dashboard → Settings → Git that it's linked to the correct repo |

If B1 or B2 fails: the Vercel deploy is broken. Ask the student to open Vercel dashboard → the project → Deployments tab, look at the most recent deployment log, and paste the error here. Most common cause: framework preset auto-detection picked the wrong framework (Lovable's default is Vite, not Next.js).

If B3 fails: the page deployed but doesn't show "Video Speed Reader". Most likely Lovable's v1 reinterpreted the prompt and used a different brand name. Ask the student to go back to Lovable, prompt it to use "Video Speed Reader" verbatim as the product name, and re-deploy.

### Section C — Landing page contents (4 checks)

Fetch the Vercel page HTML and check for required UI elements. **Heuristic** — exact selectors depend on what Lovable generated, so use multiple fallback searches.

| # | Check | How to verify |
|---|---|---|
| C1 | Hero section exists with the right value prop | HTML contains an `<h1>` or `<h2>` element near the top (within first 5000 chars of body) AND contains text matching `/三分鐘\|three.?minute\|3.?minute/i` (the value prop "三分鐘內拿到逐字稿") |
| C2 | Features section has all 3 expected features | HTML contains at least 2 of these phrases: `Whisper`, `三分鐘交付` (or `three.?minute turnaround`), `可商用` (or `commercial.?use`). If only 1 or 0 matches, Lovable likely generated generic feature copy — re-prompt. |
| C3 | Sign In / Sign Up button exists in header | HTML contains text matching `/sign.?in\|sign.?up\|登入\|註冊/i` near the top of the page |
| C4 | NO upload widget in M0 (negative check) | HTML must NOT contain `<input type="file">` or text matching `/upload your video\|drag.*drop\|拖.*影片/i`. If present, Lovable over-generated; re-prompt to remove upload widget. |

If C1–C3 fail: the Lovable v1 didn't match the prompt. Suggest re-rolling the Lovable prompt with more specific instructions.

If C4 fails: Lovable generated an upload widget that belongs in M1, not M0. Tell the student to re-prompt Lovable: "Remove any upload widget or drop zone. M0 only includes landing page + Sign In/Up + placeholder dashboard. Upload comes in M1."

### Section D — Two-way sync (1 check)

| # | Check | How to verify |
|---|---|---|
| D1 | Lovable ↔ GitHub two-way sync is live | **You can't verify this passively** — sync only proves itself if a new edit propagates. Tell the student: "請在 Lovable 隨便改一個字（例如 hero 的某個 emoji），按存檔，然後跟我說『改好了』。" When they confirm, run `gh api repos/<owner>/<repo>/commits --jq '.[0] | {sha: .sha[0:8], message: .commit.message, when: .commit.author.date}'` and check the timestamp is within the last 2 minutes. If yes ✅ — sync is alive. If the latest commit is still the same as before, sync is broken. |

This is the only check that requires the student to do an action. Worth doing — if sync is broken, M1+ will be painful because every Claude Code edit needs to land in the repo for Vercel to redeploy.

### Section E — Supabase project + auth wired up (5 checks)

This is the biggest correctness risk of the new M0. Two failure modes to catch:
1. Lovable claimed to connect Supabase, but the project lives in some shared/demo space — not in the student's own Supabase account
2. Lovable connects Supabase OK in preview, but the Vercel deploy breaks because env vars got lost in transit

E0 + E1 cover the first; E2–E4 cover the second.

| # | Check | How to verify |
|---|---|---|
| E0 | Student has a Supabase organization under their own account | **Active verification (prefer MCP):** Call `mcp__supabase_remote__get_project_url` or any `mcp__supabase_remote__*` tool — if it returns successfully, the MCP is authenticated to the student's Supabase account, which means an org exists. <br>**Fallback (no MCP):** ask the student to run `supabase orgs list` (requires `supabase login` first) and paste the output. Should show at least one org. <br>**Last resort (no CLI either):** ask the student to open https://supabase.com/dashboard and confirm they see at least one organization. |
| E1 | Supabase project `video-speed-reader` (or similar) exists inside the student's own org and matches the URL the student gave | **Active verification (prefer MCP):** Call `mcp__supabase_remote__list_tables` or similar — returns a project context. Cross-reference the project ref in the MCP output against the `<ref>` in the student-supplied Supabase URL. They must match. <br>**Fallback (CLI):** `supabase projects list` — should include a row whose ref matches the supplied URL. <br>**If the URL's project ref isn't in the student's account at all**, the Lovable connection went to a shared/demo space. Re-do `m0-landing-page` Step 5.A and pick "create new" explicitly inside the student's org. |
| E2 | Supabase project responds | Hit `curl -sI https://<ref>.supabase.co/rest/v1/` — should return HTTP 401 (without API key) or 200 (some versions). Anything else (DNS error, 5xx) means the project is paused / deleted. |
| E3 | Vercel deploy has Supabase env vars set | **Active verification (prefer Vercel MCP):** if a `mcp__vercel__*` tool for listing env vars is available, call it and grep for `SUPABASE`. <br>**Fallback (CLI):** `cd <project-dir> && vercel env ls` — should list at least one var matching `*SUPABASE_URL*` and one matching either `*SUPABASE_PUBLISHABLE_KEY*` (new Supabase naming) OR `*SUPABASE_ANON_KEY*` (legacy naming; still valid). <br>**Last resort:** ask the student to go to Vercel project → Settings → Environment Variables and paste the variable names back. <br>If missing, this is the #1 cause of "auth works in Lovable preview but not on Vercel." |
| E4 | Sign-up + sign-in actually works on the Vercel deploy + user lands in Supabase `auth.users` | **Hybrid: student does the browser test, you verify the side effect.** Tell the student: "請打開 `<vercel-url>` (用 incognito 視窗)，點 Sign Up，用一個測試 email 註冊（例如 `test-m0-<時間戳>@example.com`），登入後應該看到 `/app` 頁面顯示 'Hi {email}'。完成跟我說。" <br>When they confirm, **verify the user actually landed in Supabase**: use `mcp__supabase_remote__execute_sql` to run `SELECT email, created_at FROM auth.users ORDER BY created_at DESC LIMIT 5` — the test email should appear at the top with a `created_at` within the last few minutes. <br>If MCP unavailable, fall back to `supabase db execute` or ask student to look at Supabase dashboard → Authentication → Users. |

If E0 fails: student hasn't signed up for Supabase. Stop everything, send them to register, then resume.

If E1 fails: Lovable connected to a project that's not in the student's org. The project will eventually break (Lovable demo projects expire / get rate-limited). Re-do Step 5.A explicitly creating a new project inside the student's own org.

If E2 fails: the project URL is dead. Could be paused (free tier auto-pauses after a week of inactivity — student needs to un-pause from the dashboard), or could be a typo in the URL.

If E3 fails: copy the env vars from Lovable's `.env` file (or Supabase dashboard → Settings → API) into Vercel manually. Redeploy.

If E4 fails: walk the student through Supabase dashboard → Authentication → Providers → Email and verify "Confirm email" is OFF for v1 simplicity. If still failing, look at the browser console on the Vercel deploy — usually a CORS or env var issue.

---

## Reporting

After running all checks, output a table:

```
M0 Checklist Results
====================

Section A — GitHub repo
  A1 Repo exists                       ✅
  A2 Has commits                       ✅
  A3 Has package.json                  ✅

Section B — Vercel deploy
  B1 HTTP 200                          ✅
  B2 Real HTML page                    ✅
  B3 Product name appears              ✅
  B4 Linked to correct GitHub repo     ✅

Section C — Landing page contents
  C1 Hero section                      ✅
  C2 Features section                  ✅
  C3 Sign In / Sign Up button          ✅
  C4 No upload widget (negative)       ✅

Section D — Two-way sync
  D1 Lovable→GitHub sync works         ✅

Section E — Supabase project + auth
  E0 Student has Supabase org          ✅
  E1 Project in student's own org      ✅
  E2 Supabase project responds         ✅
  E3 Vercel has Supabase env vars      ✅
  E4 Sign-up/in works + user appears   ✅  (manually verified)

=====================================
Verdict: 17/17 pass
M0 status: READY for M1
```

If any ❌ exists: M0 is **NOT** done. Loop back into `m0-landing-page` skill at the corresponding step.

If only ⚠️ exist (manual checks): tell the student exactly what to manually verify, then re-run this checklist.

If all ✅: green-light M1. Say:
> "M0 全綠燈通過。準備好的話跟我說『啟動 M1』。"

## Why this checklist exists

Two LLM-specific failure modes this guards against:

1. **The "I think I did it" failure** — the LLM walks through the steps verbally but never actually verifies the artifacts. The student trusts the LLM, moves to M1, then M1 breaks because M0 wasn't real.
2. **The "skip the boring parts" failure** — the LLM (or student) skips connecting GitHub because "Lovable already has a preview URL." Then M1 has no repo to edit code in.

Both kill student progress invisibly. This checklist catches both before they propagate.

## TODO

- [ ] When Lovable's HTML output structure is more stable, tighten C1–C3 selectors from heuristic to specific
- [ ] Add a check for `vercel.json` or framework config files if Lovable starts generating them
- [ ] Decide if A3 should require specific dependencies (react, vite) or stay loose
