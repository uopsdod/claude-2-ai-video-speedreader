---
name: m4-serverless
description: Course 2 Milestone 4 (OPTIONAL, advanced) — replace M1's always-on EC2 worker with a fully serverless stack with NO EC2 dependency. An EventBridge schedule fires a Lambda distributor every minute; it polls Supabase for pending jobs and launches one ECS Fargate task per job via ecs:RunTask; the worker Docker image (ffmpeg baked in) is built by AWS CodeBuild straight from the GitHub repo and pushed to ECR. Handles any video length. Cost scales to $0 when idle. No new external accounts — reuses the M1 AWS account. Use when the student says "啟動 M4", "start M4", "go serverless", "Lambda + Fargate", "remove the EC2", or asks how to scale the worker without an always-on server.
---

# M4 — Fully Serverless Scaling (Lambda + Fargate, no EC2)

> **這一節是進階優化，不是必修。** M0–M3 結束時你的產品已經能上線、能收錢、有自己的網域 — 已經是一個會賺錢的 SaaS。M4 是當你流量長起來（同時跑很多影片）或想把伺服器成本壓到趨近 $0、不再付 always-on EC2 的錢時才做。M2 結束時的 EC2 + Vercel 部署模型完全可以撐你前 6 個月。

## What this skill does

Walks the student through Course 2 Milestone 4 end-to-end. By the end the student has a fully serverless worker with **no EC2 anywhere in the live path or the build path**:

1. The worker (the same `worker.py` pipeline from M1, ffmpeg baked in) packaged as a **Docker image in Amazon ECR** — built by **AWS CodeBuild**, sourced straight from the **GitHub repo**.
2. An **ECS Fargate** cluster + task definition that runs one container per job — isolated CPU/memory, no contention, **no 15-minute limit** (handles any video length).
3. A **Lambda distributor** that polls Supabase for pending jobs and calls `ecs:RunTask` per job — replacing M1's `distributor.py`.
4. An **EventBridge schedule** (`rate(1 minute)`) firing the Lambda — replacing the always-on EC2 + `tmux` loop.
5. The M1 **EC2 worker stopped** (this skill stops it; `m4-serverless-checklist` terminates it for the student once the end-to-end test passes) — and, crucially, **nothing depends on it anymore**: the build runs on CodeBuild, the work runs on Fargate, the loop runs on Lambda.

**Why this shape.** The worker shells out to the **ffmpeg binary** (download → transcode → chunk → Whisper) and pulls heavy deps (`yt-dlp`, `moviepy`, `pydub`, …). That's a poor fit for a Lambda (15-min cap, no ffmpeg, `/tmp` limits) but a perfect fit for a **container on Fargate** — ffmpeg + every dep just bake into the image, and a long video has no time ceiling. The **distributor**, by contrast, only polls a table and fires `RunTask` — tiny, fast, every minute — which is the ideal **Lambda + EventBridge** job. So M4 splits M1's EC2 into the two serverless services each half fits best. (This is also exactly what the production stack runs.)

**The one thing that's easy to get wrong — and how M4 avoids it.** Building the Docker image needs Docker on an x86_64 Linux host. The obvious-but-wrong answer is "build it on the M1 EC2" — but the whole point of M4 is to *remove* that EC2, so depending on it to build the very image that replaces it is a trap. **M4 builds the image with AWS CodeBuild** — an on-demand, managed Docker build that pulls source from GitHub, builds, pushes to ECR, and disappears. No EC2, no local Docker, no SSH. Cowork-friendly (all `call_aws`).

## GitHub's role in M4: load-bearing build source (not just documentation)

The student **edits the worker/infra files in the workspace and pushes them to their own GitHub repo.** In M4 that push is **required and active, not optional or document-only** — once the webhook is set up (Step 3f), **a `git push` to the worker is what triggers the image build.** CodeBuild reads the repo as its build source and rebuilds on push, so **if the worker code isn't pushed, no new image is built.** GitHub is the deploy trigger for business-logic code, not a "for the record" nicety.

This is the deliberate **business-logic vs infrastructure split**: the worker code (changes often) ships by `git push` → webhook → rebuild; the infrastructure (Lambda, ECS task def, EventBridge, IAM, ECR — changes rarely) is set up once via `call_aws` and left alone.

Be precise about which files are build-critical vs. record-only:

| File | Role | If not pushed |
|---|---|---|
| `worker/Dockerfile`, `worker/worker.py` (+ `requirements.txt`), `buildspec.yml` | **Build-critical** — CodeBuild checks them out and runs `docker build` against them | **CodeBuild builds nothing (or builds stale code).** The Fargate worker never gets the new image. |
| `worker/lambda_distributor.py` | **Record-only** — the Lambda is deployed via `call_aws lambda` from a zip, *not* by CodeBuild | The Lambda still works (you deploy it directly), but the repo no longer matches what's running. Push it anyway. |
| migration SQL | **Record-only** — applied via Supabase MCP/CLI | Same — applied independently; commit it so the schema is reproducible. |

So: **the worker push feeds the build; the Lambda/migration push keeps the repo honest.** Both should happen, but only the first is on the critical path to a working image.

Lambda + Fargate themselves are created/updated with `call_aws` (and CodeBuild for the image). **Deployment is manual `call_aws`, not a GitHub-triggered pipeline** — pushing feeds the build source; you trigger the build and the deploys explicitly. (Auto-build-on-push is a v2 roadmap item; CDK to make the repo the full deploy source is another.)

## When to do M4 (and when to skip it)

**Do M4 when:** you want the worker to cost $0 while you sleep · you don't want to babysit an always-on EC2 · concurrent jobs lag on the single EC2 · you want unlimited concurrency and any video length without resizing a box.

**Skip M4 when:** still validating the product · MAU < 100 · ECS/Fargate/CodeBuild names make your head hurt. The M1 EC2 worker is genuinely fine for a first product — going serverless is an optimization you earn, not a box you must tick.

## When to load this skill

Trigger phrases: 「啟動 M4」 / "start M4" · "go serverless" · "Lambda + Fargate" · 「把 EC2 關掉」 / "remove the always-on EC2" · any prompt about scaling the worker off the always-on box.

Do NOT load this for M0/M1/M2/M3 — they have their own skills. If the student hasn't finished M1 (a working EC2 worker), **stop** — M4 has nothing to migrate from.

## Required external accounts

Same six as M3 — **no new accounts**:

| # | Service | Used for | Added in |
|---|---|---|---|
| 1 | GitHub | Source control — **+ CodeBuild source (M4)** | M0 |
| 2 | Supabase | Auth + Postgres + RLS | M0 |
| 3 | Vercel | Auto-deploy Next.js | M0 |
| 4 | OpenAI | Whisper (`whisper-1`) | M1 |
| 5 | AWS | EC2 (M1) → **ECR + ECS Fargate + Lambda + EventBridge + CodeBuild (M4)** | M1 |
| 6 | Stripe (sandbox) | Checkout Sessions + webhook | M2 |

M4 adds **five AWS services** (ECR, ECS Fargate, Lambda, EventBridge, CodeBuild) inside the **same AWS account** from M1. No new signup. If the AWS connector isn't authenticated, **stop and load `m4-serverless-prerequisites` first**.

## Required Cowork connectors / MCPs

| # | Connector / MCP | Used in M4 for |
|---|---|---|
| 1 | `call_aws` / `suggest_aws_commands` (AWS API MCP) | ECR, ECS, Lambda, EventBridge, CodeBuild, IAM — the bulk of M4 |
| 2 | `mcp__supabase_remote__*` | Schema migration (`fargate_task_arn`) + checklist verification |
| 3 | `mcp__vercel__*` | Not changed by M4, but the app must stay green |
| 4 | `mcp__stripe__*` | Not used in M4, but M2/M3 must remain green |

> **No local Docker needed, and no EC2.** Unlike a build-on-EC2 design, M4 never builds a container locally or on the EC2 — **CodeBuild** does it in the cloud. The student only needs `call_aws` + a GitHub push.

## The shape of the final system

![AI Video Reader architecture](assets/ai_video_reader_structure.jpg)

*Same product as M3. The worker compute lane changes: M1's always-on EC2 (running `distributor.py` in tmux) is replaced by EventBridge → Lambda distributor → Fargate tasks, with the worker image built by CodeBuild from GitHub. Everything else — Vercel, Supabase, Stripe, the domain — is untouched.*

Text-tree of M4's two flows — **build-time** and **run-time** — neither touches the EC2:

```
BUILD-TIME (automatic on every worker code change):
  edit worker/ + buildspec.yml → git push to GitHub repo
       └─ CodeBuild GitHub webhook fires on push to main   (no manual start-build)
            └─ ephemeral x86_64 builder: docker build → docker push → ECR :latest

RUN-TIME (every job):
  [Browser] → [Vercel] → POST /api/jobs → [Supabase: jobs + job_sessions]
                                                ▲          │ status='pending'
                                                │          ▼
                          [EventBridge: rate(1 minute)]
                                                │
                          [Lambda: distributor]
                             ├─ SELECT pending WHERE fargate_task_arn IS NULL
                             ├─ ecs:RunTask  (one Fargate task per job, fire-and-forget)
                             └─ UPDATE job_sessions SET fargate_task_arn = <arn>
                                                │
                                                ▼
                          [ECS Fargate task]  (one container per job, any length)
                             ├─ image pulled from ECR (ffmpeg + deps baked in)
                             ├─ claim_job → yt-dlp → ffmpeg → OpenAI Whisper, chunked
                             └─ write subtitle_*_content back to job_sessions
```

**The two state changes vs M1:** `distributor.py` → Lambda (now async/scheduled, `ecs:RunTask` instead of a local subprocess), and the worker subprocess → a Fargate task (same `worker.py`, now containerized). **The new column:** `job_sessions.fargate_task_arn` carries the idempotency the stateless Lambda can't keep in memory — it only spawns jobs `WHERE fargate_task_arn IS NULL` and writes the ARN right after `RunTask`.

## Execution mode (single path — Claude Code driving MCPs)

| Component | Operation | How |
|---|---|---|
| **Supabase** | Add `fargate_task_arn` column | `mcp__supabase_remote__apply_migration` (Cowork) / migration file + `supabase db push` (CLI). **Always a migration file** (see [[supabase-best-practice]]). |
| **GitHub** | Push worker `Dockerfile` + `buildspec.yml` + Lambda handler | normal `git push` (auth per environment — see prereq §4a). Source of truth + CodeBuild input. |
| **CodeBuild** | Build the worker image from GitHub → ECR, auto on push | `call_aws codebuild create-project` + `import-source-credentials` + `create-webhook`; one manual `start-build` to verify, then push-triggered. **No EC2, no local Docker.** |
| **ECR** | Hold the worker image | `call_aws ecr create-repository`; CodeBuild pushes. |
| **ECS** | Cluster + task definition | `call_aws ecs create-cluster` / `register-task-definition`. |
| **IAM** | Task exec role, worker task role, Lambda role, **CodeBuild role** | `call_aws iam create-role` / `attach-role-policy` / `put-role-policy`. |
| **Lambda** | Distributor + deps layer | `call_aws lambda create-function` / `publish-layer-version`. **Cowork can't `s3 cp` the zips** (sandbox is proxy-blocked from S3 + the AWS MCP has a separate filesystem) — use CodeBuild as the courier (Step 5c). CLI mode uploads directly. |
| **EventBridge** | `rate(1 minute)` → Lambda | `call_aws events put-rule` / `put-targets`; `lambda add-permission`. |
| **EC2 (M1)** | Stop the old worker (Step 7) — termination is done by the checklist once green | `call_aws ec2 stop-instances` here; `terminate-instances` runs in `m4-serverless-checklist` Section H. Nothing depends on it after M4. |

## Conversational flow

Drive the student through the steps below. **Don't dump them all at once.** After each, **wait for confirmation**. Each step ends with a "verify before moving on" you must actually run.

> **Before Step 1:** confirm `m4-serverless-prerequisites` is done — connectors green, M1 worker confirmed working, region noted, GitHub push auth resolved.

> **⚠️ One distributor per Supabase at a time.** M1's EC2 `distributor.py` and the new Lambda **must not both poll the same Supabase** — they'll each spawn a task for the same job (double Whisper billing). Step 7 stops the EC2 distributor *before* enabling the EventBridge rule. Never enable both.

---

### Step 1 — Add the `fargate_task_arn` idempotency column

Migration (never raw SQL — see [[supabase-best-practice]]):
```sql
-- supabase/migrations/<timestamp>_add_fargate_task_arn_to_job_sessions.sql
ALTER TABLE job_sessions ADD COLUMN IF NOT EXISTS fargate_task_arn TEXT;
CREATE INDEX IF NOT EXISTS idx_job_sessions_unspawned
  ON job_sessions (id) WHERE fargate_task_arn IS NULL;
```
Apply — **Cowork:** `mcp__supabase_remote__apply_migration`; **CLI:** file + `supabase db push`.

**Verify:** `SELECT column_name FROM information_schema.columns WHERE table_name='job_sessions' AND column_name='fargate_task_arn'` returns one row.

---

### Step 2 — Add the worker `Dockerfile` + `buildspec.yml` and push to GitHub

The worker image needs the M1 `worker.py` pipeline + ffmpeg + deps. CodeBuild builds it from these two files in the repo.

> **⚠️ Before you bake it in: M1's `worker.py` reads its three Supabase/OpenAI/Anthropic creds from AWS Secrets Manager at import time** (`sm.get_secret_value(...)` on an EC2 instance profile). In M4 those creds arrive as **env vars** via the Lambda's `containerOverrides[].environment` — and the Fargate **worker task role is intentionally minimal** (Step 4b: no `secretsmanager:GetSecretValue`). So the unmodified M1 worker **crashes on import in Fargate** even though the secrets exist in the account. **Pick one before building the image:**
>
> - **(Recommended) Env-first refactor.** Make `_load_secrets()` prefer env vars when all three are present, falling back to Secrets Manager otherwise. The *same* image then runs in both M1 (EC2 instance profile + Secrets Manager) and M4 (Fargate + env). Sketch:
>   ```python
>   def _load_secrets():
>       keys = ("SUPABASE_URL", "SUPABASE_SECRET_KEY", "OPENAI_API_KEY")
>       if all(os.environ.get(k) for k in keys):          # M4 / Fargate: env-injected
>           return {k: os.environ[k] for k in keys}
>       sm = boto3.client("secretsmanager")               # M1 / EC2: Secrets Manager
>       return {... sm.get_secret_value(...) ...}
>   ```
>   (Real implementation committed as `7c19086`.)
> - **(Alternative) Grant the task role `secretsmanager:GetSecretValue`** in Step 4b and keep `worker.py` as-is. Simpler diff, but the task role is no longer minimal and the image still hard-depends on Secrets Manager.
>
> Either is fine — but the skill must pick one. Default to the env-first refactor (keeps the task role minimal and the image portable). **Do this refactor before you push in 2c**, or the first Fargate task fails on import.

**2a. `worker/Dockerfile`** (ffmpeg baked in — the worker shells out to it):
```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends ffmpeg \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY worker.py .
# JOB_ID + Supabase + OpenAI + Anthropic creds arrive as env from the Lambda's RunTask override.
CMD ["python", "worker.py"]
```

**2b. `buildspec.yml`** (at the repo root, or set its path in the CodeBuild project) — logs in to ECR, builds, pushes `latest`:
```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_URI
  build:
    commands:
      - docker build -t $ECR_URI:latest ./worker
  post_build:
    commands:
      - docker push $ECR_URI:latest
```
CodeBuild runs on x86_64 Linux, so the image matches Fargate — **no `--platform` flag needed.**

**2c. Push to GitHub** — derive the repo URL, **never hardcode it** (each student's repo differs). Auth depends on environment (Cowork = PAT pasted into the session; laptop = existing `gh`/keychain) — see prereq §4a:
```bash
git -C <repo> remote get-url origin     # discover the repo
git -C <repo> add worker/Dockerfile buildspec.yml && git -C <repo> commit -m "M4: worker image + buildspec" && git -C <repo> push origin main
```
Confirm with the student before pushing (outward action).

**Verify:** the two files are visible on the repo's default branch.

---

### Step 3 — Create the ECR repo + CodeBuild project, verify a first build, then turn on push-to-build

This step is where the **business-logic / infrastructure split** shows up concretely. The image *build* is the part that changes often (every worker code edit), so we make it **trigger on `git push`**. The CodeBuild project, role, and ECR repo are infrastructure — created once here by `call_aws`, then rarely touched.

> **The plan for this step:** create the ECR repo + role + project (one-time infra) → **manually run one build and verify it** (proves the buildspec/role/ECR wiring in isolation) → **then register a GitHub webhook** so every future worker push rebuilds automatically with no `start-build`.

**3a. ECR repo:**
```
call_aws ecr create-repository --repository-name subtitle-worker
```
Note the `repositoryUri` (`<acct>.dkr.ecr.<region>.amazonaws.com/subtitle-worker`).

**3b. CodeBuild service role** — needs ECR push + CloudWatch logs:
```
call_aws iam create-role --role-name subtitleCodeBuildRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codebuild.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
call_aws iam attach-role-policy --role-name subtitleCodeBuildRole --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
call_aws iam attach-role-policy --role-name subtitleCodeBuildRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```
> **Cowork only:** Step 5c reuses this same role to have CodeBuild upload the Lambda layer/handler zips to S3 (the "CodeBuild as courier" pattern — the Cowork bash sandbox can't reach S3 itself). If you're in Cowork, also grant `s3:PutObject` on the artifacts bucket now so Step 5 doesn't stall:
> ```
> call_aws iam put-role-policy --role-name subtitleCodeBuildRole --policy-name CodeBuildS3Upload \
>   --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:PutObject","Resource":"arn:aws:s3:::<artifacts-bucket>/*"}]}'
> ```

**3c. Connect CodeBuild to GitHub (mandatory — the webhook needs it).** Import a GitHub PAT into CodeBuild once per AWS account. **This is required even for a public repo**, because registering a webhook needs repo-admin access — so the PAT must have **`repo` + `admin:repo_hook`** scope (not just `repo`):
```
call_aws codebuild import-source-credentials --token <PAT> --server-type GITHUB --auth-type PERSONAL_ACCESS_TOKEN
```
The token is stored in CodeBuild (used at build time + to register the webhook), not in the repo or the Cowork session beyond this call. Treat it as a secret; revoke at course end.

**3d. Create the CodeBuild project** — GitHub source, Linux container, **privileged mode** (required for `docker build`):
```
call_aws codebuild create-project --name subtitle-worker-build \
  --source 'type=GITHUB,location=https://github.com/<gh-user>/<repo>.git,buildspec=buildspec.yml' \
  --source-version main \
  --artifacts 'type=NO_ARTIFACTS' \
  --environment 'type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_SMALL,privilegedMode=true,environmentVariables=[{name=ECR_URI,value=<repositoryUri>}]' \
  --service-role <subtitleCodeBuildRole ARN>
```
> **`privilegedMode=true` is mandatory** — without it the build container can't run the Docker daemon and `docker build` fails. This is the CodeBuild equivalent of the arch/Docker traps in a local build.
> **`--source-version` accepts both `main` and `refs/heads/main`** — they're equivalent. Don't be surprised when `batch-get-projects` reads the value back as `refs/heads/main` even though you passed `main`; it's the same ref, just normalized.

**3e. Run ONE manual build and verify it** — *before* trusting the webhook. This proves the buildspec, role, and ECR wiring in isolation, so any setup error surfaces here rather than as a confusing webhook failure later. **You (Claude) run and verify this directly:**
```
call_aws codebuild start-build --project-name subtitle-worker-build
call_aws codebuild batch-get-builds --ids <build-id from start-build>   # poll until buildStatus → SUCCEEDED
call_aws ecr list-images --repository-name subtitle-worker              # confirms a latest tag
```
If `buildStatus` is `FAILED`, read the project's CloudWatch log group — usual causes: missing `privilegedMode`, role lacks ECR push, `buildspec.yml` path mismatch, or the GitHub PAT in 3c lacks `repo` scope. **Fix and re-run until SUCCEEDED. Do not proceed to 3f on a failed build.**

**3f. Turn on push-to-build (the webhook).** Now that a build is proven green, register a webhook so every push to `main` that touches the worker rebuilds automatically — no more manual `start-build`:
```
call_aws codebuild create-webhook --project-name subtitle-worker-build \
  --filter-groups '[[{"type":"EVENT","pattern":"PUSH"},{"type":"HEAD_REF","pattern":"^refs/heads/main$"}]]'
```
> Optional tightening: add a `{"type":"FILE_PATH","pattern":"^worker/"}` clause to the filter group so only worker-path changes trigger a rebuild (avoids rebuilding when only docs change). Keep it simple (PUSH on `main`) if unsure.

**Verify before moving on:** `call_aws codebuild batch-get-projects --names subtitle-worker-build` → the response now has a `webhook` block with a `url` and the PUSH/`main` filter group. From here, **a `git push` to the worker is the deploy** — CodeBuild rebuilds and pushes `:latest` with no human `start-build`.

---

### Step 4 — ECS Fargate cluster + task roles + task definition

**4a. Cluster:** `call_aws ecs create-cluster --cluster-name subtitle-workers`

**4b. Two IAM roles:**
- **Task execution role** — pull image from ECR + write logs. Attach `AmazonECSTaskExecutionRolePolicy`.
- **Worker task role** — what the running worker needs (essentially nothing AWS-side; it talks to Supabase/OpenAI over HTTPS via env creds). Keep minimal, separate from exec role. **This is exactly why Step 2's env-first refactor matters:** a minimal task role has no `secretsmanager:GetSecretValue`, so an unrefactored M1 worker (which reads creds from Secrets Manager on import) crashes here. If you instead chose the "grant the task role Secrets Manager access" alternative from Step 2, attach a `secretsmanager:GetSecretValue` inline policy to *this* role — otherwise leave it minimal.
```
call_aws iam create-role --role-name subtitleTaskExecutionRole --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
call_aws iam attach-role-policy --role-name subtitleTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
call_aws iam create-role --role-name subtitleWorkerTaskRole --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

**4c. Pre-create the CloudWatch log group** (do this **before** registering the task def):
```
call_aws logs create-log-group --log-group-name /ecs/subtitle-worker
```
> **Why this is mandatory, not optional.** The obvious shortcut is `"awslogs-create-group":"true"` in the task def — *let ECS make the group on first run*. But that requires the **execution role** to have `logs:CreateLogGroup`, and `AmazonECSTaskExecutionRolePolicy` (attached in 4b) grants only the log **write** side, **not** create. So with `create-group=true` and the managed policy alone, the **first Fargate task fails before the container even starts**:
> ```
> ResourceInitializationError: failed to validate logger args:
> AccessDeniedException ... not authorized to perform: logs:CreateLogGroup
> on resource: /ecs/subtitle-worker
> ```
> Pre-creating the group (one idempotent call — `ResourceAlreadyExistsException` is harmless) sidesteps this entirely. *(Alternative: attach `CloudWatchLogsFullAccess` to `subtitleTaskExecutionRole`. Pre-create is cleaner — it keeps the exec role minimal.)* Because we pre-create, the task def below **omits `awslogs-create-group`** rather than setting it `true`.

**4d. Register the task definition** (note: **no `awslogs-create-group`** — the group already exists from 4c):
```
call_aws ecs register-task-definition --family subtitle-worker \
  --requires-compatibilities FARGATE --network-mode awsvpc --cpu 1024 --memory 2048 \
  --execution-role-arn <subtitleTaskExecutionRole ARN> --task-role-arn <subtitleWorkerTaskRole ARN> \
  --container-definitions '[{"name":"worker","image":"<repositoryUri>:latest","essential":true,
    "logConfiguration":{"logDriver":"awslogs","options":{"awslogs-group":"/ecs/subtitle-worker","awslogs-region":"<region>","awslogs-stream-prefix":"worker"}}}]'
```

**Verify + optional one-shot sanity test** (prove image/roles/env before wiring the Lambda):
```
call_aws ecs run-task --cluster subtitle-workers --task-definition subtitle-worker --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[<subnet>],assignPublicIp=ENABLED}' \
  --overrides '{"containerOverrides":[{"name":"worker","environment":[{"name":"JOB_ID","value":"<a real pending job>"},{"name":"SUPABASE_URL","value":"..."},{"name":"SUPABASE_SECRET_KEY","value":"..."},{"name":"OPENAI_API_KEY","value":"..."}]}]}'
```
`assignPublicIp=ENABLED` on a public subnet so the task reaches the internet without a NAT gateway. Watch the job advance in Supabase + `/ecs/subtitle-worker` logs (the group exists from 4c).

> **Env-var naming:** use **`SUPABASE_SECRET_KEY`** throughout (matching `worker.py`, Vercel, and Supabase's current "publishable/secret" terminology). The older `SUPABASE_SERVICE_KEY` / "service_role JWT" naming is being phased out — don't mix the two or the worker reads `None`.

> **Test-path trap (direct-INSERT jobs).** If you create a test job by inserting straight into `jobs` + `job_sessions` (instead of going through `POST /api/jobs`), make sure `jobs.current_session_id` actually points at the session row — see Step 5's `_ensure_session` note. A null `current_session_id` crashes `worker.py` at its first `update_session(...)`, which looks like an image/env failure but isn't.

---

### Step 5 — Build the Lambda distributor + its deps layer

The Lambda replaces `distributor.py`. **For M4 it does one thing: spawn pending jobs.** On each tick it `SELECT`s pending jobs `WHERE fargate_task_arn IS NULL`, ensures a session exists (and **links it**, see 5b), calls `ecs.run_task`, then writes the ARN. It imports **only** `boto3` + `supabase` — **no OpenAI/Anthropic** (it forwards those keys to the Fargate task via `containerOverrides[].environment`).

> **Scope note (M8 — what M4's Lambda does NOT do yet).** The production distributor has three passes: (1) spawn pending jobs, (2) stuck-job recovery (creates session N+1, nulls heavy content on the abandoned session), (3) daily DB-timestamp-gated storage cleanup. **M4 ships only pass (1).** Passes (2) and (3) are a **v2 roadmap** item — the precise "session N+1" semantics aren't trivial and aren't needed to prove the serverless loop end-to-end. Don't block M4 on them; build pass (1) cleanly and note (2)/(3) as future work. (If you later add recovery, the same `_ensure_session` linking rule in 5b applies to the new session.)

**5b. The handler — and the one link that's easy to miss (`_ensure_session`).** The naive spec is "find or create a `job_sessions` row, then `run_task`." Implemented literally, the Lambda **creates the session row but never sets `jobs.current_session_id` to point at it.** Production traffic dodges this because `POST /api/jobs` creates both rows atomically and sets the link — but **any direct-INSERT test job** (what you'll use to verify the pipeline structurally, see Step 4's test-path trap) then crashes `worker.py` at `update_session(...)` because `current_session_id` is null. So `_ensure_session` **must set the link after inserting:**
```python
def _ensure_session(db, job_id):
    job = db.table("jobs").select("current_session_id").eq("id", job_id).single().execute().data
    if job.get("current_session_id"):
        return job["current_session_id"]                       # already linked — reuse
    row = db.table("job_sessions").insert(
        {"job_id": job_id, "session_number": 1}).execute().data[0]
    db.table("jobs").update(                                   # ← the easy-to-miss link
        {"current_session_id": row["id"]}).eq("id", job_id).execute()
    return row["id"]
```
The `jobs.update(current_session_id=...)` line is the fix (committed as `f422557`). Without it the spawn "works" (a task launches) but the worker dies immediately on a null link. Spell this out — the trap is non-obvious.

**5c. Deps layer.** The layer is tiny (`supabase` + `boto3`). But **getting the zip into S3 differs sharply by environment** — this is the step that silently assumes S3 reach:

- **CLI mode (laptop):** straightforward. The laptop reaches S3 directly.
  ```bash
  pip download --platform manylinux2014_x86_64 --only-binary=:all: --target python supabase boto3
  zip -r deps-layer.zip python && aws s3 cp deps-layer.zip s3://<bucket>/deps-layer.zip
  ```
- **Cowork mode — `aws s3 cp` from bash does NOT work, and there is no obvious workaround.** Two hard walls: (a) the **bash sandbox is proxy-blocked from S3** (`X-Proxy-Error: blocked-by-allowlist on s3.amazonaws.com` — pre-signed PUT URLs and raw `curl` hit the same proxy, so they don't escape either); (b) the **AWS API MCP runs in a separate sandbox** with its own workdir (`/tmp/aws-api-mcp/workdir`) — files you write from bash aren't visible to it. So you cannot build the zip in bash and upload it, and you cannot hand it to `call_aws`. **The only viable path in Cowork is to make CodeBuild the courier** — let the build container (which has full AWS reach) download the deps and `s3 cp` them up itself. Reuse the existing CodeBuild project with a `--buildspec-override` pointing at a small helper buildspec:
  ```yaml
  # lambda-build.buildspec.yml  — quote EVERY command (see the :all: YAML trap below)
  version: 0.2
  phases:
    build:
      commands:
        - "pip download --platform manylinux2014_x86_64 --only-binary=:all: --target python supabase boto3"
        - "zip -r deps-layer.zip python"
        - "aws s3 cp deps-layer.zip s3://${ARTIFACTS_BUCKET}/deps-layer.zip"
        - "zip handler.zip lambda_distributor.py && aws s3 cp handler.zip s3://${ARTIFACTS_BUCKET}/handler.zip"
  ```
  ```
  call_aws codebuild start-build --project-name subtitle-worker-build \
    --buildspec-override lambda-build.buildspec.yml \
    --environment-variables-override 'name=ARTIFACTS_BUCKET,value=<bucket>,type=PLAINTEXT'
  ```
  This requires adding **`s3:PutObject`** (on the artifacts bucket) to `subtitleCodeBuildRole`. **Parameterize the bucket via `${ARTIFACTS_BUCKET}` — never hardcode it** (a hardcoded bucket breaks the moment you mirror to a second AWS account).
  > **⚠️ YAML `:all:` trap (M5).** The pip flag `--only-binary=:all:` contains a `: ` (colon-space) sequence that YAML reads as a key/value separator — CodeBuild rejects the unquoted command with `YAML_FILE_ERROR: Expected Commands[1] to be of string type: found subkeys instead`. **Wrap every command line in double quotes** (as above) and it parses.

Then publish the layer from the zip now sitting in S3:
```
call_aws lambda publish-layer-version --layer-name subtitle-distributor-deps \
  --content S3Bucket=<bucket>,S3Key=deps-layer.zip --compatible-runtimes python3.12
```
Commit `worker/lambda_distributor.py` and `lambda-build.buildspec.yml` to the repo for the record.

**5d. Lambda execution role** — `ecs:RunTask`/`DescribeTasks`/`StopTask`, `iam:PassRole` scoped to the two task roles from Step 4, plus `AWSLambdaBasicExecutionRole`.

**5e. Create the function** (the handler zip is already in S3 from 5c in Cowork; from a laptop you can also `--zip-file fileb://handler.zip` directly):
```
call_aws lambda create-function --function-name subtitle-distributor \
  --runtime python3.12 --handler lambda_distributor.handler --role <Lambda role ARN> \
  --layers <deps layer ARN> --timeout 300 --memory-size 256 \
  --code S3Bucket=<bucket>,S3Key=handler.zip \
  --environment 'Variables={SUPABASE_URL=...,SUPABASE_SECRET_KEY=...,OPENAI_API_KEY=...,ANTHROPIC_API_KEY=...,ECS_CLUSTER=subtitle-workers,TASK_DEFINITION=subtitle-worker,SUBNETS=<subnet>}'
```

**Verify:** `get-function` → `State: Active`. Manual one-shot `call_aws lambda invoke` → confirm in CloudWatch the spawn pass ran and (if a pending job exists) a Fargate task launched + ARN landed in `job_sessions.fargate_task_arn`.

---

### Step 6 — Wire the EventBridge schedule (but don't enable until Step 7)

```
call_aws events put-rule --name subtitle-distributor-schedule --schedule-expression 'rate(1 minute)'
call_aws lambda add-permission --function-name subtitle-distributor --statement-id eventbridge-invoke \
  --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn <rule ARN>
call_aws events put-targets --rule subtitle-distributor-schedule --targets 'Id=1,Arn=<Lambda ARN>'
```

---

### Step 7 — Stop the EC2 distributor, THEN confirm the rule is live

**Order matters** — never run both distributors on one Supabase. Stop EC2 *first*.

**7a. Stop the M1 EC2 distributor + the instance — do this FOR the student, via `call_aws` (no AWS console).** We **stop** (not terminate) here on purpose: until the end-to-end test in `m4-serverless-checklist` passes, the EC2 is the safety fallback. The checklist **terminates it for the student once everything is green** — so the student never opens the EC2 console. Kill the distributor process via SSM (no SSH), then stop the instance:
```
call_aws ssm send-command --instance-ids <id> --document-name "AWS-RunShellScript" --parameters 'commands=["tmux kill-session -t worker || pkill -f distributor.py || true"]'
call_aws ec2 stop-instances --instance-ids <id>
```
Tell the student: 「我已經幫你把 EC2 worker 停掉了（stop，不是 terminate）。先留著當保險，等 `驗收 M4` 全綠之後我會直接幫你把它 terminate 掉，你不用進 AWS console。」

**7b. Ensure the rule is ENABLED:** `call_aws events enable-rule --name subtitle-distributor-schedule`.

**Verify:** EC2 `stopped`; rule `ENABLED` at `rate(1 minute)`; after ~2 min, CloudWatch `/aws/lambda/subtitle-distributor` shows it firing each minute.

---

### Step 8 — End-to-end proof + redeploy loop

1. Submit a real job on the product URL; within ~1 min the Lambda spawns a Fargate task and stamps `fargate_task_arn`; the next tick skips it (idempotency).
   > **Recommended first test URL** (CloudFront, ~2 min clip, ~3 credits — a known-good baseline that rules out "the URL is the problem" before you debug M4 infra):
   > ```
   > https://dqwd87ogl0f9o.cloudfront.net/950371295/mp4/950371295_1920x1080.mp4?v=1716669006
   > ```
   > **Expected on success:** Fargate task `STOPPED` exit 0, ~3 min total; `jobs.status='done'`; `length(subtitle_txt_content) ≈ 2,200–2,400` chars (Whisper, language=zh); user balance debited 3 credits; one `credit_transactions` row with `amount=-3`. If your *first* test fails with a *different* URL, retry with this one to isolate "URL problem" from "M4 infra problem."
2. Job advances `pending → … → done` on Fargate; logs in `/ecs/subtitle-worker`. The EC2 is `stopped` throughout (still there as a fallback, but unused).
3. **Future worker changes are just a `git push`.** Edit `worker/`, `git push origin main` → the CodeBuild **webhook rebuilds + pushes `:latest` automatically** (no `start-build`); new Fargate tasks pull the new image (force-new if you want it immediately). **No EC2 ever re-enters the loop, and no manual build step.** This is the business-logic-rollout half of M4: code changes ship by push; the infrastructure underneath stays put.

Run `m4-serverless-checklist` for the full sweep. **The EC2 is still `stopped` (your fallback) at the end of this skill — the checklist terminates it for you once the end-to-end test passes.**

---

## Post-M4 state

| Layer | After M1 / M3 | After M4 |
|---|---|---|
| Distributor | Always-on EC2 `distributor.py` (~$15/mo) | Lambda on EventBridge `rate(1 min)` (~$1.50/mo) |
| Worker compute | Single EC2 CPU shared | Isolated Fargate task per job, **any video length** |
| Image build | (ran on the EC2) | **CodeBuild** from GitHub — no EC2, no local Docker |
| Concurrency | Bounded by EC2 size | Effectively unlimited |
| Cost when idle | EC2 24/7 | ~$0 (scales to zero) |
| Idempotency | In-memory `_spawned_jobs` | DB `job_sessions.fargate_task_arn` |
| EC2 | Running | **Stopped after this skill; terminated by the checklist once green — nothing depends on it either way** |
| Web / DB / Stripe / domain | — | **Unchanged** |

## Switching back to the EC2 fallback

Only if you kept the EC2 stopped (not terminated):
```
call_aws events disable-rule --name subtitle-distributor-schedule   # disable FIRST
call_aws ec2 start-instances --instance-ids <id>
call_aws ssm send-command --instance-ids <id> --document-name "AWS-RunShellScript" --parameters 'commands=["cd /home/ubuntu/app && tmux new -d -s worker ./venv/bin/python distributor.py"]'
```
Re-enable the rule only after stopping the EC2 distributor again. **One distributor per Supabase, always.** (If you terminated the EC2, the fallback is gone — that's the intended end state.)

## Future enhancements (v2 roadmap)

- **CDK / IaC.** M4 is provisioned imperatively via `call_aws` so the student sees each piece. Today the repo is CodeBuild's *build source* for the worker image, but the AWS resources (CodeBuild project, ECR, ECS, Lambda, EventBridge, IAM) are created by hand. Capture them in CDK so the whole stack is `cdk deploy`-able and the repo becomes the full deploy source, not just the image source.
- **Auto-rollout to running tasks.** Push-to-build is already on (the CodeBuild webhook rebuilds `:latest` on every worker push). The next step is auto-*rollout*: have the new image picked up without waiting for the next natural Fargate task — e.g. a post-build hook that forces-new any in-flight service, or versioned image tags + a task-def update. (M4 today relies on the next job's task pulling `:latest`.)
- **Dev + prod stacks (multi-account).** Two distributors + clusters, same code, different env vars, one per Supabase/stage. **If you provision M4 in a second AWS account** (dev→prod, or a wrong-account→correct-account migration): `import-source-credentials` is **per-account**, so re-import the same GitHub PAT into the new account's CodeBuild. Both accounts' webhooks then fire on every push to the shared repo (harmless but each incurs build minutes) — `call_aws codebuild delete-webhook` on the inactive account to silence it. And any helper buildspec (e.g. `lambda-build.buildspec.yml`) must take the artifacts bucket via `${ARTIFACTS_BUCKET}`, never a hardcoded name, or it breaks the moment you mirror accounts.
- **Lambda-only tier for short videos.** For a cheaper tier that skips Fargate entirely, a single worker Lambda can transcribe short clips directly (accepting the 15-min cap with a clear error on long videos). Useful as a low-end option; Fargate remains the any-length path.
- **Terminate the EC2 + clean up** its KeyPair/SG if you only stopped it.

## Related skills

- [[m4-serverless-prerequisites]] — connector check + GitHub push auth + M1-green confirmation
- [[m4-serverless-checklist]] — verifies M4 after this skill completes
- [[m1-ai-video-transcript]] — the EC2 worker M4 migrates away from (same `worker.py`, now containerized)
- [[aws-best-practice]] — IAM, SSM-not-SSH, region rules apply to every `call_aws` here
- [[supabase-best-practice]] — the `fargate_task_arn` column must go through a migration file
- [[project-2-ai-video-reader]] — the full M0→M4 architecture progression
