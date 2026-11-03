# Company Brain — Source Map

This document maps every external resource this repository draws on to the requirements, functionality, and architecture it shaped. Use it to answer two questions:

1. **Why does this requirement or design choice exist?** Find the requirement ID, component, or section, then follow it to its sources.
2. **What did this source change?** Find the source, then follow it to the requirements and sections it touched.

The repository is the foundation for a Company Brain along three pillars: **Requirements**, **Functionality**, and **Architecture**, with **Measurement** closing the loop. Like the rest of the set, this document is organization-neutral: it lists only public resources and contains no company names, people, hosts, addresses, or internal services.

---

## 1. Conventions

**Source IDs** are stable. Never reuse or renumber an ID; retire it instead ([§7](#7-maintaining-this-map)).

| Prefix | Category |
|---|---|
| `SRC-F` | Foundational organizational-memory research |
| `SRC-A` | Agent memory and self-improvement research |
| `SRC-I` | Reference implementations and products |
| `SRC-P` | Practitioner inputs (anonymized) |

In the tables below the `SRC-` prefix is dropped where space is short (`A01` = `SRC-A01`).

**Influence labels**

| Label | Meaning |
|---|---|
| `Direct` | Created, changed, or directly supports a requirement |
| `Supporting` | Shaped the framing or the architecture, without a requirement of its own |
| `Reviewed` | Used mainly to define boundaries, non-goals, and gaps |

**Pillar tags:** `R` Requirements · `F` Functionality · `A` Architecture · `M` Measurement.

**Document shorthand:** [01](01-requirements.md) requirements · [02](02-functionality.md) functionality · [03](03-architecture.md) architecture · [04](04-kpi-and-measurements.md) KPIs.

**Component key:** `[1]`–`[10]` in the tables are the logical components from [03-architecture.md §3](03-architecture.md#3-logical-architecture).

```
   INPUT ---------> PROCESSING ---------> MEMORY ---------> OUTPUT ----> outcome ---+
   [1] adapters     [3] processor         [4] admission     [6] projections         |
   [2] evidence log                       [5] memory store  [7] composer            |
                                                            [8] output adapters     |
     ^                                                                              |
     +--------------------------- [10] evaluation loop <----------------------------+
   [9] governance control plane applies to every arrow
```

---

## 2. Source catalog

### 2.1 Foundational organizational-memory research

| ID | Source | Label | Pillar | Contributes | Maps to |
|---|---|---|---|---|---|
| SRC-F01 | [Walsh & Ungson — Organizational Memory](https://doi.org/10.2307/258607) | Direct | R F A | Memory is distributed across retention reservoirs; it needs acquisition, retention, and retrieval, not one central database | [REQ-OBJ-01](01-requirements.md#req-obj-01), [REQ-ME-01](01-requirements.md#req-me-01), [OD-10](01-requirements.md#od-10), [01](01-requirements.md) [§2](01-requirements.md#2-non-goals); [03](03-architecture.md) [§1](03-architecture.md#1-architectural-principles), [§14.1](03-architecture.md#141-topology-position-od-10) |
| SRC-F02 | [Levitt & March — Organizational Learning](https://doi.org/10.1146/annurev.so.14.080188.001535) | Direct | R F | Lessons from experience become routines; warns about path dependence and competency traps | [REQ-ME-08](01-requirements.md#req-me-08), [REQ-ME-13](01-requirements.md#req-me-13), [REQ-SC-02](01-requirements.md#req-sc-02), [REQ-EV-02](01-requirements.md#req-ev-02); [02](02-functionality.md) [§5](02-functionality.md#5-memory-lifecycle-functional-view); [03](03-architecture.md) [§8](03-architecture.md#8-memory-lifecycle) |
| SRC-F03 | [Argote & Ingram — Knowledge Transfer](https://doi.org/10.1006/obhd.2000.2893) | Direct | R A | Knowledge lives in people, tools, tasks, and their combinations; transfer and coordination are part of the architecture | [REQ-IN-02](01-requirements.md#req-in-02), [REQ-ME-03](01-requirements.md#req-me-03), [REQ-SC-01](01-requirements.md#req-sc-01); [03](03-architecture.md) [§2](03-architecture.md#2-object-model) |
| SRC-F04 | [Lewis — Transactive Memory Systems in the Field](https://doi.org/10.1037/0021-9010.88.4.587) | Direct | R F | Expertise locator: know *who or what* knows, not only *what* is known | [REQ-ME-04](01-requirements.md#req-me-04) (owner), [REQ-SC-04](01-requirements.md#req-sc-04), [REQ-OUT-01](01-requirements.md#req-out-01); [02](02-functionality.md) [§4](02-functionality.md#4-scope-and-promotion) |

### 2.2 Agent memory and self-improvement research

| ID | Source | Label | Pillar | Contributes | Maps to |
|---|---|---|---|---|---|
| SRC-A01 | [Agentic Context Engineering (ACE)](https://arxiv.org/abs/2510.04618) | Direct | R F A | Incremental context deltas; Reflector and Curator roles; helpful/harmful counters; context collapse from full rewrites | [REQ-ME-09](01-requirements.md#req-me-09), [REQ-PR-09](01-requirements.md#req-pr-09); `[3]`, [REQ-ME-29](01-requirements.md#req-me-29); [03](03-architecture.md) [§4](03-architecture.md#4-write-path), [§16.3](03-architecture.md#163-why-the-executing-agent-never-writes-shared-memory), [§16.5](03-architecture.md#165-known-failure-modes-the-design-must-avoid) |
| SRC-A02 | [CHOIR](https://arxiv.org/abs/2509.20512) | Direct | R M | Private questions reveal real knowledge gaps but carry privacy and psychological-safety risk → aggregate gap telemetry, no employee surveillance | [REQ-IN-07](01-requirements.md#req-in-07), [REQ-IN-13](01-requirements.md#req-in-13), [REQ-GOV-09](01-requirements.md#req-gov-09), [REQ-NFR-07](01-requirements.md#req-nfr-07); [04](04-kpi-and-measurements.md) [§9](04-kpi-and-measurements.md#9-data-handling-for-metrics) |
| SRC-A03 | [Organizational Memory for Agentic Business Process Execution](https://arxiv.org/abs/2607.03228) | Direct | R F | Process atoms: applicability, action, purpose, source → schema for procedural experience and its conditions of use | [REQ-PR-07](01-requirements.md#req-pr-07), [REQ-PR-08](01-requirements.md#req-pr-08), [REQ-ME-04](01-requirements.md#req-me-04), [OD-02](01-requirements.md#od-02); [02](02-functionality.md) [§3.2](02-functionality.md#32-processing-reflection-and-extraction) |
| SRC-A04 | [Shared Organizational Memory for Enterprise Coding Agents](https://arxiv.org/abs/2608.00122) | Direct | R F | Enterprise deployment of task-triggered capture with contributor approval; shows passive capture and outcome utility still incomplete | [REQ-IN-04](01-requirements.md#req-in-04), [REQ-IN-12](01-requirements.md#req-in-12), [REQ-EV-10](01-requirements.md#req-ev-10); [02](02-functionality.md) [§3.1](02-functionality.md#31-input-capture) |
| SRC-A05 | [WikiSkill](https://arxiv.org/abs/2608.27454) | Direct | R F A | Separate raw trajectories, persistent knowledge, and executable skills; evaluate and allow rollback before a skill change is admitted | [REQ-ME-03](01-requirements.md#req-me-03), [REQ-GOV-08](01-requirements.md#req-gov-08), [OD-08](01-requirements.md#od-08); [03](03-architecture.md) [§2](03-architecture.md#2-object-model), [§16.6](03-architecture.md#166-building-blocks-not-a-brain) |
| SRC-A06 | [Agent Zero Memory](https://arxiv.org/abs/2608.29606) | Direct | R A | Provenance-linked episodic, graph, and hierarchical views; cite only evidence actually opened; abstain without evidence | [REQ-PR-04](01-requirements.md#req-pr-04), [REQ-ME-02](01-requirements.md#req-me-02), [REQ-OUT-03](01-requirements.md#req-out-03), [REQ-OUT-15](01-requirements.md#req-out-15), [REQ-OUT-17](01-requirements.md#req-out-17), [REQ-ME-16](01-requirements.md#req-me-16) |
| SRC-A07 | [Copying explains the collective behavior of AI agents in the wild](https://arxiv.org/abs/2609.09150) | Direct | R | Shared memory becomes de facto behavioral policy → authority, expiry, rollback, controlled writes | [REQ-PR-09](01-requirements.md#req-pr-09), [REQ-GOV-02](01-requirements.md#req-gov-02), [REQ-OUT-16](01-requirements.md#req-out-16); [03](03-architecture.md) [§16.3](03-architecture.md#163-why-the-executing-agent-never-writes-shared-memory), [§16.5](03-architecture.md#165-known-failure-modes-the-design-must-avoid) |
| SRC-A08 | [Glyph](https://arxiv.org/abs/2609.10430) | Supporting | A | Governed multi-strategy pipeline with ontology, provenance, and partial-failure resilience (narrow domain) | [REQ-NFR-02](01-requirements.md#req-nfr-02); [03](03-architecture.md) [§7.1](03-architecture.md#71-semantic-identity-and-relationship-typing) |
| SRC-A09 | [COBRA-Skills](https://arxiv.org/abs/2609.11682) | Direct | R F M | Risk-aware evaluation budget; version-specific utility; changed candidates must earn evidence again | [REQ-EV-01…09](01-requirements.md#req-ev-01), [REQ-EV-04](01-requirements.md#req-ev-04), [REQ-EV-16](01-requirements.md#req-ev-16), [REQ-EV-17](01-requirements.md#req-ev-17); [03](03-architecture.md) [§11](03-architecture.md#11-evaluation-loop), [§16.6](03-architecture.md#166-building-blocks-not-a-brain); [04](04-kpi-and-measurements.md) [§4.4](04-kpi-and-measurements.md#44-canary-evaluation-of-unvalidated-units) |
| SRC-A10 | [DeFA](https://arxiv.org/abs/2610.01256) | Direct | R F M | Dependency-guided failure attribution: a failed run must not teach the wrong lesson about prompt, skill, tool, or environment | [REQ-EV-06](01-requirements.md#req-ev-06); `[10]`; [03](03-architecture.md) [§11](03-architecture.md#11-evaluation-loop); [04](04-kpi-and-measurements.md) [§4.1](04-kpi-and-measurements.md#41-outcome-signal-capture-and-attribution) |
| SRC-A11 | [Learning from Revision Consequences (HMED)](https://arxiv.org/abs/2610.07979) | Direct | R M | Compare old vs revised behavior from a common start state; learn from consequences, not retrospective stories | [REQ-EV-02](01-requirements.md#req-ev-02), [REQ-EV-08](01-requirements.md#req-ev-08), [REQ-NFR-10](01-requirements.md#req-nfr-10), [REQ-EV-15](01-requirements.md#req-ev-15); [04](04-kpi-and-measurements.md) [§4.2](04-kpi-and-measurements.md#42-net-value-ab-or-paired-protocol) |
| SRC-A12 | [Agent Plasticity](https://arxiv.org/abs/2610.08902) | Direct | R M | System-level measures: held-out improvement, transfer, retention, gain relative to learning cost | [REQ-EV-03](01-requirements.md#req-ev-03), [REQ-NFR-07](01-requirements.md#req-nfr-07), [REQ-NFR-10](01-requirements.md#req-nfr-10), [AS-07](01-requirements.md#as-07); [03](03-architecture.md) [§11.1](03-architecture.md#111-net-value-accounting); [04](04-kpi-and-measurements.md) [§3.7](04-kpi-and-measurements.md#37-net-value) |
| SRC-A13 | [STRATA](https://arxiv.org/abs/2610.19386) | Direct | R F M | Typed hierarchical experience; explicit contrasts; abstention; negative-transfer tests against a no-memory baseline | [REQ-PR-07](01-requirements.md#req-pr-07), [REQ-OUT-03](01-requirements.md#req-out-03), [REQ-EV-01](01-requirements.md#req-ev-01), [REQ-EV-05](01-requirements.md#req-ev-05), [REQ-EV-12](01-requirements.md#req-ev-12), [AS-08](01-requirements.md#as-08), [REQ-EV-15](01-requirements.md#req-ev-15), [REQ-EV-16](01-requirements.md#req-ev-16); [03](03-architecture.md) [§11.2](03-architecture.md#112-research-agent-baseline); [04](04-kpi-and-measurements.md) [§4.3](04-kpi-and-measurements.md#43-research-agent-baseline) |
| SRC-A14 | [Evolving Together, Failing Apart](https://arxiv.org/abs/2610.20845) | Direct | R A | A changed skill bank can break a co-evolved policy → compatibility matrices, canaries, re-evaluation after model/harness/skill change | [REQ-EV-03](01-requirements.md#req-ev-03), [REQ-EV-04](01-requirements.md#req-ev-04), [REQ-EV-08](01-requirements.md#req-ev-08), [REQ-NFR-06](01-requirements.md#req-nfr-06), [REQ-EV-17](01-requirements.md#req-ev-17); [04](04-kpi-and-measurements.md) [§4.4](04-kpi-and-measurements.md#44-canary-evaluation-of-unvalidated-units) |
| SRC-A15 | [From Relevant Memories to Sufficient Evidence (QEMem)](https://arxiv.org/abs/2610.22668) | Direct | R F | Relevant memories can still be an insufficient evidence set → evidential roles, sufficiency checks, targeted retrieval | [REQ-OUT-01](01-requirements.md#req-out-01), [REQ-OUT-09](01-requirements.md#req-out-09); `[7]`; [03](03-architecture.md) [§5](03-architecture.md#5-read-path) |
| SRC-A16 | [MemDerive](https://arxiv.org/abs/2610.25373) | Direct | R A | Derived state, versioned rules, dependency tracking, invalidation and recomputation after source change | [REQ-ME-05](01-requirements.md#req-me-05), [REQ-ME-11](01-requirements.md#req-me-11), [REQ-ME-13](01-requirements.md#req-me-13), [REQ-IN-15](01-requirements.md#req-in-15), [REQ-OUT-10](01-requirements.md#req-out-10); `[6]`, [REQ-ME-28](01-requirements.md#req-me-28); [03](03-architecture.md) [§7](03-architecture.md#7-canonical-memory-and-retrieval-projections) |
| SRC-A17 | [GEPA-Curator](https://arxiv.org/abs/2610.37651) | Direct | R F | Admission control: filter noisy experience; allow "insufficient evidence to learn" instead of forced updates | [REQ-ME-10](01-requirements.md#req-me-10), [OD-03](01-requirements.md#od-03); `[4]`; [02](02-functionality.md) [§3.3](02-functionality.md#33-memory-governed-retention) |

### 2.3 Reference implementations and products

| ID | Source | Label | Pillar | Contributes | Maps to |
|---|---|---|---|---|---|
| SRC-I01 | [Project Lucca](https://github.com/AlienatedAliens/ProjectLucca) | Direct | R F | Retention vs grounding eligibility; autobiographical memory is not truth; proposal → decision → outcome records; shadow, replay, regression, approval gates before rule promotion; signed provenance gates (integrity, not truth) | [REQ-IN-06](01-requirements.md#req-in-06), [REQ-PR-15](01-requirements.md#req-pr-15), [REQ-ME-14](01-requirements.md#req-me-14), [REQ-ME-15…18](01-requirements.md#req-me-15), [REQ-OUT-18](01-requirements.md#req-out-18), [REQ-ME-12](01-requirements.md#req-me-12), [REQ-EV-08](01-requirements.md#req-ev-08), [AS-10](01-requirements.md#as-10), [AS-11](01-requirements.md#as-11), [OD-08](01-requirements.md#od-08), [REQ-EV-13…17](01-requirements.md#req-ev-13), [AS-12](01-requirements.md#as-12), [REQ-SEC-16…17](01-requirements.md#req-sec-16), [AS-15](01-requirements.md#as-15); [03](03-architecture.md) [§2.1](03-architecture.md#21-decision-lineage), [§2.2](03-architecture.md#22-retention-retrieval-and-grounding), [§10.1](03-architecture.md#101-record-integrity-and-attestation) |
| SRC-I02 | [Microsoft Skill Recorder](https://github.com/microsoft/skill-recorder) | Direct | R F A | Observed-work capture; evidence separate from reconstruction; employee review; privacy controls; one recording → candidate, not skill | [REQ-IN-09…14](01-requirements.md#req-in-09), [REQ-PR-13](01-requirements.md#req-pr-13), [REQ-PR-14](01-requirements.md#req-pr-14), [REQ-ME-12](01-requirements.md#req-me-12), [AS-06](01-requirements.md#as-06); [02](02-functionality.md) [§6.6](02-functionality.md#66-observed-work-becomes-a-skill-candidate-as-06); [03](03-architecture.md) [§4.1](03-architecture.md#41-observed-work-capture), [§16.6](03-architecture.md#166-building-blocks-not-a-brain) |
| SRC-I03 | [PM Brain](https://github.com/phuryn/pm-brain) | Supporting | F A | Source → ingestion → durable-knowledge split; hypothesis vs decision; reconsideration conditions; role packs over a generic core | [REQ-PR-03](01-requirements.md#req-pr-03), [REQ-ME-02](01-requirements.md#req-me-02), [REQ-SC-01](01-requirements.md#req-sc-01), [REQ-IN-16](01-requirements.md#req-in-16) |
| SRC-I04 | [Company Brain Azure](https://github.com/roy2392/company-brain-azure) | Reviewed | A | Broad reference; exposes the gap to ACL-aware retrieval, continuous ingestion, governed memory, evaluation, recovery | [01](01-requirements.md) [§2](01-requirements.md#2-non-goals); [REQ-OUT-06](01-requirements.md#req-out-06), [REQ-IN-04](01-requirements.md#req-in-04), [REQ-EV-01](01-requirements.md#req-ev-01), [REQ-NFR-02](01-requirements.md#req-nfr-02) |
| SRC-I05 | [Merge — Universal Context Layer](https://www.merge.dev/blog/universal-context-layer) | Supporting | A | Cross-tool governed context with connectors, inherited permissions, citations; shows context access ≠ experience learning | [01](01-requirements.md) [§2](01-requirements.md#2-non-goals); `[1]` `[8]`; [03](03-architecture.md) [§16.4](03-architecture.md#164-context-layer-versus-memory), [§16.6](03-architecture.md#166-building-blocks-not-a-brain) |
| SRC-I06 | [OpenViking repository](https://github.com/volcengine/OpenViking) | Supporting | A | Context database combining memory, knowledge, skills; separates canonical material from retrieval representations | `[5]` `[6]`; [03](03-architecture.md) [§7](03-architecture.md#7-canonical-memory-and-retrieval-projections) |
| SRC-I07 | [OpenViking context layers](https://docs.openviking.ai/en/concepts/03-context-layers) | Direct | R A | Progressive disclosure: compact, navigational, full detail linked to canonical evidence | [REQ-OUT-09](01-requirements.md#req-out-09), [REQ-OUT-10](01-requirements.md#req-out-10); [03](03-architecture.md) [§6](03-architecture.md#6-progressive-disclosure) |
| SRC-I08 | [Memtrace](https://www.memtrace.io/) | Supporting | F | Coding-agent memory; evidence-backed decision memory; safe absence behavior | [REQ-OUT-11](01-requirements.md#req-out-11), [REQ-IN-15](01-requirements.md#req-in-15); [02](02-functionality.md) [§6.5](02-functionality.md#65-nothing-applies-as-05) |
| SRC-I09 | [Memtrace Cortex](https://github.com/syncable-dev/memtrace-public/blob/main/docs/cortex.md) | Direct | R | `CannotProve` as an explicit unknown, never as permission | [REQ-OUT-11](01-requirements.md#req-out-11), [REQ-OUT-13](01-requirements.md#req-out-13), [AS-05](01-requirements.md#as-05) |
| SRC-I10 | [Memtrace decision-memory skill](https://github.com/syncable-dev/memtrace-public/blob/main/plugins/memtrace-skills/skills/memtrace-decision-memory/SKILL.md) | Supporting | F | Structured decision record with evidence, status, retrieval behavior, not free-form notes | [REQ-PR-02](01-requirements.md#req-pr-02), [REQ-ME-04](01-requirements.md#req-me-04), [REQ-IN-16](01-requirements.md#req-in-16) |
| SRC-I11 | [Atlan — overview](https://docs.atlan.com/get-started/what-is-atlan) | Supporting | A | Data catalog and lineage graph as a trusted data-domain subsystem and source adapter, not the whole brain | [REQ-IN-01](01-requirements.md#req-in-01); `[1]` |
| SRC-I12 | [Atlan — Context Agents Studio](https://docs.atlan.com/product/capabilities/governance/context-agents-studio) | Supporting | A | Governed agent-based enrichment inside a controlled data context | [REQ-ME-04](01-requirements.md#req-me-04); `[3]` |
| SRC-I13 | [Atlan — Remote MCP](https://docs.atlan.com/product/capabilities/atlan-ai/how-tos/remote-mcp-overview) | Direct | R A | MCP is for exploration and evidence expansion, not proof of proactive delivery before reasoning | [REQ-OUT-04](01-requirements.md#req-out-04), [REQ-OUT-07](01-requirements.md#req-out-07); `[8]`; [02](02-functionality.md) [§9](02-functionality.md#9-integration-surfaces); [03](03-architecture.md) [§12](03-architecture.md#12-integration-surfaces) |
| SRC-I14 | [GitHub Copilot — enterprise managed permissions](https://github.blog/changelog/2026-09-09-enterprise-managed-permissions-for-github-copilot-agent-operations/) | Direct | R F A | Deterministic organizational policy precedence over local agent settings | [REQ-GOV-02](01-requirements.md#req-gov-02), [AS-04](01-requirements.md#as-04); `[9]`; [02](02-functionality.md) [§6.4](02-functionality.md#64-policy-wins-everywhere-as-04); [03](03-architecture.md) [§10](03-architecture.md#10-governance-control-plane) |
| SRC-I15 | [Microsoft Ontology Playground](https://github.com/microsoft/Ontology-Playground) | Supporting | A | Ontology and RDF as semantic infrastructure for typed relationships, separate from admission and truth | [REQ-ME-05](01-requirements.md#req-me-05), [REQ-IN-15](01-requirements.md#req-in-15); [03](03-architecture.md) [§7.1](03-architecture.md#71-semantic-identity-and-relationship-typing) |

### 2.4 Practitioner inputs

Public practitioner discussions that shaped the framing. Author names are intentionally omitted, and links are given only where recorded; each entry records the idea it contributed.

| ID | Input | Label | Pillar | Contributes | Maps to |
|---|---|---|---|---|---|
| SRC-P01 | Working definition of a company brain | Direct | R | Knowledge access vs a brain that captures, manages, and redistributes experience | [REQ-OBJ-01](01-requirements.md#req-obj-01), [REQ-OBJ-02](01-requirements.md#req-obj-02); [02](02-functionality.md) [§1](02-functionality.md#1-what-the-system-is-for); [03](03-architecture.md) [§16.1](03-architecture.md#161-why-experience-not-knowledge) |
| SRC-P02 | Shared, self-improving experience units | Direct | R F | Portable experience units shared and improved across people and agents | [REQ-PR-08](01-requirements.md#req-pr-08), [REQ-SC-01](01-requirements.md#req-sc-01), [REQ-IN-16](01-requirements.md#req-in-16); [03](03-architecture.md) [§2](03-architecture.md#2-object-model) |
| SRC-P03 | Critique of "universal context layer = company brain" | Supporting | R | Governed context layer vs the full learning loop | [01](01-requirements.md) [§2](01-requirements.md#2-non-goals); [03](03-architecture.md) [§16.4](03-architecture.md#164-context-layer-versus-memory) |
| SRC-P04 | Knowledge graphs as agent memory | Supporting | A | Graph as a provenance-preserving retrieval projection, not the brain | [REQ-IN-15](01-requirements.md#req-in-15); [03](03-architecture.md) [§7](03-architecture.md#7-canonical-memory-and-retrieval-projections), [§16.6](03-architecture.md#166-building-blocks-not-a-brain) |
| SRC-P05 | Ontology and RDF in practice | Supporting | A | Stable canonical identity, typed relationships, reversible entity resolution, versioned semantics | [REQ-ME-05](01-requirements.md#req-me-05); [03](03-architecture.md) [§7.1](03-architecture.md#71-semantic-identity-and-relationship-typing) |
| SRC-P06 | Ad hoc analytical work | Supporting | R F | Capture gap in query history, notebooks, BI, and research trails | [REQ-IN-08](01-requirements.md#req-in-08); [02](02-functionality.md) [§3.1](02-functionality.md#31-input-capture) |
| SRC-P07 | [Five-layer memory and meta-memory](https://lnkd.in/p/d2ipxmdj) | Direct | R F | Working-memory continuity, scheduled and event-driven maintenance, drift checks, outcome loop | [REQ-ME-02](01-requirements.md#req-me-02), [REQ-ME-19…25](01-requirements.md#req-me-19), [AS-13](01-requirements.md#as-13), [REQ-EV-10](01-requirements.md#req-ev-10), [REQ-OUT-02](01-requirements.md#req-out-02), [REQ-ME-26…30](01-requirements.md#req-me-26), [AS-14](01-requirements.md#as-14); [02](02-functionality.md) [§5](02-functionality.md#5-memory-lifecycle-functional-view); [03](03-architecture.md) [§2.3](03-architecture.md#23-working-memory-and-continuity), [§8](03-architecture.md#8-memory-lifecycle), [§8.1](03-architecture.md#81-memory-maintenance-orchestrator) |

---

## 3. Traceability by requirement area

Reverse index: requirement area → sources that shaped it.

| Area | Requirements | Components | Functionality | Sources |
|---|---|---|---|---|
| Objective and non-goals | [REQ-OBJ-01…02](01-requirements.md#req-obj-01), [01](01-requirements.md) [§2](01-requirements.md#2-non-goals) | — | [02](02-functionality.md) [§1](02-functionality.md#1-what-the-system-is-for) | F01, P01, P03, I04, I05 |
| Input — capture | [REQ-IN-01…08](01-requirements.md#req-in-01) | `[1]` `[2]` | [02](02-functionality.md) [§3.1](02-functionality.md#31-input-capture) | F03, A02, A04, I01, I11, P06 |
| Input — observed work | [REQ-IN-09…14](01-requirements.md#req-in-09) | `[1]` `[2]` | [02](02-functionality.md) [§6.6](02-functionality.md#66-observed-work-becomes-a-skill-candidate-as-06) | I02, A02, A04 |
| Input — sources and portability | [REQ-IN-15…16](01-requirements.md#req-in-15) | `[1]` `[4]` | [02](02-functionality.md) [§3.1](02-functionality.md#31-input-capture) | A16, I15, I08, P04 ([IN-15](01-requirements.md#req-in-15)) · I03, I10, P02 ([IN-16](01-requirements.md#req-in-16)) |
| Processing | [REQ-PR-01…15](01-requirements.md#req-pr-01) | `[3]` | [02](02-functionality.md) [§3.2](02-functionality.md#32-processing-reflection-and-extraction) | A01, A03, A07, A13, I01, I02, I03, I10, I12 |
| Memory | [REQ-ME-01…30](01-requirements.md#req-me-01) | `[4]` `[5]` | [02](02-functionality.md) [§3.3](02-functionality.md#33-memory-governed-retention), [§5](02-functionality.md#5-memory-lifecycle-functional-view) | F01, F02, F03, F04, A01, A05, A06, A16, A17, I01, I02, I15, P05, P07 |
| Evaluation | [REQ-EV-01…17](01-requirements.md#req-ev-01) | `[10]` | [02](02-functionality.md) [§3.6](02-functionality.md#36-evaluation-the-feedback-loop) | A09, A10, A11, A12, A13, A14, F02, I01, P07 |
| Output — delivery | [REQ-OUT-01…18](01-requirements.md#req-out-01) | `[6]` `[7]` `[8]` | [02](02-functionality.md) [§3.4](02-functionality.md#34-output-context-delivery) | A06, A07, A15, A16, I01, I07, I08, I09, I13, F04, P07 |
| Scope and promotion | [REQ-SC-01…06](01-requirements.md#req-sc-01) | `[4]` | [02](02-functionality.md) [§4](02-functionality.md#4-scope-and-promotion) | F02, F03, F04, I03, P02 |
| Governance | [REQ-GOV-01…12](01-requirements.md#req-gov-01) | `[9]` | [02](02-functionality.md) [§3.5](02-functionality.md#35-governance-the-control-plane) | A02, A05, A07, I14 |
| Security | [REQ-SEC-01…18](01-requirements.md#req-sec-01) | `[9]`, all | [02](02-functionality.md) [§10](02-functionality.md#10-behaviors-that-hold-everywhere) | I01 (SEC-16…17); otherwise no external research source ([§7](02-functionality.md#7-agentic-flows)) |
| Non-functional | [REQ-NFR-01…11](01-requirements.md#req-nfr-01) | all | [02](02-functionality.md) [§8](02-functionality.md#8-management-dashboard) | A02, A08, A11, A12, A14, I04 |
| Measurement | [04](04-kpi-and-measurements.md) (fulfils [REQ-NFR-07](01-requirements.md#req-nfr-07), [NFR-10](01-requirements.md#req-nfr-10), [NFR-11](01-requirements.md#req-nfr-11), [EV-12](01-requirements.md#req-ev-12)) | `[10]` | [02](02-functionality.md) [§8](02-functionality.md#8-management-dashboard) | A09, A10, A11, A12, A13, A14 |

### 3.1 Acceptance scenarios and open decisions

| Item | Topic | Sources |
|---|---|---|
| [AS-01](01-requirements.md#as-01) | Basic experience transfer | P01, A04, F03 |
| [AS-02](01-requirements.md#as-02) | Contextual contradiction | F02, A13 |
| [AS-03](01-requirements.md#as-03) | Harmful memory recovery | A09, A14 |
| [AS-04](01-requirements.md#as-04) | Policy precedence | I14 |
| [AS-05](01-requirements.md#as-05) | Safe absence | I09, A06 |
| [AS-06](01-requirements.md#as-06) | Observed work to Skill Candidate | I02 |
| [AS-07](01-requirements.md#as-07) | Net value accounting | A12, A11 |
| [AS-08](01-requirements.md#as-08) | Research-agent baseline | A13 |
| [AS-09](01-requirements.md#as-09) | Management dashboard visibility | No external source |
| [AS-10](01-requirements.md#as-10) | Decision lineage | I01 |
| [AS-11](01-requirements.md#as-11) | Self-reinforcing model claim | I01, A06 |
| [AS-12](01-requirements.md#as-12) | Protected regression | I01, A13, A14 |
| [AS-13](01-requirements.md#as-13) | Session continuity | P07 |
| [AS-14](01-requirements.md#as-14) | Dependency change | P07, A16 |
| [AS-15](01-requirements.md#as-15) | Tampered record | I01 |
| [AS-16](01-requirements.md#as-16) | Purpose-dependent grounding | I01 (separation only; see [§5](#5-design-additions-without-an-external-source)) |
| [OD-02](01-requirements.md#od-02) | Minimum experience-unit schema | A03 |
| [OD-03](01-requirements.md#od-03) | Admission thresholds | A17 |
| [OD-08](01-requirements.md#od-08) | Skill compilation and promotion | A05, I01 |
| [OD-10](01-requirements.md#od-10) | Deployment topology | F01, I05 |

## 4. Traceability by concept

Quick lookup for the design ideas that recur across the documents.

| Concept | Where defined | Sources |
|---|---|---|
| Experience vs knowledge | [01](01-requirements.md) [§11](01-requirements.md#11-glossary); [03](03-architecture.md) [§16.1](03-architecture.md#161-why-experience-not-knowledge) | P01, F01, F02 |
| Trajectory as unit of capture | [REQ-IN-03](01-requirements.md#req-in-03); [03](03-architecture.md) [§4](03-architecture.md#4-write-path) | A04, A05, I01 |
| Separation of duties (executor ≠ writer) | [REQ-PR-09](01-requirements.md#req-pr-09); [03](03-architecture.md) [§16.3](03-architecture.md#163-why-the-executing-agent-never-writes-shared-memory) | A01, A07 |
| Incremental deltas, no rewrites | [REQ-ME-09](01-requirements.md#req-me-09) | A01 |
| Evidence vs interpretation | [REQ-IN-11](01-requirements.md#req-in-11), [REQ-PR-03](01-requirements.md#req-pr-03) | I02, I01, I03 |
| Skills separate from memory | [REQ-ME-03](01-requirements.md#req-me-03), [REQ-ME-12](01-requirements.md#req-me-12) | A05, I01, I02 |
| Usage bounds / applicability | [REQ-PR-07](01-requirements.md#req-pr-07) | A03, A13 |
| Deprecation as a lifecycle trigger | [REQ-ME-13](01-requirements.md#req-me-13) | A16, F02 |
| Utility ≠ validity, per version | [REQ-EV-01…04](01-requirements.md#req-ev-01) | A09, A11, A14 |
| Failure attribution | [REQ-EV-06](01-requirements.md#req-ev-06) | A10 |
| Governed memory maintenance | [REQ-ME-26…30](01-requirements.md#req-me-26); [03](03-architecture.md) [§8.1](03-architecture.md#81-memory-maintenance-orchestrator) | P07, A16, A01 |
| Session continuity and compaction | [REQ-ME-19…25](01-requirements.md#req-me-19); [03](03-architecture.md) [§2.3](03-architecture.md#23-working-memory-and-continuity) | P07 |
| Shadow evaluation and risk-calibrated promotion | [REQ-EV-13…17](01-requirements.md#req-ev-13), [REQ-ME-12](01-requirements.md#req-me-12); [03](03-architecture.md) [§11.3](03-architecture.md#113-shadow-evaluation-and-promotion); [04](04-kpi-and-measurements.md) [§4.5](04-kpi-and-measurements.md#45-shadow-evaluation) | I01, A09, A11, A13, A14 |
| Decision lineage and verified outcomes | [REQ-PR-15](01-requirements.md#req-pr-15), [REQ-ME-14](01-requirements.md#req-me-14); [03](03-architecture.md) [§2.1](03-architecture.md#21-decision-lineage) | I01 |
| Integrity is not truth (signed provenance) | [REQ-SEC-16…18](01-requirements.md#req-sec-16); [03](03-architecture.md) [§10.1](03-architecture.md#101-record-integrity-and-attestation) | I01 |
| Net value and no-memory baseline | [REQ-NFR-10](01-requirements.md#req-nfr-10), [REQ-EV-12](01-requirements.md#req-ev-12); [04](04-kpi-and-measurements.md) [§3.7](04-kpi-and-measurements.md#37-net-value), [§4.2–4.3](04-kpi-and-measurements.md#42-net-value-ab-or-paired-protocol) | A12, A13, A11 |
| Minimum sufficient context | [REQ-OUT-01](01-requirements.md#req-out-01) | A15, A06 |
| Proactive delivery before reasoning | [REQ-OUT-04…05](01-requirements.md#req-out-04) | I13 |
| Progressive disclosure L0/L1/L2 | [REQ-OUT-09](01-requirements.md#req-out-09); [03](03-architecture.md) [§6](03-architecture.md#6-progressive-disclosure) | I07 |
| Absence = unknown; fail closed | [REQ-OUT-11](01-requirements.md#req-out-11), [REQ-OUT-13](01-requirements.md#req-out-13) | I09, I08, A06 |
| Stored does not mean grounding-authoritative; autobiographical memory | [REQ-ME-15…18](01-requirements.md#req-me-15), [REQ-OUT-18](01-requirements.md#req-out-18), [AS-16](01-requirements.md#as-16); [03](03-architecture.md) [§2.2](03-architecture.md#22-retention-retrieval-and-grounding) | I01, A06 |
| Projections never authoritative | [03](03-architecture.md) [§7](03-architecture.md#7-canonical-memory-and-retrieval-projections) | A16, I06, P04 |
| Code graph as a derived source | [REQ-IN-15](01-requirements.md#req-in-15) | A16, P04, I15 |
| Semantic identity and typed relations | [REQ-ME-05](01-requirements.md#req-me-05); [03](03-architecture.md) [§7.1](03-architecture.md#71-semantic-identity-and-relationship-typing) | I15, P05, A08 |
| Policy precedence | [REQ-GOV-02](01-requirements.md#req-gov-02) | I14, A07 |
| Expertise location (who knows what) | [REQ-ME-04](01-requirements.md#req-me-04), [REQ-SC-04](01-requirements.md#req-sc-04) | F04 |
| Privacy-safe gap telemetry | [REQ-IN-13](01-requirements.md#req-in-13); [04](04-kpi-and-measurements.md) [§9](04-kpi-and-measurements.md#9-data-handling-for-metrics) | A02 |
| Context layer vs memory | [03](03-architecture.md) [§16.4](03-architecture.md#164-context-layer-versus-memory) | I05, P03 |

---

## 5. Design additions without an external source

These items come from the design work itself, not from a cited resource. They are listed so the gap is visible, not hidden.

| Item | Note |
|---|---|
| [REQ-OUT-17](01-requirements.md#req-out-17), transcript pitfall review ([03](03-architecture.md) [§5.1](03-architecture.md#51-transcript-pitfall-review), flow F5) | A new read-path mode built from existing rules (A06 abstention, A07 advisory framing apply) |
| [REQ-NFR-11](01-requirements.md#req-nfr-11), management dashboard ([02](02-functionality.md) [§8](02-functionality.md#8-management-dashboard), [03](03-architecture.md) [§12.1](03-architecture.md#121-management-dashboard), [AS-09](01-requirements.md#as-09)) | Operational need; no research source |
| Agentic flows F1–F8 ([02](02-functionality.md) [§7](02-functionality.md#7-agentic-flows), [03](03-architecture.md) [§13](03-architecture.md#13-agentic-flows-catalog)) | Composition of existing components |
| Delivery phases ([03](03-architecture.md) [§15](03-architecture.md#15-proposed-delivery-phases--proposal)) and phase gates ([04](04-kpi-and-measurements.md) [§5](04-kpi-and-measurements.md#5-phase-gates)) | Planning proposal |
| [REQ-NFR-09](01-requirements.md#req-nfr-09), scale assumption | Planning input to be set by each adopter |
| [REQ-SEC-18](01-requirements.md#req-sec-18), organization-managed keys and trust roots | From the enterprise security baseline; I01 shows signed provenance but no key-management standard |
| Grounding ceiling, per-request grounding decision, and permitted-use labels ([REQ-ME-15…16](01-requirements.md#req-me-15), [REQ-OUT-18](01-requirements.md#req-out-18), [AS-16](01-requirements.md#as-16)) | Extends I01's retention, retrieval, and grounding separation; the two-level model and the labels are ours |

## 6. Coverage notes and limits

- **Security and compliance (`REQ-SEC-*`)** reflects a common enterprise security baseline, not external research. The exception is signed provenance in `REQ-SEC-16…17`, drawn from I01; a valid signature proves origin and integrity, not that a claim is true. Each adopter should map `REQ-SEC-*` to its own policies.
- **Foundational papers (F01–F04)** were reviewed at abstract and metadata level. Recheck exact claims against the full text before academic publication.
- **Product pages, repositories, and practitioner posts** are architectural inputs, not independent proof of vendor performance claims.
- **Reviewed but excluded:** other repositories and tools were explored but did not lead to an accepted requirement, so they are not listed.
- **Working documents** (drafts, task trackers, prior versions) are destinations of this work, not evidence, and are not listed.

## 7. Maintaining this map

1. **Add a source**: give it the next free ID in its category, a label, pillar tags, a one-line contribution, and the exact `REQ` / `AS` / `OD` / component / section it touches. Update [§3](#3-traceability-by-requirement-area) and, if it adds a concept, [§4](#4-traceability-by-concept).
2. **Add a requirement**: if it has an external source, add it to that source's row and to [§3](#3-traceability-by-requirement-area); if not, add it to [§5](#5-design-additions-without-an-external-source).
3. **Retire a source**: keep the row, set the label to `Retired`, and state why. Never reuse the ID.
4. **Keep it public**: list only public resources. Do not add organization names, people, hosts, IP addresses, internal URLs, services, or policy titles.
5. **Single source of truth**: this file is the only reference list. Other documents point here instead of keeping their own list, and [03-architecture.md](03-architecture.md) [§16](03-architecture.md#16-design-rationale) must not cite a source that is missing here.
