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

In Cowork, AWS is reached through the official **AWS MCP** connector (set up in [[m1-ai-video-transcript-prerequisites]] §2.0, OAuth sign-in). Its `aws___run_script` tool runs a short Python script in a sandbox; inside it you call any AWS API with `await call_boto3(service_name=..., operation_name=..., region_name=..., params={...})`.

| Operation | CLI mode | Cowork mode (AWS MCP) |
|---|---|---|
| Run any AWS API call | `aws <service> <verb> ...` | `aws___run_script` → `await call_boto3(service_name="<service>", operation_name="<PascalCaseOperation>", region_name=REGION, params={...})` |
| Look up how to do something | `aws <service> help` + man pages | `aws___search_documentation` / `aws___read_documentation` / `aws___retrieve_skill` |
| Run a shell command on an EC2 | `aws ssm send-command ...` | `SendCommand` + poll `GetCommandInvocation` — use the `run_on_ec2` helper (Rule 2) |
| Open an interactive EC2 shell | `aws ssm start-session --target <id>` | Not available — Session Manager needs a local plugin. Use `SendCommand` (Rule 2). |
| Auth into AWS | `aws sso login` or `aws configure` | Sign in to the AWS MCP connector in the browser (OAuth). **No access key.** See Rule 8. |

**`call_boto3` conventions** (every snippet in the course follows them):
- `operation_name` is the **PascalCase API operation** (`SendCommand`, `DescribeInstances`) — not the CLI spelling (`send-command`) and not the snake_case boto3 method.
- Pass `region_name` explicitly on every regional call (EC2, SSM, Secrets Manager). IAM is global and takes none.
- JSON policy documents go in as strings (`json.dumps(...)`).
- End the script with `result = {...}` then `result` on its own line.
- Each `aws___run_script` call is **self-contained** — no variables survive between calls, so every script re-discovers what it needs (instance ID, region). See Rule 7.

**Cowork rule:** there is **no laptop filesystem** in Cowork. That means: no `.pem` file, no `~/.ssh/config`, no `scp`. SSH is therefore not a viable EC2 access path in this course — see Rule 1.

> **Old connector:** if the student still has the **AWS API MCP** connector (tools `call_aws` / `suggest_aws_commands`, authenticated with a pasted access key) installed, have them disable it in Customize → Connectors. The course no longer uses it, and keeping both makes Claude mix the two.

---

## Hard rules

### Rule 1 — Never use SSH to access EC2 — always use AWS Systems Manager (SSM)

> **The rule:** All shell commands on the EC2 go through SSM `SendCommand`. Do not open port 22. Do not generate a key pair. Do not paste a `.pem` file anywhere.

**Why:** Three reasons compound:
1. **Cowork has no laptop filesystem** to hold a `.pem` file — so the SSH path isn't even available in Cowork mode. Half the course runs in Cowork, so any EC2 access pattern that requires a key file is a non-starter.
2. **PEM files leak.** They get copy-pasted into notes, screenshots, public repos. SSM uses the EC2's IAM instance profile to authenticate — no long-lived key to leak.
3. **No inbound port = smaller attack surface.** The EC2 in M1 has *no* inbound rules — it phones home to SSM over outbound 443. CloudTrail logs every command. Audit trail is built in.

**How to apply:**
- When provisioning an EC2 (M1 prereq §2.5), attach the IAM instance profile that includes `AmazonSSMManagedInstanceCore`. Do **not** pass `KeyName`.
- When the security group is created (M1 prereq §2.4), give it zero inbound rules. SSM is outbound-only.
- Wait for SSM `DescribeInstanceInformation` to show `PingStatus=Online` before trying to run commands on the EC2. If it never appears, the IAM instance profile is the most likely culprit — re-check `IamInstanceProfile` on the instance from `ec2 DescribeInstances`.
- For any command on the EC2, use SSM `SendCommand` with `DocumentName="AWS-RunShellScript"` — see Rule 2 for the exact pattern.

If a student insists on SSH, the answer is: that's a different course path; this one is SSM-only by design. See [[feedback_ec2_commands]] for the durable user preference.

---

### Rule 2 — Send EC2 commands as a structured list, then poll for the result

> **The rule:** Pass shell commands to `SendCommand` as a real list — `"Parameters": {"commands": ["...", "..."]}` — never as a single quoted string or the CLI shorthand `'commands=[...]'`. `SendCommand` only *queues* the command; always poll `GetCommandInvocation` until the status is final before reading output.

**Why:** Two separate traps:
1. **Quoting.** The CLI shorthand (`--parameters 'commands=["..."]'`) uses brackets, equals signs, and bare commas that get re-interpreted or mangled whenever a quoting layer sits in between — symptoms range from `MalformedInput` to a silently truncated `commands` array. With `call_boto3` the commands are a Python list, so there is no shell parsing at all. (In CLI mode, use the JSON form `--parameters '{"commands":["..."]}'` for the same reason.)
2. **Async.** `SendCommand` returns immediately with a `CommandId` while the command is still `Pending` / `InProgress`. Reading output right away returns nothing, which looks like "the command did nothing."

**How to apply:** Every script that touches the EC2 includes this helper (copy it in — scripts don't share state):

```python
REGION = "us-west-2"   # the region the EC2 lives in

async def run_on_ec2(instance_id, commands, comment):
    sent = await call_boto3(
        service_name="ssm", operation_name="SendCommand", region_name=REGION,
        params={"InstanceIds": [instance_id],
                "DocumentName": "AWS-RunShellScript",
                "Comment": comment,
                "Parameters": {"commands": commands}})
    command_id = sent["Command"]["CommandId"]
    for _ in range(60):                       # up to ~5 min
        await asyncio.sleep(5)
        inv = await call_boto3(
            service_name="ssm", operation_name="GetCommandInvocation", region_name=REGION,
            params={"CommandId": command_id, "InstanceId": instance_id})
        if inv["Status"] not in ("Pending", "InProgress", "Delayed"):
            return {"status": inv["Status"],
                    "stdout": inv["StandardOutputContent"],
                    "stderr": inv["StandardErrorContent"]}
    return {"status": "StillRunning", "command_id": command_id}
```

Then: `result = await run_on_ec2(instance_id, ["echo hello", "date"], "M1: smoke test")`. If it returns `StillRunning`, the command is fine — it outlasted the wait; poll `GetCommandInvocation` again with the returned `command_id`.

---

### Rule 3 — SSM truncates command output — redirect to a file, then read it back

> **The rule:** Any command that produces a lot of output must redirect to a file on the EC2 (`> /tmp/<name>.log 2>&1`), and you read it back with a second, separate command that `tail`s the file.

**Why:** SSM caps the output it returns inline (`StandardOutputContent`) and silently truncates the rest. The cut lands in the middle of long output, which means the *interesting* line (the actual error, the "Successfully installed ..." line, the systemd status) is exactly the part you don't get to see. Worse, when this happens during a `pip install` you can't tell "failed" from "fine but truncated".

**How to apply:** For any noisy command (include `run_on_ec2` from Rule 2 and the instance lookup from Rule 7):

```python
# Step 1 — run, redirect everything to a file
await run_on_ec2(instance_id,
    ["sudo -u ubuntu bash -c \"cd /home/ubuntu/app/worker && ./venv/bin/pip install -r requirements.txt > /tmp/pip.log 2>&1\""],
    "M1: pip install")

# Step 2 — read the tail of the file in a separate command
result = await run_on_ec2(instance_id, ["tail -40 /tmp/pip.log"], "M1: read pip log")
result
```

`pip install` on a `t3.small` takes 60–90 s; `run_on_ec2` waits for it. The same pattern applies to `systemctl status` output, application tracebacks, and anything from `journalctl`. Always redirect, always read back separately.

---

### Rule 4 — Always store credentials in AWS Secrets Manager — never in `.env`, `user-data`, or the EC2 filesystem

> **The rule:** `OPENAI_API_KEY`, `SUPABASE_SECRET_KEY`, and any other application credential lives in AWS Secrets Manager. The EC2's IAM instance profile grants `secretsmanager:GetSecretValue` on exactly those secret names. The worker pulls them at runtime via `boto3.client("secretsmanager").get_secret_value(SecretId=<name>)["SecretString"]`. No `.env` file. No `cloud-init` script with the key inlined. No git commit of any value.

**Why:** Every other home for the key has a leak story:
- **`.env` on the EC2:** survives reboots, gets included in `tar` backups, gets copied into images. There's no audit trail on read.
- **`user-data` / `cloud-init`:** the script is readable from the instance metadata service (IMDS) by anyone who can run code on the EC2 — and remains visible for the lifetime of the instance.
- **Committed in git:** the worst — public repos index immediately on GitHub's secret scanner, but private repos are still exposed to anyone with read access plus all forks.

Secrets Manager: encrypted at rest with KMS, IAM-scoped (the M1 role can only read 4 specific secret names), `GetSecretValue` is logged in CloudTrail with the calling principal and timestamp, and rotation is a built-in API.

**How to apply:**
- Create one secret per credential (`openai-api-key`, `supabase-url`, `supabase-secret-key`, `supabase-publishable-key`) — the full script is in M1 prereq §2.3:
  ```python
  await call_boto3(service_name="secretsmanager", operation_name="CreateSecret",
                   region_name=REGION,
                   params={"Name": "openai-api-key", "SecretString": "<sk-...>"})
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
- **Chat-retention caveat:** the raw value briefly appears in the Cowork chat transcript on its way to `CreateSecret`. Acceptable for course-grade keys (cap your OpenAI spend, scope your Supabase to a learning project). Rotate both keys at end of course. For real production, type values into the AWS console directly — not through a chat transcript.
- **C4 in the M1 checklist actively greps for `.env`** on the EC2. A `.env` present = the student deviated from this rule = ⚠️.

---

### Rule 5 — Bake `AWS_DEFAULT_REGION` into the systemd unit — and resolve it from the actual EC2, not from a hardcoded template

> **The rule:** `m1-distributor.service` (and any future systemd unit on an EC2) declares `Environment=AWS_DEFAULT_REGION=<region>`. Before pushing the unit file, resolve the EC2's actual region and patch the template if it doesn't match.

**Why:** The committed unit file is templated to one region (in M1: `us-west-2`). If the student launched their EC2 somewhere else (for example `ap-northeast-1`, or `us-east-1` because a script left out `region_name` — the AWS MCP endpoint itself lives in `us-east-1`), boto3 inside the worker either can't find a default region and crashes with `NoRegionError`, or looks for the secrets in the wrong region. The worker silently fails on every job, and the only place the traceback surfaces is `/var/log/m1-distributor.log` — which the student isn't yet looking at.

**How to apply:**
- Resolve the real region in one script — search the candidate regions for the tagged instance:
  ```python
  found = {}
  for region in ["us-west-2", "us-east-1", "ap-northeast-1", "ap-southeast-1"]:
      desc = await call_boto3(
          service_name="ec2", operation_name="DescribeInstances", region_name=region,
          params={"Filters": [{"Name": "tag:Name", "Values": ["m1-worker"]}]})
      for r in desc["Reservations"]:
          for i in r["Instances"]:
              found[i["InstanceId"]] = i["Placement"]["AvailabilityZone"][:-1]  # us-west-2a → us-west-2
  result = found
  result
  ```
- Patch `Environment=AWS_DEFAULT_REGION=<region>` in `worker/m1-distributor.service` if it differs, commit + push, then continue.
- **C7 in the M1 checklist tests this.** It greps the deployed unit file and cross-references against the instance's actual region. Mismatch = ❌.

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

### Rule 7 — Look up the EC2 by tag, not by instance ID — in every script

> **The rule:** Every script, skill, and snippet that needs to act on the M1 worker EC2 resolves the instance ID from the `Name=m1-worker` tag, not from a hardcoded `i-...`. Because each `aws___run_script` call is self-contained, the lookup goes at the top of **every** script that needs it.

**Why:** Instance IDs are per-account and change every time the student terminates + relaunches (which they will, when something breaks badly enough). Tag-based lookup means the same skill works for the next student, on the next instance, in the next region, without edits. It also means the student never has to copy/paste a 17-character ID across the checklist.

**How to apply:**
```python
REGION = "us-west-2"
desc = await call_boto3(
    service_name="ec2", operation_name="DescribeInstances", region_name=REGION,
    params={"Filters": [{"Name": "tag:Name", "Values": ["m1-worker"]},
                        {"Name": "instance-state-name", "Values": ["running"]}]})
instances = [i for r in desc["Reservations"] for i in r["Instances"]]
instance_id = instances[0]["InstanceId"] if instances else None
```
If `instances` comes back empty, the EC2 is stopped, the tag is wrong, or `REGION` is wrong — start it (`StartInstances`, see Rule 9), fix the tag, or fix the region. Don't paste an ID.

---

### Rule 8 — Connect AWS through the AWS MCP OAuth sign-in — never paste access keys into Cowork

> **The rule:** Cowork talks to AWS only through the official **AWS MCP** connector (`https://aws-mcp.us-east-1.api.aws/mcp?oauth=initialize`), signed in through the browser (M1 prereq §2.0). Do not create an IAM user + access key for Cowork, and do not paste any `AKIA...` key into a connector, a chat, or a file.

**Why:** An access key is a long-lived credential: once it's pasted into a connector setting, a chat, or a screenshot, anyone who sees it has that account's access until someone remembers to revoke it. The OAuth sign-in leaves no key to leak — the connector holds a session, which you cut by removing the connector. It's also one less setup step for students (no IAM console detour).

**How to apply:**
- Set up per M1 prereq §2.0: Customize → Connectors → **disable AWS API MCP** if present → Add **AWS MCP** with the URL above → sign in → **Always allow**.
- Sanity check: ask Claude "How many IAM users do I have on my AWS account?" — any number (0 is normal) means the connector works.
- If AWS tools disappear or calls fail with a sign-in / auth error mid-course, the session expired: reconnect the connector and sign in again. Don't fall back to an access key.
- **End-of-course:** there is no key to delete. Remove the AWS MCP connector in Customize → Connectors to cut Cowork's access.
- **If a student already created an access key** for the old AWS API MCP: deactivate and delete it in IAM → Users → Security credentials once AWS MCP works.

---

### Rule 9 — Stop the EC2 when not actively running a lesson — and tell the student about it before they walk away

> **The rule:** At the end of every M1+ session, the worker EC2 gets stopped (not terminated) with `ec2 StopInstances`. The student should know how to do this on day one — the one-line ask is 「幫我把 m1-worker EC2 停掉」.

**Why:** A `t3.small` running 24/7 is ~$15/month — small in absolute terms, but it's *fully wasted* cost when no jobs are running, and it scales with every student multiplied by every milestone. Storage on the EBS volume persists across stop/start, so there's no setup cost to resuming the next day. Terminate is destructive (you'd have to re-run M1 prereq §2.5 onward); stop is reversible.

**How to apply:**
```python
REGION = "us-west-2"
desc = await call_boto3(
    service_name="ec2", operation_name="DescribeInstances", region_name=REGION,
    params={"Filters": [{"Name": "tag:Name", "Values": ["m1-worker"]},
                        {"Name": "instance-state-name", "Values": ["running"]}]})
ids = [i["InstanceId"] for r in desc["Reservations"] for i in r["Instances"]]
if ids:
    await call_boto3(service_name="ec2", operation_name="StopInstances",
                     region_name=REGION, params={"InstanceIds": ids})
result = {"stopping": ids}
result
```
To resume: the same script with `instance-state-name = stopped` and `StartInstances`.

- **E1 in the M1 checklist** specifically tests: "How do you stop the EC2?" Acceptable answer: ask Claude to stop the `m1-worker` EC2 (which runs `StopInstances` via AWS MCP).
- **E2 (recommended, not required):** OpenAI billing limits should be set so a runaway worker can't drain the credit balance.
- When the student says "let me sign off for the day" → remind them to stop the EC2, then run `DescribeInstances` and watch `State.Name` go `stopping → stopped`.
- **Stopped EC2 ≠ broken EC2.** When the student says "my jobs are stuck at pending," check `State.Name` first. If it's `stopped`, start it before debugging anything else.

---

## How to read EC2 logs (because journalctl won't tell you the truth)

Two log surfaces — they show **different** things:

| Surface | What it shows | What it MISSES |
|---|---|---|
| `journalctl -u m1-distributor.service` | systemd lifecycle: Started, Active, Deactivated, Stopped | Application stdout/stderr — because the unit redirects them to a file |
| `/var/log/m1-distributor.log` (or whatever the unit's `StandardOutput=` points at) | Real worker activity, tracebacks, per-job log lines | Nothing relevant — this is the source of truth for "what is the worker actually doing" |

**Rule of thumb when a job is silently stuck:** `tail -100 /var/log/m1-distributor.log` *first*, then `journalctl` only if you need to confirm the service crashed vs was killed. Most first-time M1 debugging goes the wrong way around — students check `journalctl`, see "Active (running)", and assume everything's fine while every job dies with `FileNotFoundError: 'yt-dlp'` in the log file.

Read the log via SSM (with `run_on_ec2` from Rule 2 and the lookup from Rule 7):
```python
result = await run_on_ec2(instance_id, ["tail -100 /var/log/m1-distributor.log"], "M1: read worker log")
result
```

(Use Rule 3's file-redirect pattern if the output is long enough to get truncated.)

---

## Things to actively watch out for

These are not hard rules — they're failure modes that don't have a single "always do this" answer but show up often enough that you should recognize them.

1. **Region drift between the script and the EC2.** The AWS MCP endpoint lives in `us-east-1`, but that says nothing about where your resources are. A `call_boto3` without `region_name` (or with the wrong one) hits an empty region and returns "no instances" / "secret not found" — when the resource exists, just elsewhere. Fix: set `REGION` to the student's region (M1 prereq §2.1) and pass `region_name=REGION` on every regional call. If unsure, run Rule 5's multi-region lookup.

2. **`InvalidInstanceId` when `SendCommand` runs but the instance "exists."** Usually means the SSM agent on the instance can't reach the SSM endpoint (IAM role missing `AmazonSSMManagedInstanceCore`, or the security group is somehow blocking outbound 443). Verify with SSM `DescribeInstanceInformation` — if `PingStatus` is not `Online`, the agent isn't reachable.

3. **`AccessDeniedException` on `secretsmanager:GetSecretValue`.** The EC2's IAM role doesn't have the inline policy granting read on the four secret names. Re-run the `PutRolePolicy` script from M1 prereq §2.3.

4. **`ResourceNotFoundException: Secrets Manager can't find the specified secret`.** One of the secret names wasn't created (or was created in another region). Don't assume which one — list them with `secretsmanager ListSecrets` in `REGION` (M1 prereq §2.3 **Verify** script). Re-create the missing one.

5. **AMI ID is region-specific.** Don't hardcode `ami-xxxxxxx`. Resolve the latest Ubuntu 22.04 AMI per region from SSM Parameter Store:
   ```python
   ami = await call_boto3(
       service_name="ssm", operation_name="GetParameter", region_name=REGION,
       params={"Name": "/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id"})
   ami_id = ami["Parameter"]["Value"]
   ```

6. **`RunInstances` succeeds before SSM is ready.** Just because the instance is `running` doesn't mean the SSM agent has phoned home yet. Wait for SSM `DescribeInstanceInformation` (filter `tag:Name = m1-worker`) to return a row with `PingStatus=Online` — usually 60–120 s after launch — before any `SendCommand`.

7. **Default systemd PATH (see Rule 6) is the single most common silent failure in M1.** If a worker says "running" but no jobs progress, your first suspect is missing `Environment=PATH=...`, not a logic bug.

8. **AWS MCP session expired.** AWS tools vanish or every call fails with an auth / sign-in error. Reconnect the AWS MCP connector and sign in again (Rule 8) — the EC2 and secrets are untouched.

---

## Out of scope for this course

Things that are real AWS best practice but *deliberately not enforced* here because they distract from the learning goal:

- **Least-privilege access for Cowork.** The AWS MCP sign-in carries the signed-in identity's full permissions (the course signs in with the account owner — see M1 prereq §2.0). Real-prod hardening = a dedicated identity limited to the ~15 actions across IAM/EC2/SSM/KMS/Secrets Manager the course uses. Out of scope.
- **VPC + subnets + private endpoints for SSM.** Course uses the default VPC + public subnet + outbound to SSM public endpoints. Real prod uses a private subnet + VPC endpoints. Out of scope.
- **EBS encryption keys + key rotation.** Course relies on AWS-managed KMS keys. Real prod uses customer-managed KMS keys with explicit rotation policies. Out of scope.
- **Multi-AZ / autoscaling / blue-green deploys.** Course is one EC2, one region, one student. Out of scope until M4 (Lambda + Fargate distributor).

When a student asks "shouldn't we do <thing from this list>?" the right answer is: "Yes — for production. Not for the course. The course is optimizing for 'minimum AWS surface area to get Whisper running'; production hardening is a separate exercise we'd do once the rest of the milestones are stable."

---

## Cross-references

- [[m1-ai-video-transcript-prerequisites]] — where AWS MCP is connected and the EC2 + IAM + Secrets Manager are first provisioned
- [[m1-ai-video-transcript]] — where the worker code is deployed via SSM (Step 6)
- [[m1-ai-video-transcript-checklist]] — Section C tests every rule in this skill against the live EC2
- [[whisper-best-practice]] — the *application* side of the same worker (yt-dlp, ffmpeg, Whisper limits, language hints)
- [[feedback_ec2_commands]] — durable user preference: SSM, not SSH; no per-student instructions to run commands manually
