# LLM-driven MAC Scheduling Agent (A Unified Agent Workforce)

An LLM-guided unified MAC scheduling architecture for 5G that targets:
- **URLLC latency and deadline reliability**
- **eMBB throughput and fairness**
- **Slice isolation and SLA adherence**

> Core principle: **Use LLMs for policy intelligence, not per-slot scheduling execution**.
NOTE:

It means:

* Per-slot scheduling (every 0.5–1 ms) is extremely time-critical.
* An LLM is too variable/slow for that hard real-time loop.
* So the LLM should not decide “which UE gets which PRBs this slot.
  
Instead:

* LLM works at a slower control loop (seconds/minutes),
* analyzes KPIs (URLLC misses, eMBB throughput, slice violations),
* proposes bounded policy changes (weights, slice min/max caps),
* and a deterministic scheduler applies those settings in real time.

---

## 1) Problem Statement

A 5G MAC scheduler must make strict real-time decisions while balancing conflicting goals:
- URLLC requires very low latency and low deadline-miss probability.
- eMBB requires high spectral efficiency and stable throughput.
- Multi-slice operation requires isolation and predictable SLA compliance.

A purely heuristic scheduler can be rigid under changing load and traffic mix. An LLM can improve adaptability by:
- interpreting operator intent,
- analyzing KPI trends,
- proposing bounded policy updates,
- and explaining trade-offs.

However, LLM inference variability and latency make it unsuitable for the hard real-time scheduling loop. This repository proposes a **hierarchical control architecture** where deterministic MAC execution remains in charge.

---

## 2) Recommended Development Strategy

For this project, the best approach is:

1. **Start with a Python simulation prototype** (fast iteration and risk reduction).
2. **Then integrate into an existing 5G RAN stack** (real-system validation).

If only one can be chosen at the beginning, choose **Python first** for 4–8 weeks.

### Why this sequence is best

- The highest uncertainty is in LLM policy quality and safety, not in implementing classic schedulers.
- Python enables rapid iteration on:
  - policy schema,
  - guardrails,
  - rollback logic,
  - KPI-driven arbitration.
- Directly jumping to RAN integration can hide policy flaws behind integration complexity and timing constraints.

---

## 3) Target Runtime Architecture (for Existing 5G RAN)

### 3.1 Layered Design

| Layer | Role | Timing Characteristics |
|---|---|---|
| **LLM Policy Agent** | Intent translation, KPI reasoning, policy proposals, explanation | Slow loop (seconds/minutes) |
| **Policy Coordinator + Safety Gate** | Validate, constrain, arbitrate objectives, approve/reject updates | Near-real-time control loop |
| **Deterministic MAC Scheduler** | UE selection, PRB allocation, HARQ-aware execution | Hard real-time / slot-level |

### 3.2 Key Separation Rule

- **LLM never directly allocates per-slot PRBs**.
- LLM only proposes bounded policy parameters.
- Real-time scheduler remains deterministic and deadline-safe.

---

## 4) Policy Contract Between LLM and Scheduler

The LLM outputs a constrained policy object (JSON/YAML), such as:

- per-slice minimum/maximum resource share,
- slice borrowing permissions,
- latency urgency weight bounds,
- proportional-fairness tuning bounds,
- update TTL/expiry,
- confidence and rationale.

### 4.1 Example (Conceptual)

```json
{
  "policy_version": "1.0",
  "effective_for_sec": 120,
  "slices": {
    "urllc": { "min_prb_pct": 25, "max_prb_pct": 50, "borrow": false },
    "embb":  { "min_prb_pct": 40, "max_prb_pct": 75, "borrow": true }
  },
  "weights": {
    "deadline_urgency": 0.9,
    "throughput_fairness": 0.6
  },
  "constraints": {
    "max_step_change_pct": 10,
    "rollback_on_kpi_breach": true
  },
  "rationale": "URLLC deadline misses rising in last 60s; temporary urgency increase.",
  "confidence": 0.78
}
```

---

## 5) Safety and Reliability Requirements (Mandatory)

Every proposed policy must pass deterministic checks before activation:

1. **Schema validation** (required fields, types, versioning).
2. **Range validation** (all tunables within operator-approved bounds).
3. **Feasibility checks** (sum constraints, conflicting slice rules, invalid borrow settings).
4. **Rate limiting** (max update frequency, max delta per update).
5. **TTL enforcement** (automatic expiry and reversion if not refreshed).
6. **Shadow/replay test** when possible before live application.
7. **Rollback triggers** on KPI breach (deadline miss, SLA violation spike, instability).
8. **Fail-safe fallback scheduler** always available.

### 5.1 Failure Handling

If LLM output is delayed, unavailable, or invalid:
- keep current safe policy until expiry window,
- then revert to known-good static profile,
- continue deterministic MAC operation without interruption.

---

## 6) Unified Scheduling Logic (Deterministic Core)

A practical deterministic hierarchy:

1. **Hard feasibility first**: transmission constraints, HARQ requirements, available grants.
2. **Slice contract enforcement**: floors/ceilings and explicit borrow rules.
3. **URLLC deadline urgency handling**: prioritize near-deadline packets.
4. **Residual optimization for eMBB**: proportional fairness / throughput gains.
5. **Stability controls**: dampening to avoid oscillation from frequent policy shifts.

This preserves real-time correctness while allowing policy-level adaptation.

---

## 7) Implementation Path

## Phase A — Python Digital Twin (4–8 weeks)

Build and validate:
- traffic generators for URLLC + eMBB,
- slice models and SLA definitions,
- baseline deterministic scheduler,
- LLM policy loop,
- safety gate,
- KPI evaluator and rollback pipeline.

### Exit Criteria

- URLLC: improved tail latency and deadline-miss ratio vs baseline.
- eMBB: no unacceptable throughput collapse.
- Slicing: bounded SLA/isolation violations.
- Safety: zero unsafe policy activations in test campaigns.

## Phase B — Existing 5G RAN Integration (8–12+ weeks)

Integrate policy control path into real stack:
- keep scheduler deterministic,
- add policy parameter ingestion,
- wire KPI telemetry and control loop,
- enable staged rollout (shadow → canary → controlled live).

---

## 8) Recommended Open-Source Stack for First Integration

Primary recommendation:
- **OpenAirInterface (OAI) + near-RT control integration (e.g., O-RAN/FlexRIC style workflow)**

Why:
- broad research adoption,
- modifiable MAC implementation,
- suitable for controlled policy adaptation experiments.

Alternative:
- **srsRAN** can be easier for initial engineering flow depending on team familiarity.

Selection guidance:
- choose the stack your team can instrument deeply for scheduler KPIs, policy hooks, and rollback controls.

---

## 9) KPI Framework (What to Measure)

### URLLC KPIs
- p95/p99 one-way latency,
- deadline-miss ratio,
- jitter under overload.

### eMBB KPIs
- aggregate and per-UE goodput,
- fairness index,
- spectral efficiency trends.

### Slice Isolation KPIs
- SLA violation rate per slice,
- unauthorized borrow incidents,
- recovery time after contention spikes.

### Control-Loop KPIs
- policy acceptance/rejection rate,
- rollback frequency,
- oscillation score (excessive policy churn),
- LLM inference cost/latency vs net KPI gain.

---

## 10) What “Unified Agent Workforce” Means Here

“Unified” does **not** mean one unconstrained agent making every decision.
It means coordinated specialized roles with deterministic guardrails:

- **Intent Agent**: translates operator/business objectives to machine constraints.
- **KPI Analyst Agent**: summarizes trends and anomalies.
- **Policy Proposer Agent**: suggests bounded updates.
- **Safety Gate (deterministic)**: validates and arbitrates.
- **Execution Scheduler (deterministic)**: performs real-time MAC decisions.

This structure improves explainability, safety, and reproducibility.

---

## 11) Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| LLM hallucinated/unsafe policy | SLA instability | strict schema + hard bounds + reject invalid output |
| Excessive policy churn | oscillation / jitter | update cooldown, max step change, hysteresis |
| KPI misinterpretation | wrong optimization direction | deterministic KPI definitions and confidence gating |
| Inference latency or outage | stale control | TTL + fallback baseline policy |
| Objective conflict (URLLC vs eMBB) | uneven service quality | explicit arbitration priorities and overload policy |

---

## 12) Next Milestones for This Repository

1. Define policy schema (`/spec/policy.schema.json`).
2. Implement safety gate (`/control/safety_gate.py`).
3. Implement baseline scheduler simulator (`/sim/scheduler_baseline.py`).
4. Add LLM policy adapter (`/agent/policy_agent.py`).
5. Add KPI evaluator + rollback tests (`/eval/*`).
6. Publish experiment matrix and reproducible benchmark scripts.
7. Begin RAN stack integration plan and interface contract.

---

## 13) Summary

This repository should pursue an **LLM-guided, safety-bounded, deterministic-execution MAC scheduling architecture**. The most effective path is to validate policy intelligence in Python first, then port the proven policy-control loop into an existing 5G RAN stack for production-grade evaluation.
