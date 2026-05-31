---
name: m4-serverless-prerequisites
description: One-time setup the student needs BEFORE starting M4 (Course 2 — go serverless, Lambda distributor + Fargate worker, image built by CodeBuild, NO EC2 dependency). Confirms all 4 Cowork connectors are still authenticated, confirms M1's EC2 worker actually works (M4 has nothing to migrate without it), checks the AWS account can reach ECR/ECS/Lambda/EventBridge/CodeBuild, resolves GitHub push + CodeBuild source auth, and covers the cost model. No new external accounts — M4 reuses the M1 AWS account. Use when the student says "M4 環境準備", "setup for M4", or when `m4-serverless` / `m4-serverless-checklist` detects a missing connector, a broken M1 worker, or unresolved build auth.
---

# M4 Prerequisites — serverless readiness + CodeBuild/GitHub auth

## What this skill does

Confirms the student is ready to start M4 (the **Lambda distributor + Fargate worker** design, with the Docker image built by **AWS CodeBuild** from the GitHub repo — **no EC2 in the build path or the run path**). Like M3, **M4 adds zero new external accounts and zero new Cowork connectors** — ECR, ECS Fargate, Lambda, EventBridge, and CodeBuild are all services inside the same AWS account from M1 prerequisites.

By the end of this skill the student has:

1. Confirmed all **4 Cowork connectors** (Supabase, Vercel, AWS, Stripe) are still authenticated and returning data.
2. Confirmed **M1's EC2 worker actually works** — M4 moves the *same* worker pipeline into a Fargate container. If M1 never ran a job successfully, there's nothing to migrate; fix M1 first.
3. Confirmed the AWS IAM user can reach **ECR / ECS / Lambda / EventBridge / CodeBuild** (not just EC2 from M1).
4. **Resolved GitHub auth** for both the push (the **required build source** CodeBuild checks out — not optional) and the CodeBuild→GitHub connection (`import-source-credentials` with a **`repo` + `admin:repo_hook`** PAT, **mandatory even for public repos** because the push webhook needs it) — and derived their own repo URL.
5. Understood the M4 **cost model** ($0 at idle, pay-per-job).

**Why no EC2 anywhere:** M4's whole point is to remove the always-on EC2. So the worker image is **not** built on the EC2 (that would make the new system depend on the box it replaces) — it's built by **CodeBuild**, on demand, in the cloud. The student needs **no local Docker** and the EC2 can be terminated at the end.

## Required external accounts (cumulative — unchanged from M3)

| # | Service | Used for | Added in |
|---|---|---|---|
| 1 | GitHub | Source control — **+ CodeBuild source (M4)** | M0 |
| 2 | Supabase | Auth + Postgres + RLS | M0 |
| 3 | Vercel | Auto-deploy Next.js | M0 |
| 4 | OpenAI | Whisper (`whisper-1`) | M1 |
| 5 | AWS | EC2 (M1) → **ECR + ECS Fargate + Lambda + EventBridge + CodeBuild (M4)** | M1 |
| 6 | Stripe (sandbox) | Checkout Sessions + webhook | M2 |

M4 uses five new AWS *services* but no new *account*. No new row.

## Required Cowork connectors / MCPs (read this first)

All 4 connectors from M0–M3 must still be authenticated. **M4 leans hardest on `call_aws`** — ECR, ECS, Lambda, EventBridge, CodeBuild, and IAM are all driven through it. Supabase is needed for the migration + checklist. Vercel/Stripe aren't changed by M4 but must stay green.

| # | Connector / MCP | Added in | Used in M4 for | How to verify |
|---|---|---|---|---|
| 1 | `call_aws` / `suggest_aws_commands` (AWS API MCP) | M1 | ECR, ECS, Lambda, EventBridge, CodeBuild, IAM — the bulk of M4 | `call_aws sts get-caller-identity` — must return an account/ARN |
| 2 | `mcp__supabase_remote__*` | M0 | `fargate_task_arn` migration + checklist queries | `mcp__supabase_remote__execute_sql`: `SELECT 1` — must return `1` |
| 3 | `mcp__vercel__*` | M0 | Not changed by M4, but the app must stay green | `mcp__vercel__list_projects` — must return the M0 project |
| 4 | `mcp__stripe__*` | M2 | Not used in M4, but M2/M3 must remain green | `mcp__stripe__list_prices` — `livemode: false` rows or empty |

**CLI fallback:** `aws sts get-caller-identity` · `supabase db execute` · `vercel projects ls` · `stripe prices list`.

### What if a connector has expired?

| Connector | How to re-authenticate |
|---|---|
| AWS API MCP | The AWS API MCP reads `~/.aws/credentials` — there is **no MCP settings field**. If the access key was deleted/rotated, create a new one and write it to `~/.aws/credentials` by running `claude` in a terminal and pasting the credential there, then **restart Cowork**. See [[m1-ai-video-transcript-prerequisites]] §2.0.3. |
| Supabase MCP | Cowork: reconnect via Connectors panel. CLI: `supabase login`. |
| Vercel MCP | Cowork: reconnect via Connectors panel. CLI: `vercel login`. |
| Stripe MCP | Cowork: reconnect via Connectors panel (re-authorize OAuth — **pick sandbox, not live**). CLI: `stripe login`. |

Fix all 4 before proceeding.

## Section 1 — Verify all 4 connectors

**Check 1 — AWS:** `call_aws sts get-caller-identity` — returns the M1 account `Account` + `Arn`. If `InvalidClientTokenId`: the key was rotated — re-write `~/.aws/credentials` via the `claude` CLI + restart Cowork (no MCP settings field; see [[m1-ai-video-transcript-prerequisites]] §2.0.3). If `AccessDenied`: see Section 3.

**Check 2 — Supabase:** `mcp__supabase_remote__execute_sql: SELECT current_timestamp;` — returns a timestamp.

**Check 3 — Vercel:** `mcp__vercel__list_projects` — returns the M0 project.

**Check 4 — Stripe:** `mcp__stripe__list_prices` — `livemode: false` (or empty). If `livemode: true`, re-authorize against sandbox.

| All 4 pass | → Section 2 |
|---|---|
| Any fail | → fix per the table above, re-run |

## Section 2 — Confirm M1 actually works (the thing M4 migrates)

M4 doesn't build a new pipeline — it moves the *existing* M1 worker into a Fargate container. If M1 never transcribed a video, M4 has nothing to migrate and you'd be debugging two unknowns at once.

1. Ask: 「M1 有沒有成功跑完至少一個 job？如果不確定，先跑 `m1-ai-video-transcript-checklist`。」
2. `mcp__supabase_remote__execute_sql: SELECT COUNT(*) FROM jobs WHERE status='done';` — must be ≥ 1. If `0`, **stop and fix M1 first**.
3. Confirm the worker code to containerize is the same `worker.py` M1 ran (it already shells out to `ffmpeg`/`ffprobe` — those get baked into the image).

> If credits gate transcription, make sure the student has credits — a job stuck `pending` for lack of credits is not an M4 problem.

## Section 3 — Confirm AWS reach for the five new services

M1 needed EC2 + SSM + Secrets Manager. M4 needs five more. Confirm reach (empty list is fine — `AccessDenied` is not):
```
call_aws ecr describe-repositories --max-results 1
call_aws ecs list-clusters --max-results 1
call_aws lambda list-functions --max-items 1
call_aws events list-rules --limit 1
call_aws codebuild list-projects --sort-by NAME
```
- All five returning → enough breadth. Proceed.
- Any `AccessDenied` → the M1 IAM user was scoped too tightly (rare — the course uses an admin-ish user, see [[m1-ai-video-transcript-prerequisites]] §2.0). Attach the relevant managed policies (`AmazonEC2ContainerRegistryFullAccess`, `AmazonECS_FullAccess`, `AWSLambda_FullAccess`, `AmazonEventBridgeFullAccess`, `AWSCodeBuildAdminAccess`) or broaden to admin.

> **`iam:PassRole` matters later.** The main skill creates a CodeBuild role, two Fargate task roles, and a Lambda role, and the Lambda gets `iam:PassRole` on the task roles. The IAM user needs `iam:CreateRole` / `iam:PutRolePolicy` / `iam:AttachRolePolicy` — covered by the admin-ish M1 user.

## Section 4 — How the worker image gets built (no EC2, no local Docker)

**M4 builds the worker Docker image with AWS CodeBuild, not on the EC2 and not locally.** There is nothing to "decide" about a Docker daemon — CodeBuild provides one in the cloud. The student needs only `call_aws` + a GitHub push. The single setup question is **GitHub auth** (next).

> **Why not build on the EC2?** Because M4 *removes* the EC2 at the end. Building the replacement image on the box you're deleting is a circular dependency. CodeBuild is ephemeral and EC2-free — it spins up, builds, pushes to ECR, and vanishes.

### Section 4a — GitHub auth (push + CodeBuild source)

Two distinct GitHub touchpoints — keep them straight:

1. **Push the new files to the student's repo** (`worker/Dockerfile`, `buildspec.yml`, the worker code, and `worker/lambda_distributor.py`). The worker files + `buildspec.yml` are **CodeBuild's build source — the push is required, not documentation**: no push → no image. (`lambda_distributor.py` is record-only; deployed via `call_aws`.) How the push authenticates depends on environment:
   - **Cowork (common case):** the sandbox has **no stored git credential**. The student **pastes a GitHub PAT into the Cowork chat** and Cowork pushes with it (token-embedded remote URL or `GITHUB_TOKEN` for that push). PAT needs **contents-write/`repo`** scope; a fine-grained token scoped to the one repo is ideal. Treat it as a **session secret** — never commit/echo it; revoke after the course. Confirm before pushing (outward action).
   - **CLI mode (laptop):** the M0 `gh auth login` credential (keychain) already exists — a plain `git push` works, no token paste.
   - **Always derive the repo URL, never hardcode it** (`git -C <repo> remote get-url origin`) — each student's repo differs.

2. **CodeBuild → repo (build source + push webhook):** CodeBuild reads the repo to run `docker build`, **and** registers a webhook so a `git push` triggers the build (M4's deploy mechanism for worker code). Connecting CodeBuild to GitHub is therefore **mandatory regardless of repo visibility** — even a public repo needs it, because *registering the webhook* requires repo-admin access:
   - Run **once per AWS account**: `call_aws codebuild import-source-credentials --token <PAT> --server-type GITHUB --auth-type PERSONAL_ACCESS_TOKEN`.
   - The PAT needs **`repo` + `admin:repo_hook`** scope — `repo` to read source, `admin:repo_hook` to register the push webhook. A `repo`-only token builds but **cannot create the webhook**, so push-to-build silently won't arm.
   - Stored in CodeBuild; used at build time + to manage the webhook. Treat as a secret; revoke at course end.

> **No "public repo needs nothing" shortcut anymore.** An earlier design skipped CodeBuild auth for public repos — but the webhook (M4's whole point: push → build) requires it either way, so resolve the PAT regardless. Note this PAT (`admin:repo_hook`) is for **CodeBuild→GitHub**; it's separate from the push PAT in item 1 (Cowork → GitHub). A student may use one token with both scopes for both jobs, or two scoped tokens. (The *running* Fargate worker pulls its image from **ECR**, not GitHub — only the build touches GitHub.)

## Section 5 — Cost-model awareness

> 「M4 的成本：
> - **EventBridge + Lambda distributor**：每分鐘觸發，量很小 — 約 **$1.50/月**（M1 的 always-on EC2 是 ~$15/月）。
> - **Fargate**：只有真的在跑 job 時計費，1 vCPU / 2GB 約 **$0.05/小時**（一支 job 幾分鐘 → 幾分錢）。閒置 **$0**。
> - **CodeBuild**：只有 build image 時計費，一次 build 幾分鐘、幾分錢；不 build 就 $0。每月免費額度通常就涵蓋了課程用量。
> - **ECR**：存 image，每 GB ~$0.10/月；worker image 約 270 MB → 約 $0.03/月（小，但不是 0；明細見下方）。
> - **重點**：沒流量時趨近 $0；有流量按用量付。對個人 SaaS 通常比 always-on EC2 便宜很多。
>
> EC2 在 M4 結束時可以直接 terminate（沒有東西再依賴它）— build 在 CodeBuild、run 在 Fargate、loop 在 Lambda。想保守一點就先 stop 幾週再 terminate。」

> **閒置那 ~$0.06/月 到底是哪來的？** 上面的數字（Lambda、Fargate、EventBridge）涵蓋了主要支出，但學生帳單上「明明沒在跑卻還是收幾分錢」的那一小筆，是這幾條小項目加起來的，列出來免得學生看到帳單一頭霧水：
> - **ECR image storage** — `$0.10/GB·月`。worker image 約 270 MB，所以一份大約 **$0.03/月**。
> - **孤兒 ECR images（會慢慢累積）** — 每次 push 新的 `:latest`，舊的那層會變成 untagged 但**仍然占空間計費**。沒有 lifecycle policy 的話會越積越多。建議加一條「只保留最近 N 個 image」的 lifecycle policy（v2 cleanup），否則 storage 會隨著每次 build 緩慢爬升。
> - **CloudWatch log ingestion** — `$0.50/GB`。Lambda 每分鐘一筆 log + Fargate 的 worker log，量很小但不是 0。
> - **S3 artifact bucket** — 存 `deps-layer.zip` + `handler.zip`（Cowork 模式下還會放 CodeBuild courier 上傳的東西），幾 MB，幾乎可忽略。
> - **Lambda layer storage** — 75 GB 以下**免費**，這個 layer 才幾 MB，所以 $0；列出來只是讓你知道它不收錢。
>
> 全部加起來大約就是那 **~$0.06/月** 的閒置費 — 課程規模下小到不用在意，但知道組成就不會被帳單嚇到。

## Sanity check at the end (final state)

1. ✅ All 4 Cowork connectors authenticated (Section 1).
2. ✅ M1 has completed at least one real job (Section 2).
3. ✅ IAM user can reach ECR / ECS / Lambda / EventBridge / CodeBuild (Section 3).
4. ✅ GitHub auth resolved: repo URL derived (not hardcoded); push path known (Cowork PAT / laptop gh); CodeBuild→GitHub connection planned via `import-source-credentials` with a `repo` + `admin:repo_hook` PAT — **mandatory for any repo** (the push webhook needs it), no public-repo shortcut (Section 4a).
5. ✅ Student understands the cost model (Section 5).

If any fail, fix here.

## ⛔ STOP HERE — Do NOT proceed to M4 implementation

**This skill is a prerequisite check only.** When all items above are ✅, report green and **stop**.

Do **NOT**:
- ❌ Automatically load or invoke `m4-serverless`
- ❌ Ask "ready to start M4?" or "shall we proceed to Step 1?"
- ❌ Create ECR repos, the CodeBuild project, roles, the cluster, or the Lambda
- ❌ Push files, start a build, or stop the EC2 "to get a head start"

The student explicitly invokes `m4-serverless` when ready. If they say "what's next?", **tell them the next skill is `m4-serverless`** and let them decide when.

## Related skills

- [[m1-ai-video-transcript-prerequisites]] — where the AWS account + IAM user + AWS API MCP were set up (§2.0.3 has the `~/.aws/credentials` re-auth flow)
- [[m1-ai-video-transcript-checklist]] — must be green before M4 (Section 2 depends on it)
- [[m4-serverless]] — the main M4 walkthrough this skill prepares for
- [[m4-serverless-checklist]] — verifies M4 after the main skill
- [[aws-best-practice]] — IAM, SSM-not-SSH, region rules
- [[supabase-best-practice]] — the `fargate_task_arn` column must go through a migration file
