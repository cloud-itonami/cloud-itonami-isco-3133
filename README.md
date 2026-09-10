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
:intake -> :advise -> :govern -> :decide -+-> :commit                          (:ok? true)
                                           +-> :escalate -> :request-approval   (:escalate? true, interrupt-before)
                                           +-> :hold                            (:hard? true)
```

### Proposal operations (all `:effect :propose`)

- `:log-process-reading` — routine output/parameter reading logging (e.g., reactor temperature, pressure, flow rate, conversion metrics)
- `:schedule-maintenance` — maintenance scheduling proposal (e.g., catalyst replacement, heat exchanger cleaning, valve inspection)
- `:flag-anomalous-reading` — surface an out-of-range reading; **ALWAYS escalates to human approval**
- `:coordinate-shift-handover` — shift-handover coordination note (handoff checklist, notable conditions)

### Hard invariants (ALWAYS `:hold`, never overridable)

1. **Plant provenance** — the request must name a plant, and that plant must be registered and verified before any operation.
2. **No direct actuation** — all proposals must have `:effect :propose` only (never `:commit` or `:dispatch`).
3. **No reactor/chemical control** — `:reactor-control`, `:chemical-injection`, `:process-start`, `:emergency-shutdown`, `:interlock-override` and `:safety-system-activation` are *reserved* (`chemical-ops.operation/reserved`): permanently rejected with the reason, never escalated, because no human can delegate licensed operator authority to the actor.
4. **Declared vocabulary** — an `:op` outside `chemical-ops.operation/supported` is rejected as `:undeclared-operation`. This is what makes rule 3 a boundary rather than a list of examples: until 2026-09-11 the governor was a denylist of four names, and a proposal for `:override-interlock` against a registered plant reached `:commit` and was written to the operation records.
5. **Well-formed envelope** — a proposal whose `:op` is not a keyword, or whose `:confidence` is present but not a number in `[0.0, 1.0]`, is rejected rather than compared against (measured before the fix: `:confidence 1.5` was admitted, `:confidence "high"` threw out of the governor).

### Escalation invariants (ALWAYS human sign-off, per the README robotics-premise)

6. **`:flag-anomalous-reading` ALWAYS escalates** — declared `:escalates? true` on the operation itself, no exception.
7. **Low confidence** (< `confidence-floor` = 0.7) ALWAYS escalates — LLM parse failures or uncertain advisors.

An escalation is written to the audit ledger as `:disposition :escalate` *before* the graph interrupts, so a request that waits for a human — or is never approved — has a trace. When a human resumes the thread, the resulting commit entry carries `:approved-by :human`; an automatic commit carries `:approved-by :actor`. Every ledger entry is hash-chained (`chemical-ops.ledger/verify`).

### Implementation modules

- `src/chemical_ops/operation.kotoba` — the closed vocabulary: `supported` (the four proposal operations, with `:escalates?`) and `reserved` (licensed-operator authority, with the reason). The governor is an allowlist over this.
- `src/chemical_ops/facts.kotoba` — well-formedness of the plant record, the request and the proposal envelope, as `{:rule .. :detail ..}` violations that compose with the governor's own.
- `src/chemical_ops/phase.kotoba` — verdict → phase (`:hold` / `:request-approval` / `:commit`), what each phase may do, and `refusal?`.
- `src/chemical_ops/ledger.kotoba` — hash-chained append-only ledger: `commit-entry` (with `:approved-by`), `hold-entry`, `escalate-entry`, `verify`.
- `src/chemical_ops/store.kotoba` — `Store` protocol + `MemStore`: registered plants/units, committed records, and the ledger (every `append-ledger!` extends the chain).
- `src/chemical_ops/advisor.kotoba` — `Advisor` protocol; `mock-advisor` (deterministic, default) proposes a chemical operations coordination action from a request. The advisor only ever produces a `:propose`-effect proposal, never a committed record.
- `src/chemical_ops/governor.kotoba` — `ChemicalOpsGovernor/check`: a pure function, wired as its own `:govern` node. Delegates vocabulary to `operation`, well-formedness to `facts`; hard violations route to `:hold`, escalations to `:request-approval` — an `interrupt-before` node that the graph checkpoints and only resumes on explicit human approval (`actor/approve!`).
- `src/chemical_ops/actor.kotoba` — `build-graph`, `run-request!`, `approve!`: the `langgraph.graph/state-graph` wiring itself, including the `:escalate` node that writes the ledger entry before the interrupt.
- `src/chemical_ops/sim.kotoba` — the governed-scenario harness: 12 requests through the real wired graph, asserting the phase each reaches, that no refusal wrote a record, that every escalation is in the ledger before the interrupt, that a human-approved resume is marked as such, and that every ledger verifies. **Exits 1 when the table demonstrates no refusal** — a governed actor that refuses nothing has shown nothing.

### Running it

The sources are `.kotoba` (owner-instructed rename, 2026-09-10) and are loaded by path — `require` and `clojure.tools.namespace` do not resolve the extension, so `cognitect.test-runner` and `clj-kondo --lint src` both report a clean zero on this tree. The three entry points build their own file lists and **refuse (exit 2) rather than pass** when they found nothing, ran nothing, or ran less than this README publishes:

```bash
clojure -M:test   # 56 tests / 256 assertions across 8 namespaces; exit 2 if fewer ran
clojure -M:sim    # the scenario table; exit 1 if it demonstrated no refusal
clojure -M:lint   # clj-kondo over every .kotoba file by name; exit 2 if it read none
```

`amu compile` does not yet accept these sources (the rename commit says so and calls the compiler's refusals its work list); that migration is separate from the suite running.

This is what backs this repo's `:maturity :implemented` entry in [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation).

## License

AGPL-3.0-or-later.
