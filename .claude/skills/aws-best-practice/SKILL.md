---
name: aws-best-practice
description: Hard rules and operational SOP for working with AWS in this course. Use whenever a student is provisioning EC2 / IAM / Secrets Manager, sending commands to an instance, deploying worker code, or debugging an AWS-side failure in M1+. Sourced from real M1 try-outs — every rule has an incident behind it.
---

# AWS Best Practice

Battle-tested rules from running M1 (and later milestones) of Course 2 — the part where the SaaS first reaches outside Vercel + Supabase and onto an AWS EC2 worker. Each rule has a real incident behind it; skipping any one of them costs the student 10–60 minutes of "why is the worker silent?" debugging.

When you (Claude Code) are guiding a student through AWS operations, **apply these rules proactively**. Don't wait for the student to ask. If you see them about to break one, stop and explain why.

---

## Execution mode: Cowork vs CLI

Course 2 supports two execution environments. The hard rules below apply identically in both — only the **command surface** differs.

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Run any AWS API call | `aws <service> <verb> ...` | `call_aws <service> <verb> ...` via AWS API MCP |
| Suggest a command for a fuzzy goal | `aws <service> help` + man pages | `suggest_aws_commands "<natural language>"` via AWS API MCP |
| Run a shell command on an EC2 | `aws ssm send-command ...` | `call_aws ssm send-command ...` |
| Open an interactive EC2 shell | `aws ssm start-session --target <id>` | `call_aws ssm start-session --target <id>` (rarely needed — prefer `send-command`) |
| Auth into AWS | `aws configure` (Access Key ID + Secret + region) | Paste Access Key ID + Secret + region into the AWS API MCP connector settings |

**Cowork rule:** there is **no laptop filesystem** in Cowork. That means: no `.pem` file, no `~/.ssh/config`, no `scp`. SSH is therefore not a viable EC2 access path in this course — see Rule 1.

---

## Hard rules

### Rule 1 — Never use SSH to access EC2 — always use AWS Systems Manager (SSM)

> **The rule:** All shell commands on the EC2 go through `ssm send-command` (one-shot) or `ssm start-session` (interactive). Do not open port 22. Do not generate a key pair. Do not paste a `.pem` file anywhere.

**Why:** Three reasons compound:
1. **Cowork has no laptop filesystem** to hold a `.pem` file — so the SSH path isn't even available in Cowork mode. Half the course runs in Cowork, so any EC2 access pattern that requires a key file is a non-starter.
2. **PEM files leak.** They get copy-pasted into notes, screenshots, public repos. SSM uses the EC2's IAM instance profile to authenticate — no long-lived key to leak.
3. **No inbound port = smaller attack surface.** The EC2 in M1 has *no* inbound rules — it phones home to SSM over outbound 443. CloudTrail logs every command. Audit trail is built in.

**How to apply:**
- When provisioning an EC2 (M1 prereq §2.5), attach the IAM instance profile that includes `AmazonSSMManagedInstanceCore`. Do **not** pass `--key-name`.
- When the security group is created (M1 prereq §2.4), give it zero inbound rules. SSM is outbound-only.
- Wait for `ssm describe-instance-information` to show `PingStatus=Online` before trying to run commands on the EC2. If it never appears, the IAM instance profile is the most likely culprit — re-check via `ec2 describe-instances --query 'Reservations[0].Instances[0].IamInstanceProfile'`.
- For any command on the EC2, use `ssm send-command --instance-ids <id> --document-name AWS-RunShellScript --parameters '{"commands":[...]}'`. See Rule 2 for the JSON-form requirement.

If a student insists on SSH, the answer is: that's a different course path; this one is SSM-only by design. See [[feedback_ec2_commands]] for the durable user preference.

---

### Rule 2 — Always pass `--parameters` to `ssm send-command` as JSON, never as the shorthand `commands=[...]`

> **The rule:** `--parameters '{"commands":["..."]}'` is the only form that works reliably through the Cowork MCP shell layer. The CLI shorthand `--parameters 'commands=["..."]'` parses fine in a terminal but breaks through `call_aws`.

**Why:** The AWS CLI shorthand uses brackets, equals signs, and bare commas — all of which get re-interpreted or mangled when passed through Cowork's quoting layer to the AWS API MCP. The symptoms are inconsistent: sometimes you get `MalformedInput`, sometimes the command runs but with a truncated `commands` array, sometimes nothing visible happens at all. JSON form is the unambiguous wire format that survives every layer.

**How to apply:** Every `send-command` invocation uses this form:

```
call_aws ssm send-command \
  --instance-ids "$INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --parameters '{"commands":["echo hello","date"]}'
```

If you find yourself writing `'commands=[...]'` to either work in CLI or copy-paste from an old StackOverflow answer — convert it. The rule applies to CLI mode too (consistency makes copy-paste between modes safe).

---

### Rule 3 — SSM truncates command output — redirect to a file, then read it back

> **The rule:** Any `send-command` that produces more than ~3–4 lines of stdout must redirect to a file on the EC2 (`> /tmp/<name>.log 2>&1`) and the student must read it via a separate `send-command` that `cat`s the file.

**Why:** SSM caps the inline `Output` field returned by `list-command-invocations` at a fixed size and silently truncates with `---Output truncated---`. The truncation point lands in the middle of multi-line output, which means the *interesting* line (the actual error, the "Successfully installed ..." line, the systemd status) is exactly the part you don't get to see. Worse, when this happens during a `pip install` you can't distinguish "still running" from "finished but truncated" — there's no exit code visible inline.

**How to apply:** For any noisy command:

```
# Step 1 — run, redirect everything to a file
call_aws ssm send-command --instance-ids "$INSTANCE_ID" --document-name AWS-RunShellScript \
  --parameters '{"commands":["cd /home/ubuntu/app/worker && source venv/bin/activate && pip install -r requirements.txt > /tmp/pip.log 2>&1"]}'

# Step 2 — wait long enough (pip install on t3.small is 60–90 s, not 30)
# Step 3 — read the file in a separate send-command
call_aws ssm send-command --instance-ids "$INSTANCE_ID" --document-name AWS-RunShellScript \
  --parameters '{"commands":["tail -40 /tmp/pip.log"]}'
```

The same pattern applies to `systemctl status` output, application tracebacks, and anything from `journalctl`. Always redirect, always read back separately.

---

### Rule 4 — Always store credentials in AWS Secrets Manager — never in `.env`, `user-data`, or the EC2 filesystem

> **The rule:** `OPENAI_API_KEY`, `SUPABASE_SECRET_KEY`, and any other application credential lives in AWS Secrets Manager. The EC2's IAM instance profile grants `secretsmanager:GetSecretValue` on exactly those secret names. The worker pulls them at runtime via `boto3.client("secretsmanager").get_secret_value(SecretId=<name>)["SecretString"]`. No `.env` file. No `cloud-init` script with the key inlined. No git commit of any value.

**Why:** Every other home for the key has a leak story:
- **`.env` on the EC2:** survives reboots, gets included in `tar` backups, gets copied into images. There's no audit trail on read.
- **`user-data` / `cloud-init`:** the script is readable from the instance metadata service (IMDS) by anyone who can run code on the EC2 — and remains visible for the lifetime of the instance.
- **Committed in git:** the worst — public repos index immediately on GitHub's secret scanner, but private repos are still exposed to anyone with read access plus all forks.

Secrets Manager: encrypted at rest with KMS, IAM-scoped (the M1 role can only read 4 specific secret names), `GetSecretValue` is logged in CloudTrail with the calling principal and timestamp, and rotation is a built-in API.

**How to apply:**
- Create one secret per credential (`openai-api-key`, `supabase-url`, `supabase-secret-key`, `supabase-publishable-key`):
  ```
  call_aws secretsmanager create-secret --name openai-api-key --secret-string '<sk-...>'
  ```
- Scope the EC2's IAM role to **exactly** those secret names — use the `-*` ARN suffix to match Secrets Manager's auto-generated random tail:
  ```
  "Resource": [
    "arn:aws:secretsmanager:*:*:secret:openai-api-key-*",
    "arn:aws:secretsmanager:*:*:secret:supabase-url-*",
    "arn:aws:secretsmanager:*:*:secret:supabase-secret-key-*",
    "arn:aws:secretsmanager:*:*:secret:supabase-publishable-key-*"
  ]
  ```
  Never `"Resource": "*"` on `secretsmanager:GetSecretValue` — that's a backdoor to every other secret in the account.
- **Chat-retention caveat:** the raw value briefly appears in the Cowork chat transcript on its way to `create-secret`. Acceptable for course-grade keys (cap your OpenAI spend, scope your Supabase to a learning project). Rotate both keys at end of course. For real production, type values into the AWS console directly — not through a chat transcript.
- **C4 in the M1 checklist actively greps for `.env`** on the EC2. A `.env` present = the student deviated from this rule = ⚠️.

---

### Rule 5 — Bake `AWS_DEFAULT_REGION` into the systemd unit — and resolve it from the actual EC2, not from a hardcoded template

> **The rule:** `m1-distributor.service` (and any future systemd unit on an EC2) declares `Environment=AWS_DEFAULT_REGION=<region>`. Before pushing the unit file, resolve the EC2's actual region from `describe-instances` and patch the template if it doesn't match.

**Why:** The committed unit file is templated to one region (in M1: `us-west-2`). If the student launched their EC2 in `us-east-1` (the AWS API MCP default if region isn't set), boto3 inside the worker can't find a default region and crashes with `NoRegionError` on the first Secrets Manager call. The worker silently fails on every job, and the only place the traceback surfaces is `/var/log/m1-distributor.log` — which the student isn't yet looking at.

**How to apply:**
- Resolve the real region in one call before pushing:
  ```
  call_aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' --output text
  # → e.g. us-east-1a — strip the trailing letter to get the region (us-east-1)
  ```
- Patch `Environment=AWS_DEFAULT_REGION=<region>` in `worker/m1-distributor.service` if it differs, commit + push, then continue.
- **C7 in the M1 checklist tests this.** It greps the deployed unit file and cross-references against `describe-instances`. Mismatch = ❌.

The same rule applies to any AWS-aware service you deploy onto an EC2 — always state the region explicitly in the service's environment, and always resolve it from the actual instance, never from your memory of "I think I launched in us-west-2."

---

### Rule 6 — systemd's default PATH does NOT include the venv — always set `Environment=PATH=...` with the venv `bin/` first

> **The rule:** Any systemd unit that runs a Python process which `subprocess.run`s a binary installed into a venv (e.g. `yt-dlp`, `ffmpeg-python` wrappers, `playwright`) must declare:
> ```
> Environment=PATH=/home/ubuntu/app/worker/venv/bin:/usr/local/bin:/usr/bin:/bin
> ```

**Why:** systemd does *not* inherit the calling user's `PATH`. It defaults to `/usr/bin:/bin`, which does not include the venv's `bin/`. So `yt-dlp` (installed via `pip install yt-dlp` into the venv) is invisible to `subprocess.run(["yt-dlp", ...])` even though it's clearly installed — and the worker dies with `FileNotFoundError: 'yt-dlp'` on every single job. The killer detail: this error only appears in the application's log file (`/var/log/m1-distributor.log`), **not** in `journalctl`, so a student checking `journalctl -u m1-distributor.service` sees "Started", "Active", and no obvious failure. This was the #1 silent failure for first-time M1 students.

**How to apply:**
- The `Environment=PATH=...` line is mandatory in every systemd unit that spawns subprocesses from a venv.
- **C6 in the M1 checklist actively greps for this line.** Missing or wrong = ❌.
- When debugging "the service is running but no jobs progress" — `tail -100 /var/log/m1-distributor.log` is the right next step. `journalctl` will lie to you here.

---

### Rule 7 — Look up the EC2 by tag, not by instance ID — and resolve it once per session

> **The rule:** Every script, skill, and snippet that needs to act on the M1 worker EC2 resolves the instance ID from the `Name=m1-worker` tag, not from a hardcoded `i-...`. Resolve once, reuse via a shell variable.

**Why:** Instance IDs are per-account and change every time the student stops + terminates + relaunches (which they will, when something breaks badly enough). Tag-based lookup means the same skill works for the next student, on the next instance, in the next region, without edits. It also means the student doesn't have to copy/paste a 17-character ID twelve times across the checklist.

**How to apply:**
```
INSTANCE_ID=$(call_aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=m1-worker" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
```
Then reuse `"$INSTANCE_ID"` in every subsequent `send-command`. If `INSTANCE_ID` comes back empty, the EC2 is stopped or the tag is wrong — `call_aws ec2 start-instances` or fix the tag, don't paste an ID.

---

### Rule 8 — Use a dedicated IAM user for AWS API MCP — never root account keys

> **The rule:** Create a named IAM user (in M1: `cowork-m1`) with `AdministratorAccess`, generate one access key for it, paste that into Cowork's AWS API MCP. Never use AWS root account keys.

**Why:** Root keys have no scoping, no per-key audit trail, and no easy rotation. If a root key leaks, the only safe response is "rotate the root password, regenerate all keys, hope nothing was destroyed in between." A named IAM user is the AWS-recommended floor — auditable, deletable, and you can have multiple per-purpose users.

**How to apply:**
- IAM → Users → Create user → name `cowork-m1`, no console access.
- Attach `AdministratorAccess` directly (for a learning course this is the right trade — the alternative is enumerating ~15 IAM/EC2/SSM/KMS actions, which is a separate "production hardening" exercise).
- Security credentials → Create access key → use case "Application running outside AWS" → copy both halves once (AWS only shows the secret once).
- Paste into the AWS API MCP connector settings.
- **End-of-course:** delete the access key. The MCP stops working; that's the desired terminal state.
- **If you ever leak the key by pasting it into chat / a commit / a screenshot:** delete it immediately and create a new one. Anyone who sees the `AKIA...` + matching secret has full AWS access until you revoke.

---

### Rule 9 — Stop the EC2 when not actively running a lesson — and tell the student about it before they walk away

> **The rule:** At the end of every M1+ session, the worker EC2 gets stopped (not terminated) via `call_aws ec2 stop-instances --instance-ids "$INSTANCE_ID"`. The student should know this command on day one.

**Why:** A `t3.small` running 24/7 is ~$15/month — small in absolute terms, but it's *fully wasted* cost when no jobs are running, and it scales with every student multiplied by every milestone. Storage on the EBS volume persists across stop/start, so there's no setup cost to resuming the next day. Terminate is destructive (you'd have to re-run M1 prereq §2.5 onward); stop is reversible.

**How to apply:**
- **E1 in the M1 checklist** specifically tests: "How do you stop the EC2?" Acceptable answer: the `stop-instances` MCP call.
- **E2 (recommended, not required):** OpenAI billing limits should be set so a runaway worker can't drain the credit balance.
- When the student says "let me sign off for the day" → remind them: `call_aws ec2 stop-instances --instance-ids "$INSTANCE_ID"`, watch the state flip to `stopping → stopped`, done.
- **Stopped EC2 ≠ broken EC2.** When the student says "my jobs are stuck at pending," check whether the EC2 is running first — `call_aws ec2 describe-instances ... --query '...State.Name'`. If it's `stopped`, start it before debugging anything else.

---

## How to read EC2 logs (because journalctl won't tell you the truth)

Two log surfaces — they show **different** things:

| Surface | What it shows | What it MISSES |
|---|---|---|
| `journalctl -u m1-distributor.service` | systemd lifecycle: Started, Active, Deactivated, Stopped | Application stdout/stderr — because the unit redirects them to a file |
| `/var/log/m1-distributor.log` (or whatever the unit's `StandardOutput=` points at) | Real worker activity, tracebacks, per-job log lines | Nothing relevant — this is the source of truth for "what is the worker actually doing" |

**Rule of thumb when a job is silently stuck:** `tail -100 /var/log/m1-distributor.log` *first*, then `journalctl` only if you need to confirm the service crashed vs was killed. Most first-time M1 debugging goes the wrong way around — students check `journalctl`, see "Active (running)", and assume everything's fine while every job dies with `FileNotFoundError: 'yt-dlp'` in the log file.

Read the log via SSM:
```
call_aws ssm send-command --instance-ids "$INSTANCE_ID" --document-name AWS-RunShellScript \
  --parameters '{"commands":["tail -100 /var/log/m1-distributor.log"]}'
```

(Use Rule 3's file-redirect pattern if the tail is bigger than ~30 lines.)

---

## Things to actively watch out for

These are not hard rules — they're failure modes that don't have a single "always do this" answer but show up often enough that you should recognize them.

1. **Region drift between the MCP and the EC2.** AWS API MCP defaults to `us-east-1` if region isn't set; if you launched the EC2 in `us-west-2` and didn't set region in MCP, half your `call_aws` calls will hit the empty `us-east-1` and return "instance not found" — when it does exist, just in another region. Fix: set the MCP's region to match the EC2's, OR pass `--region <region>` on every call.

2. **`InvalidInstanceId` when `send-command` runs but the instance "exists."** Usually means the SSM agent on the instance can't reach the SSM endpoint (IAM role missing `AmazonSSMManagedInstanceCore`, or the security group is somehow blocking outbound 443). Verify with `ssm describe-instance-information` — if `PingStatus` is not `Online`, the agent isn't reachable.

3. **`AccessDeniedException` on `secretsmanager:GetSecretValue`.** The EC2's IAM role doesn't have the inline policy granting read on the four secret names. Re-run the `put-role-policy` from M1 prereq §2.3.

4. **`ResourceNotFoundException: Secrets Manager can't find the specified secret`.** One of the secret names wasn't created. Don't assume which one — list them: `call_aws secretsmanager list-secrets --query 'SecretList[].Name'`. Re-create the missing one.

5. **AMI ID is region-specific.** Don't hardcode `ami-xxxxxxx`. Resolve the latest Ubuntu 22.04 AMI per region from SSM Parameter Store:
   ```
   call_aws ssm get-parameters \
     --names /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
     --query 'Parameters[0].Value' --output text
   ```

6. **`run-instances` succeeds before SSM is ready.** Just because the instance is `running` doesn't mean the SSM agent has phoned home yet. Wait for `ssm describe-instance-information --filters "Key=tag:Name,Values=m1-worker"` to return a row with `PingStatus=Online` — usually 60–120 s after launch — before trying any `send-command`.

7. **Default systemd PATH (see Rule 6) is the single most common silent failure in M1.** If a worker says "running" but no jobs progress, your first suspect is missing `Environment=PATH=...`, not a logic bug.

---

## Out of scope for this course

Things that are real AWS best practice but *deliberately not enforced* here because they distract from the learning goal:

- **Least-privilege IAM beyond `AdministratorAccess`.** Course IAM user is admin (see Rule 8). Real-prod hardening = enumerate the ~15 minimum actions across IAM/EC2/SSM/KMS/Secrets Manager. Out of scope.
- **VPC + subnets + private endpoints for SSM.** Course uses the default VPC + public subnet + outbound to SSM public endpoints. Real prod uses a private subnet + VPC endpoints. Out of scope.
- **EBS encryption keys + key rotation.** Course relies on AWS-managed KMS keys. Real prod uses customer-managed KMS keys with explicit rotation policies. Out of scope.
- **Multi-AZ / autoscaling / blue-green deploys.** Course is one EC2, one region, one student. Out of scope until M4 (Lambda + Fargate distributor).

When a student asks "shouldn't we do <thing from this list>?" the right answer is: "Yes — for production. Not for the course. The course is optimizing for 'minimum AWS surface area to get Whisper running'; production hardening is a separate exercise we'd do once the rest of the milestones are stable."

---

## Cross-references

- [[m1-ai-video-transcript-prerequisites]] — where the EC2 + IAM + Secrets Manager are first provisioned
- [[m1-ai-video-transcript]] — where the worker code is deployed via SSM (Step 6)
- [[m1-ai-video-transcript-checklist]] — Section C tests every rule in this skill against the live EC2
- [[whisper-best-practice]] — the *application* side of the same worker (yt-dlp, ffmpeg, Whisper limits, language hints)
- [[feedback_ec2_commands]] — durable user preference: SSM, not SSH; no per-student instructions to run commands manually
