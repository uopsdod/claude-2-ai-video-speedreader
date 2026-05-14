---
name: m1-ai-video-transcript-prerequisites
description: One-time external-account + EC2 setup the student needs BEFORE starting M1 (Course 2 module 2.2 — 打造 AI 影片摘要核心功能). Adds OpenAI + AWS on top of M0's accounts, launches one t3.small Ubuntu EC2 entirely through MCP (Supabase + AWS API MCP) — no SSH, no key pair, no terminal. Access the EC2 via AWS Systems Manager (SSM) Session Manager. Use when the student says "M1 環境準備", "setup for M1", or when `m1-ai-video-transcript` / `m1-ai-video-transcript-checklist` detects a missing OpenAI key / EC2.
---

# M1 Prerequisites — OpenAI + AWS EC2 (SSM-only)

## What this skill does

Adds two accounts on top of M0's four (GitHub / Lovable / Supabase / Vercel) and provisions one Ubuntu EC2 instance that runs the worker for the rest of M1. **No SSH. No key pair. No PEM file. Everything goes through MCP** — Supabase MCP for DB writes, AWS API MCP (`suggest_aws_commands` + `call_aws`) for EC2 lifecycle, AWS Systems Manager (SSM) for in-instance commands.

| # | Service | Used for |
|---|---|---|
| 5 | **OpenAI** | Whisper transcription (`whisper-1`) |
| 6 | **AWS** | One EC2 instance + IAM instance profile for SSM |

By the end of this skill the student has:

1. An OpenAI API key (`sk-...`) with at least \$5 of credit.
2. An AWS account with **AWS API MCP** installed and authenticated in Cowork (or `aws` CLI authenticated, in CLI mode — but the SSM commands work identically either way).
3. One running `t3.small` Ubuntu 22.04 EC2 instance, tagged `Name=m1-worker`, with an IAM instance profile that grants SSM Session Manager + SSM Run Command.
4. `ffmpeg`, `python3.12`, `python3.12-venv`, `git` installed on the EC2 via `ssm send-command` (no SSH).
5. The M0 GitHub repo cloned to `/home/ubuntu/app` on the EC2.
6. A Python venv pre-created at `/home/ubuntu/app/worker/venv` ready for the M1 worker to install requirements into.

**No port 22 is open. There is no SSH key.** Everything you do on the EC2 from now on goes through `call_aws ssm send-command` or `call_aws ssm start-session`.

## How M1 will deploy code to this EC2 (preview)

![AI Video Reader architecture](assets/ai_video_reader_structure.jpg)

*User → GitHub repo → Vercel-hosted Next.js writes to Supabase; the EC2 you provision here reads pending jobs, runs OpenAI Whisper, writes transcripts back.*

This prereq builds the EC2; the **main M1 skill** writes the worker code and starts deploying to it. So you (or the student) understands what infrastructure to optimize for: the deploy model is **GitHub-pull + systemd**, no Docker, no CI, no SSH.

- The M1 worker code lives in the **same GitHub repo as the M0 web app** — under a new `worker/` subdirectory the main skill creates in Step 5. (Vercel auto-deploys `app/` as before; it ignores `worker/`.)
- Code reaches the EC2 by `git pull` *into* `/home/ubuntu/app` (which §2.7 below cloned). The pull is triggered through `call_aws ssm send-command` from Cowork — no `scp`, no `ssh`, no PEM file.
- The distributor process runs as a **systemd service** (`m1-distributor.service`) installed by the main skill in Step 6. systemd handles auto-start on EC2 reboot + auto-restart on crash. Updates: pull → `pip install` → `systemctl restart m1-distributor.service`. All three steps go through `ssm send-command`.

That means the EC2 you build here just needs: SSM agent reachable (handled by §2.2 IAM role), `git` + Python venv (handled by §2.6/§2.7), and read access to the four secrets in Secrets Manager (handled by §2.3). Nothing else. The main skill never has to install Docker, configure a CI runner, or open inbound ports.

## Required MCPs (read this first)

This skill assumes you have:

- ✅ **Supabase MCP** (`mcp__supabase_remote__*`) — already installed for M0.
- ✅ **AWS API MCP** (`call_aws`, `suggest_aws_commands`) — published by Amazon Web Services, available in the Cowork plugin/connector directory under "AWS API MCP Server". Install it via the Cowork connector directory, then authenticate it in **Section 2.0** below (you need an AWS account first — Section 2.1).

If AWS API MCP isn't installed: pause this skill, install it in Cowork, then resume. The whole point of this path is to keep the student out of the AWS console and the terminal — without AWS API MCP, you're stuck doing AWS-console clicks manually.

> **Why no SSH?** SSH on EC2 is fine when you have a laptop + a `.pem` file + a stable IP. In Cowork there's no laptop filesystem to hold the PEM, and the network egress story is fiddly. AWS Systems Manager (SSM) gives us a better answer: the EC2 attaches to SSM via an IAM role; we send commands from Cowork via `call_aws ssm send-command` and open a shell (if needed) via `call_aws ssm start-session`. No inbound port. No key. Everything audited in CloudTrail. This is the same pattern the production worker in this repo uses (see `feedback_ec2_commands.md`).

## Section 1 — OpenAI API key

1. Go to https://platform.openai.com/, sign up (or log in).
2. **Settings → Billing → Add payment method**, add a card.
3. **Add to credit balance** → top up at least **\$5**.
4. **Settings → Limits** → set a **soft monthly cap** (e.g. \$10). This is your runaway-worker insurance.
5. **API keys → Create new secret key**. Copy the `sk-...` value — OpenAI only shows it once.

Hold onto this `sk-...`. We'll inject it into the EC2 via AWS Secrets Manager in Section 2.3.

## Section 2 — AWS account + EC2 (all via MCP)

### 2.1 — AWS account + region

Sign up at https://aws.amazon.com/ if needed. Pick **one region** and stick with it for the whole course (we recommend `us-west-2`, `ap-northeast-1`, or `ap-southeast-1` for latency). The region you pick here is the same one you'll plug into Cowork's AWS API MCP settings in Section 2.0 next.

### 2.0 — Wire AWS credentials into Cowork

> Prerequisite: complete Section 2.1 above (AWS account + region pick), then come back here. Numbered 2.0 — not 2.2 — so this section is the FIRST thing you do inside the AWS account, before any `call_aws ...` invocation in Sections 2.2+.

Cowork's AWS API MCP needs an AWS access key + secret to call `call_aws` on your behalf. The fastest course path is to create a dedicated IAM user with admin permissions, generate one access key, paste it into Cowork's MCP settings, and set a calendar reminder to delete the key when the course is done.

> **Why a dedicated IAM user (not your AWS root account):** never use root for day-to-day work — even for solo learning. Root keys have no scoping, no audit per-key, no easy rotation. A named IAM user is the AWS-recommended floor.

#### 2.0.1 — Create the IAM user

In the AWS console (browser):

1. **IAM → Users → Create user**
   - User name: `cowork-m1`
   - Do **not** check "Provide user access to the AWS Management Console" — this user is for API access only.
2. **Permissions → Attach policies directly → AdministratorAccess** → Next → Create user.

> **About `AdministratorAccess`:** for a learning course this is the right call — Sections 2.2–2.7 of this skill touch IAM, EC2, SSM, and KMS, and the exact-minimum policy is ~15 actions across those services. We trade tighter IAM scope for a smoother first-time AWS experience. **At end of course, delete the key (2.0.5 below).** A future "production hardening" appendix can replace `AdministratorAccess` with the minimum scope.

#### 2.0.2 — Generate the access key

1. Click into the freshly-created `cowork-m1` user → **Security credentials** tab → **Create access key**.
2. Use case: **Application running outside AWS** → Next → Create access key.
3. AWS shows two values **once and only once**:
   - **Access key ID** — looks like `AKIA...` (20 chars)
   - **Secret access key** — looks like `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
4. Copy both somewhere safe (password manager). If you close this page without copying the secret, AWS won't show it again — you'll have to delete the key and make a new one.

#### 2.0.3 — Paste into Cowork's AWS API MCP settings

1. In Cowork: open the AWS API MCP connector / plugin settings (the same screen you used to install the connector).
2. Fill in:
   - **AWS Access Key ID:** `AKIA...` from 2.0.2
   - **AWS Secret Access Key:** `wJalrXUt...` from 2.0.2
   - **Region:** the same one you picked in 2.1 (`us-west-2`, `ap-northeast-1`, or `ap-southeast-1`)
3. Save. Cowork should now show the AWS API MCP as connected.

#### 2.0.4 — Verify (one MCP call)

Run a harmless read to confirm credentials work:

`call_aws sts get-caller-identity`

Expect a JSON response with:
- `UserId` ending in `:cowork-m1`
- `Arn` ending in `user/cowork-m1`
- An `Account` number matching the AWS account you signed up with

If you get `Unable to locate credentials` or `InvalidClientTokenId`: the key/secret were typo'd or got truncated on copy. Delete the key in IAM and create a new one.

#### 2.0.5 — End-of-course cleanup (set this reminder NOW)

Long-lived access keys are the single biggest source of accidentally-leaked AWS credentials. Reduce the blast radius:

- **After every M1 lecture session:** nothing to do — Cowork holds the key, the student doesn't paste it anywhere else.
- **End of M4 / end of course:** **IAM → Users → cowork-m1 → Security credentials → Access keys → Actions → Delete**. The MCP will stop working; that's fine, you're done with the course.
- **If you ever paste the key into a chat / commit / screenshot by accident:** delete it immediately (same path) and create a new one. Anyone who sees `AKIA...` + the matching secret has full AWS access until you revoke.

Add a calendar reminder for ~6 weeks out: "Delete cowork-m1 access key if Course 2 is done."

#### 2.0.6 — CLI-mode variant (skip if you're in Cowork)

CLI-mode students do the same 2.0.1 + 2.0.2, then instead of pasting into Cowork:

```bash
aws configure
# AWS Access Key ID:     <paste>
# AWS Secret Access Key: <paste>
# Default region name:   us-west-2
# Default output format: json
```

`aws sts get-caller-identity` does the same verify as 2.0.4.

### 2.2 — Create the IAM instance profile (one MCP call)

The EC2 needs an IAM role that grants:
- `AmazonSSMManagedInstanceCore` (the managed policy that gives SSM agent permission to phone home — that's how Cowork will run shell commands on the EC2 in 2.6 + 2.7 without SSH)
- Read on the four Secrets Manager secrets we'll create in 2.3 (`openai-api-key`, `supabase-url`, `supabase-secret-key`, `supabase-publishable-key`) — added in Section 2.3 below as a separate inline policy on this same role

Ask AWS API MCP to do it:

```
call_aws iam create-role --role-name m1-worker-role --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}'

call_aws iam attach-role-policy --role-name m1-worker-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

call_aws iam create-instance-profile --instance-profile-name m1-worker-profile
call_aws iam add-role-to-instance-profile --instance-profile-name m1-worker-profile --role-name m1-worker-role
```

(`suggest_aws_commands "create an instance profile with SSM access named m1-worker"` will produce similar — the canonical recipe is above.)

**Verify:** `call_aws iam get-instance-profile --instance-profile-name m1-worker-profile` returns a JSON object with the role attached.

### 2.3 — Store OpenAI + Supabase secrets in AWS Secrets Manager

#### Why the EC2 needs three secrets, not one

The M1 architecture has two writers to Supabase:

```
[User browser] ──submit─▶ [Vercel: Next.js]  ──W: insert job row──▶ [Supabase]
                                                                       ▲    │
                                                              R: poll  │    │ W: write transcript back
                                                              pending  │    ▼
                                                                   [AWS EC2 worker]  ──▶ [OpenAI Whisper]
```

- **Vercel** writes new `jobs` rows when a user submits the form. It already reads its own Supabase env vars (set in M1 main skill Step 4) — nothing new for Vercel here.
- **The EC2 worker** does *both* reads and writes against Supabase: it polls `jobs` for pending rows, claims one, runs Whisper, then writes the transcript back to `job_sessions`. So the EC2 needs the Supabase URL + **Secret key** (`sb_secret_...`) at runtime — not the Publishable key (`sb_publishable_...`). The Publishable key is RLS-gated and would return zero rows when the worker queries `select * from jobs where status='pending'` (no user session = nothing visible). The Secret key bypasses RLS so the worker can see and update jobs across all users.

Cowork having a Supabase MCP doesn't help the EC2 — that connector is how *Cowork* talks to Supabase, not how the EC2 does. The EC2 is a separate Python process with no MCP, just `boto3` + `supabase-py`. We give it Supabase access by storing the credentials in AWS Secrets Manager and granting the EC2's IAM role permission to read them.

OpenAI is the third secret — the worker calls Whisper directly with that key.

#### How the secret values flow from your eyes into Secrets Manager

You paste each value into the Cowork chat as part of your prompt to Claude. Claude wraps it in a `call_aws secretsmanager create-secret` call. The value flows: your message → Cowork → AWS API MCP → AWS Secrets Manager (encrypted at rest with KMS). Once stored, you don't need the plaintext value again — only the EC2 reads it.

> **Chat-retention caveat:** the raw `sk-...` / `sb_secret_...` value briefly appears in your Cowork chat transcript on its way to `create-secret`. Cowork retains chat history; assume the value is recoverable from your account history for as long as that conversation is retained. This is acceptable for course-grade keys (the OpenAI cap is small, the Supabase secret key is scoped to a learning project), but you should still **rotate both keys at end of course** (revoke the OpenAI key in OpenAI dashboard → Settings → API Keys; rotate the Supabase secret key in Supabase dashboard → Project Settings → API Keys → roll). For a real production deployment you'd type the values into AWS Console → Secrets Manager via your browser, never through the chat — out of scope for this course.

#### Tell Claude

> 「請把這四個 secret 存到 AWS Secrets Manager：
>   - `openai-api-key` = `<paste your sk-... here>`
>   - `supabase-url` = `https://<your-ref>.supabase.co`
>   - `supabase-secret-key` = `<paste your Supabase Secret key — `sb_secret_...` — from Supabase dashboard → Project Settings → API Keys. This is the RLS-bypass key the EC2 worker uses.>`
>   - `supabase-publishable-key` = `<paste your Supabase Publishable key — `sb_publishable_...` — from the same page. RLS-gated, browser-safe.>`」

> **Why store both Supabase keys?** The M1 EC2 worker only reads the **Secret key** (it needs to bypass RLS to see all users' pending jobs). The **Publishable key** is stored alongside it as a convenience: future milestones may want to give the EC2 an RLS-respecting client (e.g. when the worker acts on behalf of a specific user), and having both in one place means students don't have to re-fetch from the Supabase dashboard later. M1 itself doesn't read it.

Claude will run four `call_aws secretsmanager create-secret` invocations, one per secret:

```
call_aws secretsmanager create-secret --name openai-api-key --secret-string '<sk-...>'
call_aws secretsmanager create-secret --name supabase-url --secret-string 'https://<ref>.supabase.co'
call_aws secretsmanager create-secret --name supabase-secret-key --secret-string '<sb_secret_...>'
call_aws secretsmanager create-secret --name supabase-publishable-key --secret-string '<sb_publishable_...>'
```

> If a secret already exists from a previous run, `create-secret` errors with `ResourceExistsException`. Use `put-secret-value --secret-id <name> --secret-string '<new value>'` to update an existing secret in place.

Then add the IAM permission to read these to the worker role:

```
call_aws iam put-role-policy --role-name m1-worker-role --policy-name m1-secrets-read --policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": [
      "arn:aws:secretsmanager:*:*:secret:openai-api-key-*",
      "arn:aws:secretsmanager:*:*:secret:supabase-url-*",
      "arn:aws:secretsmanager:*:*:secret:supabase-secret-key-*",
      "arn:aws:secretsmanager:*:*:secret:supabase-publishable-key-*"
    ]
  }]
}'
```

> The `-*` suffix on each ARN matches Secrets Manager's auto-appended random suffix (e.g. `openai-api-key-AbCdEf`). It scopes the role to *exactly* these four secret names without granting `secretsmanager:GetSecretValue *`.

> **Why Secrets Manager (not Parameter Store)?** Both encrypt at rest with KMS, both can be scoped via IAM. Secrets Manager is the AWS-native home for application credentials — it has rotation hooks, audit-logging built in, and a more obvious name for "this is a secret." Parameter Store works fine for non-secret config; we use Secrets Manager here because the OpenAI key + Supabase keys are unambiguously credentials.

**Verify:** `call_aws secretsmanager list-secrets --query 'SecretList[?Name==\`openai-api-key\` || Name==\`supabase-url\` || Name==\`supabase-secret-key\` || Name==\`supabase-publishable-key\`].Name'` returns the four names. (Don't fetch the `SecretString` values unless you're debugging — they're still plaintext at that point.)

### 2.4 — Create a minimal security group

No inbound rules at all. SSM is fully outbound — the EC2 phones home to SSM endpoints over HTTPS (443).

```
call_aws ec2 create-security-group --group-name m1-worker-sg --description "M1 worker — outbound only, SSM-managed"
```

Note the returned `GroupId` (looks like `sg-0abc...`). Default outbound (allow all) is fine — no inbound edits needed.

### 2.5 — Launch the EC2 (one MCP call)

```
call_aws ec2 run-instances \
  --image-id <latest Ubuntu 22.04 AMI for your region> \
  --instance-type t3.small \
  --iam-instance-profile Name=m1-worker-profile \
  --security-group-ids <sg-0abc...> \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=m1-worker}]' \
  --count 1
```

**To find the AMI ID:** `call_aws ssm get-parameters --names /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id --query 'Parameters[0].Value' --output text` returns the latest Canonical-published Ubuntu 22.04 image in your region.

**Wait for SSM to register the instance** (usually 1–2 min after launch):

```
call_aws ssm describe-instance-information --filters "Key=tag:Name,Values=m1-worker"
```

When the instance appears with `PingStatus=Online`, it's ready for `send-command`. If it never appears, the IAM instance profile is the most likely culprit — confirm `m1-worker-profile` is attached via `call_aws ec2 describe-instances --filters "Name=tag:Name,Values=m1-worker" --query 'Reservations[0].Instances[0].IamInstanceProfile'`.

### 2.6 — Install ffmpeg / Python / git via SSM (no SSH)

Grab the instance ID:

```
INSTANCE_ID=$(call_aws ec2 describe-instances --filters "Name=tag:Name,Values=m1-worker" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].InstanceId' --output text)
```

Run the install script via Run Command:

```
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --comment "M1: install ffmpeg, python3.12, git" \
  --parameters 'commands=[
    "sudo apt-get update",
    "sudo apt-get install -y python3.12 python3.12-venv python3-pip ffmpeg git",
    "ffmpeg -version | head -1",
    "python3.12 --version"
  ]'
```

The call returns a `CommandId`. Wait ~30 s, then read the output:

```
call_aws ssm list-command-invocations --command-id <CommandId> --details --query 'CommandInvocations[0].CommandPlugins[0].Output' --output text
```

You should see `ffmpeg version 4.x...` and `Python 3.12.x` at the end. If apt errors, retry — Ubuntu's apt sometimes needs a moment after first boot.

### 2.7 — Clone the M0 repo + create the venv

Same shape — one more `send-command`:

```
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --comment "M1: clone repo + create venv" \
  --parameters 'commands=[
    "sudo -u ubuntu bash -c \"cd /home/ubuntu && git clone https://github.com/<student-username>/<repo-name>.git app\"",
    "sudo -u ubuntu bash -c \"mkdir -p /home/ubuntu/app/worker && cd /home/ubuntu/app/worker && python3.12 -m venv venv && ./venv/bin/pip install --upgrade pip\"",
    "ls /home/ubuntu/app",
    "ls /home/ubuntu/app/worker/venv/bin/python"
  ]'
```

Replace `<student-username>/<repo-name>` with the GitHub repo URL from M0. If the repo is private, the student needs a deploy key or a personal-access-token clone URL — pause and walk through that, then retry.

Verify the command output via `list-command-invocations` again.

## Verify (one-shot)

The end-of-prereq health check, all via MCP, no SSH:

```
# 1. EC2 is running + SSM-managed
call_aws ssm describe-instance-information --filters "Key=tag:Name,Values=m1-worker"
# Expect: one row, PingStatus=Online

# 2. ffmpeg + python installed
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["ffmpeg -version | head -1 && python3.12 --version && ls /home/ubuntu/app/worker/venv/bin/python"]'
# (then list-command-invocations to read the output)

# 3. Secrets in AWS Secrets Manager
call_aws secretsmanager list-secrets --query 'SecretList[?Name==`openai-api-key` || Name==`supabase-url` || Name==`supabase-secret-key` || Name==`supabase-publishable-key`].Name'
# Expect: all four names present
```

If everything looks good, you're ready for the M1 main skill.

## Cost reminder

`t3.small` always-on ≈ **\$15/month** (us-west-2 on-demand as of writing). EBS 20 GiB gp3 ≈ \$1.60/mo whether the instance is running or stopped.

**Stop the EC2 when not actively running an M1 lecture:**

```
call_aws ec2 stop-instances --instance-ids "$INSTANCE_ID"
```

To resume:

```
call_aws ec2 start-instances --instance-ids "$INSTANCE_ID"
```

A **stopped** instance pays ~\$0 for compute (EBS only). The instance ID stays the same, so the same `send-command` workflow keeps working.

> **No Elastic IP needed.** SSM doesn't care about the instance's public IP — it talks to AWS endpoints via the SSM agent. Skip the EIP cost entirely.

## CLI-mode variant (if the student isn't in Cowork)

Everything in this skill works identically with the `aws` CLI in a local terminal — replace each `call_aws <subcommand>` with `aws <subcommand>`. The only difference: in CLI mode the student authenticates `aws` (via `aws configure` with an IAM access key, or `aws sso login`); in Cowork mode the AWS API MCP is already authenticated through Cowork's connector setup.

The **SSH/PEM/EC2-console path is intentionally NOT documented** in this skill. SSM is the only access pattern. If a student insists on SSH (or their org disallows SSM), they'll need a different M1 prereq variant — not in scope here.

## When done

Tell the student:

> 「M1 環境準備完成 — OpenAI key 設好了、EC2 跑著、SSM 可以送指令進去、ffmpeg/python/git 都裝好、repo 也 clone 了。整個過程不需要 SSH、不需要 PEM、不需要打開 terminal。可以開始 M1 主流程 — 跟我說『啟動 M1』。」

## Why this exists as its own skill

- **Reusability:** M2 Stripe + M3 Domain + M4 Serverless all assume the same AWS account + the same SSM access pattern. M3 also reuses the IAM role for Route 53; M4 reuses the AWS account for Lambda + Fargate.
- **Recoverability:** if the EC2 gets terminated or the secrets rotate, the student re-runs this skill instead of re-reading all of M1.
- **Smaller M1 main skill:** the main walkthrough stays focused on schema + worker code + web upload + service management, not "how do I touch AWS at all."
