---
name: whisper-best-practice
description: Hard rules and operational SOP for using OpenAI Whisper (`whisper-1`) in this course. Use whenever a student is writing or debugging the M1 worker, picking a test URL for transcription, hitting the 25 MB limit, getting gibberish output, or wondering why YouTube fails from an EC2 cloud IP. Sourced from real M1 try-outs.
---

# Whisper Best Practice

Battle-tested rules for `whisper-1` — the OpenAI Speech-to-Text endpoint used in M1's worker (`/v1/audio/transcriptions`). Each rule maps to an incident from a real try-out session — students hit these in the *first hour* of M1, before they've internalized that "Whisper API" is a thin wrapper with very specific input requirements.

When you (Claude Code) are guiding a student through any Whisper-touching code, **apply these rules proactively**. Don't wait for the student to ask. If you see them about to break one, stop and explain why.

This skill is the *application-layer* sibling of [[aws-best-practice]] — that one covers EC2 / IAM / Secrets Manager / SSM; this one covers what happens inside `worker.py` once the EC2 is running.

---

## Execution mode: Cowork vs CLI

Whisper is a stateless API; the calling code runs identically in both modes. The two relevant deltas:

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Set / rotate `OPENAI_API_KEY` | `aws secretsmanager put-secret-value --secret-id openai-api-key --secret-string '<sk-...>'` | `call_aws secretsmanager put-secret-value --secret-id openai-api-key --secret-string '<sk-...>'` |
| Read recent Whisper errors from the worker | `aws ssm send-command ... 'grep -i "whisper\|openai" /var/log/m1-distributor.log'` | Same, via `call_aws` |
| Check OpenAI usage / quotas | Browser → https://platform.openai.com/usage | Same browser |

**The API key never sits in `.env`.** It lives in AWS Secrets Manager (see [[aws-best-practice]] Rule 4). The worker pulls it at startup via `boto3` and instantiates one `OpenAI(api_key=...)` client.

---

## Hard rules

### Rule 1 — Always pre-process audio with ffmpeg before sending to Whisper — never upload the raw video

> **The rule:** Whisper accepts video containers, but you should not feed them in directly. Convert to **64 kbps mono 16 kHz mp3** with ffmpeg first, then upload that. The M1 worker does this in one ffmpeg pass after `yt-dlp` finishes.

**Why:** Three compounding reasons:
1. **The 25 MB limit is a hard wall (see Rule 2).** A 10-minute 1080p mp4 is easily 100+ MB. The same audio re-encoded to 64 kbps mono mp3 is ~5 MB — a 20× reduction with no measurable accuracy loss for speech.
2. **Whisper is mono-only internally.** Sending a stereo file means OpenAI's pipeline downmixes server-side anyway. You're paying upload bandwidth and storage for data that gets discarded.
3. **16 kHz is the model's native rate.** Speech information lives below 8 kHz; the 16 kHz sample rate satisfies Nyquist exactly. 44.1 / 48 kHz studio rates add nothing the model can use.

**How to apply:**
```python
subprocess.run([
    "ffmpeg", "-y", "-i", src_path,
    "-ac", "1",          # mono
    "-ar", "16000",      # 16 kHz
    "-b:a", "64k",       # 64 kbps
    "-f", "mp3",
    out_path
], check=True)
```
Confirm `ffmpeg` is installed on the EC2 (M1 prereq §2.6). The C4 checklist check looks for the binary; if it's missing, every job dies at the conversion step.

---

### Rule 2 — Whisper's per-request file-size limit is 25 MB — chunk audio that exceeds it, never just retry

> **The rule:** `whisper-1` returns `413 Request Entity Too Large` for any file over 25 MB. The worker splits the audio into ≤25 MB chunks at silence boundaries (or fixed time boundaries as a fallback) and transcribes each chunk separately, then concatenates.

**Why:** At the Rule-1 codec (64 kbps mono mp3), 25 MB is roughly **52 minutes of speech**. Most short lectures fit in one request. But a 90-minute keynote will hit the wall — and retrying won't help; the file is still 25+ MB. Chunk-and-concat is the only correct response.

**How to apply:**
- Split with ffmpeg's segment muxer:
  ```python
  subprocess.run([
      "ffmpeg", "-y", "-i", mp3_path,
      "-f", "segment", "-segment_time", "1800",  # 30 min chunks at 64kbps ≈ 14 MB
      "-c", "copy",
      "/tmp/chunk_%03d.mp3",
  ], check=True)
  ```
  30-minute chunks at 64 kbps give a comfortable safety margin under 25 MB even for content with a higher average bitrate.
- Transcribe each chunk in order, accumulate the `text` field:
  ```python
  full_text = ""
  for chunk_path in sorted(glob.glob("/tmp/chunk_*.mp3")):
      with open(chunk_path, "rb") as f:
          r = client.audio.transcriptions.create(model="whisper-1", file=f, language=language)
      full_text += r.text + "\n"
  ```
- **Don't try to split mid-call by streaming** — Whisper has no streaming endpoint for transcription. The split happens before the API call.

---

### Rule 3 — Never use YouTube URLs for cloud-IP smoke tests — YouTube rate-limits / bot-checks AWS / GCP / Azure IP ranges

> **The rule:** Smoke-test URLs and student-facing examples in M1 must point at hosts that allow anonymous downloads from cloud IPs. Internet Archive, Wikimedia Commons direct media, and direct `.mp4` / `.mp3` URLs work. YouTube does not.

**Why:** YouTube's anti-bot defense actively rejects requests originating from known cloud-provider IP ranges. `yt-dlp` from an EC2 fails with `Sign in to confirm you're not a bot` — even though the same `yt-dlp` command works from a residential connection. This is by design on YouTube's side; it's not a `yt-dlp` bug, not a fixable-by-cookies problem in the M1 context (cookies expire, students would have to refresh them, the whole experience falls apart). Picking YouTube as the example URL turned the first M1 try-out into 20 minutes of "why is yt-dlp broken?" when nothing was broken — just the URL choice.

**How to apply:**
- **Default smoke-test URL** (works from any EC2 IP): the Internet Archive MLK "I Have a Dream" mp3 — short, public domain, English, clear:
  ```
  https://ia801501.us.archive.org/29/items/MLKDream/MLKDream.mp3
  ```
- Other reliable hosts (used to also work in production; verify before using in a public example):
  - **Wikimedia Commons direct media** — `https://upload.wikimedia.org/wikipedia/commons/<path>.mp4`
  - **Any direct `.mp4` / `.mp3` / `.webm` URL** — `yt-dlp`'s generic extractor handles them
  - **Internet Archive video items** — `https://archive.org/download/<id>/<file>.mp4`
- **The form placeholder in `/upload`** should suggest direct media URLs, not `https://youtube.com/watch?v=...`. The M1 main skill enforces this; if you see a student about to paste a YouTube URL, redirect to one of the alternatives.
- If a future milestone genuinely needs YouTube support: the answer is a residential-IP proxy or a cookie-refresh sidecar — both out of scope for M1.

---

### Rule 4 — Always pass the `language` parameter when you know it — don't rely on auto-detect

> **The rule:** If you know the audio language, pass `language="<ISO-639-1>"` to `transcriptions.create`. Only omit it when the audio language is genuinely unknown or mixed.

**Why:** Whisper's auto-detect is accurate for most major languages, but it's done by listening to the first ~30 seconds. If a Mandarin lecture opens with an English brand name ("Welcome to Claude Code") followed by Mandarin content, the detector can lock onto English and transcribe the rest of the file phonetically as English — gibberish. Explicit `language="zh"` skips the detection step entirely.

**How to apply:**
- In M1, the worker can take an optional `language` field from the `jobs` row; if set, pass it through. If unset, omit the parameter and let Whisper detect.
- For the M1 demo, the student should set `language` to match their test audio:
  - MLK mp3 smoke test → `language="en"`
  - Mandarin lecture → `language="zh"`
  - Japanese lecture → `language="ja"`
- **Symptom mapping:** "the transcript is gibberish English-looking text" + audio is non-English → first suspect is missing/wrong `language`. Re-submit with the correct value before assuming the audio is the problem.

ISO-639-1 codes — `en`, `zh`, `ja`, `ko`, `es`, `fr`, `de`, `pt`, `ru`. Full list on OpenAI's [supported languages page](https://platform.openai.com/docs/guides/speech-to-text/supported-languages).

---

### Rule 5 — Track Whisper cost: ~$0.006 / audio-minute — set a billing cap before opening the floodgate

> **The rule:** Before any student submits "a long file" or "a batch of files," they have a billing cap set on their OpenAI key. The course-default cap is $10/month — enough for hundreds of test transcriptions, low enough that a runaway worker can't drain the credit balance.

**Why:** Whisper's `whisper-1` is priced at $0.006 per audio-minute (as of 2026, verify current pricing on OpenAI's pricing page). That's cheap per file — a 10-minute lecture is $0.06 — but a stuck retry loop that re-transcribes the same hour-long file 1000 times overnight is real money. The OpenAI dashboard's soft + hard caps stop that scenario.

**How to apply:**
- **One-time setup:** OpenAI dashboard → Settings → Limits → set a usage limit (course-default $10/mo soft, $20/mo hard).
- **E2 in the M1 checklist** asks for this; it's recommended-not-required, so mark ⚠️ rather than ❌ if missing.
- **Rule of thumb for sizing:** 1 hour of audio = 60 minutes × $0.006 = $0.36 per transcription. A class of 30 students running M1 three times each = ~$1 of Whisper cost per student. Small, but not zero.
- **Cost driver to watch:** Rule 2's chunking does NOT cost extra — Whisper bills per audio-minute, not per request. 6 × 10-min chunks = 60 audio-minutes = $0.36, same as one 60-min file would have cost.

---

### Rule 6 — Wrap Whisper calls in retry-with-backoff for 5xx and 429 — but NOT for 4xx

> **The rule:** Transient failures (`429 Too Many Requests`, `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`) get retried with exponential backoff. Permanent failures (`400 Bad Request`, `401 Unauthorized`, `413 Request Entity Too Large`, `415 Unsupported Media Type`) do not — they need a code fix, not a retry.

**Why:** Whisper occasionally returns 429 (rate limit) or 5xx (capacity) — these clear on a retry seconds later. Retrying a 413 ("file too big") just bills you for the same upload again with the same outcome; retrying a 401 ("bad key") amplifies whatever leaked-key situation you've gotten into. The distinction matters for both cost and correctness.

**How to apply:**
```python
import openai, time

def transcribe_with_retry(client, audio_path, language=None, max_retries=4):
    for attempt in range(max_retries):
        try:
            with open(audio_path, "rb") as f:
                return client.audio.transcriptions.create(
                    model="whisper-1",
                    file=f,
                    language=language,
                )
        except openai.RateLimitError:
            wait = 2 ** attempt
            time.sleep(wait)
        except openai.APIStatusError as e:
            if 500 <= e.status_code < 600:
                wait = 2 ** attempt
                time.sleep(wait)
            else:
                raise  # 4xx — don't retry
    raise RuntimeError(f"Whisper failed after {max_retries} retries")
```
For M1's scale (one job at a time), 4 retries with backoff (1s → 2s → 4s → 8s) is plenty.

---

### Rule 7 — Don't cleanup downloaded video / audio files until the job is marked `done` — keep them on disk for re-runs

> **The rule:** During a transcription job, write the downloaded video and the converted audio to `/tmp/<job_id>/` (or any per-job dir). Don't delete them on success — let them age out via the EC2's normal `/tmp` rotation. Definitely don't delete them on failure.

**Why:** When a job fails after the download (e.g. ffmpeg crash, Whisper 5xx, Supabase write error), the operator's first move is to re-run the same job. If you've already deleted the source files, re-running means re-downloading — which on YouTube-style hosts can fail with a different error the second time, on a 500 MB file is just slow, and on a flaky source URL might not succeed at all. Keeping the files on disk turns a re-run from "30 minutes" to "30 seconds."

**How to apply:**
- Use a per-job temp dir: `os.makedirs(f"/tmp/{job_id}", exist_ok=True)`.
- Stage downloads + ffmpeg outputs inside that dir.
- On success: don't `rm -rf`. Linux's `/tmp` reaper handles aging files; the EC2 has 30 GB EBS, which holds tens of jobs of working files.
- On failure: also don't `rm -rf` — operators will want to inspect.
- **If `/tmp` fills up** (rare on M1, but possible): a one-shot SSM cleanup is acceptable: `find /tmp -mindepth 1 -mtime +1 -delete`. Don't bake aggressive cleanup into the worker.

This rule has a soft form: in a high-volume production worker, you'd absolutely clean up to control disk pressure. M1's "one job at a time" pace makes the trade-off cleanly favor "keep around for debugging."

---

## What Whisper is NOT good at — set student expectations early

`whisper-1` is a transcription model, not a magic intelligence layer. The M1 demo intentionally shows the **raw** Whisper output, warts and all — students who haven't used Whisper before usually assume it's worse than it is at audio quality and better than it is at semantic correctness. Calibrate:

| What Whisper IS good at | What Whisper is NOT good at |
|---|---|
| Clean speech, single speaker, minimal background noise | Heavy overlap, multiple speakers shouting over each other |
| Standard accents in major languages | Strong regional dialects, technical jargon it hasn't seen |
| Punctuation and casing (mostly) | Speaker diarization ("Speaker 1:" / "Speaker 2:") — that's a different model |
| Standard pronunciation of names | Brand names, product names, codenames — frequent misspellings |
| Background music behind speech | Sung lyrics during loud music — output is unreliable |

The production stack in this repo runs Whisper output through Claude + GPT-4o for block-combining, typo correction, semantic correction, and contextual proofreading — *because* raw Whisper isn't good enough on its own. M1 stops at raw Whisper deliberately, so the student sees the gap and understands why the later milestones exist.

---

## Things to actively watch out for

These show up often enough to recognize without being hard rules.

1. **The 25 MB limit applies to the *uploaded* file, not the original.** Rule 1 + Rule 2 together fix this. If a student is hitting 413 errors, ask: are you converting to 64 kbps mono mp3 first? They probably aren't.

2. **`ffmpeg` not installed on the EC2** → conversion step crashes with `FileNotFoundError: 'ffmpeg'`. Same shape as the yt-dlp PATH failure ([[aws-best-practice]] Rule 6), but a different fix — install ffmpeg via apt (`sudo apt-get install -y ffmpeg`), not via pip.

3. **`yt-dlp` invisible to systemd's PATH** is *not* a Whisper rule — it's an AWS / systemd rule. See [[aws-best-practice]] Rule 6. Mentioned here only because the symptom looks like "Whisper isn't running" when really nothing has reached Whisper yet.

4. **OpenAI client construction order:** the `OpenAI(api_key=...)` client must be created *after* you've pulled the key from Secrets Manager. The temptation is to put `client = OpenAI()` at module top — that reads from `os.environ` at import time, before your `_get_secret` calls have populated anything. Construct the client inside whatever bootstrap function does the secret fetch.

5. **Empty / silence-only audio** → Whisper returns empty `text`. Not an error from OpenAI's side. Decide whether the worker treats that as `done` (transcript is genuinely empty) or `error` (likely a bad input). M1 treats it as `done` with an empty transcript — but mention it to the student so they don't panic when their silent test clip "succeeded" with no output.

6. **Mixing `whisper-1` (the API model) with `openai-whisper` (the open-source PyPI package).** They are *different things*. The PyPI package runs Whisper locally and downloads ~3 GB of model weights — way too much for a t3.small. The API model is `client.audio.transcriptions.create(model="whisper-1", ...)`. M1 uses the API model, not the package. See [[feedback_no_openai_whisper]] for the durable user preference: the local `openai-whisper` repo is deprecated, don't use it.

7. **Response `text` field, not `transcript`.** OpenAI's API returns `{"text": "..."}` for the default `response_format`. Students who reflexively type `r.transcript` get an `AttributeError`. The full response object also has `language`, `duration`, and (if requested) `segments` / `words`.

8. **`response_format` choices.** Default is `json` (just `text`). Pass `response_format="verbose_json"` if you want segment timestamps; `"srt"` or `"vtt"` if you want a ready-to-render subtitle string. M1 uses default `json` because we only output TXT in this milestone; later milestones may switch to `verbose_json` for word-level alignment.

---

## Out of scope for this course

Real Whisper-in-production concerns we deliberately don't enforce in M1:

- **Speaker diarization** — needs a separate model (e.g. pyannote). Out of scope.
- **Word-level alignment + word-level confidence** — `verbose_json` exposes it but M1 doesn't use it. Future milestone.
- **Custom vocabulary / prompt biasing** — Whisper's `prompt` parameter lets you nudge it toward expected terms. Useful for jargon-heavy lectures; not introduced until the proofreading milestones in the production stack.
- **Local Whisper (open-source package)** — explicitly out of scope and discouraged ([[feedback_no_openai_whisper]]). Don't suggest as a "free alternative."
- **Real-time / streaming transcription** — `whisper-1` is batch-only. The closest API surface is the Realtime API, but that's a different product and a different price point.

---

## Cross-references

- [[aws-best-practice]] — the infrastructure side: where the OpenAI key lives, how the worker reaches it, why systemd is hiding yt-dlp from you
- [[m1-ai-video-transcript]] — where the worker code is actually written (Step 5)
- [[m1-ai-video-transcript-checklist]] — Section C tests the worker, Section D runs an end-to-end transcribe
- [[feedback_no_openai_whisper]] — durable user preference: don't use the local `openai-whisper` package
- [[project_pricing]] — how Whisper cost folds into the per-minute pricing the SaaS charges
- OpenAI docs: https://platform.openai.com/docs/guides/speech-to-text
- yt-dlp options: https://github.com/yt-dlp/yt-dlp
