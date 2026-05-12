---
name: m0-landing-page
description: Course 2 Milestone 0 — generate the v1 SaaS landing page with Lovable, push to GitHub, deploy to Vercel. Use when the student says "啟動 M0", "start M0", "build the M0 landing page", "Lovable 入口網站", or any variant that maps to Course 2 module 2.1 (快速搭建第一版可登入的 SaaS 入口網站).
---

# M0 — Landing Page (對應 2.1 快速搭建第一版可登入的 SaaS 入口網站)

## What this skill does

Walks the student through Course 2 Milestone 0 end-to-end. By the end the student has:

1. A live URL hosted on Vercel (e.g. `https://my-saas-xyz.vercel.app`)
2. A GitHub repo that auto-deploys to Vercel on every push
3. A Lovable project two-way-synced to that GitHub repo
4. A v1 landing page for **Video Speed Reader** with: hero ("上傳影片，三分鐘內拿到逐字稿"), three feature cards (高準確度 / 三分鐘交付 / 可商用授權), a Sign In / Sign Up button, and a copyright footer.
5. **Lovable's built-in Supabase auth wired up** — students can actually sign up, sign in, and sign out. No custom schema yet (only the default `auth.users` table that Supabase auto-creates). No upload widget, no AI pipeline.

**Out of scope for M0:** Custom Supabase tables / RPC functions / migrations, upload widget, database schema for the product, API routes for the AI pipeline, Stripe. Those are M1+.

## When to load this skill

Trigger phrases:
- "啟動 M0"
- "start M0" / "begin M0"
- "M0 跑起來"
- "幫我蓋 M0 的 landing page"
- "Lovable 入口網站"
- Any prompt the student gives that references Course 2 module 2.1

Do NOT load this skill for M1–M5 — they have their own skills.

## Required external accounts

Before starting, the student must have:

| # | Service | Used for |
|---|---|---|
| 1 | GitHub | Lovable will create a repo here; Vercel will import from here |
| 2 | Lovable (`lovable.dev`) | Generates the v1 UI + scaffolds Supabase auth |
| 3 | Supabase (`supabase.com`) | Stores `auth.users`, handles sign up / sign in / sign out |
| 4 | Vercel | Auto-deploys the GitHub repo |

If any of the four is missing, **stop and ask the student to register first**. Do not try to proceed without all four.

## The product being built

M0 builds **Video Speed Reader** — the exact same product as this course's reference repo. Hard-coded, no per-student variation. The v1 in M0 is **a real, working signed-in SaaS**: the landing page sells "upload a video, get a transcript", and the Sign In / Sign Up button leads to a working Supabase-backed auth flow. After signing in, the user lands on a placeholder authenticated page (e.g. "Hi {email}, your dashboard is coming soon"). The actual upload widget and AI pipeline come in M1.

## Conversational flow

The skill is conversational — you (Claude Code) drive the student through 6 steps. Don't dump all the steps at once. After each step, **wait for confirmation** before moving to the next.

> **Before Step 1:** confirm the student has done the one-time CLI + MCP setup. If not, **load the `m0-landing-page-prerequisites` skill first** and walk them through it. The checklist at the end of M0 will need `gh` / `vercel` / `supabase` CLIs (and ideally Vercel + Supabase MCPs) — if they're missing, the verification will stall at the worst moment. A quick sanity check before starting:
>
> ```bash
> gh auth status && vercel whoami && supabase projects list 2>/dev/null | head -1
> ```
>
> If any of those error out, switch to `m0-landing-page-prerequisites` skill and come back.

### Step 1 — Create the Lovable project AND a separate Supabase project (do not connect them yet)

This step has two parallel sub-steps. The student creates the Lovable project AND a Supabase project, but they are NOT wired together yet. Lovable's v1 will auto-use **Lovable Cloud** (Lovable's own managed Supabase) as the backend; we'll switch over to the student's own Supabase project in **Step 5** as a clean one-shot operation. Connecting Supabase to Lovable now would just create a half-state where the student thinks auth runs against their project but it actually runs against Lovable Cloud.

**Order verified by walking through Lovable:** create a Lovable project → create a Supabase project in parallel (not connected to Lovable) → give the full Video Speed Reader prompt (auth runs on Lovable Cloud) → connect GitHub → deploy to Vercel → swap backend to the student's own Supabase in one go.

Tell the student (verbatim, with the exact menu paths):

> **A. 建立 Lovable project**
> 1. 到 https://lovable.dev/，登入後從 dashboard 點 **+ New project**（或類似的「建立新專案」按鈕）
> 2. 如果 Lovable 要求你打一個初始 prompt，輸入最簡單的 placeholder：`Create a blank project for a SaaS product called Video Speed Reader.` — 這只是讓你進到專案 workspace，**不要在這一步就貼正式 prompt**
> 3. 等 Lovable 跑完初始 scaffold（30秒到一分鐘）
> 4. 把 Lovable project URL 貼回來給我（格式 `https://lovable.dev/projects/<id>`）
>
> **B. 在 Supabase dashboard 建一個 project（不要在 Lovable 連）**
> 1. 到 https://supabase.com/dashboard
> 2. 點 **New project**（在你的 org 底下）
> 3. Name 填 `video-speed-reader`，region 選離你近的，database password 隨意（之後 M1 才會用到）
> 4. 等 Supabase 把 project 啟動好（通常 1–2 分鐘）
> 5. 把 Supabase project URL 貼回來給我（格式 `https://<ref>.supabase.co`）
>
> ⚠️ **不要**在 Lovable 點 Connect Supabase。我們現在故意讓 Lovable 跑在它預設的 Lovable Cloud 上，等 Step 5 再一次性切換到你的 Supabase project。如果在這一步就連 Lovable + Supabase，Lovable 的 v1 還是會用 Lovable Cloud，反而會產生混淆狀態。

**Verify before moving on:** the student must have given you both URLs (Lovable project URL + Supabase project URL) before Step 2.

> **Note for Claude Code:** If the student asks about modifying the Supabase schema during this step (e.g. "should I add a `profiles` table?"), say **no — M0 only uses the default `auth.users` table that Supabase auto-creates**. Custom tables come in M1. Also if the student insists on connecting Supabase to Lovable now, gently push back — explain Lovable Cloud will still take precedence and we'll do the swap cleanly in Step 5.

### Step 2 — Give Lovable the full Video Speed Reader prompt

Lovable will generate v1 using **Lovable Cloud as the auth backend** (its default). That's fine for now — Step 5 will swap to the student's own Supabase. Paste this prompt into the chat for the student to copy into Lovable verbatim:

```
Build a SaaS landing page + authenticated app shell for Video Speed Reader, a product that turns any video into an accurate transcript in three minutes, targeted at content creators, educators, and engineers who record long-form video and need a fast, clean transcript to repurpose into blog posts, course notes, or searchable archives.

The site must include:

1. A public landing page (`/`) with:
   - Hero section: product name "Video Speed Reader" prominently displayed, value prop "上傳影片，三分鐘內拿到逐字稿。" (English subtitle: "Upload your video, get a clean transcript in three minutes."), and a primary CTA button labeled "Sign in / 登入" in the top-right header
   - Features section with exactly 3 feature cards:
     * Card 1: "高準確度逐字稿 (High-accuracy transcripts)" — powered by OpenAI Whisper, supports Chinese and English
     * Card 2: "三分鐘交付 (Three-minute turnaround)" — processed in the background, you get an email when it's ready
     * Card 3: "可商用授權 (Commercial-use ready)" — you own the output, use it however you like
   - Footer with copyright "© 2026 Video Speed Reader"

2. Authentication using Lovable's built-in Supabase-style auth (use whatever auth backend Lovable provides by default — Lovable Cloud is fine for this v1; we'll swap to a user-owned Supabase project in a later step):
   - Sign Up page with email + password
   - Sign In page with email + password
   - Sign Out functionality
   - Email confirmation can be disabled for simplicity in this v1

3. An authenticated app shell at `/app` that the user lands on after signing in:
   - Greets the signed-in user by email: "Hi {user.email}"
   - A placeholder message: "Your dashboard is coming soon. Upload functionality will be added in the next milestone."
   - A Sign Out button in the header

Design requirements:
- Modern, professional dark theme (purple/violet accent on a near-black background)
- Use Inter or a similar sans-serif font
- Mobile responsive
- Tasteful subtle animations (fade-in on scroll is fine; don't overdo it)

Out of scope for this v1: video upload widget, transcript display, payment, custom database tables (do NOT create a `profiles` or `videos` table — only use Supabase's default `auth.users`). Those come in later milestones. Stick to landing page + auth + placeholder dashboard.
```

Tell the student verbatim:

> 「請複製這段 prompt 貼到 Lovable 的 prompt 框，按送出。Lovable 大概 2-3 分鐘會跑完。跑完之後在 Lovable preview 試一次 sign up + sign in，確認真的能用、登入後會跳到 /app 顯示 "Hi {你的 email}"。確認 OK 再跟我說。」

**Verify before moving on:** the student must confirm sign-up + sign-in works in the Lovable preview. Debug here rather than waiting for Vercel re-deploys — Lovable iterates faster.

### Step 3 — Connect Lovable to GitHub

Now that the v1 has been generated, push the code to GitHub so Vercel can pick it up.

Tell the student (verbatim, with the exact menu path):

> 在 Lovable 的專案頁面：
> 1. 右上角點 **GitHub icon** → 選 **Connect GitHub**（或從 Project Settings → Git → GitHub）
> 2. 授權 Lovable access 你的 GitHub workspace
> 3. 點 **Connect** 旁邊你想要建 repo 的 workspace
> 4. Lovable 會自動建一個 GitHub repo（名字通常會是 `video-speed-reader` 或 `video-speed-reader-xyz`）
> 5. 把 GitHub repo URL 貼回來給我

**Verify before moving on:** the GitHub repo URL contains `video-speed-reader` or similar. Quickly confirm it really has the code by running `gh repo view <owner>/<repo>` — should show recent commits.

### Step 4 — Deploy to Vercel

#### Step 4.A — Convert the Lovable project to a Vite SPA (preemptive, one prompt)

Lovable's v1 may use **TanStack Start with `@lovable.dev/vite-tanstack-config`**, which is Cloudflare-Workers-targeted. Deploying that as-is to Vercel results in a green build that 404s on every route. To avoid that detour entirely, **run one preemptive Lovable prompt to convert to a plain Vite SPA before we touch Vercel.**

Tell the student verbatim:

> 「在 Vercel deploy 之前，先在 Lovable 跑這個 prompt，把專案改成 Vite + React SPA（Vercel 才能正確 serve）：」

Then paste this prompt for the student to copy into Lovable:

```
Convert this project to a plain Vite + React + shadcn SPA suitable for static deployment on Vercel. Specifically:

1. Remove the `@lovable.dev/vite-tanstack-config` import in vite.config.ts and replace with a standard `@vitejs/plugin-react` setup. The final vite.config.ts should look roughly like:
   import { defineConfig } from "vite";
   import react from "@vitejs/plugin-react";
   import path from "path";
   export default defineConfig({
     plugins: [react()],
     resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
   });
2. Remove any Cloudflare-specific build scripts and dependencies from package.json (e.g. `wrangler`, `build:cloudflare`, `@cloudflare/*`).
3. Remove any TanStack Start server entry points (typically `app/router.tsx`, `app/ssr.tsx`, `app/api/*`, or similar SSR handlers). Keep the React client code under `src/`.
4. Ensure `npm run build` (or `bun run build`) produces a static `dist/` folder with `index.html` and assets — NO server functions, NO Workers handlers.
5. Keep all the existing UI (landing page, Sign In / Sign Up pages, /app shell) and the Supabase auth integration. Only the build/deploy target changes.

After this conversion, Vercel will auto-detect Framework Preset = Vite and the site will deploy correctly.
```

**Verify before moving on:**
- `vite.config.ts` no longer imports `@lovable.dev/vite-tanstack-config`
- Sign-up / sign-in still works in the Lovable preview after the conversion
- The conversion commit has synced to GitHub (`gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message'` should show a recent commit)

If sign-up broke during the conversion, re-prompt Lovable: 「Sign-up flow broke after the Vite SPA conversion. Restore the Supabase auth flow using the already-connected `@supabase/supabase-js` client. Keep the project as a pure Vite SPA — no SSR, no TanStack Start.」

#### Step 4.B — Import GitHub repo to Vercel

Tell the student:

> 1. 到 https://vercel.com/new
> 2. 點 **Import Git Repository**
> 3. 找到剛剛 Lovable 建的 repo，點 **Import**
>    - **如果 Vercel 看不到 repo**：你可能是第一次從這個 GitHub 帳號用 Vercel — Vercel 會要你裝 **"Vercel for GitHub" App** 並選給它哪些 repo 的權限。選 Lovable 建的那個 repo（或勾「All repositories」較方便）。
> 4. 進到 project 設定頁。Framework Preset 現在應該會自動偵測成 **Vite**（因為 Step 4.A 已經轉成 Vite SPA）— 確認一下，如果還是被偵測成 "Other" 或別的，手動改成 **Vite**
> 5. **Build & Output settings** — 維持預設值即可：Build Command = `npm run build`（或 `vite build`），Output Directory = `dist`，Root Directory = `./`
> 6. **Environment Variables**：M0 在 Step 4 的時候 Lovable 還用 Lovable Cloud，所以你 .env 裡看到的 Supabase 相關 env vars 是 Lovable Cloud 注入的。先把這些值（無論變數名叫 `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` / `VITE_SUPABASE_PUBLISHABLE_KEY`）複製進 Vercel（讓 Vercel build 不會 fail）。Step 5 切換成學員自己的 Supabase 之後會再把這些值換掉。
> 7. 點 **Deploy**
> 8. 等 ~1 分鐘，deploy 完成後 Vercel 會給一個 `https://<repo-name>-<random>.vercel.app` 的 URL

Ask the student to paste the Vercel deploy URL back here. Then test the sign-up / sign-in flow on the Vercel URL too — confirm it still works after deploy (env var typos are the #1 cause of "works in Lovable preview but broken on Vercel").

> **Heads up — auth right now is on Lovable Cloud, not the student's own Supabase.** Even though Step 1.B "connected Supabase," Lovable's v1 actually uses its **Lovable Cloud** managed backend by default (a Lovable-owned Supabase project, not the one in the student's org). Test users from sign-up will land in Lovable Cloud, NOT in `auth.users` of the student's own Supabase project. We fix this in Step 5 before the final checklist — otherwise checklist E1/E4 will fail because the test user won't appear in the student's Supabase dashboard.

### Step 5 — Switch backend from Lovable Cloud to the student's own Supabase

Until now, sign-up / sign-in works — but the users are stored in **Lovable Cloud** (Lovable's managed backend), not the Supabase project the student created in Step 1.B. We need to repoint Lovable at the student's own Supabase before M1 can do anything meaningful (M1 will write custom tables to a Supabase the student doesn't control = bad).

Confirm with the student first:

> 「現在 sign up / sign in 雖然能跑，但使用者其實是存在 Lovable Cloud（Lovable 自己的 Supabase），不是你 Step 1.B 連的那個 Supabase project。我們現在要切過去你自己的 Supabase。」

#### Step 5.A — Have the student grab their own Supabase project's URL + publishable key

Tell the student:

> 1. 到 https://supabase.com/dashboard
> 2. 點進 Step 1.B 建的 `video-speed-reader` project
> 3. 左邊 sidebar → **Project Settings** → **API Keys**（新版 Supabase）或 **API**（舊版）
> 4. 複製這兩個值貼回來給我：
>    - **Project URL**（格式 `https://<ref>.supabase.co`）
>    - **Publishable key**（開頭是 `sb_publishable_...`）
>      - 如果 dashboard 還用舊版命名，這個欄位可能標 "anon public" — 是同一個 key、可以放心用。
>      - **不要**複製到 "Secret key" / "service_role" — 那是 server-only，絕對不能放在前端 / Vercel 的 `VITE_*` env var 裡。

(MCP fallback: if Supabase MCP is wired, you can also call `mcp__supabase_remote__get_project_url` and `mcp__supabase_remote__get_publishable_keys` to fetch these directly without bothering the student. Confirm they match what the student would see in dashboard.)

#### Step 5.B — Give Lovable the switchover prompt

Paste this prompt for the student to copy into Lovable verbatim, replacing `<URL>` and `<PUBLISHABLE_KEY>` with the values from Step 5.A:

```
Switch this project's backend from Lovable Cloud to the user's own Supabase project. Do NOT keep any Lovable Cloud references.

Specifically:

1. Find every place the project currently uses Lovable Cloud's Supabase client (Lovable's auto-provisioned `supabase` client, typically in `src/integrations/supabase/client.ts` or similar). Update it to use these credentials instead:

   VITE_SUPABASE_URL=<URL>
   VITE_SUPABASE_PUBLISHABLE_KEY=<PUBLISHABLE_KEY>

   Note: Supabase's newer "publishable key" (`sb_publishable_*`) replaces what used to be called the "anon key". They're the same role (browser-safe, RLS-gated). If the codebase already uses `VITE_SUPABASE_ANON_KEY`, you can either:
   - Rename to `VITE_SUPABASE_PUBLISHABLE_KEY` for consistency with current Supabase naming, OR
   - Keep `VITE_SUPABASE_ANON_KEY` as the variable name but put the new `sb_publishable_*` value in it (works fine; it's just an env var name).
   Pick one and apply consistently across `.env`, the client init code, and any docs.

2. Update the `.env` file (or `.env.local`) to use the values above. Remove any Lovable Cloud env vars (e.g. anything prefixed with `LOVABLE_CLOUD_*` or that points to a Lovable-owned Supabase ref).

3. Make sure the Supabase client is initialized exactly once and reads from `import.meta.env.VITE_SUPABASE_URL` and the matching key env var — no hardcoded URLs.

4. Disconnect / remove the Lovable Cloud integration if there's a UI toggle for it (Project Settings → Integrations → Lovable Cloud → disconnect). If you can't toggle it, at least make sure the code only references the new Supabase project.

5. Keep the Sign Up / Sign In / Sign Out flow exactly as it is. Only the backend target changes.

After this change, sign-up should create users in the student's own Supabase auth.users table — verify by signing up a NEW test user in the Lovable preview, then checking the student's Supabase dashboard → Authentication → Users — the new email should appear there.
```

**Verify before moving on:**

1. **In Lovable preview**: sign up with a NEW test email (e.g. `test-switch-<timestamp>@example.com`). Sign-in should still work.
2. **In the student's Supabase dashboard** (or via Supabase MCP): the new test email must appear in `auth.users`. You can verify via `mcp__supabase_remote__execute_sql`:
   ```sql
   SELECT email, created_at FROM auth.users ORDER BY created_at DESC LIMIT 3;
   ```
   The new test email should be at the top, with a `created_at` within the last few minutes.
3. **Vercel will auto-redeploy** when Lovable pushes the conversion commit to GitHub. Wait for that, then test sign-up on the Vercel URL one more time — the new user should land in the student's Supabase too.
4. **Vercel Environment Variables** still need to match: confirm `VITE_SUPABASE_URL` and the publishable key env var (`VITE_SUPABASE_PUBLISHABLE_KEY` or `VITE_SUPABASE_ANON_KEY` — whichever name Lovable settled on) in Vercel project → Settings → Environment Variables match the new values from Step 5.A. If Lovable's `.env` got pushed to git (it shouldn't — `.env` should be gitignored — but check anyway), the values may have been baked in. Either way, set them in Vercel and re-deploy.

If the test user doesn't appear in the student's Supabase, the switchover didn't fully take. Re-prompt Lovable: 「Sign-up is still going to Lovable Cloud instead of the configured Supabase project. Confirm the Supabase client in `src/integrations/supabase/client.ts` reads ONLY from `import.meta.env.VITE_SUPABASE_URL` and the publishable-key env var (the one set in Step 5.B), and disconnect Lovable Cloud entirely.」

### Step 6 — Final verification

Once the student gives you the Vercel URL, run the M0 checklist skill (`m0-landing-page-checklist`) to verify every artifact exists and is correctly wired.

If anything fails the checklist, walk the student through fixing it. Don't declare M0 done until the checklist is 100% green.

## Things to watch out for (common mistakes)

LLMs (and students following along) tend to drift in these ways:

1. **Skipping Lovable, going straight to scaffolding the UI with Claude Code's Write tool.**
   M0 is specifically about using Lovable, not Claude Code, to generate the UI. The point is that the student experiences a no-code UI generator first. If you find yourself about to call the Write tool to make `.tsx` files for the landing page, **STOP** — that's M0 done wrong.

2. **Asking the student to install Vite/Next.js locally before Lovable.**
   Lovable handles all the project scaffolding. Don't suggest `npm create vite@latest` etc.

3. **Adding upload widget / video processing / custom DB tables in M0.**
   M0 stops at: landing page + Sign In / Sign Up / Sign Out + a placeholder authenticated page. NO upload widget, NO video transcript display, NO custom Supabase tables, NO API calls to OpenAI / Anthropic. Those are M1. If Lovable's v1 includes an upload zone or starts generating SQL migrations, tell the student to re-prompt: "Remove the upload widget and any database migrations beyond Supabase's default auth.users table — this is M0, only auth + placeholder dashboard."

4. **Forgetting GitHub.**
   It's tempting to go Lovable → "publish to lovable.app" directly. Skip that. We want code on GitHub so Vercel can auto-deploy and M1+ can edit it with Claude Code.

5. **Vercel framework preset wrong.**
   If Vercel guesses wrong (e.g. picks "Other" instead of Vite), the deploy will fail. Tell the student to manually pick Vite if the auto-detect is wrong.

6. **Letting the student rename the product.**
   M0 hard-codes "Video Speed Reader" because the rest of the course (M1–M5) all assume that product. If the student wants to rename it, fine — but tell them they'll need to do a find-and-replace across the whole repo in every later milestone. Default answer: keep "Video Speed Reader" for now, rename later if they really want.

7. **Modifying Supabase schema beyond `auth.users` in M0.**
   Lovable will sometimes propose creating a `profiles` table, a `videos` table, or RPC functions for "later use". **Block this in M0** — say no. M0 uses only the default `auth.users` table that Supabase auto-creates. Custom tables / RPCs / migrations are M1, and they must follow the migration-file-first rule (see `supabase-best-practice` skill). Letting Lovable create ad-hoc schema in M0 means the student has untracked schema state going into M1.

8. **Not testing sign-up after Vercel deploy.**
   Sign-up works in Lovable preview but breaks on Vercel? Almost always means the Supabase env vars (`VITE_SUPABASE_URL` + the publishable-key env var, whether named `VITE_SUPABASE_PUBLISHABLE_KEY` or the legacy `VITE_SUPABASE_ANON_KEY`) didn't get copied into Vercel's Environment Variables panel. Re-check there. **The checklist (`m0-landing-page-checklist`) has a specific test for this — don't skip it.**

9. **Skipping Step 4.A (the Vite SPA conversion) and going straight to Vercel import.**
   Lovable's v1 often uses TanStack Start + `@lovable.dev/vite-tanstack-config` (Cloudflare-Workers-targeted SSR). If you let the student import that directly into Vercel, the build succeeds but every route 404s — and the student spends an hour debugging Vercel config thinking it's a framework preset issue. **Always run Step 4.A first** (the preemptive Lovable conversion prompt) before touching Vercel, even if the student says "I just want to deploy now." Default course path is Vite + Vercel, NOT TanStack Start + Cloudflare.

## Expected duration

A well-paced student should finish M0 in **30–60 minutes**. If they're past 90 minutes and still stuck, something is wrong — review the error with them step by step rather than letting them grind.

## Next step

After M0 checklist passes, tell the student:
> "M0 完成了！你的網站可以註冊、登入、登出了。下一步是 M1，把 AI 影片摘要 pipeline 接進來，讓登入後的使用者真的能上傳影片、拿到逐字稿。準備好的話跟我說『啟動 M1』。"

## Reference

- Lovable docs: https://docs.lovable.dev/introduction/getting-started
- Lovable + GitHub: https://docs.lovable.dev/integrations/git-integration
- Vercel import flow: https://vercel.com/docs/git/vercel-for-github

## TODO (filled in by future iterations)

- [ ] Add screenshots of the Lovable UI (Connect GitHub button location, deploy success view)
- [ ] Verify the Vercel framework preset auto-detection actually works for Lovable's default Vite output
- [ ] Add a fallback prompt for when Lovable's v1 output is visually broken (re-roll instructions)
- [ ] Decide whether to demo Lovable's "Publish" button vs. forcing students through GitHub→Vercel (some students may push back wanting the easier path)
