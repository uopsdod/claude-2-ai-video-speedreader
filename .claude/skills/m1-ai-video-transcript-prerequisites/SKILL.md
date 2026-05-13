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

## Required MCPs (read this first)

This skill assumes you have:

- ✅ **Supabase MCP** (`mcp__supabase_remote__*`) — already installed for M0.
- ✅ **AWS API MCP** (`call_aws`, `suggest_aws_commands`) — published by Amazon Web Services, available in the Cowork plugin/connector directory under "AWS API MCP Server". Install + authenticate it via your Cowork settings before running this skill.

If AWS API MCP isn't installed: pause this skill, install it in Cowork, then resume. The whole point of this path is to keep the student out of the AWS console and the terminal — without AWS API MCP, you're stuck doing AWS-console clicks manually.

> **Why no SSH?** SSH on EC2 is fine when you have a laptop + a `.pem` file + a stable IP. In Cowork there's no laptop filesystem to hold the PEM, and the network egress story is fiddly. AWS Systems Manager (SSM) gives us a better answer: the EC2 attaches to SSM via an IAM role; we send commands from Cowork via `call_aws ssm send-command` and open a shell (if needed) via `call_aws ssm start-session`. No inbound port. No key. Everything audited in CloudTrail. This is the same pattern the production worker in this repo uses (see `feedback_ec2_commands.md`).

## Section 1 — OpenAI API key

1. Go to https://platform.openai.com/, sign up (or log in).
2. **Settings → Billing → Add payment method**, add a card.
3. **Add to credit balance** → top up at least **\$5**.
4. **Settings → Limits** → set a **soft monthly cap** (e.g. \$10). This is your runaway-worker insurance.
5. **API keys → Create new secret key**. Copy the `sk-...` value — OpenAI only shows it once.

Hold onto this `sk-...`. We'll inject it into the EC2 via SSM Parameter Store in Section 2.

## Section 2 — AWS account + EC2 (all via MCP)

### 2.1 — AWS account + region

Sign up at https://aws.amazon.com/ if needed. Pick **one region** and stick with it for the whole course (we recommend `us-west-2`, `ap-northeast-1`, or `ap-southeast-1` for latency). Make sure AWS API MCP is configured against that region — it usually defaults to whatever's in the student's AWS profile.

### 2.2 — Create the IAM instance profile (one MCP call)

The EC2 needs an IAM role that grants:
- `AmazonSSMManagedInstanceCore` (the managed policy that gives SSM agent permission to phone home)
- Read on **one** SSM Parameter Store path (where we'll put OpenAI + Supabase keys)

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

### 2.3 — Store OpenAI + Supabase secrets in SSM Parameter Store

We **never** type these onto the EC2 directly. Instead we put them in SecureString parameters and let the worker fetch them at boot.

```
call_aws ssm put-parameter --name /m1/OPENAI_API_KEY --type SecureString --value '<the sk-... from Section 1>'
call_aws ssm put-parameter --name /m1/SUPABASE_URL --type String --value 'https://<ref>.supabase.co'
call_aws ssm put-parameter --name /m1/SUPABASE_SERVICE_KEY --type SecureString --value '<the service_role key from Supabase dashboard>'
```

Add the IAM permission to read these to the worker role:

```
call_aws iam put-role-policy --role-name m1-worker-role --policy-name m1-ssm-read --policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["ssm:GetParameter", "ssm:GetParameters"],
    "Resource": "arn:aws:ssm:*:*:parameter/m1/*"
  }]
}'
```

> **Why SecureString?** Parameter Store SecureString encrypts at rest with KMS; only roles with `ssm:GetParameter` *and* `kms:Decrypt` on the default `alias/aws/ssm` key can read it. Plain env-var injection into `user-data` would leak the keys into the EC2's metadata service.

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

# 3. Secrets in Parameter Store
call_aws ssm get-parameters --names /m1/OPENAI_API_KEY /m1/SUPABASE_URL /m1/SUPABASE_SERVICE_KEY --with-decryption --query 'Parameters[].Name'
# Expect: all three names present
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
