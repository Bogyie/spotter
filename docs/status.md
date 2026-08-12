# Spotter Status

> **Purpose:** the fastest way to answer three questions: **What works today? What blocks the project now? What comes next?**  
> Runtime details: [Architecture](architecture.md) · Sequence and evidence gates: [Roadmap](roadmap.md)

---

## 30-second summary

Spotter is currently a **Hook-based research prototype**. It already has real trajectory journals, deterministic gates, Git snapshots, fork/replay, a shadow reviewer, an audit ledger, labeling/metrics, and a counterfactual experiment harness.

The target product is a standalone runtime:

```text
CURRENT
Codex hooks
   ↓
a new Spotter process per hook
   ↓
journal / gate / snapshot / periodic shadow review

TARGET
Codex TUI
   ↓
External Codex App Server
   ↕ events / steer / interrupt
spotterd
   ↓
PreToolUse Hook only
(deterministic synchronous enforcement)
```

The roadmap no longer uses `P0–P9` / `E0–E5`. It is organized by named outcomes:

```text
Runtime → Observe → Detect → Intervene → Recover → Harden
```

The **current blocker** is [#78](https://github.com/spotter-agent/spotter/issues/78): prove that ordinary `codex` and Spotter can use the same external App Server, observe the same thread/turn, and that `turn/steer` reaches the real active user turn.

---

## Current focus

### Runtime

Before building the standalone daemon around App Server assumptions, prove the control boundary itself.

[#78](https://github.com/spotter-agent/spotter/issues/78) must establish:

- TUI and Spotter see the same thread id;
- Spotter sees the same active turn id over multiple actions;
- the needed runtime events are available;
- `turn/steer` affects the real user-visible turn;
- concurrent Codex sessions can be distinguished;
- embedded/disconnected modes are diagnosable as degraded.

If no viable path passes, revise the architecture before proceeding.

In parallel, the highest-value evidence foundations are:

- [#21](https://github.com/spotter-agent/spotter/issues/21) — mechanically scored task set;
- [#42](https://github.com/spotter-agent/spotter/issues/42) — replay/fork fidelity and noise floor;
- [#37](https://github.com/spotter-agent/spotter/issues/37) — observability ceiling baseline/post-migration measurement;
- [#33](https://github.com/spotter-agent/spotter/issues/33) — runtime cost/timing/outcome telemetry.

---

## Quick capability status

Legend: ✅ implemented · 🟡 partial/shadow · 🧪 proof required · 🎯 target · ❌ not implemented

| Area | Status | What exists now | Next concrete step |
| --- | --- | --- | --- |
| Hook ingestion | ✅ | `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse` are journaled | Replace broad observation with App Server events after #78 |
| Deterministic gate | ✅ | Shell-aware checks for destructive commands, path/dependency constraints, fail-open ambiguity handling | Move bounded enforcement behind daemon IPC and re-measure latency |
| Journal | ✅ | Crash-tolerant JSONL, locking, fsync, torn-tail recovery | Keep as durable history/recovery, not hot state |
| Snapshot | ✅ | Git-backed snapshots, deduplication, pruning, detached restore | Integrate into managed runtime lifecycle |
| Fork / replay | ✅ | Continue Codex from a shared prefix in a detached worktree | Measure fidelity/noise floor in #42 |
| Shadow reviewer | ✅ | Produces `CONTINUE`, `VERIFY`, `NUDGE`; verdicts are recorded only | Move to event-driven candidates, then live delivery |
| Audit ledger | 🟡 | Claim/evidence state and stale propagation where outcomes are observable | Move independent state into `spotterd` (#31) |
| Labels / metrics | ✅ | Coverage-aware labeling and precision/FP metrics | New repository label taxonomy + broader cost/miss/harm metrics |
| Counterfactual harness | ✅ | Control/guidance same-prefix pairs can be prepared/run | Add mechanically scored tasks (#21) |
| Standalone runtime | ❌ | No long-lived owner of live supervision state | Resolve #78, then build runtime boundary |
| App Server primary observation | 🧪 | Target design only | Prove shared-server path in #78 |
| Event-driven detection | ❌ | Reviewer is still cadence-based | #28 after Runtime/Observe |
| Live `VERIFY` / `NUDGE` | ❌ | Reviewer decisions stop at the journal | #22 via `turn/steer` |
| `INTERRUPT` / `RESTART` | ❌ | No live recovery path | #26 + #30 after soft intervention is understood |
| Product lifecycle | 🎯 | Current paths are source/plugin installs | standalone setup/status/doctor/teardown under Runtime/Harden |

---

## Roadmap at a glance

### Runtime

Prove and establish the standalone App Server/`spotterd` boundary.

### Observe

Use App Server events as the primary trajectory source and maintain live supervision state. Measure what is actually observable early enough to act on.

### Detect

Trigger semantic review from cheap candidate signals. Measure precision, miss rate, and detection delay together.

### Intervene

Deliver `VERIFY` / `NUDGE` to the correct active turn. Measure benefit, harm, recovery, ignored guidance, and wrong-nudge susceptibility.

### Recover

Add `INTERRUPT` / `RESTART` only with stronger evidence and explicit side-effect/reversibility handling.

### Harden

Make setup, upgrades, schemas, retention, cleanup, recovery, and long-term operation predictable.

See [Roadmap](roadmap.md) for detailed exit criteria and linked issues.

---

## Evidence status

Implementation progress and research evidence remain separate.

| Question | Evidence today |
| --- | --- |
| Can Spotter collect real coding-agent trajectories? | Yes |
| Can deterministic gates catch concrete policy violations? | Yes, with precision/miss-rate work still ongoing |
| Can Spotter produce plausible semantic reviewer verdicts? | Yes, in shadow mode |
| Can Spotter branch a shared prefix for counterfactual experiments? | Yes |
| Is the fork instrument's causal noise floor known? | **No — #42** |
| Is there a reproducible mechanically scored task corpus? | **No — #21** |
| Has live Spotter guidance been shown to improve outcomes? | **No** |
| Is the App Server target architecture operationally viable? | **Not yet — #78 is the blocker** |

A mechanism being implemented does not prove it improves outcomes. Null and negative results are first-class outcomes for this project.

---

## Issue triage

Repository issues use a small label vocabulary defined in [Repository Conventions](conventions.md) and `.github/labels.json`.

The useful filters are:

- `type:*` — what kind of work is this?
- `priority:*` — how urgently should it be selected?
- `area:*` — which part of Spotter does it belong to?
- `status:blocked` — is meaningful progress currently blocked?

Roadmap stages are **not labels**. They are durable product/evidence outcomes; priorities are expected to change as the project advances.

---

## Documentation map

| If you want to know... | Read |
| --- | --- |
| What Spotter is and why it exists | [Concept](concept.md) |
| What is implemented right now | **This document** |
| Exact process/data/control boundaries | [Architecture](architecture.md) |
| Install → setup → run → recover → upgrade → remove | [Lifecycle](lifecycle.md) |
| What should be built in what order | [Roadmap](roadmap.md) |
| Prior work, hypotheses, and evidence | [Research](research.md) |
| Repository/issue conventions | [Conventions](conventions.md) |
