---
name: supabase-best-practice
description: Hard rules and operational SOP for working with Supabase in this course. Use whenever a student is modifying Supabase schema, querying tables, switching between local/remote, or wiring Supabase into a Lovable project. Sourced from a real production Lovable + Supabase project (orangebox-insider-marketplace).
---

# Supabase Best Practice

Battle-tested rules from running a real production Lovable + Supabase project (`uopsdod/orangebox-insider-marketplace`). Each rule has a real incident behind it — skipping any of them costs hours of debugging.

When you (Claude Code) are guiding a student through Supabase operations, **apply these rules proactively**. Don't wait for the student to ask. If you see them about to break one, stop and explain why.

---

## Execution mode: Cowork vs CLI

Course 2 supports two execution environments. The hard rules below apply in both, but the **mechanics** differ.

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Apply a migration | `supabase db push` (after `supabase link`) | `mcp__supabase_remote__apply_migration` (or `mcp__supabase_local__apply_migration` for local) |
| Inspect schema | `supabase db dump --schema public` / dashboard | `mcp__supabase_remote__list_tables`, `list_extensions`, `list_migrations` |
| Run a one-off SELECT | `supabase db execute "SELECT ..."` | `mcp__supabase_remote__execute_sql` |
| Get URL / publishable key | `supabase status --output json` | `mcp__supabase_remote__get_project_url`, `mcp__supabase_remote__get_publishable_keys` |
| Local dev stack (`supabase start`, docker volumes) | Native | **Not available in Cowork** — use the remote/branch project, or `mcp__supabase_remote__create_branch` for an ephemeral isolated environment |

**Cowork rule:** treat all `supabase` CLI invocations in this skill as **CLI-only**. Replace them with the equivalent MCP call from the table above. `supabase login`, `supabase link`, and `supabase start` have no Cowork equivalent — use the remote MCP directly (or a Supabase branch via MCP for isolated changes).

---

## Hard rules

### Rule 1 — Never modify Supabase schema with direct SQL — always use migration files

> **The rule:** All schema changes (CREATE TABLE / ALTER TABLE / CREATE INDEX / CREATE POLICY / CREATE FUNCTION / etc.) must be added as a new migration file under `supabase/migrations/` first. Apply the migration. Never run schema-modifying SQL directly in the Supabase dashboard or via `execute_sql` MCP for production changes.

**Why:** A schema change applied directly to remote Supabase has no record in git. Three weeks later, when you (or a Lovable session) try to reproduce the project from scratch, the missing schema causes silent runtime failures. The migration file is what makes the project reproducible — without it, "works on my laptop" becomes terminal.

**How to apply:** When a student / Lovable wants to change schema:
1. Write a new migration file with a timestamp prefix (`YYYYMMDDHHMMSS_<short_description>.sql`).
2. Commit it to git.
3. Apply it:
   - **CLI mode:** `supabase db push`
   - **Cowork mode:** `mcp__supabase_remote__apply_migration` (pass the migration name + SQL); for isolated testing, create a branch first via `mcp__supabase_remote__create_branch` and apply there.

Read-only one-off queries (SELECT) are fine to run directly — only schema mutations are gated.

---

### Rule 2 — Always look up existing RPC functions before creating new ones, and never query tables directly for joins

> **The rule:** For any new data-fetching requirement, first list existing RPC functions in the Supabase project and check whether one already covers the case. If yes, reuse. If no, create a new RPC. Do **not** build the feature as direct table queries from the client — especially for cross-table joins.

**Why:** Two problems with direct client-side queries:

1. **RLS silently filters cross-table joins.** Row Level Security policies apply per-table. A `.from('table_a').select('*, table_b(*)')` gets filtered on `table_a` *and* `table_b` independently — and if the user fails RLS on either side, the join returns empty with **no error**. The student sees "no data" and assumes the data doesn't exist, when actually RLS quietly killed the response. Half-day debugging exercise. RPCs run with `SECURITY DEFINER` and can bypass RLS in a controlled way.

2. **Lovable proliferation.** Left alone, Lovable creates a new RPC for every screen. You end up with `get_listings_v1`, `get_listings_v2`, `get_my_listings`, `get_listings_with_filter` — three of which return the same data. Schema changes have to be applied four places. The DB becomes ungrep-able.

**How to apply:** Before approving any data-fetch change:
1. Tell the student / Lovable to enumerate existing RPCs: `Supabase dashboard → Database → Functions`, or `mcp__supabase_*__list_extensions` + manual review.
2. If a reusable RPC exists → reuse it.
3. If not → create a new RPC. Migration-file gated (see Rule 1). Name it descriptively (`get_user_video_jobs`, not `get_data_v3`).
4. Never approve a client-side `.from(a).select('*, b(*)')` cross-table join — block it, suggest an RPC.

---

### Rule 3 — Verify "local or remote?" at the start of every Supabase-related session

> **The rule:** Before running any Supabase command (migration, query, schema change), ask the user / student: "Are we working against local Supabase or remote? Confirm." Do not assume.

**Why:** This project has two Supabase environments — `local` (dockerized, dev) and `remote` (production). Commands targeting one accidentally hit the other when the env vars or CLI link point at the wrong place. Applying a half-baked migration to remote prod because you assumed it was local is unrecoverable (or recoverable via painful rollback). The 30 seconds spent confirming saves a 2-hour incident.

**How to apply:** At the start of every new conversation that touches Supabase, ask:
> "我們現在是 local Supabase 還是 remote？確認一下。"

In this course, there are MCP tools for both: `mcp__supabase_local__*` and `mcp__supabase_remote__*`. Pick the right one based on the user's confirmation.

---

## Operational SOPs

### Local development startup

> **Cowork mode: skip this section entirely.** No docker, no `supabase start`. Use a remote Supabase branch (`mcp__supabase_remote__create_branch`) when you need an isolated test environment.

**CLI mode:**

```bash
SUPABASE_PROJECT_REF=<your-project-ref>     # e.g. aonhrhzuntjkskglqdwv
supabase login
supabase link --project-ref $SUPABASE_PROJECT_REF --debug
supabase start
```

This boots the local dockerized Supabase stack. Use it for any non-trivial schema change before mirroring to remote.

### How to clean up local Supabase (when local state goes stale)

> **CLI mode only** — Cowork has no local docker stack to clean.

```bash
docker volume ls --filter label=com.supabase.cli.project=<your-project-name>
docker volume rm <the-volumes-listed>
```

Then re-run `supabase start`. Useful when migrations get into a weird half-applied state, or when you want a clean slate without affecting remote.

### How to get the publishable key (formerly "anon key")

Supabase recently renamed the browser-safe key from "anon key" to **"publishable key"** (`sb_publishable_*`). They're the same role — RLS-gated, safe to ship in client code. Older docs / `supabase status` output may still call it `ANON_KEY`.

- **CLI mode:**
  ```bash
  supabase status --output json
  ```
  Extract `ANON_KEY` (legacy CLI output name) OR `PUBLISHABLE_KEY` (new) from the JSON — whichever is present.
- **Cowork mode:** call `mcp__supabase_remote__get_publishable_keys` (or `mcp__supabase_local__get_publishable_keys`). Returns the same value without a CLI roundtrip.

This value goes into your client env var, conventionally `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` or the legacy `NEXT_PUBLIC_SUPABASE_ANON_KEY` (either name works — it's just an env var name; pick one and apply consistently).

**Never confuse this with the secret / service_role key.** That one is server-only — exposing it in browser code = full bypass of RLS = total compromise.

---

## What this skill is NOT

- Not a Supabase tutorial — go to https://supabase.com/docs for that.
- Not RLS policy advice — RLS design is project-specific, not a global rule.
- Not a Supabase auth provider comparison — that's M1-specific (covered in `m1-*` skills when they exist).

## Source

These rules come from `uopsdod/orangebox-insider-marketplace/README_Supabase.md`. Each rule has a real incident behind it. Don't re-derive them by ignoring them.

## TODO (fill in after running M0 + M1 a few times)

- [ ] Add specific MCP tool usage examples once we know which tools are most-used in Course 2
- [ ] Document the local↔remote switching SOP in this project (already partly in `switch-to-local-mode` / `switch-to-remote-mode` commands — link them)
- [ ] Add Lovable-to-Supabase connection step (Lovable's "Connect Supabase" button) for M0
