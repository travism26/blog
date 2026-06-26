# Eval-Driven Agent Development: How I Stopped Tuning Prompts on Vibes

> **Series context:** This is a follow-up to [How I Automate Parts of My SDLC with AI Agents](https://dev.to/rickjms/how-i-automate-parts-of-my-software-development-lifecycle-with-ai-agents-43h7). Earlier posts covered the pipeline overview, the Validate phase, agent state management, and rate-limit resilience. This one is about the part that holds the whole thing together: how I know whether a change to my harness actually made it _better_.

---

## The Question I Couldn't Answer

I changed a prompt. The next run looked better. Did the prompt help, or did I get lucky?

For a long time my honest answer was "I think so?" I'd tweak a slash command, run my pipeline on a feature, watch it succeed, and ship the change. That's vibes-based prompt engineering, and almost everyone building agents does it — because the alternative feels impossible.

Here's the trap. A coding agent is non-deterministic. Run the same task twice and you get two different trajectories. So a single good run tells you almost nothing: you can't separate "my change helped" from "the dice came up nice this time." And open-ended feature work has no single right answer, so you can't just write a unit test for "did the agent build the feature well."

That's two problems stacked on top of each other: **non-determinism** and **no ground-truth**. If you don't solve both, every harness change is a coin flip you can't see.

This post is how I solved it — by stealing the discipline from ML evaluation and pointing it at my own harness.

---

## Where Eval Sits in the Pipeline

Quick orientation for anyone new to the series. My ADW harness runs a feature through seven phases:

```plaintext
research → plan → build → validate → test → review → document
```

Eval isn't one of those phases. Eval is _meta_ — it wraps the whole harness and asks a different question:

```plaintext
                  ┌─────────────────────────────────────┐
   harness  ─────►│  run the full pipeline on N tasks,  │
   change         │  M times each, score every run      │────► verdict:
                  │  (code graders + LLM judge)         │      better / worse / noise
                  └─────────────────────────────────────┘
```

The phases build software. The eval framework measures whether _my changes to how the phases work_ are improvements or regressions. It's testing the tester.

---

## The Core Idea: A Frozen Benchmark You A/B Against

The whole thing rests on one move borrowed from ML: **fix everything except the variable you're testing.**

A few terms I'll use (this harness has its own vocabulary, so here are the one-liners):

- **Target** — a sample codebase a task runs against, _vendored_: copied and frozen into the repo so scores stay comparable over months. If the target drifts, your scores aren't measuring your harness anymore.
- **Task** — one benchmark item: a prompt plus an **oracle**.
- **Oracle / acceptance checks** — the deterministic definition of "correct": shell commands that must exit `0`. No oracle, no task.
- **Variant** — a labeled config under test: a planner, a flag, a branch. This is the thing you're A/B-ing.
- **Trial** — one run of one task. Because agents are non-deterministic, every task runs N times (default 3).

I run two difficulty tiers:

- **Tier 1** — a tiny hermetic notes CLI with pytest. Cheap regression gate, ~5 min/trial.
- **Tier 2** — a frozen **~69K-LOC Express/TypeScript backend** with Postgres + Redis. Realistic headroom, ~20 min/trial (longer when the build loop has to retry). This is where the interesting failures live.

Thirteen tasks across the two tiers — features, bugs, and chores. Each trial copies the target to a clean temp dir, runs the full SDLC against it, and grades the result. Same tasks, same targets, every time. Now a change is measurable.

---

## How I Grade a Run (and Why I Use Two Judges)

Each trial gets scored two completely different ways, on purpose.

**Code graders — deterministic.** These are the things a machine can check without an opinion:

- test pass-ratio (did the target's own suite go green?)
- behavioral acceptance oracle (the task-specific "what correct looks like")
- phases completed, test-retry count
- **cost** — the agent-under-test's token spend, pulled from the API-recorded usage
- wall-clock time
- diff size (did it change three files or thirty?)

**LLM judge — probabilistic.** A separate, fixed model scores what determinism can't: spec quality (0–5) and implementation fidelity (0–5). One combined call, **memoized** so re-runs don't re-pay, and its cost is itemized _separately_ from the agent under test — you never want your judge's spend polluting the number you're optimizing.

Why both? Because they catch different failures. The execution signal (tests) tells you the code _runs_; the judge tells you the code is _good_. A change can make tests pass while quietly trashing code quality, or write beautiful code that fails a behavioral check. The two signals are complementary, not redundant — lean on only one and you'll optimize a blind spot.

---

## The Metric That Actually Matters: pass@k vs pass^k

This is the part I wish someone had told me earlier.

If you average your trials, you get a mean. **Means lie about agents.** A task that succeeds 2 out of 3 times and a task that succeeds 3 out of 3 times can show the same "67% vs 100%" gap that looks like noise — when actually one is reliable and one is a coin flip.

So I report two reliability numbers instead:

- **pass@k** — did _at least one_ of k trials pass? This is the **capability ceiling**: can the agent do this task _at all_?
- **pass^k** — did _all_ k trials pass? This is **consistency**: can it do it _every time_?

A concrete example from my own suite — Task 08, a CRUD feature on the tier-2 backend, run three times:

```plaintext
Task 08 (seasonal-tips CRUD, tier-2 backend):
   trials       → fail, pass, pass
   mean success = 0.67   ← "mostly fine." Looks shippable.
   pass@3       = 1.00   ← it CAN do this task.
   pass^3       = 0.00   ← but not every time. Not reliable.
```

A mean would've called Task 08 "mostly fine" at 0.67. But pass^3 is all-or-nothing — every trial passes or it's a zero — and here it's a zero: the capability is there, the _reliability_ isn't. That's a completely different engineering problem, and you can't fix what your metric hides.

The rule I follow: **a difference smaller than the spread across trials is noise, not signal.** If variant A scores 0.71 and variant B scores 0.68 but the trial-to-trial spread is ±0.15, you've discovered nothing. Run more trials or make the change bigger.

---

## Reading a Scorecard

Here's the format my `compare.py` spits out when I A/B two planners on the same tasks — the kind of comparison I use to settle a question like _is OpenSpec actually a better planner than my original two-agent approach, or do I just like it?_

> _The numbers below are illustrative — they show the **shape** of the answer `compare.py` gives, not a published result. The point is what each row tells you, not these exact values._

```plaintext
A/B: travis (baseline) vs openspec        tasks=13  trials=3
======================================================================
                       travis            openspec          Δ
----------------------------------------------------------------------
pass@3  (capability)   0.85              0.92             +0.07
pass^3  (reliability)  0.54              0.77             +0.23  ◄ the real win
mean spec quality      3.9 / 5           4.3 / 5          +0.4
mean impl fidelity     3.7 / 5           4.1 / 5          +0.4
avg SUT cost / task    $0.71             $0.63            -$0.08
avg diff size          214 LOC           158 LOC          -56
avg wall time          8m12s             7m41s            -31s
----------------------------------------------------------------------
verdict: openspec wins on reliability and cost; capability gap
         within trial spread (±0.06) — treat as tied there.
```

The row to read for isn't capability — if both planners can _eventually_ solve most tasks, pass@3 lands in the same place and tells you little. The row that decides it is **pass^3**: a jump there means a planner didn't make the agent smarter, it made it _consistent_. That's the kind of conclusion you simply cannot reach by eyeballing a couple of runs — and it's exactly what I built the scorecard to surface before I promote a planner to the default slot.

That's the whole payoff. A scorecard like this turns "I think it's better" into "it's +X on pass^3 at lower cost, and the capability gap is within noise" — or it tells you there's no difference worth shipping, which is just as useful.

---

## Keeping the Benchmark Honest: Saturation and the Capture Loop

A benchmark has a failure mode of its own: **saturation.** When your tasks get easy enough that every variant aces them, the suite stops discriminating — good change and bad change score the same, and you're back to flying blind.

Two things keep mine sharp.

**1. Hard-mode tasks with no greppable answer.** My nastiest task is a _real, unplanted_ latent defect I found in the tier-2 backend: a Sequelize `findAndCountAll` combined with a `hasMany` include that inflates `count` by the number of JOINed rows. There's no magic keyword to grep for — the agent has to actually understand ORM semantics to find it. And there are _two_ instances of the bug while the prompt only reports one, so the task also grades whether the agent sweeps for siblings or fixes the one and leaves. That single task discriminates harnesses that "look right" from harnesses that investigate.

**2. The capture loop.** When a real run produces a buggy result on a vendored target, I mint it into a new permanent task with `/capture_eval_task`. The acceptance oracle is harvested from _my_ fix, and before I trust the task I run a **saturation check** — a quick A/B that only earns the task a spot if the current harness _doesn't already ace it_. In other words: every real failure becomes a permanent guard, and the benchmark gets harder exactly as fast as the agent gets better. The suite can't go stale because my own mistakes keep feeding it.

This is the closed loop that turns "a pile of tasks" into a system that stays useful.

---

## Bonus: Finding the Cheapest Thing That Still Works

Once you can score variants, a fun question opens up: what's the _cheapest_ model-and-effort setting that still passes a task?

I run a model×effort sweep — each cell of the grid is just another eval variant — and `report` prints a Pareto view plus the cheapest cell that still gets a full pass. That's how I reason about which model each phase gets: today the harness runs Sonnet for build and the execution-heavy phases (validate, test, review, document) and reserves Opus for planning, where spec quality moves pass^k the most. The sweep is what turns "can I drop a tier here?" from a vibe into a lookup — it shows exactly where going cheaper stops being free.

---

## Skeptic's Corner

**"This is a lot of infrastructure for a side project."** It is. But it's also the single highest-leverage thing I built. Every other improvement to the harness — every prompt tweak, every new planner, every phase change — is now measurable instead of hopeful. The eval framework pays for itself the first time it stops you from shipping a regression you were _sure_ was an improvement.

**"Thirteen tasks isn't statistically significant."** Correct, and I don't pretend otherwise — that's exactly why I report spread and refuse to call anything inside the noise band a win. The goal isn't a p-value, it's to stop fooling myself. Thirteen real tasks scored honestly beats one impressive demo every time.

**"The LLM judge is graded by an LLM — isn't that circular?"** That's why the judge is fixed, memoized, cost-isolated, and _paired with deterministic graders_. The execution signal is the ground truth; the judge only scores the things tests can't see. If they disagree, that disagreement is itself a signal worth reading.

---

## The Bigger Picture

Coding agents are easy to demo and hard to trust. The thing that moved my harness from "neat demo" to "system I iterate on with confidence" wasn't a smarter prompt or a bigger model — it was deciding to **measure**. Frozen targets, real tasks, two kinds of graders, and reliability numbers that don't let a flaky 2-of-3 hide behind a friendly average.

Most "AI agent" projects skip this part because it's unglamorous. That's exactly why it's worth doing. The discipline is the moat.

---

## What's Next

The next post in the series gets back into the pipeline itself: the **Review phase** — how the review agent compares what was built against the original spec, categorizes issues by severity, and how I auto-patch blockers. After that, a deeper look at the **OpenSpec planner integration** — and how I'm using the scorecard above to decide whether it earns the default slot.

If you're building your own harness, the one thing I'd steal first isn't any single phase — it's the eval loop. Build the thing that tells you whether you're improving, and everything else gets easier.

---

_Tags: `#ai` `#agents` `#claudecode` `#testing` `#softwaredevelopment`_
