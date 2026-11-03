# Company Brain — Foundation Documents

A Company Brain captures the **experience** created during real work (what was tried, what failed, what was corrected, why a decision changed), converts it into **governed memory**, and delivers the **minimum sufficient context** to the right person or agent at the moment of decision or action. It is not a search layer over documents.

These documents are organization-neutral. They contain no company names, people, hosts, addresses, or internal services, and can be used as the starting point for any organization building a Company Brain.

```
   INPUT ---> PROCESSING ---> MEMORY ---> OUTPUT ---> (what happened next) ---+
     ^                                                                        |
     +------------------------------------------------------------------------+
   governance (identity, permissions, policy precedence, audit) applies to every arrow
```

## Documents

| File | Type | Read it for |
|---|---|---|
| [01-requirements.md](01-requirements.md) | Normative | What the system must do: objective, non-goals, actors, functional requirements by function, governance, security baseline, non-functional requirements, acceptance scenarios, open decisions, glossary |
| [02-functionality.md](02-functionality.md) | Descriptive | What users and agents experience: capabilities per function, scope and promotion, lifecycle, journeys, agentic flows, dashboard, integration surfaces, behaviors that hold everywhere |
| [03-architecture.md](03-architecture.md) | Normative where marked | How it is built: object model, ten components, write and read paths, projections, lifecycle, governance plane, evaluation loop, deployment and topology, delivery phases, design rationale, constraints checklist |
| [04-kpi-and-measurements.md](04-kpi-and-measurements.md) | Normative for the catalog | How it is measured: principles, KPI catalog by layer, measurement protocols (attribution, net value A/B, research-agent baseline, canaries), phase gates, dashboard mapping, reporting cadence, anti-metrics |
| [05-source-map.md](05-source-map.md) | Informative | Why it is shaped this way: every public source the design draws on, mapped to the requirements, functionality, and architecture it shaped; reverse indexes by requirement area and by concept |

Suggested reading order: README → [01 §1–4](01-requirements.md#1-objective) → [02](02-functionality.md) → [03](03-architecture.md) → [01](01-requirements.md) (rest) → [04](04-kpi-and-measurements.md). Use [05](05-source-map.md) to trace any requirement back to its sources.

## Conventions

- **MUST / MUST NOT / SHOULD / MAY** follow RFC 2119.
- Requirement IDs are stable: `REQ-<AREA>-<nn>`. Acceptance scenarios: `AS-nn`. Open decisions: `OD-nn`. Agentic flows: `F1`–`F8`.
- Component numbers `[1]`–`[10]` refer to the logical architecture in [03-architecture.md](03-architecture.md) [§3](03-architecture.md#3-logical-architecture).
- Sections marked **Proposal** are recommended approaches or starting thresholds awaiting owner approval.
- Diagrams are plain ASCII so they survive copy into any tracker, wiki, or email. Read them in a monospace font.

## The ten components in one line each

| # | Component | One-line responsibility |
|---|---|---|
| [1] | Source adapters | Connect systems of record and work surfaces; emit evidence with identity, time, source, authorization |
| [2] | Evidence and trajectory log | Immutable, append-only audit root for everything derived |
| [3] | Experience processor | Detect boundaries, reflect, extract, classify, deduplicate, find contradictions; produce candidates |
| [4] | Admission and promotion service | Validate, handle sensitivity, assign owners, apply scope and policy gates; versioned admission |
| [5] | Managed memory store | Canonical, versioned, linked, scoped units with lifecycle state; temporary working-memory state, crash-safe continuity checkpoints, and an action journal, kept apart from shared memory; maintained by a governed orchestrator that proposes changes and never silently rewrites |
| [6] | Retrieval projections | Derived vector, graph, text, and relational indexes; rebuilt and invalidated from [5] |
| [7] | Retrieval and context composer | Deterministic filters, ranking, budgeted selection, L0/L1 composition, selection logging |
| [8] | Output adapters | Harness hooks, MCP, APIs, UI; deliver in-tool; collect outcome signals |
| [9] | Governance control plane | Identity, permissions, policy precedence, audit, expiry, deletion, rollback; managed integrity and authenticity controls; governs which maintenance actions may run automatically |
| [10] | Evaluation loop | Verify outcomes against explicit criteria, attribute them along decision lineage by evidence of influence, not mere exposure, update utility per version, trigger weakening, run canaries and shadow evaluation, net value, baseline |

## The ten rules that never bend

1. The executing agent never writes shared memory.
2. No monolithic rewrite of accumulated memory.
3. Deterministic filters (identity, permission, policy, validity, authority, scope, grounding eligibility for this request's purpose) run before ranking.
4. Memory reaches the model before it reasons; exploration tools are never the only path.
5. Projections are derived, invalidated with canonical memory, and inherit its grounding restrictions; never authoritative.
6. Empty, filtered, or unavailable results mean unknown; authority-requiring actions fail closed.
7. Policies are human-owned and take deterministic precedence everywhere.
8. Utility is measured per version and context; versions never inherit reputation; promotion needs evidence across distinct contexts and is blocked by a protected regression.
9. Forgetting preserves history; deletion is a separate governed operation.
10. Interfaces are harness-neutral.

## References

All public sources the design draws on, with what each contributed and where it lands, are in [05-source-map.md](05-source-map.md).
