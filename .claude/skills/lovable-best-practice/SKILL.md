---
name: lovable-best-practice
description: Hard rules and workflow tips for working with Lovable in this course. Use whenever a student is generating, re-rolling, or editing a Lovable project — applies to any Lovable session through the course. Sourced from a real production Lovable + Supabase project (orangebox-insider-marketplace).
---

# Lovable Best Practice

This skill is loaded any time the student is interacting with Lovable. It lives at two levels:

1. **Hard rules** — non-negotiable, learned from a real production Lovable + Supabase project (the `orangebox-insider-marketplace` repo). Each rule has a real incident behind it. Skipping these costs the student hours of debugging later.
2. **General workflow tips** — Vercel framework preset gotchas, re-roll discipline, git commit habits.

When you (Claude Code) are guiding a student through Lovable steps, **apply these rules proactively** — don't wait for the student to ask. If you see the student about to break one, stop them and explain why.

---

## Execution mode: Cowork vs CLI

Course 2 supports two execution environments. Lovable itself works the same way in both (it's a hosted web app), but the **verification + git steps around Lovable** differ.

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Confirm a Lovable iteration synced to GitHub | `gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message'` | GitHub MCP (if installed), Lovable's built-in Git panel, or open `https://github.com/<owner>/<repo>/commits` in a browser |
| PII grep over changed files (Rule 3) | `grep -rE '@(gmail|yahoo|hotmail|outlook)\.com' .` locally | No local checkout — review the Lovable diff in the Lovable UI before saving, or open the diff in GitHub web after it pushes |
| Edit Lovable's generated files directly | Use Claude Code Edit/Write on local clone | Edit through Lovable's own editor; there's no local working copy to patch |

The **hard rules** (Rules 1–4) apply identically in both modes — only the verification mechanics change. Wherever this skill says `gh ...`, treat that as **CLI-only** and substitute the Cowork equivalent above.

---

## Hard rules (apply to every Lovable session in this course)

These came from real incidents on a multi-page Lovable + Supabase project. They become relevant the moment the student adds Supabase in M1 — but state them in M0 so the student already has the right mental model when they get there.

### Rule 1 — Reuse existing Supabase RPC functions before creating new ones

> **The rule:** Whenever a Lovable feature needs to fetch data from Supabase, first list the existing RPC functions in the Supabase project and consider reusing one. Only create a new RPC if no existing function fits.

**Why:** Lovable, left alone, will happily create a new RPC for every screen it builds. You end up with `get_listings_v1`, `get_listings_v2`, `get_listings_with_filter`, `get_all_listings` — three of which return the same data. The DB becomes ungrep-able, and any schema change has to be applied four places instead of one.

**How to apply:** Before approving any Lovable change that touches Supabase, ask the student:
- "Lovable 想新建一個叫 `<name>` 的 RPC — 你的專案裡有沒有已經能做這件事的 RPC？查一下 `Supabase dashboard → Database → Functions`。"
- If yes → tell Lovable to reuse it.
- If no → only then approve the new RPC.

This rule doesn't apply in M0 (no Supabase yet), but say it once during M0 so the student has the principle before they need it in M1.

---

### Rule 2 — Use RPC for any cross-table query — NOT direct table queries

> **The rule:** If a feature needs data from two or more Supabase tables (a JOIN), build it as an RPC. Do not let Lovable build it as a direct multi-table query from the client.

**Why:** Supabase Row Level Security (RLS) silently filters rows the current user doesn't have access to. Direct cross-table queries from the client get filtered at every table, and the join result drops to empty — but **no error is thrown**. The student sees "no data" and assumes the data isn't there, when actually RLS quietly killed the response. Debugging this can take half a day. RPCs run with `SECURITY DEFINER` and bypass RLS in a controlled way, so the join actually works.

**How to apply:** When reviewing Lovable's diff:
- Look for `.from('table_a').select('*, table_b(*)')` — that's a client-side join, will hit RLS silently. Block it.
- Replace with `.rpc('get_my_data', { ... })` where the RPC does the join server-side.
- This applies from M1 onward.

---

### Rule 3 — Never commit PII (email, real names, addresses) in code or test data

> **The rule:** When Lovable generates demo data, mock users, or feature copy that includes example emails / names, replace them with obvious fakes (`user@example.com`, `Test User`) before committing.

**Why:** Lovable will sometimes auto-fill demo data with whatever email is signed into your account, or copy a real-looking email it saw in your prompt. Once that hits the GitHub repo, it's permanently in git history — even if you delete the file, anyone with a clone has it. For a course product you'll eventually share publicly, this is a compliance and trust issue.

**How to apply:** Before any commit triggered by Lovable, do a PII scan on changed files:
- **CLI mode:** `grep -rE '@(gmail|yahoo|hotmail|outlook)\.com' .` on the local checkout.
- **Cowork mode:** there's no local checkout — review the diff in Lovable's UI before saving, or open the just-pushed commit in the GitHub web UI and scan the diff there.

If it finds real-looking emails, swap to `example.com` placeholders first.

---

### Rule 4 — Filenames use PascalCase for components (CamelCase enforced)

> **The rule:** React component files in the project must be named `PascalCase.tsx` — e.g. `CreateListingForm.tsx`, `VideoUploader.tsx`. Not `create-listing-form.tsx`, not `createListingForm.tsx`.

**Why:** Lovable's default file-naming is inconsistent — it sometimes generates kebab-case, sometimes camelCase, sometimes PascalCase, occasionally mixed in the same project. Once the project gets to 30+ files, finding `<CreateListingForm />` in a folder of `create-listing-form.tsx` / `createListingForm.tsx` / `CreateListingForm.tsx` becomes a real productivity tax. Pick one and tell Lovable to enforce it.

**How to apply:** When Lovable creates a new file:
- Component files (anything that exports a React component): `PascalCase.tsx`
- Non-component files (utilities, hooks, types): keep Lovable's default (usually camelCase like `useAuthState.ts`)
- If Lovable creates a kebab-case `.tsx` file, tell it: "rename to PascalCase to match convention."

---

## General Lovable workflow tips

These apply to any Lovable session in this course, not just one milestone.

### Tip 1 — Vercel framework preset auto-detection sometimes picks wrong

Lovable's default output is Vite + React. Vercel's auto-detect usually catches that, but ~10% of the time it picks "Other" or guesses Next.js, and the deploy fails with a cryptic build error.

**If a Vercel deploy fails right after Lovable connects to GitHub:** go to Vercel project → Settings → General → Framework Preset → manually set to **Vite**. Redeploy.

### Tip 2 — Free tier credit budget — re-roll discipline

Lovable's free tier limits you to a small number of generations per day (subject to change; verify current quota in your account). Each "fix this and regenerate" eats one. Students who don't think before prompting can burn the daily budget in 15 minutes and then be stuck for 24 hours.

**Discipline:**
- First prompt should be the full prompt verbatim from the relevant milestone skill (e.g. `m0-landing-page` Step 1). Don't try shorter versions first.
- If v1 has small issues (wrong color, missing copyright text), use Lovable's "edit in place" features rather than full regeneration where available.
- If you need to regenerate, write down all the changes you want in a single follow-up prompt — don't do 3 small re-rolls.

### Tip 3 — Always commit to GitHub before iterating in Lovable

Lovable two-way syncs with GitHub once connected. But if Lovable's next generation overwrites something you liked, you want a git commit to roll back to.

**Habit:** after every Lovable iteration the student is happy with, the GitHub repo should have a commit. Lovable usually does this automatically once connected, but verify the latest commit message matches the iteration:
- **CLI mode:** `gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message'`
- **Cowork mode:** open `https://github.com/<owner>/<repo>/commits` in the browser, or use the GitHub MCP if installed; Lovable's own Git panel also shows the sync status.

---

## What this skill is NOT

- Not a course-3 / course-4 Lovable tutorial — only applies to Course 2 (Video Speed Reader).
- Not a replacement for `m0-landing-page` skill — that one has the actual step-by-step. This skill provides the *rules* the step-by-step assumes.
- Not Lovable documentation — go to https://docs.lovable.dev for that. This is opinionated guidance for this specific course.

## Source

- Hard rules 1–4 came from `uopsdod/orangebox-insider-marketplace`'s `README_Lovable.md`. They're real lessons from a multi-page Lovable + Supabase project. Each rule has a real incident behind it; don't re-derive them by ignoring them.
- M0 tips come from official Lovable docs + Course 2 design choices.

## TODO (fill in after running M0 a few times)

- [ ] Update M0-T4 with the actual current Lovable free-tier limit once verified
- [ ] Add M0-T7+ for any new pitfalls discovered while running M0 with the first cohort
- [ ] Add specific Lovable prompt wording that reliably gets v1 without a login form on the first try
