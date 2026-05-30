---
name: m4-serverless-checklist
description: Course 2 Milestone 4 verification (Lambda distributor + Fargate worker, image built by CodeBuild, no EC2 dependency) — checks every artifact is real and correctly wired: the `fargate_task_arn` idempotency column, the CodeBuild project + a SUCCEEDED build, the worker image in ECR, the ECS cluster + task definition + IAM roles, the Lambda distributor + deps layer, the EventBridge rule, the M1 EC2 stopped/terminated with nothing depending on it, and a real job running end-to-end on Fargate. Guards the "one distributor per Supabase" rule. Use when the student says "驗收 M4", "check M4", "M4 done?", or after the `m4-serverless` skill completes Step 8.
---

# M4 — Serverless Scaling Checklist (Lambda + Fargate, CodeBuild)

## What this skill does

Verifies the student actually completed M4 — and, critically, that **nothing in the live system still depends on the EC2**. M4 failures are sneaky: the CodeBuild build silently fails (no image), the image is wrong, both distributors run and double-bill, or the EC2 "looks" gone but the distributor process is still alive. This checklist tests every layer from the schema column through to a real job completing on Fargate, with the EC2 confirmed out of the loop.

**The invariant this checklist protects above all else:** exactly **one** distributor polls a given Supabase. If both the M1 EC2 `distributor.py` and the M4 Lambda are live, every pending job gets spawned twice (double Whisper billing). Section F gates on this.

**Run this AFTER `m4-serverless` Step 8, or any time the student claims M4 is done.**

![AI Video Reader architecture, M4 view](assets/ai_video_reader_structure.jpg)

*What this checklist verifies: the idempotency schema (A), the CodeBuild→ECR image (B), the ECS+IAM compute lane (C), the Lambda+EventBridge distributor (D), a real end-to-end Fargate job (E), and the single-distributor / EC2-out-of-the-loop invariant (F).*

## Execution mode: Cowork vs CLI

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Schema | `mcp__supabase_remote__execute_sql` | same |
| B — CodeBuild / ECR | `aws codebuild/ecr ...` | `call_aws codebuild/ecr ...` |
| C — ECS / IAM | `aws ecs/iam ...` | `call_aws ecs/iam ...` |
| D — Lambda / EventBridge | `aws lambda/events/logs ...` | `call_aws lambda/events/logs ...` |
| E — End-to-end job | browser + `mcp__supabase_remote__execute_sql` + CloudWatch | same |
| F — Single-distributor invariant | `aws ec2/events ...` + SSM | `call_aws ec2/events ...` + SSM |

## How to run

Invoked directly by the student (`驗收 M4`). You (Claude) **actively execute** each check.

### Step 1: Collect inputs

1. **AWS region**
2. **CodeBuild project name** (default `subtitle-worker-build`) + **ECR repo** (default `subtitle-worker`)
3. **ECS cluster** (default `subtitle-workers`) + **task family** (default `subtitle-worker`)
4. **Lambda function** (default `subtitle-distributor`) + **EventBridge rule** (default `subtitle-distributor-schedule`)
5. **M1 EC2 instance ID** (should be stopped or terminated)
6. **Supabase project URL**

### Step 2: Preflight

`call_aws sts get-caller-identity` returns the M1 account. Connectors available: `call_aws` (B–D, F), `mcp__supabase_remote__*` (A, E).

---

## Checklist

### Section A — Idempotency schema (2 checks)

| # | Check | How to verify |
|---|---|---|
| A1 | `job_sessions.fargate_task_arn` column exists | `SELECT column_name FROM information_schema.columns WHERE table_name='job_sessions' AND column_name='fargate_task_arn'` — one row. |
| A2 | Added via a migration file | `mcp__supabase_remote__list_migrations` (or `supabase/migrations/`) — a migration mentioning `fargate_task_arn` exists. Raw-SQL changes violate [[supabase-best-practice]]. |

### Section B — CodeBuild image build (4 checks)

| # | Check | How to verify |
|---|---|---|
| B1 | CodeBuild project exists, GitHub source, privileged | `call_aws codebuild batch-get-projects --names <project>` — `source.type=GITHUB` with the student's repo URL, `environment.privilegedMode=true` (required for `docker build`), `ECR_URI` env var set. |
| B2 | The latest build SUCCEEDED | `call_aws codebuild list-builds-for-project --project-name <project>` → newest id → `call_aws codebuild batch-get-builds --ids <id>` → `buildStatus: SUCCEEDED`. A FAILED/absent build = no image was produced. |
| B3 | Worker image exists in ECR with `latest` | `call_aws ecr list-images --repository-name <repo>` — includes a `latest` tag, pushed at/after B2's build time. |
| B4 | The build did NOT run on the EC2 | Confirm the image came from CodeBuild, not a leftover EC2 build: B2 SUCCEEDED is the positive proof. (Sanity: the EC2 has no ECR-push role and is stopped/terminated per Section F — so it *couldn't* have built it.) |

If B2 is FAILED: read the project's CloudWatch log group. Usual causes: missing `privilegedMode`, CodeBuild role lacks ECR push, `buildspec.yml` path wrong, or (private repo) `import-source-credentials` never run.

### Section C — ECS + IAM compute lane (5 checks)

| # | Check | How to verify |
|---|---|---|
| C1 | ECS cluster ACTIVE | `call_aws ecs describe-clusters --clusters <cluster>` → `status: ACTIVE`. |
| C2 | Task definition ACTIVE, points at the ECR image | `call_aws ecs describe-task-definition --task-definition <family>` → `status: ACTIVE`; container `image` = `<repositoryUri>:latest`; `requiresCompatibilities` includes `FARGATE`; both `executionRoleArn` + `taskRoleArn` set. |
| C3 | Task execution role can pull ECR + write logs | `call_aws iam list-attached-role-policies --role-name <execRole>` includes `AmazonECSTaskExecutionRolePolicy`. |
| C4 | Worker task role exists | `call_aws iam get-role --role-name <taskRole>` — minimal/empty is fine. |
| C5 | CodeBuild role can push to ECR | `call_aws iam list-attached-role-policies --role-name <codeBuildRole>` includes ECR push (e.g. `AmazonEC2ContainerRegistryPowerUser`) + a logs policy. |

If C2 image mismatch: the task def points at a stale tag — re-register pointing at current `latest`.

### Section D — Lambda distributor + EventBridge (5 checks)

| # | Check | How to verify |
|---|---|---|
| D1 | Lambda exists + Active | `call_aws lambda get-function --function-name <fn>` → `State: Active`, handler `lambda_distributor.handler`. |
| D2 | Deps layer attached | Same response: `Layers` includes the `subtitle-distributor-deps` ARN. |
| D3 | Env wired; forwards AI keys but doesn't call AI | `call_aws lambda get-function-configuration --function-name <fn> --query 'Environment.Variables'` — has `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `ECS_CLUSTER`, `TASK_DEFINITION`, `SUBNETS`, and the `OPENAI_API_KEY`/`ANTHROPIC_API_KEY` it forwards to Fargate. (Distributor imports only boto3+supabase.) |
| D4 | EventBridge rule ENABLED at `rate(1 minute)`, targets the Lambda | `call_aws events describe-rule --name <rule>` → `State: ENABLED`, `rate(1 minute)`; `list-targets-by-rule` → the Lambda ARN. |
| D5 | Lambda is firing each minute | `call_aws logs describe-log-streams --log-group-name /aws/lambda/<fn> --order-by LastEventTime --descending --max-items 1` — latest event within ~2 min. |

If D5 has no recent invocations but D4 is ENABLED: confirm `lambda add-permission` granted `events.amazonaws.com` (`call_aws lambda get-policy --function-name <fn>`).

### Section E — End-to-end job on Fargate (4 checks)

| # | Check | How to verify |
|---|---|---|
| E1 | A recent job has `fargate_task_arn` set | `SELECT j.id,j.status,s.fargate_task_arn FROM jobs j JOIN job_sessions s ON s.id=j.current_session_id WHERE s.fargate_task_arn IS NOT NULL ORDER BY j.created_at DESC LIMIT 3` — ≥1 recent row with a non-null ARN. |
| E2 | That Fargate task ran cleanly | `call_aws ecs describe-tasks --cluster <cluster> --tasks <arn>` — `lastStatus` RUNNING or STOPPED with exit 0; **not** `CannotPullContainerError` (→ exec role / image, C2/C3) or `ResourceInitializationError`. |
| E3 | Worker logged to CloudWatch | `call_aws logs filter-log-events --log-group-name /ecs/subtitle-worker --start-time <recent ms> --max-items 5` — real worker lines (Whisper progress, status transitions). |
| E4 | Job reached `done` with content | `SELECT status,(subtitle_txt_content IS NOT NULL) AS has_txt FROM job_sessions WHERE id=(SELECT current_session_id FROM jobs WHERE id='<E1 job>')` → `done`, `has_txt=true`. Pipeline runs intact in the container. |

**Idempotency spot-check (recommended):** confirm the *next* tick did not spawn a second task for the same job — one task per job; the distributor log for the following minute shows it skipping the already-ARN'd session.

### Section F — Single-distributor invariant + EC2 out of the loop (3 checks)

> **The most expensive M4 mistake.** Two live distributors on one Supabase = every job spawned twice, silently.

| # | Check | How to verify |
|---|---|---|
| F1 | M1 EC2 is `stopped` or `terminated` | `call_aws ec2 describe-instances --instance-ids <id> --query 'Reservations[].Instances[].State.Name'` — `stopped` (kept as fallback) or `terminated` (intended end state). `running` → see F2. |
| F2 | No `distributor.py` alive (only relevant if `running`) | If `stopped`/`terminated`, satisfied. If `running`: `call_aws ssm send-command --instance-ids <id> --document-name AWS-RunShellScript --parameters 'commands=["pgrep -fa distributor.py || echo NONE"]'` — must print `NONE`. A live `distributor.py` + ENABLED rule (D4) = double-spawn. |
| F3 | Exactly one distributor enabled | EventBridge rule ENABLED (D4) **XOR** EC2 distributor running (F1/F2). Steady-state serverless: rule ENABLED + EC2 stopped/terminated. **Never both.** |

If F1 is `running` and the rule is ENABLED: **stop now** — double-billing. Stop the EC2 distributor (steady-state) or disable the rule (EC2 fallback), per `m4-serverless` Step 7 / "Switching back."

### Section G — Regression + hygiene (4 checks)

| # | Check | How to verify |
|---|---|---|
| G1 | Web app + M2 credits still work | `mcp__vercel__list_deployments` shows a READY prod deploy; `/credits` loads; `credit_products` has its active rows. |
| G2 | No secrets in git | `git log -p --all -S 'SUPABASE_SERVICE_KEY' | head` / `... -S 'sk-' | head` — none. M4 creds live in Lambda env / RunTask overrides / CodeBuild env, never the repo. |
| G3 | M4 build source + handler are in the repo | `git -C <repo> ls-files | grep -E 'worker/Dockerfile|buildspec.yml|worker/lambda_distributor.py'` — all present. `worker/Dockerfile` + `buildspec.yml` are **CodeBuild's required source** (B2 couldn't have SUCCEEDED without them on the built branch); `lambda_distributor.py` is record-only. If B2 passed but these are missing locally, the student built from an un-pushed branch — reconcile. |
| G4 | CodeBuild PAT (if used) is scoped + noted for cleanup | If a private-repo PAT was imported (`import-source-credentials`), remind the student it's stored in CodeBuild and should be a repo-scoped token they revoke at course end. |

---

## Verdict

If A1–F3 all pass: **M4 is complete, with no EC2 dependency.** The worker image is built on demand by **CodeBuild** from the GitHub repo, runs as isolated **Fargate** tasks (any video length), launched by a **Lambda** distributor on an **EventBridge** schedule. The M1 EC2 is stopped or terminated and **nothing depends on it** — build, run, and loop all moved to serverless services. Cost scales to ~$0 when idle. Nothing about the web app, database, payments, or domain changed.

If anything failed:

- **Schema (A)** → `m4-serverless` Step 1
- **CodeBuild / image (B)** → Steps 2–3 (push + buildspec + project + privilegedMode + ECR perms)
- **ECS / IAM (C)** → Step 4
- **Lambda / EventBridge (D)** → Steps 5–6
- **End-to-end (E)** → trace B (image) → C (roles) → D (Lambda) → worker logs
- **Single-distributor invariant (F)** → Step 7 — fix immediately, it's a billing risk
- **Regression (G)** → the relevant earlier-milestone checklist

Re-run after every fix until green.

## Related skills

- [[m4-serverless]] — the main M4 walkthrough this verifies
- [[m4-serverless-prerequisites]] — connector + GitHub/CodeBuild auth this assumes is done
- [[m1-ai-video-transcript-checklist]] — the worker pipeline M4 containerized must still pass (Section E is the same pipeline)
- [[aws-best-practice]] — IAM, SSM-not-SSH, region rules
- [[supabase-best-practice]] — the `fargate_task_arn` column must come from a migration file (A2)
