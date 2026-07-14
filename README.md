# cloud-itonami-isic-0910

Open Occupation Blueprint for **ISIC-08 0910**: Support activities for petroleum and natural gas extraction.

This repository designs a forkable OSS business for a petroleum services contractor: a coordination robot performs service order intake, crew dispatch scheduling, and site logistics coordination under a governor-gated actor, so the contractor maintains independent operational records and safety auditing instead of renting a closed field-services SaaS.

## Critical domain boundary: CONTRACTOR, NOT OPERATOR

**This actor supports a petroleum SERVICES CONTRACTOR's back-office operations — NOT the drilling operator's well operations.**

Scope: Contractor-side coordination
- ✓ Service order intake and verification
- ✓ Crew dispatch scheduling
- ✓ Site logistics coordination (transport, crew movement, supplies)
- ✓ Safety incident logging (always escalated to human review)

Out of scope: Operator-exclusive decisions (hard-blocked, no override path)
- ✗ Drilling decisions (bit selection, mud weight, hole geometry)
- ✗ Well-control operations (shut-in procedures, pressure management)
- ✗ Completion decisions (perforation, cementing, artificial lift)
- ✗ Hazardous-material handling authorization

This boundary is enforced as a hard invariant in the governor (`petroleum-services.governor`): any proposal flagged with operator-class ops (`:drill`, `:well-control`, `:hazmat-handle`, etc.) is instantly blocked with no override path.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a coordination robot performs service order
intake, crew dispatch, and logistics scheduling under an actor that proposes
actions and an independent **Petroleum Services Governor** that gates them.
The governor never dispatches operations itself; `:safety-incident` logging
or `:high`-risk site dispatch require human sign-off. Drilling/well-control/hazmat decisions
are operator-exclusive and permanently blocked.

## Core Contract

```text
service request + crew roster + well-site database
        |
        v
Services Advisor -> Services Governor -> intake/dispatch/logistics, or human sign-off
        |
        v
coordination actions (gated) + operating records + audit ledger
```

No automated advice can dispatch an operation the governor refuses, suppress
an operating record, or disclose sensitive data without governor approval and
audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISIC-08 `0910`). Required capabilities:

- :robotics
- :identity
- :forms
- :dmn
- :bpmn
- :audit-ledger

## Reference implementation (`:maturity :implemented`)

Full itonami Actor pattern (per ADR-2607011000 / CLAUDE.md's Actors
section): a real [`kotoba-lang/langgraph`](https://github.com/kotoba-lang/langgraph)
`StateGraph`, with the Advisor and Governor as distinct graph nodes and
human-in-the-loop interrupt/resume via checkpointing.

```text
:intake -> :advise -> :govern -> :decide -+-> :commit            (:ok? true)
                                           +-> :request-approval   (:escalate? true, interrupt-before)
                                           +-> :hold               (:hard? true)
```

- `src/petroleum_services/store.cljc` — `Store` protocol + `MemStore`:
  registered contractors, registered well-sites, committed records, an append-only audit ledger.
- `src/petroleum_services/advisor.cljc` — `Advisor` protocol; `mock-advisor`
  (deterministic, default) proposes a coordination operation from a
  request; `llm-advisor` wraps a `langchain.model/ChatModel` — either
  way the advisor only ever produces a `:propose`-effect proposal,
  never a committed record, and LLM parse failures always yield
  `confidence 0.0` (forces escalation, never fabricated confidence).
- `src/petroleum_services/governor.cljc` — `PetroleumServicesGovernor/check`: a pure
  function, wired as its own `:govern` node. Hard invariants
  (unregistered contractor, unregistered well-site, a proposal whose `:effect`
  isn't `:propose`, or any operator-class op) always route to `:hold`.
  Escalation invariants (`:log-safety-incident`, high-risk site dispatch, or low advisor
  confidence) always route to `:request-approval` — an `interrupt-before` node that the graph
  checkpoints and only resumes on explicit human approval (`actor/approve!`).
- `src/petroleum_services/actor.cljc` — `build-graph`, `run-request!`,
  `approve!`: the `langgraph.graph/state-graph` wiring itself.

```bash
clojure -M:test
```

This is what backs this repo's `:maturity :implemented` entry in
[`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation).

## License

AGPL-3.0-or-later.
