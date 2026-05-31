---
name: m4-serverless-checklist
description: Course 2 Milestone 4 verification (Lambda distributor + Fargate worker, image built by CodeBuild + auto-rebuilt on git push, no EC2 dependency) — checks every artifact is real and correctly wired: the `fargate_task_arn` idempotency column, the CodeBuild project + a SUCCEEDED build + the push-to-build webhook, the worker image in ECR, the ECS cluster + task definition + IAM roles, the Lambda distributor + deps layer, the EventBridge rule, the M1 EC2 stopped with nothing depending on it, and a real job running end-to-end on Fargate. Then, ONLY if everything (including the end-to-end test) passes, it terminates the M1 EC2 for the student via call_aws so they never touch the AWS console. Guards the "one distributor per Supabase" rule. Use when the student says "驗收 M4", "check M4", "M4 done?", or after the `m4-serverless` skill completes Step 8.
---

# M4 — Serverless Scaling Checklist (Lambda + Fargate, CodeBuild)

## What this skill does

Verifies the student actually completed M4 — and, critically, that **nothing in the live system still depends on the EC2**. M4 failures are sneaky: the CodeBuild build silently fails (no image), the image is wrong, both distributors run and double-bill, or the EC2 "looks" gone but the distributor process is still alive. This checklist tests every layer from the schema column through to a real job completing on Fargate, with the EC2 confirmed out of the loop.

**Then it finishes the job for the student.** Once everything passes — including a real end-to-end job on Fargate — **Section H terminates the now-unneeded EC2 directly via `call_aws`**, so the student never has to open the AWS console to decommission it. (If anything is red, the EC2 stays `stopped` as a fallback until it's fixed.)

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

### Section B — CodeBuild image build + push trigger (5 checks)

| # | Check | How to verify |
|---|---|---|
| B1 | CodeBuild project exists, GitHub source, privileged | `call_aws codebuild batch-get-projects --names <project>` — `source.type=GITHUB` with the student's repo URL, `environment.privilegedMode=true` (required for `docker build`), `ECR_URI` env var set. |
| B2 | The latest build SUCCEEDED | `call_aws codebuild list-builds-for-project --project-name <project>` → newest id → `call_aws codebuild batch-get-builds --ids <id>` → `buildStatus: SUCCEEDED`. A FAILED/absent build = no image was produced. (May be the manual verify-build from Step 3e or a later webhook-triggered build — either is fine.) **Cowork caveat:** if the student used the CodeBuild-courier for the Lambda zips (Step 5c), the *newest* build may be a `--buildspec-override lambda-build.buildspec.yml` run that uploads zips and does **not** push the worker image. Pair B2 with B3 — the `latest` ECR image is the real proof an image build succeeded; don't accept a courier build as the image build. |
| B3 | Worker image exists in ECR with `latest` | `call_aws ecr list-images --repository-name <repo>` — includes a `latest` tag, pushed at/after B2's build time. |
| B4 | **Push-to-build webhook is registered** | Same `batch-get-projects` response → a `webhook` block with a `url` and a `filterGroups` entry matching `EVENT=PUSH` on `^refs/heads/main$`. This is M4's deploy trigger for worker code; without it, "edit + push" won't rebuild. |
| B5 | The build did NOT run on the EC2 | Confirm the image came from CodeBuild, not a leftover EC2 build: B2 SUCCEEDED is the positive proof. (Sanity: the EC2 has no ECR-push role and is stopped per Section F — so it *couldn't* have built it.) |

If B2 is FAILED: read the project's CloudWatch log group. Usual causes: missing `privilegedMode`, CodeBuild role lacks ECR push, `buildspec.yml` path wrong, or the GitHub PAT (`import-source-credentials`) is missing/under-scoped.
If B4 has no webhook: `import-source-credentials` was never run, or its PAT lacked **`admin:repo_hook`** scope (a `repo`-only token builds but can't register the webhook). Re-import with the right scope, then `call_aws codebuild create-webhook` per `m4-serverless` Step 3f.

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
| D3 | Env wired; forwards AI keys but doesn't call AI | `call_aws lambda get-function-configuration --function-name <fn> --query 'Environment.Variables'` — has `SUPABASE_URL`, `SUPABASE_SECRET_KEY`, `ECS_CLUSTER`, `TASK_DEFINITION`, `SUBNETS`, and the `OPENAI_API_KEY`/`ANTHROPIC_API_KEY` it forwards to Fargate. (Distributor imports only boto3+supabase.) **Use `SUPABASE_SECRET_KEY`** — matching `worker.py` / Vercel / Supabase's current publishable/secret naming. The older `SUPABASE_SERVICE_KEY` (service_role JWT) name is being phased out; if you see it, it's drift — reconcile so all surfaces use one name or the worker reads `None`. |
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

> **First end-to-end test fails? Don't guess — work the triage table in [Appendix — common end-to-end failure modes](#appendix--common-end-to-end-failure-modes).** The first real Fargate task is where three setup bugs surface at once (log-group permission, worker Secrets-Manager hardcode, null session link), and they look alike from the outside. The appendix tells the three apart by their distinct symptoms. Also: if your test fails on a URL you picked, retry with the known-good baseline `https://dqwd87ogl0f9o.cloudfront.net/950371295/mp4/950371295_1920x1080.mp4?v=1716669006` (~2 min, ~3 credits) to rule out "the URL is the problem" before debugging infra.

### Section F — Single-distributor invariant + EC2 out of the loop (3 checks)

> **The most expensive M4 mistake.** Two live distributors on one Supabase = every job spawned twice, silently.

| # | Check | How to verify |
|---|---|---|
| F1 | M1 EC2 is `stopped` (or already `terminated` on a re-run) | `call_aws ec2 describe-instances --instance-ids <id> --query 'Reservations[].Instances[].State.Name'` — expected `stopped` at this point (Section H terminates it *after* the verdict). `terminated` is also fine (means H already ran on a prior pass). `running` → see F2. |
| F2 | No `distributor.py` alive (only relevant if `running`) | If `stopped`/`terminated`, satisfied. If `running`: `call_aws ssm send-command --instance-ids <id> --document-name AWS-RunShellScript --parameters 'commands=["pgrep -fa distributor.py || echo NONE"]'` — must print `NONE`. A live `distributor.py` + ENABLED rule (D4) = double-spawn. |
| F3 | Exactly one distributor enabled | EventBridge rule ENABLED (D4) **XOR** EC2 distributor running (F1/F2). Steady-state serverless: rule ENABLED + EC2 stopped/terminated. **Never both.** |

If F1 is `running` and the rule is ENABLED: **stop now** — double-billing. Stop the EC2 distributor (steady-state) or disable the rule (EC2 fallback), per `m4-serverless` Step 7 / "Switching back."

### Section G — Regression + hygiene (4 checks)

| # | Check | How to verify |
|---|---|---|
| G1 | Web app + M2 credits still work | `mcp__vercel__list_deployments` shows a READY prod deploy; `/credits` loads; `credit_products` has its active rows. |
| G2 | No secrets in git | Scan history for **every** credential format this stack can leak, not just the two legacy ones. One pass: `git log -p --all \| grep -nE 'sb_secret_\|sb_publishable_\|SUPABASE_SERVICE_KEY\|github_pat_\|ghp_\|AKIA\|ASIA\|sk-proj-\|sk-' \| head` — all should return nothing. The patterns cover: this project's actual Supabase keys (`sb_secret_`/`sb_publishable_`), the legacy `SUPABASE_SERVICE_KEY`, GitHub PATs (`github_pat_` fine-grained, `ghp_` classic), AWS access keys (`AKIA`) and STS session tokens (`ASIA`), and OpenAI keys (`sk-proj-` new, `sk-` old). M4 creds live in Lambda env / RunTask overrides / CodeBuild env, never the repo. (The old check only looked for `SUPABASE_SERVICE_KEY` + `sk-`, which misses the `sb_secret_` format this project uses, plus any AWS/GitHub leak.) |
| G3 | M4 build source + handler are in the repo | `git -C <repo> ls-files | grep -E 'worker/Dockerfile|buildspec.yml|worker/lambda_distributor.py'` — all present. `worker/Dockerfile` + `buildspec.yml` are **CodeBuild's required source** (B2 couldn't have SUCCEEDED without them on the built branch); `lambda_distributor.py` is record-only. If the student used the **Cowork CodeBuild-courier** path for the Lambda zips (Step 5c), `lambda-build.buildspec.yml` should also be committed. If B2 passed but the build-critical files are missing locally, the student built from an un-pushed branch — reconcile. |
| G4 | CodeBuild PAT is scoped right + noted for cleanup | A PAT was imported via `import-source-credentials` (mandatory for the webhook, any repo). Confirm it has **`repo` + `admin:repo_hook`** scope (B4 webhook present is the proof it could register), remind the student it's stored in CodeBuild, and that it should be a repo-scoped token they revoke at course end. |

---

## Appendix — common end-to-end failure modes

The first real Fargate task is where every "happy-path-only" setup bug shows up — and they're easy to confuse because they all present as "the task didn't transcribe." Three of them are *production-affecting* (they'd hit your very first real user); one only bites direct-INSERT test jobs. Match the symptom, apply the fix, re-run Section E.

| Symptom (where you see it) | Root cause | Fix | Skill ref |
|---|---|---|---|
| Task never starts. `describe-tasks` shows `ResourceInitializationError: ... not authorized to perform: logs:CreateLogGroup on /ecs/subtitle-worker`. **No** worker log lines (the group was never created). | Task def used `awslogs-create-group=true`, but `AmazonECSTaskExecutionRolePolicy` grants only log **write**, not `logs:CreateLogGroup`. | Pre-create the group: `call_aws logs create-log-group --log-group-name /ecs/subtitle-worker` and drop `awslogs-create-group` from the task def. *(Or attach `CloudWatchLogsFullAccess` to the exec role.)* | `m4-serverless` Step 4c |
| Container starts then **crashes on import**; CloudWatch shows a `secretsmanager`/`AccessDenied` or `GetSecretValue` traceback before any pipeline line. | M1's `worker.py` reads creds from Secrets Manager at import, but the minimal Fargate task role has no `secretsmanager:GetSecretValue` — creds actually arrive as **env vars**. | Env-first refactor of `_load_secrets()` (prefer env vars, fall back to Secrets Manager) — commit `7c19086`. *(Or grant the task role `secretsmanager:GetSecretValue`.)* | `m4-serverless` Step 2 |
| Container starts, claims the job, then **dies immediately** at `update_session(...)` with a null/`NoneType` error on `current_session_id`. Production jobs work; only your direct-INSERT **test** job fails. | The Lambda's `_ensure_session` created a `job_sessions` row but never set `jobs.current_session_id` to point at it. `POST /api/jobs` sets it atomically, so real users dodge it. | Add the link after the session insert: `db.table("jobs").update({"current_session_id": row["id"]}).eq("id", job_id).execute()` — commit `f422557`. | `m4-serverless` Step 5b |
| `describe-tasks` shows `CannotPullContainerError`. | Exec role can't pull from ECR, or the task-def image tag is stale/absent. | Check C2 (image tag) + C3 (exec role has `AmazonECSTaskExecutionRolePolicy`); confirm B3 (a `latest` image exists). | Steps 3–4 |
| Task runs but the job stays `pending` and **no** task ever spawns. | Lambda not firing, or no pending job matches `fargate_task_arn IS NULL`. | Work Section D (D4 rule ENABLED, D5 firing, D3 env). | Steps 5–6 |
| Job hangs `pending` with credits at issue. | Not an M4 bug — the user has insufficient credits. | Top up credits; retry. | — |

If the symptom doesn't match any row, fall back to the layer-by-layer trace in the Verdict's "If anything failed" list: image (B) → roles (C) → Lambda (D) → worker logs (E3).

## Verdict

If A1–G4 all pass — **including the real end-to-end Fargate job in Section E** — **M4 is complete, with no EC2 dependency.** The worker image is built on demand by **CodeBuild** from the GitHub repo, runs as isolated **Fargate** tasks (any video length), launched by a **Lambda** distributor on an **EventBridge** schedule. Build, run, and loop have all moved to serverless services. Cost scales to ~$0 when idle. Nothing about the web app, database, payments, or domain changed.

**→ When (and only when) everything above is green, run Section H to decommission the EC2 for the student.**

If anything failed:

- **Schema (A)** → `m4-serverless` Step 1
- **CodeBuild / image / webhook (B)** → Steps 2–3 (push + buildspec + project + privilegedMode + ECR perms + `import-source-credentials` + `create-webhook`)
- **ECS / IAM (C)** → Step 4
- **Lambda / EventBridge (D)** → Steps 5–6
- **End-to-end (E)** → trace B (image) → C (roles) → D (Lambda) → worker logs
- **Single-distributor invariant (F)** → Step 7 — fix immediately, it's a billing risk
- **Regression (G)** → the relevant earlier-milestone checklist

Re-run after every fix until green.

## Section H — Decommission the EC2 (run ONLY after the verdict is green)

The EC2 was kept `stopped` as a fallback while serverless was unproven. Now that A1–G4 pass — the real end-to-end job in Section E ran on Fargate, and Section F confirmed nothing depends on the EC2 — **terminate it for the student via `call_aws`, so they never have to open the AWS console.**

> **Gate — do not skip.** Only proceed if **every check above passed, including E1–E4 (a real job reached `done` on Fargate)**. If anything is red — especially the end-to-end test — **leave the EC2 `stopped`** and fix the failure first. Terminating before serverless is proven would remove the only fallback.

**H1 — Confirm the safe state one more time** (belt-and-suspenders before an irreversible action):
- Section F passed: rule `ENABLED`, EC2 `stopped`, exactly one distributor.
- Section E passed: a real job reached `done` on Fargate.

**H2 — Terminate the instance for the student:**
```
call_aws ec2 terminate-instances --instance-ids <id>
call_aws ec2 describe-instances --instance-ids <id> --query 'Reservations[].Instances[].State.Name'   # → shutting-down, then terminated
```
Tell the student: 「M4 全部驗收通過、真實 job 已經在 Fargate 上跑完，serverless 確定穩定了 — 我已經直接幫你把舊的 EC2 worker **terminate** 掉（不用你進 AWS console）。從現在起 build 在 CodeBuild、run 在 Fargate、排程在 Lambda，閒置成本趨近 $0。」

> **Termination is irreversible** (the instance + its instance-store data are gone; the EBS root volume is deleted unless the student changed the default). That's the intended end state — but if the student explicitly says they want to keep the fallback a while longer, **respect that and leave it `stopped`** instead; note that a stopped instance still incurs a small EBS charge.

**H3 — (optional) Mention leftover cleanup:** the EC2's KeyPair and security group (from M1 CDK/setup) are now unused. Removing them is harmless tidy-up but not required; flag it as a v2 cleanup rather than doing it automatically.

After H2, the EC2 is gone and M4 is fully done — purely serverless, no console visits required of the student.

## Related skills

- [[m4-serverless]] — the main M4 walkthrough this verifies
- [[m4-serverless-prerequisites]] — connector + GitHub/CodeBuild auth this assumes is done
- [[m1-ai-video-transcript-checklist]] — the worker pipeline M4 containerized must still pass (Section E is the same pipeline)
- [[aws-best-practice]] — IAM, SSM-not-SSH, region rules
- [[supabase-best-practice]] — the `fargate_task_arn` column must come from a migration file (A2)
