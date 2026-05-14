---
name: m1-ai-video-transcript-checklist
description: Course 2 Milestone 1 verification — checks every artifact (Next.js conversion, jobs schema, /upload page, /api/jobs route, EC2 worker, end-to-end transcribe) is real and correctly wired. Use when the student says "驗收 M1", "check M1", "M1 done?", or after the `m1-ai-video-transcript` skill completes Step 7.
---

# M1 — AI Video Transcript Checklist

## What this skill does

Verifies the student has actually completed M1 — not just *thinks* they have. M1 has more moving parts than M0 (web ↔ Supabase ↔ EC2 ↔ OpenAI), so steps drift quietly. This checklist tests every artifact and reports pass/fail per item.

**Run this AFTER `m1-ai-video-transcript` Step 7, or any time the student claims M1 is done.**

![AI Video Reader architecture](assets/ai_video_reader_structure.jpg)

*The pieces this checklist verifies, working outward from Cowork (Claude Code): the GitHub repo (Token-pushed), Vercel-hosted Next.js (Connector), Supabase database (Connector / MCP), and the AWS EC2 worker (Connector / SSM) that runs OpenAI Whisper. Sections A–E below map onto these boxes.*

## Execution mode: Cowork vs CLI (read this first)

Every check in this skill goes through MCP — no SSH, no terminal required (consistent with the SSM-only path established in `m1-ai-video-transcript-prerequisites`).

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Schema | `supabase db execute` / dashboard | `mcp__supabase_remote__list_tables` / `execute_sql` |
| B — Web app | `curl` + `gh api` | `mcp__vercel__*` + `mcp__playwright__browser_*` if available; otherwise student paste-back |
| C — EC2 worker host | `aws ssm send-command ...` from local terminal | `call_aws ssm send-command ...` via AWS API MCP |
| D — End-to-end transcribe | `mcp__supabase_remote__execute_sql` + browser submit + `aws ssm send-command` log read | `mcp__supabase_remote__execute_sql` + browser submit + `call_aws ssm send-command` log read |
| E — Cleanup hygiene | `aws ec2 describe-instances` / `stop-instances` | `call_aws ec2 describe-instances` / `stop-instances` |

The conversational flow is the same in both modes — only the tool choices differ. **There is no SSH path** in either mode.

## How to run

This skill is meant to be invoked directly by the student (e.g. they type `驗收 M1`). You (Claude) **actively execute** each check via MCP — Supabase MCP for DB, Vercel MCP for the web side, AWS API MCP (`call_aws ssm send-command`) for the EC2.

### Step 1: Collect inputs from the student (one message)

Ask up front for these values:

1. GitHub repo URL (same as M0).
2. Vercel deploy URL (same as M0; should now serve the Next.js conversion).
3. Supabase project URL (same as M0; same project).

EC2 inputs are auto-discovered via tag filter (`Name=m1-worker`), so the student doesn't need to paste an instance ID or IP.

If any of these three URLs is missing → stop and refer to `m1-ai-video-transcript` Step 1 (for web URLs) or `m1-ai-video-transcript-prerequisites` (for AWS).

### Step 2: Preflight

Confirm the right MCPs are available:

- `mcp__supabase_remote__*` (required for Sections A + D)
- `mcp__vercel__*` (required for Section B)
- `call_aws` from AWS API MCP (required for Sections C + E)

Resolve the EC2 instance ID up front so later checks can reuse it:

```
INSTANCE_ID=$(call_aws ec2 describe-instances --filters "Name=tag:Name,Values=m1-worker" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].InstanceId' --output text)
```

If `INSTANCE_ID` is empty: the EC2 is stopped or the tag is missing. Start it (`call_aws ec2 start-instances --instance-ids <id>`) or go back to M1 prereq.

Confirm SSM can reach it:

```
call_aws ssm describe-instance-information --instance-information-filter-list "key=InstanceIds,valueSet=$INSTANCE_ID"
```

`PingStatus` must be `Online`. If not, the IAM instance profile is misconfigured — back to M1 prereq §2.2.

Confirm the four M1 secrets exist in Secrets Manager:

```
call_aws secretsmanager list-secrets --query 'SecretList[?Name==`openai-api-key` || Name==`supabase-url` || Name==`supabase-secret-key` || Name==`supabase-publishable-key`].Name'
```

Must return all four names. If any of the first three (`openai-api-key`, `supabase-url`, `supabase-secret-key`) is missing, the worker will crash on first poll with `ResourceNotFoundException`. If `supabase-publishable-key` is missing, the worker still runs (M1 doesn't read it) but follow-on milestones may break — re-create it via M1 prereq §2.3.

---

## Checklist

### Section A — Supabase schema (4 checks)

| # | Check | How to verify |
|---|---|---|
| A1 | `jobs` table exists with M1 columns | `mcp__supabase_remote__list_tables` returns a `jobs` row with at least: `id`, `user_id`, `video_source_url`, `topic`, `language`, `status`, `current_session_id`, `created_at`, `updated_at` |
| A2 | `job_sessions` table exists with M1 columns | Same MCP call returns a `job_sessions` row with at least: `id`, `job_id`, `session_number`, `subtitle_txt_content`, `created_at` |
| A3 | **Negative check — M1 stayed minimal.** None of the deferred columns exist | `mcp__supabase_remote__execute_sql`: `SELECT column_name FROM information_schema.columns WHERE table_schema='public' AND table_name IN ('jobs','job_sessions') AND column_name IN ('subtitle_srt_content','subtitle_vtt_content','subtitle_txt_content_reviewed','subtitle_srt_content_reviewed','subtitle_vtt_content_reviewed','error_message','reviewed_subtitle_path')`. **Must return zero rows.** If any row appears, the student copied a different (production) migration — re-apply M1's. |
| A4 | RLS is enabled on both tables | `mcp__supabase_remote__execute_sql`: `SELECT relname, relrowsecurity FROM pg_class WHERE relname IN ('jobs','job_sessions')`. Both `relrowsecurity` values must be `true`. |

If A3 fails: **stop**. The student likely cribbed the production schema, which has SRT/VTT/reviewed columns. M1 is intentionally smaller — re-apply the M1-only migration from `m1-ai-video-transcript` Step 2.

### Section B — Web app (Vercel + Next.js + auth-gated /upload + /api/jobs) (7 checks)

| # | Check | How to verify |
|---|---|---|
| B1 | Vercel deploy still returns HTTP 200 | `curl -sI <vercel-url> \| head -1` shows `HTTP/2 200`. (Cowork: `mcp__vercel__*` deployment query, or student opens the URL.) |
| B2 | Repo is now Next.js (not Vite) | `gh api repos/<owner>/<repo>/contents/package.json --jq '.content' \| base64 -d \| grep -E '"next"\|"vite"'` — must show `"next": "^16` (or similar). Must NOT show a `"vite":` dependency. (Cowork: open `package.json` in github.com.) |
| B3 | `/upload` is auth-gated | `curl -sI <vercel-url>/upload \| head -3` returns either a 200 (with login form rendering) or a 30x redirect to `/sign-in`. Manually open in incognito → confirms it kicks back to sign-in. |
| B4 | `POST /api/jobs` rejects unauthenticated requests | `curl -s -X POST <vercel-url>/api/jobs -H 'Content-Type: application/json' -d '{"video_source_url":"x"}'` returns `{"error":"unauthorized"}` with HTTP 401. |
| B5 | `POST /api/jobs` rejects missing body field | Same curl but with `-d '{}'` (signed in via cookie — easiest is to just trust the student's manual browser test) returns 400 with `video_source_url required`. Skip this if simulating an authenticated curl is too painful; the form-validation in Step 3 already catches empty submits. |
| B6 | `/upload` jobs table has a Transcript column | `gh api repos/<owner>/<repo>/contents/app/upload/page.tsx --jq '.content' \| base64 -d \| grep -i 'transcript'` — must match the header literal "Transcript" AND a string like `/api/jobs/` (the download link target). (Cowork: open `app/upload/page.tsx` on github.com and visually confirm both.) The student's first AI-generated pass often omits this column entirely; this check catches that. |
| B7 | `GET /api/jobs/[id]/transcript` route exists and gates auth | `curl -sI <vercel-url>/api/jobs/00000000-0000-0000-0000-000000000000/transcript` — must return 401 (unauthorized) or 404 (not found), NEVER 200 or a redirect. A 405 (method not allowed) means the route file is missing — re-run Step 4b. |

If B1 fails: Vercel deploy is broken. Check the most recent deployment log in Vercel dashboard — most likely cause is the Vite→Next conversion left build errors.

If B2 still shows Vite: the Vite→Next.js conversion didn't take. Re-run Claude Code in the cloned repo per `m1-ai-video-transcript` Step 1, then commit + push.

### Section C — EC2 worker host (7 checks, all via SSM)

All Section C checks run as one consolidated `send-command` per check (via `call_aws` in Cowork or `aws` CLI in CLI mode). Read each command's output via `call_aws ssm list-command-invocations --command-id <id> --details --query 'CommandInvocations[0].CommandPlugins[0].Output' --output text`.

| # | Check | How to verify |
|---|---|---|
| C1 | EC2 reachable via SSM | `call_aws ssm send-command --instance-ids "$INSTANCE_ID" --document-name AWS-RunShellScript --parameters '{"commands":["echo reachable"]}'` returns `reachable` after polling for the result. (If `send-command` itself fails with `InvalidInstanceId`, the instance isn't SSM-managed — back to M1 prereq.) Use the JSON form, not `'commands=[...]'` shorthand — the shorthand parser chokes on brackets through the MCP shell layer. |
| C2 | ffmpeg installed | `commands=["ffmpeg -version \| head -1"]` returns a version line starting with `ffmpeg version`. |
| C3 | Python 3.12 + venv ready | `commands=["ls /home/ubuntu/app/worker/venv/bin/python && /home/ubuntu/app/worker/venv/bin/python --version"]` returns the python path and `Python 3.12.x`. |
| C4 | Worker code + service file present | `commands=["ls /home/ubuntu/app/worker/worker.py /home/ubuntu/app/worker/distributor.py /home/ubuntu/app/worker/requirements.txt /home/ubuntu/app/worker/m1-distributor.service && [ ! -f /home/ubuntu/app/worker/.env ] && echo 'no .env (good)' \|\| echo 'WARNING: .env exists, secrets should live in AWS Secrets Manager, not on disk'"]` lists all four worker files **and** confirms there is no `.env` file (M1 reads secrets from AWS Secrets Manager via `boto3`; a `.env` on disk indicates the student copy-pasted from an older version of the skill). |
| C5 | Distributor service running | `{"commands":["sudo systemctl is-active m1-distributor.service","sudo tail -10 /var/log/m1-distributor.log"]}` returns `active` followed by the last 10 worker-log lines. **Read `/var/log/m1-distributor.log`, NOT `journalctl`** — the unit file redirects stdout/stderr to that file, so journalctl only shows lifecycle events (Started/Stopped). The log should include a recent `distributor: polling every 10s` or `spawned worker for job ...` line (proof the loop is alive, not just the process started). |
| C6 | Unit file has `Environment=PATH=` including the venv bin dir | `{"commands":["grep '^Environment=PATH' /etc/systemd/system/m1-distributor.service"]}` must return a line containing `/home/ubuntu/app/worker/venv/bin`. Without this, systemd's default PATH (`/usr/bin:/bin`) hides `yt-dlp`, and every job dies with `FileNotFoundError: 'yt-dlp'` — visible in `/var/log/m1-distributor.log` but NOT in `journalctl`, and NOT caught by C5 (the service is still "active"). |
| C7 | Unit file's `AWS_DEFAULT_REGION` matches this EC2's actual region | `{"commands":["grep '^Environment=AWS_DEFAULT_REGION' /etc/systemd/system/m1-distributor.service"]}` returns the region. Cross-check against `call_aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' --output text` (strip the trailing letter). They must match. Mismatch → `NoRegionError` on first Secrets Manager call → worker can't authenticate. |

If C5 fails but C1–C4 pass: the service is installed but not running. Start it via SSM: `{"commands":["sudo systemctl start m1-distributor.service"]}`. If it crashes immediately, read `/var/log/m1-distributor.log` (most-likely causes: `AWS_DEFAULT_REGION` wrong → see C7; IAM role missing the `m1-secrets-read` inline policy granting `secretsmanager:GetSecretValue`; one of the three secret names — `openai-api-key`, `supabase-url`, `supabase-secret-key` — not yet created in Secrets Manager).

If C6 fails: edit `worker/m1-distributor.service` in the repo to include `Environment=PATH=/home/ubuntu/app/worker/venv/bin:/usr/local/bin:/usr/bin:/bin`, push, then re-run Step 6a + 6b in the main skill (the EC2 needs to `cp` the updated unit into `/etc/systemd/system/` and `daemon-reload`).

If C7 fails: same fix loop as C6 — patch the region in the unit file in the repo, push, re-run 6a + 6b.

### Section D — End-to-end transcribe (4 checks)

This is the actual functional test — submit a real job, watch it walk to `done`, verify the output.

Tell the student:

> 「請開 `<vercel-url>/upload`（用 incognito 視窗 + 重新登入）。在 Video URL 貼一段 30 秒到 1 分鐘的測試影片（YouTube 短片或一個直接的 .mp4 連結都可以；可以用 https://www.youtube.com/watch?v=jNQXAC9IVRw 這支 19 秒的測試影片）。Topic 隨便填、Language 選對應的語言。按 Transcribe。完成跟我說『我送出了』。」

When the student confirms:

| # | Check | How to verify |
|---|---|---|
| D1 | Job row created with `status='pending'` | `mcp__supabase_remote__execute_sql`: `SELECT id, status, video_source_url, created_at FROM jobs ORDER BY created_at DESC LIMIT 1`. The `created_at` must be within the last 2 minutes; status starts at `pending`. |
| D2 | Within 10–20 s, status flips to `downloading` | Re-run the query. If still `pending`, the distributor isn't picking up — go back to C5. |
| D3 | Within 1–3 minutes, status reaches `done` | Re-run the query every 30 s. Walks `pending → downloading → transcribe → done`. If it stops at `downloading` or `transcribe`, read the worker log via SSM: `call_aws ssm send-command --instance-ids "$INSTANCE_ID" --document-name AWS-RunShellScript --parameters '{"commands":["tail -50 /var/log/m1-distributor.log"]}'`. Most common errors: `FileNotFoundError: 'yt-dlp'` (C6 — PATH missing), `NoRegionError` (C7 — region mismatch), `OPENAI_API_KEY` typo in Secrets Manager, `ffmpeg` missing, or **the student tried a YouTube URL** (YouTube rate-limits cloud IPs — use Internet Archive / Wikimedia / a direct .mp4 URL instead). |
| D4 | The transcript is plausible | `mcp__supabase_remote__execute_sql`: `SELECT length(subtitle_txt_content) AS chars, left(subtitle_txt_content, 200) AS preview FROM job_sessions WHERE job_id = (SELECT id FROM jobs ORDER BY created_at DESC LIMIT 1)`. `chars` should be > 50; the preview should look like real text in the language the student selected (not English when `zh` was chosen, etc.). |
| D5 | Transcript column on `/upload` shows a working download link for the done job | Manually: the student opens `/upload` signed in and clicks the `.txt` link in the Transcript column for the just-completed job. A file named `transcript-<short>.txt` should download, opening to readable text matching D4's preview. If the click does nothing or downloads an error JSON, B6/B7 are misconfigured even though the backend is fine. |

If D4 produces gibberish or wrong language: the student probably submitted a `language` that doesn't match the audio. Whisper is forgiving but not magic. Re-submit with the correct language.

If D5 fails but D4 passes: the transcript exists in the DB but the user can't get to it. Re-check Step 3 (Transcript column wiring) and Step 4b (GET handler).

### Section E — Cleanup hygiene (2 checks)

Not strictly functional — but the student needs to understand cost.

| # | Check | How to verify |
|---|---|---|
| E1 | Student knows how to stop the EC2 | Ask: 「上完課要怎麼把 EC2 停掉？」 Acceptable answer: `call_aws ec2 stop-instances --instance-ids "$INSTANCE_ID"` (or the same via `aws` CLI / AWS console — but the MCP one-liner is the canonical course answer since the rest of the milestone uses MCP). Wrong answer: "I'll just leave it running" — explain the ~\$15/mo cost. Bonus: confirm by running it together and watching `call_aws ec2 describe-instances ... --query 'Reservations[0].Instances[0].State.Name'` flip to `stopping` then `stopped`. |
| E2 | OpenAI key has billing limits set | Open https://platform.openai.com/settings/organization/limits — confirm there's a soft limit (e.g. \$10/month) so a runaway worker doesn't drain the credit balance. Not technically required to pass M1 but strongly recommended; mark ⚠️ if missing rather than ❌. |

---

## Reporting

After running all checks, output a results table:

```
M1 Checklist Results
====================

Section A — Supabase schema
  A1 jobs table with M1 columns        ✅
  A2 job_sessions with M1 columns      ✅
  A3 No SRT/VTT/reviewed columns       ✅
  A4 RLS enabled on both               ✅

Section B — Web app
  B1 Vercel HTTP 200                   ✅
  B2 Repo is Next.js (not Vite)        ✅
  B3 /upload is auth-gated             ✅
  B4 POST /api/jobs rejects no auth    ✅
  B5 POST /api/jobs validates body     ✅
  B6 /upload has Transcript column     ✅
  B7 GET transcript route exists+auth  ✅

Section C — EC2 worker host
  C1 EC2 reachable via SSM             ✅
  C2 ffmpeg installed                  ✅
  C3 Python 3.12 venv ready            ✅
  C4 Worker code + service file + no on-disk .env  ✅
  C5 m1-distributor.service is active  ✅
  C6 Unit PATH includes venv bin       ✅
  C7 Unit region matches EC2 region    ✅

Section D — End-to-end transcribe
  D1 Job row created (pending)         ✅
  D2 Status flips to downloading       ✅
  D3 Status reaches done               ✅
  D4 Transcript is plausible           ✅
  D5 Transcript download link works    ✅

Section E — Cleanup hygiene
  E1 Knows how to stop EC2             ✅
  E2 OpenAI billing limit set          ⚠️  (recommended, not required)

=====================================
Verdict: 24/25 pass, 1 advisory
M1 status: READY for M2
```

**Pass criteria:** all ❌ resolved. ⚠️ items are advisory and don't block.

If any ❌ exists: M1 is **NOT** done. Loop back into `m1-ai-video-transcript` at the corresponding step.

If all ✅: green-light M2:

> 「M1 全綠燈通過 — 你的 SaaS 真的會跑 Whisper 了。準備好的話跟我說『啟動 M2』。」

## Why this checklist exists

Two M1-specific failure modes this guards against:

1. **The "I think it's done" failure** — student submitted a job, saw the page refresh, but never actually checked the transcript landed. Without D4, M2 (Stripe) will be metering jobs that produced no output.
2. **The "schema drift" failure** — student copy-pasted the wrong migration (e.g. cribbed from this repo's production schema) and got SRT/VTT/reviewed columns they don't need. Section A3 catches this before it propagates into M2's Stripe-credit-deduction logic.

## TODO

- [ ] Wrap the C2–C5 SSM-send-command pattern into a single composite check with a polling helper, instead of one `send-command` per row, to cut latency.
- [ ] Tighten D4 from "looks like text in the right language" into a deterministic language-detect call.
- [ ] Add a check that grep'ing the repo for `sk-proj-` / `sk-` / `eyJhbGciOi` returns zero hits (defense-in-depth — secrets should only live in AWS Secrets Manager, never in committed code).
