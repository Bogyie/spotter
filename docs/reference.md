# Reference

> **Purpose:** collect papers, systems, and implementation precedents that inform Spotter. This document answers **what already exists, what Spotter can learn from it, and what should not be inherited blindly**.

Spotter currently treats every project in this document as a **reference, not a runtime dependency**. In particular, Spotter does **not** depend on Shepherd, NVIDIA NeMo Relay, or OpenClaw. Their value is to reduce design uncertainty and avoid repeating integration, lifecycle, interception, and evaluation mistakes that have already been explored elsewhere.

For Spotter's own hypotheses, evidence state, research questions, and evaluation plan, see [Research](research.md).

---

## 30-second reference map

| Work / system | What it demonstrates | What Spotter takes from it |
| --- | --- | --- |
| **Wink** | asynchronous course correction during coding trajectories | persistent observer, selective guidance, silence by default |
| **SWE-PRM** | process-level SWE failure taxonomy and trajectory correction | structured failure candidates and interpretable verdicts |
| **AgentForesight** | prefix-only online auditing and early failure prediction | timing/lead-lag measurement, no hindsight assumptions |
| **FailFast-RestartSmart** | early stop/restart of failing SWE trajectories | `INTERRUPT` / `RESTART` as distinct control actions |
| **Calibration Is Not Control** | risk prediction is not the same as useful intervention | action-conditioned intervention advantage |
| **interwhen** | deterministic verification can replace model judgment for some checks | verifier-first fast path |
| **Grounded Continuation** | claim/evidence dependencies can be invalidated transitively | audit ledger and stale propagation |
| **AgentProcessBench** | exploration can be neutral rather than erroneous | conservative supervision and abstention |
| **HarnessFix** | normalize traces and diagnose harness failures separately | Trace IR and adapter boundaries |
| **Refute-or-Promote** | falsification can outperform model consensus | evidence-first `VERIFY` behavior |
| **ECLoop** | evidence-conditioned execution can stop premature commitment and improve SWE outcomes | execution-time evidence gates; proof that runtime intervention can be beneficial |
| **Shepherd** | live meta-agent supervision with `steer/revert` over execution traces | supervisor behavior and experimental design |
| **NVIDIA NeMo Relay** | mature interception/middleware/observability plumbing around agent runtimes | implementation patterns for boundaries, lifecycle, telemetry, diagnostics |
| **OpenClaw Codex supervision** | serious Codex App Server integration, session control, steer and interrupt | App Server connection/control/race-condition implementation reference |

---

# Research references

## 1. Wink — asynchronous course correction for coding agents

**Wink: Recovering from Misbehaviors in Coding Agents**  
Rahul Nanda et al., 2026  
https://arxiv.org/abs/2602.17037

### Why it matters

Wink is a close precedent for Spotter's original behavioral idea: observe a coding agent while it works and inject targeted guidance when its trajectory begins to go wrong.

The useful framing is not “another reviewer model,” but **asynchronous runtime course correction**.

### What Spotter borrows

- persistent online observation instead of post-hoc-only review;
- semantic review running mostly outside Main's execution path;
- compact coding-agent failure categories;
- targeted guidance rather than a competing implementation;
- the principle that the observer should usually stay silent.

### Spotter extension

Spotter additionally cares about a native coding-agent attachment model, deterministic fast-path checks, explicit intervention timing, and eventually speculative supervision that can finish before Main reaches the relevant decision boundary.

---

## 2. SWE-PRM — process-level SWE trajectory correction

**When Agents go Astray: Course-Correcting SWE Agents with PRMs**  
Shubham Gandhi et al., 2025  
https://arxiv.org/abs/2509.02360

### Why it matters

SWE-PRM treats failures as **trajectory-level process problems**: redundant exploration, loops, premature stopping, and other execution inefficiencies rather than only final-answer failures.

### What Spotter borrows

- attach failure categories to the execution process;
- correct at inference time without retraining Main;
- keep reviewer output structured and interpretable;
- evaluate efficiency as well as task success.

A structured Spotter verdict should remain closer to:

```json
{
  "candidate": "POSSIBLE_SPEC_DRIFT",
  "decision": "VERIFY",
  "target": "constraint:C2",
  "reason": "The current edit changes files outside the requested scope.",
  "probe": "Re-read the explicit scope constraint before continuing."
}
```

than unrestricted reviewer prose.

---

## 3. AgentForesight — prefix-only online auditing

**AgentForesight: Online Auditing for Early Failure Prediction in Multi-Agent Systems**  
Boxuan Zhang et al., 2026  
https://arxiv.org/abs/2605.08715

### Why it matters

A runtime supervisor cannot use hindsight. It has to decide from the trajectory prefix visible **now**.

### What Spotter borrows

- judge from the current prefix only;
- separate eventual detection from detection early enough to matter;
- measure the first point where intervention becomes observable, warranted, decided, and delivered.

This motivates metrics such as detection delay, supervision lead/lag, stale-intervention rate, and wasted actions before correction.

---

## 4. FailFast-RestartSmart — stop and restart bad trajectories

**Fail-Fast, Restart-Smart: Early Failure Prediction and Restart for SWE Agentic Tasks**  
Chenyu Wang et al., 2026  
https://arxiv.org/abs/2608.03222

### Why it matters

Sometimes continuing inside the same reasoning context is worse than abandoning it and restarting from selected state.

### What Spotter borrows

- `RESTART` is distinct from `NUDGE`;
- reasoning context can become contaminated by stale assumptions;
- repository state and reasoning state do not have to be retained together;
- stronger interventions need much stricter false-positive budgets.

Spotter should not implement automatic restart until it has a trustworthy state lineage, external-side-effect model, and evidence that restart actually outperforms continuation.

---

## 5. Calibration Is Not Control — intervention advantage

**Calibration Is Not Control: Why LLM-Agent Oversight Needs Intervention**  
Chubin Zhang et al., 2026  
https://arxiv.org/abs/2606.21399

### Why it matters

Predicting that a trajectory is likely to fail is not the same as knowing an intervention will improve it.

Spotter therefore cares about an action-conditioned quantity closer to:

```text
Advantage(VERIFY | prefix)
Advantage(NUDGE | prefix)
Advantage(BLOCK | proposal)
Advantage(INTERRUPT | prefix)
```

rather than risk scores alone.

---

## 6. interwhen — verifier-first supervision

**interwhen: A Generalizable Framework for Verifiable Reasoning with Test-time Monitors**  
Vishak K. Bhat et al., 2026  
https://arxiv.org/abs/2602.11202  
Code: https://github.com/microsoft/interwhen

### Why it matters

Some runtime properties are directly verifiable and do not need broad semantic model judgment.

### What Spotter borrows

- extract deterministic properties into executable checks;
- prefer external evidence over model debate;
- keep semantic model calls selective;
- avoid forcing all reasoning into a rigid formal schema.

This is the main precedent for keeping deterministic enforcement bounded and separate from asynchronous semantic supervision.

---

## 7. Grounded Continuation — claim/evidence dependency tracking

**Grounded Continuation: A Linear-Time Runtime Verifier for LLM Conversations**  
Qisong He, Yi Dong, Xiaowei Huang, 2026  
https://arxiv.org/abs/2605.14175

### Why it matters

Hypotheses depend on evidence. When evidence is invalidated, downstream conclusions should become stale rather than remaining silently authoritative.

Coding example:

```text
E1: timeout correlates with high concurrency
        ↓ supports
H1: Redis pool exhaustion causes timeout
        ↓ motivates
P1: change Redis pool configuration

new evidence
E2: stack trace points to upstream HTTP client

→ H1 stale
→ P1 requires revalidation
```

Spotter's typed audit ledger and stale propagation are informed by this line of work, while remaining constrained by what the runtime can actually observe.

---

## 8. AgentProcessBench — exploration is not automatically failure

**AgentProcessBench: Diagnosing Step-Level Process Quality in Tool-Using Agents**  
Shengda Fan et al., 2026  
https://arxiv.org/abs/2603.14465  
Code/data: https://github.com/RUCBM/AgentProcessBench

### Why it matters

Real tool-use contains steps that are neither clearly useful nor clearly wrong. Exploration may look inefficient from a short window while still being justified.

### What Spotter borrows

- preserve a neutral/uncertain state;
- treat repetition as a signal rather than an automatic verdict;
- intervene conservatively around ambiguous exploration;
- separate cheap detection candidates from semantic decisions.

---

## 9. HarnessFix — normalize traces and diagnose the harness

**From Failed Trajectories to Reliable LLM Agents: Diagnosing and Repairing Harness Flaws**  
Mengzhuo Chen et al., 2026  
https://arxiv.org/abs/2606.06324

### Why it matters

A runtime failure may belong to Main, Spotter, or the integration layer. Backend-specific events should not leak through every supervision component.

### What Spotter borrows

```text
Codex raw events
  ↓
CodexAdapter
  ↓
Trace IR
  ↓
Spotter core
```

The normalized representation should preserve provenance and make it possible to distinguish Main failure from observer/harness failure.

---

## 10. Refute-or-Promote — falsification before intervention

**Refute-or-Promote: An Adversarial Stage-Gated Multi-Agent Review Methodology for High-Precision LLM-Assisted Defect Discovery**  
Abhinav Agarwal, 2026  
https://arxiv.org/abs/2604.19049

### Why it matters

Multiple models can agree on the same wrong claim. A small empirical probe can be more useful than additional model consensus.

### What Spotter borrows

- try to falsify consequential hypotheses;
- use another model as an independent challenger, not a source of truth;
- promote intervention only when evidence supports it;
- prefer a decisive verification step over vague disagreement.

---

## 11. ECLoop — evidence-conditioned execution against premature commitment

**Preventing Premature Commitment in Coding Agents with an Evidence-Conditioned Execution Layer**  
Yisen Xu, Chenglin Li, Zehao Wang, Jinqiu Yang, Tse-Hsun Chen, 2026  
https://arxiv.org/abs/2607.28815

### Why it matters

ECLoop identifies a concrete coding-agent failure mode: the agent starts editing or submitting before it has inspected enough repository evidence to justify the action. It inserts an execution layer between the agent and repository, compiles task-specific evidence conditions from the issue and repository structure, tracks whether those conditions have been satisfied, and postpones unsupported actions.

On SWE-bench Verified, the paper reports **+4.8 to +11.8 percentage points Pass@1** across two models and two agent scaffolds, with no model retraining or scaffold changes. It also reports up to **12.1% lower average token consumption**, because unsupported trajectories are redirected before more work is wasted.

### What Spotter borrows

- runtime intervention can target **premature commitment**, not only safety or destructive actions;
- evidence requirements can be represented explicitly rather than only as prose feedback;
- some unsupported actions should be delayed until evidence is gathered;
- runtime control can improve both success and efficiency rather than necessarily imposing a cost;
- intervention quality should be evaluated against real SWE task outcomes.

### Where Spotter differs

ECLoop is primarily an **evidence-conditioned execution gate**. Spotter's broader target is an independent pair/Navigator that reasons about goals, architecture, assumptions, exploration quality, repeated failure, and likely future actions—not only whether predefined evidence conditions have been satisfied.

Spotter also wants semantic supervision to stay mostly **off Main's critical path**. ECLoop is therefore best treated as strong evidence for runtime gating and a source of condition/state-machine ideas, not as the target supervision architecture.

---

# Implementation references

## 12. Shepherd — supervisor behavior without native attachment

Repositories:

- https://github.com/shepherd-agents/shepherd
- https://github.com/shepherd-agents/shepherd-experiments

Shepherd is a runtime substrate for inspection, reversibility, and supervision. Its framework owns execution: tasks run inside Shepherd-managed workspaces and produce retained outputs that can later be selected, applied, or discarded.

Its live-intervention experiment is especially relevant. A meta-agent supervises worker agents through an effect stream and can intervene with `steer/revert` when it detects coordination failure.

This validates much of Spotter's initial behavioral intuition:

```text
worker execution
      ↓
effect / trajectory stream
      ↓
meta-agent supervisor
      ↓
selective intervention
```

The important difference is **execution ownership**. Shepherd is framework-first: the user runs the workload through Shepherd. Spotter wants to attach to the coding agent the user already runs while leaving that workflow unchanged.

Target Spotter experience:

```text
before Spotter installation
$ codex

after Spotter installation
$ codex
```

### What to study from Shepherd

- live-supervisor behavior and intervention taxonomy;
- effect-stream-driven supervision;
- managed-vs-unmanaged experiment design;
- steer/revert semantics and intervention budgets;
- reversible execution and counterfactual experiment patterns.

### What not to inherit blindly

- framework ownership of the user's execution flow;
- requirement to express the workload as a Shepherd task;
- sandbox/workspace ownership as a prerequisite for basic supervision.

Shepherd is a **behavioral and experimental reference**, not Spotter's runtime substrate.

---

## 13. NVIDIA NeMo Relay — interception/runtime plumbing without Navigator behavior

Repository:

https://github.com/NVIDIA/NeMo-Relay

NeMo Relay is the strongest reference for generic agent-runtime interception and middleware plumbing. It provides scopes, lifecycle events, managed LLM/tool boundaries, middleware, plugins, event subscribers, trajectory/observability formats, host integration, diagnostics, and multi-harness support.

Its value to Spotter is primarily engineering precedent:

```text
"Where should observation/interception boundaries live,
and how do you operate them reliably across agent runtimes?"
```

Relay deliberately does not define Spotter's higher-level question:

```text
"What should an independent second model notice, predict,
and say to Main — and when should it stay silent?"
```

Its public Codex support also differs from Spotter's intended low-latency path: the integration is centered on host hooks and provider/gateway plumbing, while Spotter wants direct Codex App Server observation/control for `turn/steer` and `turn/interrupt`.

### What to study from NeMo Relay

- interceptor and middleware boundary design;
- normalized event/trajectory representations;
- transparent wrapper and host-integration patterns;
- health, diagnostics, rollback, trust, and lifecycle patterns;
- ATOF / ATIF / OpenTelemetry conventions;
- adapter patterns for multiple agent harnesses.

### Dependency decision

**Spotter does not use NeMo Relay as a dependency.**

A required dependency would add another compatibility/lifecycle boundary without removing Spotter's hardest work: App Server ownership, live Navigator state, speculative reasoning, intervention relevance, and low-latency steering. Relay can also modify provider/host configuration as part of its integrations, while Spotter's stronger product requirement is that native Codex behavior and latency remain as unchanged as possible.

Treat Relay as an **implementation reference only** unless a future, measured use case justifies integration.

---

## 14. OpenClaw Codex supervision — App Server control without an independent pair supervisor

Repository and useful implementation references:

- https://github.com/openclaw/openclaw
- https://github.com/openclaw/openclaw/blob/main/docs/plugins/codex-harness-runtime.md
- https://github.com/openclaw/openclaw/blob/main/extensions/codex/src/supervision-tools.ts

OpenClaw is the strongest public implementation reference for the **Codex App Server side** of Spotter.

Its Codex harness leaves the native model loop, thread resume, tool continuation, and compaction with Codex while OpenClaw builds routing, tools, approvals, transcript mirroring, and controls around that boundary.

The supervision implementation exposes primitives equivalent to:

```text
list sessions
read session
send / start / steer
interrupt
```

That makes the App Server observation/control mechanism itself much less speculative. Spotter can study real code for:

- App Server process and connection management;
- stdio / Unix / WebSocket endpoint handling;
- thread discovery and source kinds;
- active-thread ownership and race conditions;
- security checks around remote endpoints;
- steer and interrupt semantics;
- separation between Codex-owned and adapter-owned tool surfaces;
- conservative behavior when ownership is uncertain.

The difference is purpose. OpenClaw supervision exists to make Codex a richer OpenClaw harness/backend and expose Codex sessions to OpenClaw control. It does not implement a persistent autonomous second model whose sole job is to pair with Main.

Spotter's target remains:

```text
                           ┌──────────────────┐
                           │ Spotter Navigator│
                           │ independent model│
                           └───────▲─────┬────┘
                                   │     │
                             observe     │ reason ahead
                                   │     │
User → native Codex → App Server ──┘     │
                       ▲                 │
                       └──── steer ──────┘
```

OpenClaw is therefore an **App Server integration/control reference**, not the product Spotter is trying to build.

---

# Implementation strategy implied by these references

The default rule is:

> **Study and reuse proven patterns before adopting dependencies.**

| Area | Best reference | Spotter action |
| --- | --- | --- |
| supervisor behavior | Shepherd / Wink | borrow intervention and silence patterns |
| reversible/counterfactual execution | Shepherd | compare with existing snapshot/fork/replay machinery |
| evidence-conditioned action gating | ECLoop | study condition compilation, state tracking, and delay semantics |
| generic interception/middleware | NeMo Relay | study boundaries and event design |
| host install/lifecycle/diagnostics | NeMo Relay | borrow proven patterns selectively |
| normalized trajectory/telemetry | NeMo Relay / HarnessFix | interoperate where useful; avoid unnecessary format invention |
| Codex App Server client | OpenClaw | study connection and request/event handling |
| thread ownership / races | OpenClaw | incorporate known failure modes early |
| steer / interrupt control | OpenClaw | study control semantics and edge cases |
| autonomous pair supervision | no complete precedent identified | Spotter core |
| speculative async supervision | no complete precedent identified | Spotter core |
| zero-workflow-change native Codex UX | no complete precedent identified | Spotter product requirement |

The deliberately small direct implementation path remains:

```text
native Codex
    ↕
Codex App Server
    ↕
CodexAdapter
    ↕
spotterd
    ├─ trajectory state
    ├─ Navigator model
    ├─ speculative lookahead
    ├─ intervention cache
    └─ steer / interrupt controller
```

Do not introduce Shepherd, NeMo Relay, OpenClaw, or ECLoop as required runtime dependencies unless a concrete measured benefit later justifies the coupling.

---

# Remaining gap

These references narrow Spotter's remaining hypothesis substantially:

- Wink and Shepherd show that runtime supervision and selective intervention are meaningful directions.
- ECLoop shows that execution-time evidence gating can materially improve SWE-agent success and reduce waste.
- NeMo Relay shows that interception/middleware/observability plumbing can be engineered cleanly around existing agent runtimes.
- OpenClaw shows that Codex App Server can serve as a serious native observation/control surface.

What remains unimplemented as a complete product pattern is:

> **Attach transparently to native Codex, keep an independent Navigator reasoning alongside Main, predict and evaluate likely near-future decisions while Main continues, and inject only timely interventions into the active turn.**

The latency distinction matters:

```text
Reactive reviewer

Main action
   ↓
start supervisor inference
   ↓
wait / arrive late
   ↓
allow / steer
```

versus:

```text
Speculative supervisor

Main        ─────────────────────────────▶

Navigator      predict / evaluate ───────▶
                 in parallel

Main reaches likely decision boundary
                 ↓
          precomputed judgment
                 ↓
            steer if needed
```

That leads directly into the research question tracked in [Research](research.md):

> **Can runtime supervision be moved off the coding agent's critical path through speculative, asynchronous lookahead?**
