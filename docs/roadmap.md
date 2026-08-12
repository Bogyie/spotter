# Roadmap

> Spotter's roadmap is organized by **named outcomes**, not phase numbers.  
> Each stage combines implementation with the evidence required to justify moving forward.

---

## 30-second summary

The roadmap is:

```text
Runtime
  ↓
Observe
  ↓
Detect
  ↓
Intervene
  ↓
Recover
  ↓
Harden
```

GitHub Milestones are the source of truth for **which stage owns an issue's completion**. This document owns the meaning and evidence gate of each stage.

The stages are not release versions and they are not strict single-threaded sprints. Work can overlap when dependencies allow. The names answer a more useful question than `P4` or `E2`: **what capability are we trying to make trustworthy next?**

The current blocker is [#78](https://github.com/spotter-agent/spotter/issues/78): prove that ordinary Codex and Spotter can share the same external App Server, observe the same thread/turn, and steer the real active turn.

If that PoC fails, revise the runtime architecture before building further on the App Server assumption.

---

## Stage map

| Stage | Product outcome | Evidence gate |
| --- | --- | --- |
| **Runtime** | Spotter has a viable standalone runtime/control boundary | same real Codex thread/turn is observable and steerable; normal `codex` UX remains viable |
| **Observe** | App Server events and live state replace broad Hook-era observation | important failures are observable early enough; missing evidence is explicit |
| **Detect** | cheap signals target semantic review | precision, miss rate, and detection delay are defensible |
| **Intervene** | `VERIFY` / `NUDGE` reach the correct active turn | intervention helps more than it harms and Main can reject bad supervision |
| **Recover** | Spotter can interrupt or restart a contaminated trajectory | strong actions meet stricter harm/precision budgets and external effects remain visible |
| **Harden** | installation, upgrades, retention, recovery, and long-term operation are predictable | lifecycle failures are diagnosable, migrations are explicit, stale resources are bounded |

Evaluation is deliberately **inside** the roadmap rather than a separate `E0–E5` track. A mechanism is not considered ready merely because its code exists.

---

# Runtime

## Outcome

Establish the smallest standalone boundary that can own live supervision state and control a real Codex session.

The architectural target remains:

```text
Codex TUI
    ↓
External Codex App Server
    ↕ events / steer / interrupt
spotterd
    ↓
PreToolUse Hook only where synchronous deterministic enforcement is required
```

## Current gate — App Server viability

[#78](https://github.com/spotter-agent/spotter/issues/78) must answer:

- can ordinary `codex` reuse or attach to an externally reachable App Server?
- can Spotter attach as another client to that same server?
- do both clients see the same thread and active turn?
- does `turn/steer` reach the actual user-visible turn?
- are concurrent sessions distinguishable?
- can embedded/disconnected modes be surfaced as degraded instead of silently healthy?

Do not treat the App Server architecture as settled until this is proven end to end.

## After the gate

Build the standalone foundation around the proven integration strategy:

- `spotterd` lifecycle and control RPC;
- App Server client and capability negotiation;
- thread / turn / runtime-attachment identity;
- bounded Hook ↔ daemon IPC for deterministic enforcement;
- multi-thread isolation;
- daemon reconnect/recovery;
- setup/status/doctor/teardown behavior sufficient for ordinary use.

[Issue #31](https://github.com/spotter-agent/spotter/issues/31) tracks moving independent live supervision state into `spotterd` once the boundary is proven.

## Exit condition

Runtime is credible when:

- normal use does not require manually starting Spotter before every Codex session;
- one long-lived runtime owns live state for multiple threads;
- the real active Codex turn can be observed and controlled;
- a dead/degraded Spotter does not break the coding session and is not reported as healthy;
- setup and teardown have explicit ownership.

---

# Observe

## Outcome

Make the App Server event stream the primary observation source and maintain trustworthy live state without rebuilding the world from journals on every Hook.

Target flow:

```text
App Server event
      ↓
Trace IR normalization
      ↓
live ThreadState
      ├─ goal / constraints
      ├─ hypotheses / evidence
      ├─ touched scope
      ├─ validation / failures
      └─ intervention history
      ↓
durable journal
```

The journal remains durable history and recovery input; it is not the normal hot-state database.

## Work

- move lifecycle/user-message/tool/result/diff observation to App Server events;
- remove `SessionStart`, `UserPromptSubmit`, and `PostToolUse` Hooks where coverage is proven;
- retain `PreToolUse` only for bounded synchronous enforcement where required;
- implement and hydrate live state ([#31](https://github.com/spotter-agent/spotter/issues/31));
- record runtime cost/timing/provenance ([#33](https://github.com/spotter-agent/spotter/issues/33));
- measure the actual observability ceiling ([#37](https://github.com/spotter-agent/spotter/issues/37)).

## Evidence gate

The important number is not raw event count. It is:

> **What fraction of consequential failures become visible early enough that Spotter could still change the trajectory?**

The existing Hook-era outcome visibility figure is only a baseline. Re-measure after the observation migration.

## Exit condition

- App Server is the primary trajectory source;
- goal/constraint/tool-result coverage is measured;
- missing information is represented as unknown rather than inferred;
- the observability ceiling is high enough to justify semantic detection work on this surface.

---

# Detect

## Outcome

Replace periodic “look every N steps” review with cheap candidate detection followed by semantic judgment only when useful.

Target flow:

```text
runtime event
    ↓
cheap signal
    ↓
possible issue?
    ├─ no → done
    └─ yes
         ↓
     reviewer
```

A signal is a hypothesis, not a verdict.

## Work

- event-driven candidate signals ([#28](https://github.com/spotter-agent/spotter/issues/28));
- repeated-action / failure / scope-growth / stale-premise / validation-gap signals;
- reviewer inputs grounded in constraints, evidence, candidate hypothesis, prior interventions, and available probes;
- precision and miss-rate measurement ([#38](https://github.com/spotter-agent/spotter/issues/38));
- detection delay and wasted-action measurement ([#24](https://github.com/spotter-agent/spotter/issues/24)).

## Evidence gate

Report together:

- precision / false-positive rate;
- miss rate or recall estimate;
- abstention / blind-spot rate;
- detection delay / wasted actions;
- reviewer call and token cost.

A detector that almost never fires does not earn trust through precision alone.

## Exit condition

- reviewer calls are primarily evidence-triggered rather than cadence-triggered;
- precision and misses are both measured;
- detections arrive early enough to plausibly save work;
- model cost is visible alongside benefit.

---

# Intervene

## Outcome

Deliver soft supervision to the correct live turn without confusing Spotter guidance with user intent.

## Work

- live `VERIFY` / `NUDGE` via `turn/steer` ([#22](https://github.com/spotter-agent/spotter/issues/22));
- target-turn freshness and stale/discard policy;
- duplicate-delivery prevention and delivery provenance;
- human-visible supervision provenance and feedback ([#45](https://github.com/spotter-agent/spotter/issues/45));
- wrong-nudge susceptibility ([#23](https://github.com/spotter-agent/spotter/issues/23));
- intervention advantage, harm, recovery, and ignored rate ([#34](https://github.com/spotter-agent/spotter/issues/34)).

## Evidence gate

The central comparison is same-prefix when possible:

```text
control: continue
vs
intervention: VERIFY / NUDGE
```

Classify:

```text
intervention better
control better (harm)
tied
infrastructure failure / inconclusive
```

Also test the opposite failure direction: Main must be able to reject plausible but wrong Spotter guidance when evidence contradicts it.

## Exit condition

- guidance reaches the intended active turn exactly once;
- stale guidance does not leak into unrelated turns;
- intervention benefit and harm are measured against a mechanically scored or otherwise defensible outcome;
- wrong-nudge compliance is low enough for the intended policy;
- live intervention remains conservative when evidence is insufficient.

---

# Recover

## Outcome

Add stronger control only after soft intervention is understood.

## Work

- action reversibility and external side-effect tracking ([#30](https://github.com/spotter-agent/spotter/issues/30));
- `turn/interrupt` with shadow-first policy;
- restart from a deliberately small verified-state payload ([#26](https://github.com/spotter-agent/spotter/issues/26));
- checkpoint/lineage tracking;
- explicit disclosure of external effects that recovery cannot undo.

A restart payload should remain intentionally small:

```text
user goal
explicit constraints
verified evidence
current repository/checkpoint state
explicitly retained artifacts
```

## Evidence gate

Stronger actions require stronger evidence than `VERIFY` or `NUDGE` because false positives cost more.

Measure:

- would-interrupt precision before active interruption;
- harm and recovery by action type;
- retained useful work after restart;
- unreverted external effects.

## Exit condition

- interruption and restart have explicit evidence budgets;
- strong actions are auditable and attributable to a specific turn/state;
- restart never implies external rollback that did not happen.

---

# Harden

## Outcome

Make months of use boring: install, upgrade, recover, clean up, and reinstall without hidden state or mystery failures.

## Work

- persisted schema/version contracts and migrations ([#47](https://github.com/spotter-agent/spotter/issues/47));
- package/setup/update/teardown compatibility;
- daemon/App Server reconnect and crash recovery;
- journal/snapshot/worktree/log retention;
- repository-aware purge;
- uninstall/reinstall fixtures;
- capability negotiation across Codex updates;
- multi-agent integration lifecycle when another adapter becomes real.

## Operational validation

Once enough of the product exists, run the real-session configuration comparison in [#36](https://github.com/spotter-agent/spotter/issues/36). That experiment asks a different question from same-prefix intervention tests:

> **Does keeping Spotter enabled during real work improve outcomes enough to justify its total cost and operational complexity?**

## Exit condition

- upgrades and supported migrations are tested;
- stale resources are discoverable and bounded;
- `status` / `doctor` explain partial failures;
- uninstall and integration teardown have predictable ownership;
- operational A/B results can be reported with uncertainty rather than anecdotes.

---

# Cross-cutting evidence infrastructure

Some work supports several stages rather than belonging to one stage. Its Milestone indicates the stage by which its current completion gate is needed, not the only area it benefits.

## Mechanically scored tasks

[#21](https://github.com/spotter-agent/spotter/issues/21) builds a reproducible task set with objective checks. It is useful as early as possible because Detect, Intervene, and later configuration comparisons all benefit from judgeable outcomes.

## Replay/fork fidelity

[#42](https://github.com/spotter-agent/spotter/issues/42) measures the noise floor and environmental drift of the fork instrument. Do not make causal claims from fork deltas before the instrument itself is measured.

## Cost and timing

[#33](https://github.com/spotter-agent/spotter/issues/33) makes Main cost, Spotter cost, latency, and outcome visible. Cost should be reported next to claimed benefit, not in a separate afterthought.

---

# Issue selection

Roadmap stage, work urgency, size, and problem domain are deliberately separate dimensions. Use GitHub's native metadata rather than encoding them in labels:

- **Milestone** — the roadmap stage that owns the issue's completion/evidence gate;
- **Priority** — `Urgent`, `High`, `Medium`, or `Low` for current sequencing pressure;
- **Effort** — `XS` through `XL` for change surface, validation difficulty, and uncertainty, not elapsed-time prediction;
- **Area** — the stable primary product/problem domain;
- **Dependencies** — actual `blocked by` / `blocking` relationships.

Keep `Urgent` rare. If most open work is `High`, priority has stopped helping choose the next issue. A dependency should mean the downstream issue cannot meaningfully complete without the blocker, not merely that the issues are related.

---

# Revised MVP

A useful Spotter does not require a graph database, learned policy, or automatic external rollback engine.

It must prove this loop:

```text
1. Observe the real active coding-agent trajectory.
2. Maintain independent live state for goal, constraints, hypotheses, evidence, and progress.
3. Generate cheap candidate failure signals.
4. Use semantic review only where deterministic evidence is insufficient.
5. Deliver VERIFY/NUDGE to the correct live turn or deterministic BLOCK before execution.
6. Record timing, cost, freshness, delivery, and outcome.
7. Demonstrate that intervention helps more than it harms.
```

Everything after that should make the loop safer, stronger, or easier to operate—not obscure whether it works.
