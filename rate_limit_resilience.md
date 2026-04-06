# Rate Limiting and SDK Resilience: When Your Agent Dies Mid-Run

> **Series context:** Follow-up to [How I Automate Parts of My SDLC with AI Agents](https://dev.to/rickjms/how-i-automate-parts-of-my-software-development-lifecycle-with-ai-agents-43h7). Previous post covered agent state management. This one covers what happens when the SDK itself crashes on a rate limit — and how to build the resilience layer to handle it.

---

## The Crash I Didn't Expect

I was running a long autonomous pipeline — multiple phases, 10+ minutes of work. Phase 6 hits a rate limit. Not a graceful 429 with a retry-after header. A hard SDK crash that killed the process.

All the upstream work was intact (because I had state management). But the agent was dead. No retry. No backoff. Just an exception and an exit.

This is different from HTTP 429 handling you'd write in a normal API client. The Claude SDK can surface rate limits as exceptions at unexpected points — mid-stream, mid-tool-call. You need to handle it at the orchestration layer, not just around the API call.

---

## The Problem with Naive Retry

The first instinct is to wrap the SDK call in a try/except and retry. But there are two failure modes that naive retry makes worse:

**1. Retry storms** — If multiple agents hit rate limits simultaneously and all retry at the same interval, you create a thundering herd. They all come back at the same time, all get rate-limited again.

**2. Non-idempotent side effects** — If your agent was in the middle of writing a file or updating state when it crashed, a blind retry may duplicate work or corrupt partial output.

The resilience layer needs to handle both.

---

## What I Built

### Exponential backoff with jitter

```python
import random
import time

def backoff_delay(attempt: int, base: float = 1.0, max_delay: float = 60.0) -> float:
    delay = min(base * (2 ** attempt), max_delay)
    jitter = random.uniform(0, delay * 0.2)
    return delay + jitter

def run_with_retry(fn, max_attempts: int = 5):
    for attempt in range(max_attempts):
        try:
            return fn()
        except RateLimitError as e:
            if attempt == max_attempts - 1:
                raise
            delay = backoff_delay(attempt)
            print(f"Rate limited (attempt {attempt + 1}). Retrying in {delay:.1f}s...")
            time.sleep(delay)
        except SDKCrashError as e:
            # Not a rate limit — don't retry, surface immediately
            raise
```

The jitter (±20% of the delay) is what prevents retry storms. Each agent wakes up at a slightly different time.

### Distinguishing rate limits from other SDK errors

Not all SDK exceptions should be retried. A rate limit is transient. A malformed prompt or context window overflow is not — retrying wastes time and tokens.

```python
RETRYABLE_ERRORS = {
    "rate_limit_exceeded",
    "overloaded",
    "timeout",
}

def is_retryable(error: Exception) -> bool:
    error_type = getattr(error, "error_type", "")
    return error_type in RETRYABLE_ERRORS
```

Build a small taxonomy of your SDK's error types. Retry transient errors. Fail fast on everything else.

---

## Phase-Level vs. Call-Level Retry

There are two places you can put retry logic:

**Call-level:** wraps individual SDK calls. Good for transient errors on a single tool call mid-phase.

**Phase-level:** restarts the entire phase if it fails. Good for phases that are cheap to re-run and where partial completion is hard to detect.

I use both. Call-level retry handles the mid-stream crashes. Phase-level retry handles the cases where I'd rather restart clean than reason about partial state.

```python
# Phase-level: orchestrator retries the whole phase up to N times
for attempt in range(phase.max_retries):
    result = run_phase(phase)
    if result.success:
        break
    if not is_retryable(result.error):
        raise result.error
    time.sleep(backoff_delay(attempt))
```

---

## Making Phases Safe to Retry

For phase-level retry to work, phases need to be idempotent — or close to it. The design principle I follow: **overwrite, don't append**.

If a phase writes a spec file, it writes to a deterministic path based on the workflow ID. On retry, it overwrites the previous partial output. No duplicate files, no corrupted state.

```python
# Deterministic output paths based on workflow ID
spec_path = f"specs/feature-{workflow_id}.md"
# Overwrite on retry — don't check if it exists
with open(spec_path, "w") as f:
    f.write(spec_content)
```

This also means you can manually inspect partial output from a failed run before retrying — the files are still there.

---

## Observability: Knowing When the Retry Layer Is Working

If your retry logic is silent, you have no idea whether it's saving you or masking a deeper problem. Log every retry:

```
[10:14:33] validate: attempt 1 — rate limited. Retrying in 4.2s...
[10:14:37] validate: attempt 2 — rate limited. Retrying in 9.7s...
[10:14:47] validate: attempt 3 — SUCCESS
```

Track retry counts in your phase state too. If you're consistently hitting 3+ retries on a phase, it's a signal to restructure that phase — break it into smaller chunks, reduce token usage, or add a delay before it runs.

---

## The Bigger Picture

Rate limit resilience is boring infrastructure. Nobody ships a feature because they got backoff/jitter right. But without it, long-running autonomous agents are brittle — they work 90% of the time and fail mysteriously the other 10%.

The principle: your orchestration layer should be more fault-tolerant than any individual agent. Agents fail. The system shouldn't.

---

## What's Next

The next post covers the **Review Phase** — how the review agent compares what was built against the original spec, categorizes issues by severity, and how I use it to auto-patch blockers without human intervention.

---

_Tags: `#ai` `#agents` `#claudecode` `#infrastructure` `#resilience`_
