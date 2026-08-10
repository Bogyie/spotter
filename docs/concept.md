# Concept

## What is Spotter?

Spotter is a runtime supervision layer for coding agents.

A primary agent remains responsible for doing the work. A second, independently configured model observes the primary agent's execution trajectory and intervenes only when intervention is likely to improve the outcome.

The metaphor is intentional: a good spotter does not perform the lift. They watch closely, avoid unnecessary interference, and step in before a recoverable mistake becomes a failure.

> **The user defines the goal. Spotter reviews the path.**

## Problem

Coding agents increasingly operate over long trajectories:

```text
understand → inspect → hypothesize → edit → run → observe → revise → validate
```

Many failures are not single incorrect outputs. They are **trajectory failures**.

A weak assumption can become a plan. The plan can trigger unnecessary edits. Those edits can create secondary failures. The agent then spends more time compensating for consequences of the original mistake. A final-diff reviewer may detect the problem, but only after most of the cost has already been incurred.

Common examples include:

- drifting away from the requested scope
- treating an unverified hypothesis as fact
- continuing to use a premise after new evidence invalidates it
- repetitive exploration that produces little new information
- repeatedly retrying a failing tool strategy
- expanding a local fix into an unnecessary refactor
- forgetting a user constraint midway through a long run
- editing before gathering enough evidence
- making changes without validating the behavior they affect

Spotter aims to catch these failures **while the trajectory is still recoverable**.

## Core principles

### 1. Always observe, rarely interrupt

Silence is a valid and desirable Spotter action.

An observer that comments on every imperfect decision becomes another source of noise and latency. Spotter should intervene only when the expected value of intervention exceeds the expected value of letting the main agent continue.

### 2. Review the process, not only the artifact

Static review asks whether the code is correct.

Spotter also asks:

- Why is this file being edited?
- What evidence supports the current hypothesis?
- Is this action still connected to the user's goal?
- Did the agent already invalidate the premise behind this plan?
- Is this exploration producing new information?
- Has the agent accumulated changes without validating them?

### 3. Prefer falsification over opinion

When Main and Spotter disagree, the preferred resolution is not a longer conversation between models.

Spotter should look for the cheapest discriminating evidence:

- run a focused test
- inspect the stack trace
- search the call sites
- check the type system or compiler
- inspect repository state
- query logs

The purpose of the reviewer is not to produce a competing answer. It is to expose fragile assumptions and move the trajectory toward evidence.

### 4. Deterministic facts should have deterministic checks

Known constraints do not need an LLM judge.

Examples:

- “Do not add dependencies” → inspect manifest diffs
- “Only modify this directory” → inspect affected paths
- “Tests must pass” → inspect test result state
- forbidden commands → match the pending tool call

LLM review should be reserved for semantic ambiguity: scope drift, unsupported reasoning, missed requirements, premature abstraction, and similar judgments.

### 5. Main and Spotter should fail differently

Spotter should be independently configurable and preferably use a different model from the main coding agent.

Its context should also preserve some independence. Main's conclusions must not automatically become Spotter's facts. A claim can be recorded as a hypothesis with supporting and contradicting evidence rather than copied into shared state as truth.

### 6. Intervention should be incremental

Spotter uses an escalation ladder:

```text
CONTINUE
   ↓
VERIFY
   ↓
NUDGE
   ↓
BLOCK
   ↓
INTERRUPT
   ↓
RESTART
```

The weakest sufficient intervention is preferred.

## Intervention semantics

### CONTINUE

No meaningful issue is detected, or an issue exists but intervention is unlikely to help. Spotter remains silent.

### VERIFY

A consequential decision depends on a weakly supported assumption. Spotter asks Main to gather a small amount of discriminating evidence before compounding the assumption.

### NUDGE

The trajectory shows early drift, inefficiency, or omission. Spotter injects a concise course correction without taking control of the task.

### BLOCK

A pending action clearly violates a known constraint or crosses a configured execution boundary. The action is stopped before execution.

### INTERRUPT

The current turn has entered a trajectory where continued execution is likely to compound waste or damage. The turn is stopped and Main must reassess instead of resuming the same plan unchanged.

### RESTART

The current reasoning context itself is considered contaminated by stale assumptions or cascading failures. A fresh rollout begins from the user's goal, verified evidence, current repository state, and explicitly retained artifacts.

This should be rare.

## Pair-aware Main

Runtime supervision changes the environment in which the primary agent operates. Main therefore needs a small execution contract.

It should understand that:

- Spotter signals are runtime supervision, not new user requirements.
- Spotter feedback should not be accepted blindly.
- disagreement should preferably be resolved with external evidence.
- a blocked action should not be retried through superficial reformulation.
- an interrupt requires reassessment before continuing.
- Main should not wait for Spotter approval during normal execution.

A useful rule is:

> **Pair feedback is a request to re-evaluate the path, not authority to redefine the goal.**

## Working state

Spotter needs more than a raw transcript, but the first implementation does not need a large knowledge-graph system.

A compact independent audit state is enough:

```text
Goal
Constraints
Current hypotheses
Evidence for / against
Open questions
Current action
Files touched
Validation state
Recent failures
Intervention history
```

The important relationship is between **claims and evidence**. If evidence is invalidated, downstream hypotheses and plans that depended on it should be marked stale and revalidated.

## What Spotter is not

Spotter is not:

- a second coding agent racing Main to solve the same task
- a static code-review bot
- a security-only guardrail
- a mandatory approval gate for every tool call
- a multi-agent debate system
- a mechanism for maximizing the amount of reasoning

Its purpose is to reduce **wasted progress**.

## Trajectory Engineering

Spotter is an experiment in a broader layer we call **Trajectory Engineering**.

- Prompt engineering shapes the instruction.
- Context engineering shapes what the model can see.
- Harness engineering shapes the execution environment, tools, and orchestration.
- **Trajectory engineering shapes what happens while the agent is already executing.**

Its concerns include observation, state tracking, verification, intervention timing, rollback, restart, and recovery.

The working hypothesis behind Spotter is simple:

> Better agents are not only agents that reason better. They are agents whose mistakes are detected early enough that they remain cheap to correct.
