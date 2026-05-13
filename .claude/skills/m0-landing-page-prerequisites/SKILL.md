---
name: m0-landing-page-prerequisites
description: One-time CLI + MCP setup the student needs BEFORE starting M0 (Course 2 Milestone 0). Splits by execution mode — Cowork (MCP-only, no local shell) vs CLI (local terminal with `gh` / `vercel` / `supabase`). Use when the student is about to start M0 for the first time, or when the `m0-landing-page` / `m0-landing-page-checklist` skill detects a missing CLI / MCP and refers the student back here.
---

# M0 Prerequisites — CLI + MCP Setup

## Execution mode: Cowork vs CLI (read this first)

Course 2 supports two execution environments. Confirm which one the student is using **before** doing anything else, and keep applying the right column for the rest of M0–M5.

| | **Cowork mode** | **CLI mode** |
|---|---|---|
| What it is | Claude Code running in the hosted Cowork environment — no local shell, no `brew`/`npm install -g`, no local files | Claude Code running on the student's own laptop with a real terminal |
| How to do GitHub | GitHub MCP if available, else GitHub web UI / Lovable's built-in Git panel | `gh` CLI |
| How to do Vercel | `mcp__vercel__*` (required) | `mcp__vercel__*` preferred, `vercel` CLI fallback |
| How to do Supabase | `mcp__supabase_remote__*` / `mcp__supabase_local__*` (required) | MCP preferred, `supabase` CLI fallback |
| Interactive `*-login` browser flows | Skip — auth comes via the MCP install (OAuth in the Cowork UI) | Student runs them in their own terminal |
| Sanity-check commands like `gh auth status`, `vercel whoami`, `supabase projects list` | **Skip** — they don't exist here. Verify by listing available MCP tools instead | Required at the end of this skill |

**Ask the student up front:** 「你是用 Cowork 還是本機 CLI 跑 Claude Code？」

- **Cowork** → skip everything in the "Install CLIs by OS" and "Login (student runs these)" sections. Go straight to "Install MCP servers" and the Cowork-mode verify step.
- **CLI** → do the whole skill end-to-end.

If the student is unsure, ask: "你在 Lovable 旁邊有沒有開一個 terminal 視窗、可以 `brew install` 東西？" Yes = CLI. No / "我都在瀏覽器裡" = Cowork.

## What this skill does

Installs and logs in the three CLIs (`gh`, `vercel`, `supabase`) and configures the two MCP servers (Vercel, Supabase) that the M0 implementation skill (`m0-landing-page`) and verification skill (`m0-landing-page-checklist`) need to work end-to-end.

**Run this ONCE before starting M0.** Subsequent milestones (M1–M5) reuse the same tools, so this skill doesn't need to be re-run between milestones.

## When to load this skill

Trigger phrases:
- "M0 環境準備"
- "setup CLIs for M0"
- "install course 2 tools"
- Any time the student starts M0 (`啟動 M0`) and you (Claude Code) detect missing CLIs / MCPs during the checklist preflight

Or auto-loaded when `m0-landing-page` reaches its Step 0 reference / when `m0-landing-page-checklist` preflight fails on a missing tool.

## Skip check (do this first)

Ask the student: 「你之前用過 `gh`、`vercel`、`supabase` CLI 嗎？都登入過了嗎？另外你 Claude Code 有裝 Vercel + Supabase 的 MCP server 嗎？」

If yes to all → skip this skill, mention it can always be re-run later if any tool breaks.

If no / partial → walk through the sections below for whatever is missing.

## Tool priority (prefer MCP > CLI)

| Service | Preferred | Fallback |
|---|---|---|
| GitHub | `gh` CLI (no MCP equivalent yet) | — |
| Vercel | `mcp__vercel__*` (if Vercel MCP installed in Claude Code) | `vercel` CLI |
| Supabase | `mcp__supabase_remote__*` (if Supabase MCP installed in Claude Code) | `supabase` CLI |

**Install both MCPs (preferred) and CLIs (fallback)** so the checklist works regardless of which is available in a given session. CLIs are also useful in their own right (e.g. `vercel env add` is the easy way to add env vars).

## Install CLIs by OS

> **Cowork mode: skip this entire section.** No local shell, nothing to install. Jump to "Install MCP servers" below.

**Detecting the student's OS:** if you (Claude Code) don't already know, ask: "你用的是 macOS、Windows、還是 Linux？" Then paste the relevant block below.

### macOS (Homebrew)

```bash
brew install gh
brew install vercel-cli
brew install supabase/tap/supabase
```

### Windows (winget / Scoop)

```powershell
# Using winget (Windows 10+)
winget install --id GitHub.cli
winget install --id Vercel.Vercel
winget install --id Supabase.cli

# Or using Scoop
scoop install gh
scoop install vercel
scoop install supabase
```

### Linux (Debian/Ubuntu)

```bash
# gh
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install gh

# vercel
npm install -g vercel

# supabase
curl -fsSL https://github.com/supabase/cli/releases/latest/download/supabase_linux_amd64.tar.gz | sudo tar -xz -C /usr/local/bin supabase
```

## Login (student runs these)

> **Cowork mode: skip this entire section.** No CLIs to log in. The Cowork-installed MCP servers handle auth via the Cowork OAuth UI, not via terminal browser flows.

After install, each CLI needs login. **The student must run these themselves** in their own terminal — these are interactive flows that open a browser, and you (Claude Code) cannot complete them inside a Bash tool call.

Tell the student to run these one at a time (suggest using `! <command>` in the prompt to run inline in their Claude Code session):

```bash
gh auth login                  # follow the browser prompt; pick HTTPS + browser auth
vercel login                   # opens browser for email-link login
supabase login                 # opens browser for OAuth login (only if not using Supabase MCP)
```

After each login, the student should see a "Logged in as ..." confirmation. Ask them to paste it back so you can confirm before moving on.

## Install MCP servers (preferred over CLI for Vercel + Supabase)

Vercel and Supabase both have MCP servers that Claude Code can use natively — no CLI roundtrips, no auth re-prompts mid-checklist. **Install these if at all possible**, then keep the CLIs as fallback.

Install instructions:

- **Vercel MCP:** follow the Vercel plugin / MCP setup in your Claude Code settings. The official Vercel plugin (`vercel-plugin`) bundles MCP tools. See https://vercel.com/docs (search "MCP" / "Claude Code") for the latest install path.
- **Supabase MCP:** Supabase ships official MCP servers. Add `mcp__supabase_remote__*` (for the remote project) to `~/.claude.json` or via `claude mcp add`. See https://supabase.com/docs/guides/getting-started/mcp for the latest install path.

After installing, **restart Claude Code** for the new MCP tools to appear in the tool list.

## Verify

### CLI mode

Once the student confirms all three logins and (optionally) the MCPs are installed, sanity-check with:

```bash
gh auth status
vercel whoami
supabase projects list 2>/dev/null || echo "supabase CLI not logged in (OK if using MCP)"
```

All three CLIs should return account info. If any errors, walk through the failing one before continuing.

### Cowork mode

There are no CLI commands to run. Verify by listing the available MCP tools in the Cowork tool list:

- `mcp__vercel__*` — at least one tool present → Vercel MCP wired
- `mcp__supabase_remote__*` (or `mcp__supabase_local__*`) — at least one tool present → Supabase MCP wired
- GitHub: if a GitHub MCP is installed in Cowork, look for its tools. If not, the student will use Lovable's Git panel + the GitHub web UI for the few GitHub steps in M0 (no `gh` CLI needed).

If `mcp__vercel__*` or `mcp__supabase_*__*` tools are missing, the MCP isn't installed — the student needs to add it via the Cowork MCP/plugin UI (not via `~/.claude.json`, which doesn't apply in Cowork). Pause M0 until both are present.

## When done

Tell the student:

> 「環境準備完成。可以開始 M0 了 — 跟我說『啟動 M0』。」

## Why this exists as its own skill

- **Reusability:** M1–M5 use the same tools. Splitting this out means students don't see CLI setup mixed with M0-specific steps.
- **Recoverability:** if a CLI or MCP breaks mid-milestone (token expired, MCP server crashed), the student can re-run just this skill instead of re-reading the entire M0 doc.
- **Smaller M0 skill:** the M0 implementation skill now stays focused on Lovable/Supabase/GitHub/Vercel application steps, not infrastructure plumbing.
