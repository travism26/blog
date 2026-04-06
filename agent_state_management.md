# Agent State Management: How I Stopped Losing Work Mid-Run

> **Series context:** This is a follow-up to [How I Automate Parts of My SDLC with AI Agents](https://dev.to/rickjms/how-i-automate-parts-of-my-software-development-lifecycle-with-ai-agents-43h7). In that post I walked through the full ADW pipeline. Here I'm going deeper on one of the less glamorous but most important pieces: state management.

---

## The Problem Nobody Talks About

When you run a single agent for 30 seconds, state doesn't matter. When you run a multi-phase pipeline for 10+ minutes, it becomes everything.

Here's what happens without proper state management:

- Phase 4 fails after Phase 3 ran for 8 minutes
- You fix the bug in Phase 4
- You have to restart from Phase 1
- You just wasted 8 minutes of compute and API tokens

This is the unsexy problem that kills autonomous agent workflows in production. Nobody writes blog posts about it because it's not flashy. But if you're building agents that do real work, you'll hit it.

---

## What State Actually Needs to Track

Before writing any code, I had to decide: what does "state" even mean for a multi-phase agent pipeline?

After a few painful restarts, I landed on this:

```python
{
  "workflow_id": "a1b2c3d4",
  "created_at": "2025-01-15T10:00:00Z",
  "phases": {
    "plan": {
      "status": "completed",
      "completed_at": "2025-01-15T10:02:30Z",
      "output_path": "specs/feature-a1b2c3d4.md"
    },
    "build": {
      "status": "completed",
      "completed_at": "2025-01-15T10:08:12Z"
    },
    "validate": {
      "status": "failed",
      "attempts": 2,
      "last_error": "golangci-lint: 3 violations found after max retries"
    },
    "test": { "status": "pending" },
    "review": { "status": "pending" },
    "document": { "status": "pending" }
  }
}
```

Three possible statuses per phase: `pending`, `completed`, `failed`. That's it. I resisted the urge to add more.

---

## The Two Design Decisions That Matter

### 1. Write state after each phase completes — not at the end

This sounds obvious but it's easy to get wrong. If you batch state writes, a crash in Phase 6 means you lose everything. Write state to disk immediately after each phase succeeds.

```python
def run_phase(self, phase_name: str) -> PhaseResult:
    result = self._execute_phase(phase_name)
    if result.success:
        self.state.mark_completed(phase_name, result.output_path)
        self.state.save()  # write immediately
    return result
```

### 2. Overwrite, don't append

When re-running a phase, overwrite the existing state rather than deleting and re-creating it. This preserves the workflow ID, timestamps for completed phases, and output paths. I learned this the hard way — deleting state on retry caused me to lose references to spec files that later phases still needed.

```python
def mark_completed(self, phase: str, output_path: str = None):
    self.data["phases"][phase]["status"] = "completed"
    self.data["phases"][phase]["completed_at"] = datetime.utcnow().isoformat()
    if output_path:
        self.data["phases"][phase]["output_path"] = output_path
    # Don't delete anything — just update what changed
```

---

## Resume From Any Phase

Once state is tracked correctly, resuming from a specific phase becomes trivial:

```bash
# Something failed at validate — resume from there
uv run travis_sdlc.py "Add rate limiting" --resume-from validate

# Or just re-run the whole thing — completed phases are skipped
uv run travis_sdlc.py "Add rate limiting"
```

The orchestrator checks state at startup:

```python
for phase in self.phases:
    if self.state.is_completed(phase):
        print(f"  ⏭  {phase}: already completed, skipping")
        continue
    result = self.run_phase(phase)
    if not result.success:
        break
```

This is the part that saves the most time in practice. A 10-minute pipeline that fails at minute 9 can resume from minute 9, not minute 0.

---

## What State Doesn't Handle: Output Passing Between Phases

State tracks *that* a phase completed. But how does Phase 3 (validate) know where Phase 1 (plan) wrote its spec file?

I solved this by having each phase write its primary output path into state, and having downstream phases read it:

```python
# Plan phase writes:
state.mark_completed("plan", output_path="specs/feature-a1b2c3d4.md")

# Build phase reads:
spec_path = state.get_output_path("plan")
# passes spec_path into the build agent's prompt
```

This makes the data flow explicit and auditable. If the build agent used the wrong spec, you can see exactly which file was passed.

---

## The Bigger Picture

State management is what separates "agent demos" from "agent infrastructure." A demo runs once, succeeds, looks great. Infrastructure handles failures, resumes work, and doesn't make you pay twice for the same API calls.

The principle: treat your multi-agent pipeline like a distributed system. Assume any phase can fail at any time. Design for recovery, not just the happy path.

---

## What's Next

The next post covers something related: **rate limiting and SDK resilience** — what happens when your agent is mid-run and hits a 429, and how to build the retry layer that prevents a crash from losing all your work.

---

_Tags: `#ai` `#agents` `#claudecode` `#infrastructure` `#softwaredevelopment`_
