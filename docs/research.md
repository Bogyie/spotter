# Research

Spotter is not based on a single paper. It combines ideas from several recent lines of work on runtime supervision, process-level evaluation, online auditing, restart/recovery, verifiable reasoning, and harness diagnosis.

This document tracks the most relevant prior work and the concrete ideas Spotter intends to borrow.

## 1. Wink — asynchronous course correction for coding agents

**Wink: Recovering from Misbehaviors in Coding Agents**  
Rahul Nanda et al., 2026  
https://arxiv.org/abs/2602.17037

### Why it matters

Wink is the closest direct precedent for Spotter's core product idea.

It studies production coding-agent trajectories and identifies three broad misbehavior classes:

- specification drift
- reasoning problems
- tool-call failures

The paper reports that these misbehaviors appear in roughly 30% of observed production trajectories. Wink adds a lightweight asynchronous observer that watches the coding agent and injects targeted course-correction guidance when it detects a problem. The evaluation covers more than 10,000 real-world trajectories; the paper reports that about 90% of misbehaviors requiring a single intervention were recovered.

### What Spotter borrows

- persistent online observation rather than post-hoc review
- course correction during the active coding trajectory
- a concrete taxonomy of coding-agent misbehavior
- the principle that the observer should be lightweight and mostly asynchronous

### Where Spotter extends it

Wink primarily nudges after observing trajectory steps. Spotter additionally explores:

- pre-action blocking for deterministic violations
- stronger interruption/restart actions
- independent claim/evidence state
- action-conditioned intervention choice
- explicit handling of external side effects

---

## 2. SWE-PRM — process reward models for SWE trajectory correction

**When Agents go Astray: Course-Correcting SWE Agents with PRMs**  
Shubham Gandhi et al., 2025  
https://arxiv.org/abs/2509.02360

### Why it matters

SWE-PRM treats software-engineering agent failures as trajectory-level problems rather than only bad final outputs. It uses inference-time process supervision to detect redundant exploration, loops, premature stopping, and other execution inefficiencies, then feeds interpretable feedback back into the running agent.

On SWE-bench Verified, the paper reports an improvement from 40.0% to 50.6% resolution for its strongest closed-source PRM configuration.

### What Spotter borrows

- process-level failure taxonomy
- inference-time correction without modifying the main model policy
- lightweight, interpretable feedback rather than verbose alternate solutions
- evaluation of trajectory efficiency as well as final success

### Design lesson

Taxonomy-guided feedback performs better than unconstrained reviewer chatter. Spotter's reviewer output should therefore be structured around a small set of failure hypotheses and control actions.

---

## 3. AgentForesight — alarm at the earliest decisive error

**AgentForesight: Online Auditing for Early Failure Prediction in Multi-Agent Systems**  
Boxuan Zhang et al., 2026  
https://arxiv.org/abs/2605.08715

### Why it matters

AgentForesight reframes failure attribution as **online auditing**. At each step, the auditor sees only the current trajectory prefix and must decide whether to continue or alarm at the earliest decisive error, without access to future steps.

Its benchmark includes coding, math, and general agentic trajectories.

### What Spotter borrows

- prefix-only judgment: the reviewer must not depend on hindsight
- first-error localization as a separate objective
- the distinction between detecting failure eventually and detecting it early enough to intervene

### Design lesson

A useful Spotter should measure **time-to-intervention** and **wasted actions before intervention**, not only whether it eventually spotted a problem.

---

## 4. FailFast-RestartSmart — abort bad trajectories and restart cleanly

**Fail-Fast, Restart-Smart: Early Failure Prediction and Restart for SWE Agentic Tasks**  
Chenyu Wang et al., 2026  
https://arxiv.org/abs/2608.03222

### Why it matters

FailFast-RestartSmart uses a small monitor to predict failure from observable trajectory prefixes. When risk becomes high enough, it terminates the current run and starts a fresh rollout without the old prompt history, while optionally preserving useful repository edits as an overlay.

The paper reports 14.6%–20.4% execution-token savings at a 5% false-positive target across transferred policies, and a resolution increase from 66.6% to 71.8% for one studied policy under a more aggressive restart setting.

### What Spotter borrows

- `RESTART` as a control primitive distinct from a nudge
- recognition that a reasoning context can become contaminated
- preservation of useful code state without preserving the entire failed reasoning history
- explicit false-positive budgets for strong intervention

### Design lesson

Sometimes telling an agent to "try again" inside the same context is weaker than discarding the trajectory and starting from verified state.

---

## 5. Calibration Is Not Control — predict the value of intervention

**Calibration Is Not Control: Why LLM-Agent Oversight Needs Intervention**  
Chubin Zhang et al., 2026  
https://arxiv.org/abs/2606.21399

### Why it matters

The paper argues that scalar failure probability is the wrong control target for runtime oversight.

Two trajectory prefixes can have the same estimated chance of failure while requiring different actions: one may be highly recoverable through intervention, while another may not benefit from intervention at all.

It proposes **intervention advantage** — the expected utility difference between intervening and continuing — as the relevant decision quantity, and uses same-prefix counterfactual branching to evaluate it.

### What Spotter borrows

- ask "will intervention help?" rather than only "is this risky?"
- intervention should be action-conditioned (`continue`, `nudge`, `interrupt`, etc.)
- same-prefix branching as an evaluation technique for intervention policy

### Design lesson

A reviewer that accurately predicts failure can still be a bad controller.

Spotter should optimize for **useful intervention**, not maximum detection rate.

---

## 6. interwhen — verifier-first runtime reasoning supervision

**interwhen: A Generalizable Framework for Verifiable Reasoning with Test-time Monitors**  
Vishak K. Bhat et al., 2026  
https://arxiv.org/abs/2602.11202  
Code: https://github.com/microsoft/interwhen

### Why it matters

interwhen verifies partial reasoning during generation rather than only testing the final answer. It separates verifiable properties from general free-form reasoning and checks them with external or self-verifiers only when needed.

### What Spotter borrows

- identify which runtime properties can be made executable
- verify deterministic constraints outside the reviewer LLM
- avoid forcing the whole reasoning process into a rigid step schema
- steer only when a verifiable invariant is actually violated

### Design lesson

Do not spend an expensive semantic reviewer call on facts that can be checked by code.

---

## 7. Grounded Continuation — claim/evidence dependency tracking

**Grounded Continuation: A Linear-Time Runtime Verifier for LLM Conversations**  
Qisong He, Yi Dong, Xiaowei Huang, 2026  
https://arxiv.org/abs/2605.14175

### Why it matters

Grounded Continuation maintains an explicit graph that records which claims depend on which evidence. When evidence is retracted, invalidation propagates to conclusions that depended on it.

The original work targets long conversations, but the mechanism maps naturally to coding-agent trajectories.

### What Spotter borrows

- evidence-backed hypotheses instead of copying Main's conclusions as truth
- explicit stale-premise detection
- transitive invalidation of dependent plans/actions

### Coding example

```text
E1: timeout appears under high concurrency
        ↓
H1: Redis pool exhaustion is the cause
        ↓
P1: modify Redis pool configuration

new evidence:
E2: stack trace points to upstream HTTP client

→ H1 becomes stale
→ P1 requires revalidation
```

---

## 8. AgentProcessBench — neutral exploration is not failure

**AgentProcessBench: Diagnosing Step-Level Process Quality in Tool-Using Agents**  
Shengda Fan et al., 2026  
https://arxiv.org/abs/2603.14465  
Code/data: https://github.com/RUCBM/AgentProcessBench

### Why it matters

AgentProcessBench provides human-labeled step-level annotations for realistic tool-using trajectories and explicitly uses a ternary label scheme to distinguish useful, neutral/exploratory, and erroneous behavior.

A key finding is that current models struggle to distinguish neutral exploration from actual error.

### What Spotter borrows

- `NEUTRAL` / uncertainty must exist in the reviewer worldview
- repeated reads or exploration are not automatically loops
- early intervention policy must be conservative around ambiguous exploratory steps

### Design lesson

"The agent is taking a while" is not itself a failure signal.

---

## 9. HarnessFix — normalize traces and diagnose the harness too

**From Failed Trajectories to Reliable LLM Agents: Diagnosing and Repairing Harness Flaws**  
Mengzhuo Chen et al., 2026  
https://arxiv.org/abs/2606.06324

### Why it matters

HarnessFix compiles raw execution traces and harness code into a Harness-aware Trace Intermediate Representation (HTIR), then attributes recurring failures to specific trajectory steps and harness layers.

### What Spotter borrows

- normalize raw runtime streams into a backend-independent Trace IR
- track provenance between runtime behavior and harness configuration
- distinguish Main failure from Spotter/harness failure

### Future direction

Repeated Spotter diagnoses may eventually reveal that the right fix is not another runtime intervention, but a change to the agent harness itself.

---

## 10. Refute-or-Promote — falsification and empirical gates

**Refute-or-Promote: An Adversarial Stage-Gated Multi-Agent Review Methodology for High-Precision LLM-Assisted Defect Discovery**  
Abhinav Agarwal, 2026  
https://arxiv.org/abs/2604.19049

### Why it matters

This work uses adversarial reviewers to try to kill candidate findings before they are promoted. It highlights correlated reviewer errors and reports an instructive case where multiple reviewers agreed on a nonexistent vulnerability that was eliminated by a single empirical test.

### What Spotter borrows

- Spotter should try to **falsify** a consequential Main hypothesis rather than produce a competing full solution
- cross-model review may reduce correlated blind spots
- empirical evidence outranks model consensus

### Design lesson

Ten models agreeing is weaker evidence than one decisive test.

---

## Synthesis

The core Spotter design can be seen as the intersection of these ideas:

```text
Wink
  asynchronous coding-agent observer
          +
SWE-PRM
  trajectory failure taxonomy
          +
AgentForesight
  earliest-error online auditing
          +
FailFast-RestartSmart
  interrupt + clean restart
          +
Calibration Is Not Control
  intervention-value decision policy
          +
interwhen
  deterministic verifier-first design
          +
Grounded Continuation
  claim/evidence dependency state
          +
AgentProcessBench
  neutral exploration awareness
          +
HarnessFix
  Trace IR / harness attribution
          +
Refute-or-Promote
  falsification + empirical gates
```

## Research questions for Spotter

### RQ1 — Intervention timing

How early should a coding-agent trajectory be reviewed?

Compare:

- periodic asynchronous review
- event-triggered review
- synchronous pre-action gate

### RQ2 — Intervention policy

When is a `NUDGE` better than `INTERRUPT`? When is `RESTART` better than continuing inside the same context?

### RQ3 — Model diversity

Does a different reviewer model reduce correlated failures compared with same-model review?

Compare:

- same model / isolated context
- different model / isolated context
- small specialized reviewer

### RQ4 — Reviewer harm

How often does Spotter interrupt a trajectory that would have succeeded on its own?

This should be treated as a first-class metric, not a footnote.

### RQ5 — Deterministic vs semantic supervision

What fraction of useful interventions can be triggered by cheap executable checks before an LLM reviewer is needed?

## Evaluation metrics

Spotter should report more than final task success:

- task resolution rate
- token usage
- tool-call count
- wall-clock latency
- interventions per run
- intervention precision
- intervention harm rate
- recovery rate after intervention
- time-to-intervention
- wasted actions before intervention
- ignored-intervention rate
- restart recovery rate

The core benchmark comparison should eventually include:

```text
A. Vanilla coding agent
B. Periodic async reviewer (Wink-style)
C. Event-triggered Spotter
D. Spotter + pre-action deterministic gates
E. Spotter + interrupt/restart
```

## Notes on research status

This is a fast-moving area. Several papers listed here are 2026 preprints rather than established production standards. Spotter should treat them as design evidence and experimental hypotheses, not as proof that every mechanism will generalize to Codex or every repository/task class.

The project should therefore keep its architecture modular enough to remove mechanisms that do not survive empirical evaluation.
