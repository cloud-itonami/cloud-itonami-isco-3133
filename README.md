# cloud-itonami-isco-3133

Open Occupation Blueprint for **ISCO-08 3133**: Chemical Processing Plant Controllers.

This repository designs a forkable OSS business for chemical processing plant operations coordination: a document-handling and telemetry-monitoring robot performs routine operational logging, maintenance scheduling, and anomalous-condition flagging under a governor-gated actor, so a plant operator keeps its own operational and maintenance history instead of renting a closed plant-management platform.

## What This Does NOT Do

**CRITICAL SCOPE BOUNDARY:** This actor supports plant-operator **back-office coordination workflow only**.

- ✗ This actor does **NOT** directly control chemical reactors, process units, or injection systems.
- ✗ This actor does **NOT** dispatch chemical dosing, reactor temperature control, pressure regulation, or feed rate control commands.
- ✗ This actor does **NOT** have authority over emergency shutdown, process control interlocks, or safety system activation.

Those capabilities remain exclusively under **licensed plant engineers' and operators' human authority** and are outside this system's vocabulary. The Governor enforces this hard boundary (see `chemical-ops.governor`).

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs the physical domain work**. Here a telemetry-monitoring robot performs routine process readings, maintenance scheduling, and anomalous-condition flagging under an actor that proposes coordination actions and an independent **Chemical Operations Governor** that gates them. The governor never dispatches a plant operation itself; `:high`/`:safety-critical` actions (such as anomalous-reading escalation, or any proposal touching reactor/chemical control) require human sign-off.

## Core Contract

```text
plant registration + operational parameters + telemetry observation
        |
        v
Chemical Operations Advisor -> Chemical Operations Governor -> operational record, maintenance scheduling, escalation, or human approval
        |
        v
robot actions (gated) + operation records + escalation records + audit ledger
```

No automated advice can dispatch a plant operation the governor refuses, suppress an operational record, escalate a safety concern without governor approval, or propose reactor/chemical control without hard rejection and audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation) (ISCO-08 `3133`). Required capabilities:

- :robotics
- :identity
- :forms
- :dmn
- :bpmn
- :audit-ledger
- :telemetry

See [`docs/business-model.md`](docs/business-model.md) and [`docs/operator-guide.md`](docs/operator-guide.md).

## Reference implementation (`:maturity :implemented`)

Full itonami Actor pattern (per ADR-2607011000 / CLAUDE.md's Actors section), alongside `cloud-itonami-isco-3131`, `-isco-3132`, `-isco-2411`, `-isco-2166`, and others: a real [`kotoba-lang/langgraph`](https://github.com/kotoba-lang/langgraph) `StateGraph`, with the Advisor and Governor as distinct graph nodes and human-in-the-loop interrupt/resume via checkpointing.

```text
:intake -> :advise -> :govern -> :decide -+-> :commit            (:ok? true)
                                           +-> :request-approval   (:escalate? true, interrupt-before)
                                           +-> :hold               (:hard? true)
```

### Proposal operations (all `:effect :propose`)

- `:log-process-reading` — routine output/parameter reading logging (e.g., reactor temperature, pressure, flow rate, conversion metrics)
- `:schedule-maintenance` — maintenance scheduling proposal (e.g., catalyst replacement, heat exchanger cleaning, valve inspection)
- `:flag-anomalous-reading` — surface an out-of-range reading; **ALWAYS escalates to human approval**
- `:coordinate-shift-handover` — shift-handover coordination note (handoff checklist, notable conditions)

### Hard invariants (ALWAYS `:hold`, never overridable)

1. **Plant provenance** — the request's plant-id must be registered and verified before any operation.
2. **No direct actuation** — all proposals must have `:effect :propose` only (never `:commit` or `:dispatch`).
3. **No reactor/chemical control** — any proposal whose `:op` is `:reactor-control`, `:chemical-injection`, `:process-start`, or `:emergency-shutdown` is permanently rejected (those are exclusively licensed operator authority).

### Escalation invariants (ALWAYS human sign-off, per the README robotics-premise)

4. **`:flag-anomalous-reading` ALWAYS escalates** — high-safety signal, no exception.
5. **Low confidence** (< `confidence-floor` = 0.7) ALWAYS escalates — LLM parse failures or uncertain advisors.

### Implementation modules

- `src/chemical_ops/store.kotoba` — `Store` protocol + `MemStore`: registered plants/units, committed records, append-only audit ledger.
- `src/chemical_ops/advisor.kotoba` — `Advisor` protocol; `mock-advisor` (deterministic, default) proposes a chemical operations coordination action from a request; `llm-advisor` wraps a `langchain.model/ChatModel` — either way the advisor only ever produces a `:propose`-effect proposal, never a committed record, and LLM parse failures always yield `confidence 0.0` (forces escalation, never fabricated confidence).
- `src/chemical_ops/governor.kotoba` — `ChemicalOpsGovernor/check`: a pure function, wired as its own `:govern` node. Hard invariants (unregistered/unverified plant, a proposal whose `:effect` isn't `:propose`, or reactor/chemical/emergency-shutdown commands) always route to `:hold`. Escalation invariants (`:flag-anomalous-reading`, or low advisor confidence) always route to `:request-approval` — an `interrupt-before` node that the graph checkpoints and only resumes on explicit human approval (`actor/approve!`).
- `src/chemical_ops/actor.kotoba` — `build-graph`, `run-request!`, `approve!`: the `langgraph.graph/state-graph` wiring itself.

```bash
clojure -M:test
```

This is what backs this repo's `:maturity :implemented` entry in [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation).

## License

AGPL-3.0-or-later.
