# Roadmap

Spotter should start narrow and earn complexity through evaluation.

The first objective is not to build a complete self-healing agent runtime. It is to prove that a separate observer can improve coding-agent trajectories **without becoming a noisy source of latency and false positives**.

## Phase 0 — Research baseline

Status: **current**

Deliverables:

- project concept and terminology
- Codex runtime capability inventory
- related-work map
- intervention taxonomy
- initial evaluation design

Exit criterion:

- a stable, testable definition of Spotter's MVP

## Phase 1 — Passive Spotter

Build observation without intervention.

### Scope

- connect to Codex runtime events
- normalize events into a simple Trace IR
- run a separately configured reviewer model
- record candidate trajectory problems
- never modify Main behavior

### Initial failure classes

Start with a small set that is both useful and observable:

1. specification / scope drift
2. repeated low-information exploration
3. repeated tool-failure loop
4. consequential unverified assumption
5. validation missing after meaningful edits

### Output

For each detected event, record:

```text
trajectory step
candidate failure class
reviewer verdict
confidence
proposed intervention
supporting evidence
```

### Goal

Measure reviewer quality before allowing it to interfere with coding.

## Phase 2 — Nudge mode

Enable only soft intervention.

### Runtime actions

- `CONTINUE`
- `VERIFY`
- `NUDGE`

### Principles

- interventions are concise
- reviewer does not provide a full alternate implementation
- Main remains autonomous
- disagreement should produce a verification request where possible

### Metrics

- recovery rate
- ignored intervention rate
- success-rate delta
- added token cost
- added wall-clock latency
- intervention harm rate

## Phase 3 — Deterministic pre-action gates

Add synchronous blocking only for constraints that can be checked reliably without semantic LLM judgment.

Examples:

- forbidden dependency changes
- forbidden paths
- destructive command patterns
- external-write boundaries
- explicit user restrictions

### Runtime action

- `BLOCK`

### Goal

Demonstrate that strong intervention can be both fast and high precision when backed by executable constraints.

## Phase 4 — Event-driven semantic review

Replace naive periodic review with a two-stage system.

```text
runtime event
    ↓
cheap signal / verifier
    ↓
possible issue?
    ├─ no → continue
    └─ yes
         ↓
    Pair Reviewer
```

Potential signals:

- repeated equivalent actions
- failure streaks
- sudden scope growth
- stale hypothesis reuse
- prolonged edit-without-validation windows

The signal layer generates hypotheses; it does not make semantic verdicts.

## Phase 5 — Interrupt and recovery

Add strong runtime control for high-confidence trajectory failure.

### Runtime actions

- `INTERRUPT`
- `RESTART`

### Required supporting features

- last-sound-step tracking
- checkpoint metadata
- external side-effect ledger
- explicit retained-state selection for restart

### Restart payload

A fresh Main should receive only:

- original user goal
- explicit constraints
- verified evidence
- current repository state
- intentionally preserved diff/artifacts

It should not inherit the full failed reasoning history by default.

## Phase 6 — Evidence dependency state

Introduce lightweight claim/evidence relationships.

```text
Evidence
   ↓ supports
Hypothesis
   ↓ motivates
Plan / Action
```

When evidence is invalidated, dependent hypotheses and pending plans become stale.

This should begin as an in-memory structure, not a graph database.

## Phase 7 — Experience and adaptation

Only after intervention quality is measurable, allow Spotter to learn from previous runs.

Potential directions:

- model-specific failure profiles
- repository-specific intervention patterns
- retrieval of previous productive / harmful interventions
- offline learning of intervention policy

Do not introduce adaptive behavior until there is a reliable evaluation harness capable of detecting reviewer regressions.

---

# Evaluation plan

## Baselines

Compare at least:

```text
A. Vanilla Codex

B. Periodic reviewer
   async, fixed interval, nudge only

C. Event-driven Spotter
   cheap trigger + semantic reviewer

D. Spotter + Gate
   event-driven + deterministic PreToolUse blocking

E. Full Spotter
   gate + nudge + interrupt/restart
```

## Model matrix

One of Spotter's central hypotheses is that reviewer diversity matters.

Evaluate:

```text
Main A + Reviewer A
Main A + Reviewer B
Main A + smaller specialized reviewer
```

Keep reviewer context isolated even in the same-model condition.

## Primary metrics

### Outcome

- task success / resolution
- regression rate

### Efficiency

- total tokens
- tool calls
- elapsed time
- repeated-action count

### Intervention quality

- interventions per run
- intervention precision
- recovery after intervention
- reviewer false-positive rate
- intervention harm rate

### Timing

- first-error localization accuracy
- steps between first meaningful deviation and intervention
- wasted tool calls before intervention

### Recovery

- success after nudge
- success after interrupt
- success after restart
- useful work retained across restart

## Counterfactual evaluation

Where runtime replay is practical, checkpoint a shared trajectory prefix and branch it:

```text
same prefix
    │
    ├─ continue normally
    ├─ VERIFY / NUDGE
    ├─ BLOCK
    └─ INTERRUPT / RESTART
```

This allows measurement of whether the intervention itself improved the outcome rather than merely correlating with a later success.

## False-positive budget

Strong interventions should have explicit budgets.

A plausible initial philosophy:

- `VERIFY`: tolerate moderate uncertainty
- `NUDGE`: conservative
- `BLOCK`: near-deterministic only
- `INTERRUPT`: very high confidence
- `RESTART`: extremely rare

Exact thresholds should be learned from evaluation rather than hard-coded from intuition.

---

# MVP definition

A useful first public version of Spotter does **not** need rollback, learned policies, or a graph engine.

It needs to demonstrate this loop reliably:

```text
1. Observe an active Codex task.
2. Keep an independent compact view of goal, constraints, hypotheses, and progress.
3. Detect one of a small set of trajectory problems.
4. Ask a separately configured model to verify the suspected problem.
5. Intervene with VERIFY/NUDGE or deterministic BLOCK.
6. Measure whether the intervention helped or harmed.
```

If this loop works, the more ambitious runtime-control features have a foundation worth building on.
