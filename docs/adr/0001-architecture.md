# ADR-0001: Core Architecture — Petroleum Services Contractor Coordination Actor

Date: 2026-07-14
Status: Accepted

## Summary

The cloud-itonami ISIC-08 0910 actor is a langgraph-clj StateGraph that gates
contractor-side coordination operations (service order intake, crew dispatch,
logistics) through a governor-enforced invariant: operators make drilling/well-control
decisions; contractors make field coordination decisions under human-in-the-loop
escalation for safety.

## Context

ISIC-08 0910 classifies **support activities for petroleum and natural gas extraction**,
covering contractor services (drilling rigs, wireline, completions crews, logistics),
NOT the operator's well-control authority. A petroleum services company needs to:

1. Intake and verify service orders from clients (well operators)
2. Schedule crew dispatch to well-sites
3. Coordinate site logistics (transport, supplies, crew movement)
4. Log safety incidents (always escalating for review)

The actor must enforce a hard boundary: no contractor software makes drilling,
well-control, or hazmat decisions — those are operator-exclusive. Violations are
instant hard blocks with no override path.

## Design

### Domain Entities

**Contractor**: A registered petroleum services company (`:client-id`, `:name`, `:safety-rating`).

**Well-site**: A registered well operated by one of the contractor's clients
(`:site-id`, `:operator-id`, `:location`, `:risk-level` ∈ `:low`/`:medium`/`:high`).

**Record**: A committed operational action (service order, dispatch, logistics,
safety log) — write-once via `commit-record!`, never mutated in place.

**Ledger**: Append-only audit trail of all proposals, verdicts, and dispositions,
regardless of outcome (commit or hold).

### StateGraph Flow

```
:intake → :advise → :govern → :decide ─┬─→ :commit            (ok? ∧ ¬escalate)
                                         ├─→ :request-approval  (escalate?)
                                         └─→ :hold              (hard?)
```

- **:intake**: Noop; entry point to initialize request/context channels.
- **:advise**: Calls `Advisor/-advise` to propose an operation. Default is
  `mock-advisor` (deterministic); `llm-advisor` wraps a real LLM with parse
  failure → confidence 0.0 (never fabricated).
- **:govern**: Calls `Governor/check` to assess the proposal. Returns
  `{:ok? :violations :confidence :hard? :escalate?}`. Pure function, never
  mutates store.
- **:decide**: Routes on `:hard?` / `:escalate?` to `:hold`, `:request-approval`,
  or `:commit`.
- **:request-approval**: Checkpoint node with `interrupt-before` — the run
  pauses and only resumes on explicit human call to `approve!`.
- **:commit**: Writes the record and appends ledger entry.
- **:hold**: Appends ledger entry only; no write. Irreversible.

### Governor Rules

**Hard Invariants** (`:hard? true` → `:hold`, no override):
1. Contractor provenance — request's `:client-id` must be registered.
2. Well-site provenance — dispatch/logistics ops must reference a registered site.
3. No actuation — proposal `:effect` must be `:propose` (no direct store write).
4. **No operator-class ops** — any proposal with `:op` ∈
   `{:drill, :perforate, :cement, :well-control, :hazmat-handle, ...}`
   is instantly blocked. This is the core domain boundary.

**Escalation Invariants** (`:escalate? true` → `:request-approval`, human sign-off):
1. `:log-safety-incident` — ALL safety logging escalates.
2. High-risk site — dispatch to a site with `:risk-level :high` escalates.
3. Low confidence — advisor confidence < 0.6 escalates (LLM parse failures = 0.0).

### Scope Exclusion: What This Actor Does NOT Do

**Contractor-side (✓ in scope)**:
- Service order intake and client/site verification
- Crew dispatch scheduling
- Site logistics coordination
- Safety incident logging

**Operator-side (✗ hard-blocked, not in scope)**:
- Drilling decisions (wellpath, bit type, mud weight)
- Well-control operations (shut-in, pressure management)
- Completion decisions (perforation, cementing, artificial lift design)
- Hazardous-material handling authorization
- Subsurface data interpretation

Any proposal attempting to venture into operator decisions is flagged
`:drilling-class-blocked` and routed to `:hold` with no escalation override.

### Portability and Swappable Components

- **Store**: Protocol-based; `MemStore` is default (zero deps); Datomic-backed or
  kotoba-server backend can swap in without touching actor/governor logic.
- **Advisor**: Protocol-based; `mock-advisor` (deterministic) or `llm-advisor`
  (real LLM, parse failures → confidence 0.0).
- **Checkpointer**: Injected; in-memory default; can be swapped for
  persistent checkpoint storage.

### Audit and Traceability

Every node that makes a decision appends to the `:audit` channel:
```clojure
{:node :advise :request request :proposal proposal}
{:node :govern :verdict verdict}
{:node :commit :record record}
{:node :hold :verdict verdict}
```

Additionally, `:commit` and `:hold` directly call `store/append-ledger!` to
record disposition in the store's audit trail, ensuring a tamper-proof
business record independent of any checkpointing.

## Consequences

- **Operator decisions are unreachable** in contractor software, reducing
  attack surface and legal liability for the contractor.
- **Safety incidents always surface** (cannot be silently held); human review
  is mandatory.
- **Audit trail is immutable** and append-only, meeting regulatory expectations
  for the oil & gas industry.
- **LLM parse failures never fabricate confidence**, preventing false confidence
  from leading to auto-commitment of safety-critical operations.
- **Contractor data is independent** of SaaS providers; the actor can be
  self-hosted or deployed into private cloud infrastructure.

## References

- ADR-2607011000 (cloud-itonami Actors pattern)
- CLAUDE.md / Actors section
- ISIC-08 Classification: https://unstats.un.org/unsd/publication/seriesM/seriesm_4rev4e.pdf
