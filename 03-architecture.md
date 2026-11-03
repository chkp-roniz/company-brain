# Company Brain — Architecture

This document describes **how the Company Brain is built**: the object model, the ten logical components, the write and read paths, canonical memory and projections, the lifecycle, the governance plane, the evaluation loop, deployment and topology, delivery phases, and the design rationale. Requirement IDs (`REQ-…`) refer to [01-requirements.md](01-requirements.md). Behavior from the user's view is in [02-functionality.md](02-functionality.md). Metrics are in [04-kpi-and-measurements.md](04-kpi-and-measurements.md).

Normative where RFC 2119 keywords are used. Sections marked **Proposal** are recommended approaches awaiting owner approval.

## 1. Architectural principles

1. **Experience, not documents.** The unit of capture is the trajectory; the unit of memory is the experience unit; delivery is a budgeted selection, not a search result.
2. **Separation of duties.** The agent that does the work never writes shared memory. Reflection proposes, admission decides, memory stores, evaluation corrects.
3. **Governance first, semantics second.** Identity, permissions, policy, validity, and authority filters run before any ranking. Projections are never authoritative. Absence means unknown.
4. **Evidence is the root.** Everything derived points back to immutable evidence. Projections and compact views are rebuildable from canonical memory and evidence.
5. **Harness-neutral.** Interfaces are tool-independent contracts. No component assumes a specific agent runtime ([REQ-NFR-06](01-requirements.md#req-nfr-06)).

## 2. Object model

Seven classes stay distinct. Memory objects are derived from evidence and linked to, never merged with, skills, policies, and authoritative references.

```
   IMMUTABLE EVIDENCE                 GOVERNED MEMORY                      SEPARATE OBJECT CLASSES
   (raw, never rewritten)             (versioned, scoped, owned)           (linked to memory, never merged)

   +----------------+                                                      +----------------------+
   | Evidence event |--+                                                   | Policy               |
   +----------------+  |              +------------------+                 | human-owned          |
   +----------------+  +------------->| Trajectory       |                 | -> CONSTRAINS        |
   | Evidence event |--+              | ordered events   |                 +----------------------+
   +----------------+  |              +---------+--------+                 +----------------------+
   +----------------+  |                        | reflect / extract        | Skill                |
   | Evidence event |--+                        | derived_from             | validated, versioned |
   +----------------+                           v supported_by             | -> EXECUTES          |
                                      +---------+--------+                 +----------------------+
                                      | Experience unit  |                 +----------------------+
                                      | contextual lesson|                 | Authoritative ref    |
                                      +---------+--------+                 | official source of   |
                                                | promoted_from            | truth -> REFERENCED  |
                                                v evidence across contexts +----------------------+
                                      +---------+--------+
                                      | Durable knowledge|
                                      | claim + evidence |
                                      +------------------+

   cross-links between memory units: contradicts . supersedes . relevant_to
   memories and lessons ADVISE . skills EXECUTE . policies CONSTRAIN . authoritative sources stay OFFICIAL
```

| Class | What it is | Mutability | Key attributes |
|---|---|---|---|
| Evidence event | A source event: message, tool call and result, decision, correction, review, outcome, observation | Immutable | id, time, actor, source system, authorization context, classification, payload reference |
| Trajectory | Ordered evidence describing one piece of work | Append-only until closed | id, objective, events[], boundaries (completion, failure, correction, decision), outcome |
| Experience unit | A contextual lesson inferred from one or more trajectories | Versioned | all [REQ-ME-04](01-requirements.md#req-me-04) attributes, usage bounds, lifecycle state, source class, retention and retrieval decisions, grounding ceiling and permitted uses ([§2.2](#22-retention-retrieval-and-grounding)) |
| Durable knowledge | A claim supported across sufficient evidence and contexts | Versioned | as above, plus the set of supporting experience units and contexts |
| Skill | An executable capability | Versioned, own lifecycle | validation status, rollout state, link to the knowledge it was compiled from |
| Policy | A human-owned, authoritative constraint | Human-edited only | owner, precedence, scope of application, version |
| Authoritative reference | Pointer to the official source of truth | Reference | system, locator, last verified |

Related derived objects: a **Reconstruction** (interpretation of observed work, linked step by step to evidence); a **Skill Candidate** (a generalized procedure that has not passed the [REQ-ME-12](01-requirements.md#req-me-12) gates); a **code-graph entity or relation** (tagged with repository, commit, and builder version); **decision-lineage records** (Proposal, Decision, Execution, Outcome; [§2.1](#21-decision-lineage)).

Required relationships: `derived_from`, `supported_by`, `contradicts`, `supersedes`, `relevant_to`, `promoted_from`, and the decision-lineage links in [§2.1](#21-decision-lineage). Relationships are typed, directional, versioned, and carry provenance and confidence.

### 2.1 Decision lineage

A trajectory records decisions and outcomes as evidence events. Decision lineage turns them into linked records, so the Brain can tell what was suggested, what an authorized actor chose, what actually ran, and what was later verified ([REQ-PR-15](01-requirements.md#req-pr-15), [REQ-ME-14](01-requirements.md#req-me-14)).

```
   proposer                          deciding actor              executing                    evaluator
   (person or agent)                 + authority                 identity                     + criterion
         ^                                 ^                           ^                            ^
         | proposed_by                     | decided_by                | executed_by                | evaluated_by
   +-----+------+                    +-----+------+              +-----+------+               +-----+------+
   | Proposal   |<--decision_selects-| Decision   |<--implements-| Execution  |--resulted_in->| Outcome    |
   | what was   |                    | what an    |              | what       |               | what was   |
   | suggested  |                    | authorized |              | actually   |               | observed   |
   +-----+------+                    | actor chose|              | ran        |               +------------+
         |                           +------------+              +------------+
         | informed_by   exposure only: the unit version was available,
         v               not proof that it changed the Proposal
   memory unit versions

   a Proposal no Decision selects stays in history, never shown as the decision taken
   Outcome with a criterion checked --> VERIFIED OUTCOME; without one --> unverified observation
   every record: actor . authority . timestamp . evidence links; a missing stage stays unknown
   a correction is a new linked record; the original is never overwritten
```

| Record | Created by | Required attributes |
|---|---|---|
| Proposal | [3], at a decision boundary | proposer (person or agent), content, time, evidence; `informed_by` links to the memory unit versions supplied at that moment |
| Decision | [3], at a decision boundary | deciding actor, authority, the Proposal selected (if any), time, evidence |
| Execution | [3], from tool calls and results | executing identity, the Decision it implements, what ran, time, evidence |
| Outcome | [10], from later evidence | evaluator, acceptance criterion, result, uncertainty, verified flag, time, evidence |

Rules:

- A record is created only when evidence supports its stage. Missing stages stay unknown; the chain is never completed by inference.
- An Outcome is verified only when an evaluator applied an explicit acceptance criterion. A positive signal without one is an unverified observation.
- Lineage records are append-only. A correction is a new record linked to the original; nothing is overwritten.
- Edges point from the later or dependent record to the earlier one or to its actor; only `resulted_in` points forward. Actors are linked by `proposed_by`, `decided_by`, `executed_by`, and `evaluated_by`; the Decision-to-Proposal edge is `decision_selects`.
- `informed_by` records exposure only. It is the starting point for attribution, not its result ([§11](#11-evaluation-loop)).
- Lineage records follow the visibility of their most restrictive evidence ([REQ-GOV-10](01-requirements.md#req-gov-10)).

### 2.2 Retention, retrieval, and grounding

Retention, retrieval, and grounding are separate decisions ([REQ-ME-15](01-requirements.md#req-me-15)). Retention and retrieval rules and a grounding ceiling are stored on the unit; grounding for a specific request is decided at read time. Storing or finding a unit never makes it trusted.

| Decision | Question | Decided by | Inputs |
|---|---|---|---|
| Retention | Is the unit kept? | [4] at admission; lifecycle in [5] | scope, policy, lifecycle state |
| Retrieval | May it be returned to this consumer? | [7] step 3 | identity, permissions, visibility, scope, validity |
| Grounding ceiling | What is the most it may ever support, and which non-grounding uses may it serve? | [4] at admission; versioned in [5] | source class, supporting evidence (including verified outcomes, [§2.1](#21-decision-lineage)), authority, validity, policy ([REQ-ME-16](01-requirements.md#req-me-16)) |
| Grounding decision | May it support this claim or action, for this requester and purpose, now? | [7] step 3; recorded in the selection log | the ceiling, freshness, visibility, identity, purpose, task context, consequence of the action, policy |

Source classes include authoritative reference, policy, tool observation, person statement, verified outcome, and **autobiographical** (assistant-generated beliefs, summaries, self-assessments, prior utterances). Autobiographical units are non-grounding by default ([REQ-ME-17](01-requirements.md#req-me-17)).

A unit delivered without grounding carries a **permitted-use label** ([REQ-OUT-18](01-requirements.md#req-out-18)):

| Label | Use |
|---|---|
| `continuity` | Resume or explain what was said or done before |
| `hypothesis` | A lead to check; never a premise |
| `disputed` | One side of an open contradiction, shown with the other side |
| `audit` | Context for reviewing past behavior |
| `reflection` | Input to the experience processor [3] |

Owners may add labels; none of them permits grounding.

Rules:

- The grounding ceiling and its reason are versioned; raising or lowering it is a new version with the evidence linked. A per-request decision is not a unit change: it is logged with the selection (read path step 8, [§5](#5-read-path)) and never exceeds the ceiling.
- A unit derived from an autobiographical unit (a summary, a merge, a reconstruction) inherits its non-grounding status unless independent evidence is linked ([REQ-ME-18](01-requirements.md#req-me-18)).
- Repetition is not corroboration. The count of statements, retrievals, or copies never enters the grounding decision.
- A unit's grounding ceiling is never higher than the ceiling of the evidence it rests on.

### 2.3 Working memory and continuity

Working memory is temporary state for one session, task, or handoff ([REQ-ME-19](01-requirements.md#req-me-19)). It lives in a separate working-memory area of [5] and never mixes with shared memory.

```
   SESSION / TASK   working memory: objective, state, constraints, open decisions
          |
          |  during work: checkpoint update on each material state change + periodic
          |               irreversible action -> ACTION JOURNAL: intent + key, then result
          |  trigger: session end . handoff . reset . compaction   ([8] hook)
          v
   1. DETERMINISTIC CLEANUP   remove obsolete and duplicate items by rule;
          |                   no model call; every removal names its rule
          v
   2. SEMANTIC COMPACTION     summarize what remains; keep provenance, roles,
          |                   authority boundaries, open state, evidence links,
          |                   grounding status
          v
   3. CONTINUITY CHECKPOINT  ---------->  [5] working-memory area
      COMPACTION RECORD      ---------->  [2] as evidence
          |                               retained . removed . transformed . uncertain
          v
   4. RESUME   [7] loads the checkpoint for the same task only
               after an abrupt end: latest checkpoint + journal, reconciled with
               external state; intent without result -> check, never re-run blindly
               resume tests: invented state . repeated work . lost constraints
          |
          +--> durable memory only through the normal write path: [3] -> [4]
```

| Object | What it is | Where | Key attributes |
|---|---|---|---|
| Working-memory state | Temporary operational state of one session or task | [5] working-memory area | owner, session or task id, scope, expiry |
| Continuity checkpoint | What the task needs to resume ([REQ-ME-20](01-requirements.md#req-me-20)) | [5] working-memory area | objective, completed and pending work, unresolved decisions, constraints, dependencies, evidence links, authority boundaries, next actions, source session, time, age |
| Action journal entry | One irreversible or externally visible action of a task ([REQ-ME-31](01-requirements.md#req-me-31)) | [5] working-memory area, append-only | task id, action, target, idempotency key, intent time, result, result time |
| Compaction record | Audit of one compaction ([REQ-ME-23](01-requirements.md#req-me-23)) | [2] | trigger, retained, removed (with rule), transformed, uncertain, checkpoint id, time |

Rules:

- [7] returns a checkpoint only to resume its own task, to identities allowed on that task. It is never returned as organizational knowledge ([REQ-ME-21](01-requirements.md#req-me-21)).
- Nothing in a checkpoint is admitted or promoted automatically. Content that should last goes through [3] and [4] like any other evidence ([REQ-PR-09](01-requirements.md#req-pr-09), [REQ-SC-02](01-requirements.md#req-sc-02)).
- Deterministic cleanup runs before semantic compaction ([REQ-ME-22](01-requirements.md#req-me-22)).
- Compaction keeps grounding status: autobiographical content stays autobiographical in the checkpoint ([§2.2](#22-retention-retrieval-and-grounding)).
- Uncertain items stay marked as uncertain in the checkpoint and the compaction record; a clean-looking summary is not a substitute ([REQ-ME-23](01-requirements.md#req-me-23)).
- Resume tests run in [10]; a failure is recorded as evidence ([REQ-ME-24](01-requirements.md#req-me-24)).
- Hidden model reasoning is not stored as required memory; decisions, evidence, corrections, and observable outcomes are ([REQ-ME-25](01-requirements.md#req-me-25)).
- **Crash safety.** [8] updates the checkpoint after each material state change and on a timer, not only at end triggers. Each irreversible or externally visible action is journaled with its intent and idempotency key before it runs and with its result after. On resume after an abrupt end, the checkpoint and journal are reconciled with external state; an action with an intent and no result is checked, never re-run blindly. Checkpoint age is recorded against the task class's recovery point ([REQ-ME-31](01-requirements.md#req-me-31)).
- Working-memory state, checkpoints, and journals expire with their scope under [9] retention rules.

## 3. Logical architecture

Ten components. The write path runs left to right along the top; the read path runs right to left below it; the evaluation loop closes the cycle.

```
   +-------------------------------------------------------------------------------------------+
   | [9] GOVERNANCE CONTROL PLANE                                                              |
   | identity . authorization . policy precedence . audit . expiry . rollback . scope control  |
   +-------+------------------+------------------+------------------+------------------+-------+
           |                  |                  |                  |                  |
           v                  v                  v                  v                  v
   +---------------+  +---------------+  +---------------+  +---------------+  +---------------+
   | [1] Source    |  | [2] Evidence  |  | [3] Experience|  | [4] Admission |  | [5] Managed   |
   | adapters      |->| & trajectory  |->| processor     |->| & promotion   |->| memory store  |
   | systems of    |  | log           |  | reflect,      |  | service       |  | canonical,    |
   | record +      |  | immutable,    |  | extract,      |  | validate,     |  | versioned,    |
   | work trails   |  | append-only   |  | classify,     |  | owner, scope, |  | linked,       |
   |               |  |               |  | contradict    |  | policy gates  |  | scoped units  |
   +---------------+  +---------------+  +---------------+  +---------------+  +-------+-------+
                                                                                       |  project / index
                                                                                       v
                                         +---------------+  +---------------+  +---------------+
                                         | [8] Output    |  | [7] Retrieval |  | [6] Retrieval |
   agents, automations, apps, humans   <-| adapters      |<-| & context     |<-| projections   |
                                         | (hooks, MCP,  |  | composer      |  | graph, vector,|
                                         |  API, UI)     |  | session-aware,|  | text, SQL     |
                                         |               |  | budgeted,     |  | derived from  |
                                         |               |  | scoped        |  | [5], rebuild- |
                                         |               |  |               |  | able          |
                                         +-------+-------+  +---------------+  +---------------+
                                                 |  outcome signals: helped / misled / ignored / corrected
                                                 v
                                         +-------------------+
                                         | [10] Evaluation   |--> outcome events to [2] as new evidence;
                                         | loop              |    utility history to units in [5]
                                         | attribution,      |
                                         | utility by version|
                                         +-------------------+
```

| # | Component | Responsibility | Primary interfaces | Key requirements |
|---|---|---|---|---|
| [1] | Source adapters | Connect systems of record and work surfaces; emit normalized evidence events carrying identity, time, source, and authorization context. Includes harness exporters, repository, ticket, document, and database connectors, the code-graph adapter, the memory import adapter, and the work-observation recorder | Write-once ingest API into [2] | [REQ-IN-01…16](01-requirements.md#req-in-01), [REQ-GOV-03](01-requirements.md#req-gov-03) |
| [2] | Evidence and trajectory log | Immutable, append-only store of events and trajectories; the audit root for everything derived; holds integrity attestations for verified outcome records ([§10.1](#101-record-integrity-and-attestation)) | Write-once ingest; read by [3], [10], and by L2 evidence expansion | [REQ-IN-03](01-requirements.md#req-in-03), [REQ-IN-06](01-requirements.md#req-in-06), [REQ-PR-04](01-requirements.md#req-pr-04), [REQ-GOV-05](01-requirements.md#req-gov-05), [REQ-SEC-16](01-requirements.md#req-sec-16) |
| [3] | Experience processor | Boundary detection, reflection, extraction, classification, deduplication, contradiction detection, generalization of reconstructions; produces candidate units | Consumes [2]; emits candidates to [4]; calls approved LLM endpoints only ([REQ-SEC-08](01-requirements.md#req-sec-08)) | [REQ-PR-01…15](01-requirements.md#req-pr-01) |
| [4] | Admission and promotion service | Validation, sensitivity handling, ownership, scope and policy gates; versioned admission; promotion and demotion workflows; Skill Candidate gating; signs admission decisions and verifies attestations before promotion | Consumes candidates; writes to [5]; owner review queue | [REQ-PR-10](01-requirements.md#req-pr-10), [REQ-ME-10](01-requirements.md#req-me-10), [REQ-ME-12](01-requirements.md#req-me-12), [REQ-SC-01…06](01-requirements.md#req-sc-01), [REQ-GOV-02…04](01-requirements.md#req-gov-02), [REQ-SEC-16](01-requirements.md#req-sec-16) |
| [5] | Managed memory store | Canonical, itemized, versioned, linked memory units with lifecycle state; a separate working-memory area for temporary state, continuity checkpoints, and action journals ([§2.3](#23-working-memory-and-continuity)); attestation and verification status per high-impact unit version ([§10.1](#101-record-integrity-and-attestation)) | Versioned CRUD; change feed to [6]; rollback API | [REQ-ME-01…31](01-requirements.md#req-me-01), [REQ-SEC-16](01-requirements.md#req-sec-16) |
| [6] | Retrieval projection layer | Derived indexes (vector, graph, full-text, relational) rebuilt from [5]; invalidated on any canonical change; repaired by the maintenance orchestrator ([§8.1](#81-memory-maintenance-orchestrator)) | Change feed from [5]; query API for [7] | [REQ-OUT-10](01-requirements.md#req-out-10), [§7](#7-canonical-memory-and-retrieval-projections) |
| [7] | Retrieval and context composer | Objective inference, deterministic filtering (including grounding eligibility and integrity verification), ranking, budgeted selection, L0/L1 composition, selection logging; transcript pitfall review mode | Retrieval API called by [8]; reads [6], [5], [2] | [REQ-OUT-01…18](01-requirements.md#req-out-01), [REQ-SEC-16](01-requirements.md#req-sec-16) |
| [8] | Output adapters | Harness hook handlers, MCP server, APIs, UI, export; deliver context in-tool; collect outcome signals; trigger continuity checkpoints at session end, handoff, reset, and compaction, and update them during work; journal irreversible actions | `SessionStart` / `UserPromptSubmit` equivalents; MCP tools; REST or gRPC | [REQ-OUT-04…07](01-requirements.md#req-out-04), [REQ-OUT-16](01-requirements.md#req-out-16), [REQ-OUT-17](01-requirements.md#req-out-17) |
| [9] | Governance control plane | Identity, permissions, policy objects and precedence, audit, expiry, deletion, rollback, deprecation transitions; managed integrity and authenticity controls (signing identities, trust roots, rotation, revocation; [§10.1](#101-record-integrity-and-attestation)); governs the maintenance orchestrator and its low-risk action list ([§8.1](#81-memory-maintenance-orchestrator)) | Consulted by every component; policy and owner administration; management dashboard | [REQ-GOV-01…12](01-requirements.md#req-gov-01), REQ-SEC-* , [REQ-NFR-11](01-requirements.md#req-nfr-11) |
| [10] | Evaluation loop | Join selection logs with outcomes; verify outcomes against explicit acceptance criteria; attribute by evidence level (exposure, influence, causal) along decision lineage; update utility per version; trigger weakening or suspension; manage safe canaries; run shadow evaluation and risk-calibrated promotion evidence; net value accounting; research-agent baseline | Reads [8] selection logs and [2] outcomes; writes utility to [5] | [REQ-EV-01…19](01-requirements.md#req-ev-01), [REQ-NFR-10](01-requirements.md#req-nfr-10) |

Interfaces between components MUST be tool-independent. No component may assume a specific agent harness ([REQ-NFR-06](01-requirements.md#req-nfr-06)).

## 4. Write path

The write path converts work into admitted memory. It is asynchronous by default and optimizes for evidence quality and safe admission, not latency ([REQ-PR-12](01-requirements.md#req-pr-12)).

```
   EXECUTION            REFLECTION             CURATION / ADMISSION         MEMORY
   does the work        learns from it         decides what enters          stores and versions

   +----------------+   +------------------+   +------------------------+   +------------------+
   | agent / human  |   | Reflector        |   | Curator / Admission    |   | Managed memory   |
   | performs task  |   | detects boundary |   | links to evidence and  |   | store            |
   | emits events   |-->| (done / failed / |-->| usage bounds           |-->| adds / updates   |
   | (tool calls,   |   |  corrected /     |   | dedup + contradiction  |   | affected units   |
   |  corrections,  |   |  decided)        |   | check vs existing      |   | only; keeps      |
   |  outcomes)     |   | extracts         |   | sensitivity, owner,    |   | prior versions   |
   |                |   | candidate lessons|   | scope, policy gates    |   | (rollback)       |
   +----------------+   +------------------+   +------------------------+   +------------------+
   no direct writes     proposes only          admits / rejects / defers    incremental deltas
   to shared memory     (no authority)         per policy and owner         never bulk rewrites
```

Steps:

1. Capture events and trajectories ([1] → [2]).
2. Normalize and classify sensitive content ([REQ-PR-10](01-requirements.md#req-pr-10)).
3. Detect meaningful completion, failure, correction, or decision boundaries ([REQ-PR-11](01-requirements.md#req-pr-11)).
4. Reflect on the trajectory and generate candidate lessons ([REQ-PR-01](01-requirements.md#req-pr-01), [REQ-PR-02](01-requirements.md#req-pr-02)).
5. Link every candidate to supporting evidence and usage bounds ([REQ-PR-04](01-requirements.md#req-pr-04), [REQ-PR-07](01-requirements.md#req-pr-07)).
6. Compare with existing memory for duplicates and contradictions ([REQ-PR-05](01-requirements.md#req-pr-05), [REQ-PR-06](01-requirements.md#req-pr-06)).
7. Apply validation, ownership, scope, and admission rules ([4]; thresholds per [OD-03](01-requirements.md#od-03)).
8. Add or update only the affected memory units ([REQ-ME-09](01-requirements.md#req-me-09)).
9. Preserve previous versions and the ability to roll back ([REQ-ME-11](01-requirements.md#req-me-11)).

Design rules:

- The executing agent has no credential that can write to [5]. Reflection and curation run as separate services with their own scoped identities ([REQ-PR-09](01-requirements.md#req-pr-09), [REQ-SEC-10](01-requirements.md#req-sec-10)).
- Candidates are idempotent: re-processing the same trajectory produces the same candidate identity, so retries never duplicate memory.
- At a decision boundary, [3] creates or extends the decision-lineage chain ([§2.1](#21-decision-lineage)); lineage records pass admission like any other candidate. Outcome and verification records from [10] enter [2] as new evidence first.
- Admission writes are versioned deltas. There is no code path that rewrites the whole store or a whole scope with a single model call.
- Every admitted unit records the admission rule and actor that let it in, so an admission rule can later be found and rolled back.
- Imported memory ([REQ-IN-16](01-requirements.md#req-in-16)) enters at step 1 through a source adapter, keeps its original provenance, and passes steps 2 to 9 like any other input.

### 4.1 Observed-work capture

Observation is the richest input the system has and the most sensitive. The pipeline below applies to every observation-based adapter, from harness hooks to a screen-level recorder.

```
   OBSERVE real work on the workstation
   screen, window and application switches, pages visited, clipboard previews,
   terminal commands and output, tool calls, optional narration
        |  recording is visible; the employee can pause, stop, or discard it
        v
   RAW EVIDENCE  (immutable events with provenance; secrets and personal data
        |         redacted ON DEVICE before anything leaves the workstation)
        v
   RECONSTRUCTION  (interpretation: one intent + ordered steps; every step linked
        |           to the evidence that supports it; stored BESIDE the evidence,
        |           never instead of it; unsupported steps marked as inferred)
        v
   EMPLOYEE REVIEW  (correct, redact, or reject; the decision is itself evidence;
        |            review is not admission, not policy clearance, not a security
        |            guarantee)
        v
   GENERALIZE  (express steps by intent; prefer native tools - CLI, API, connector -
        |       over replaying clicks; parameterize the single observed case;
        |       mark the steps that stay UI-bound)
        v
   CANDIDATE  (Experience unit or SKILL CANDIDATE - never a Skill, durable knowledge,
        |      or policy at this point)
        v
   GATES  a Skill Candidate becomes a Skill only when ALL pass; each result is recorded
     [ ] compared with additional examples of the same intent
     [ ] checked against applicable policy
     [ ] evaluated on real outcomes in a safe environment, canary, or shadow;
         evidence across distinct contexts meets the risk-set threshold;
         no protected regression
     [ ] approved by an owner
     [ ] versioned
     [ ] reversible: can be disabled and rolled back
        |
        v
   SKILL  (executes; own lifecycle; still subject to weakening, suspension, rollback)
```

Design rules:

- The reconstruction is an interpretation object ([REQ-PR-03](01-requirements.md#req-pr-03)) with `derived_from` links to its evidence events. Weakening or deleting a reconstruction never touches the evidence; expiring evidence ([REQ-GOV-07](01-requirements.md#req-gov-07)) invalidates every reconstruction built on it.
- Employee review happens before admission, in the employee's own tool, and records edits as new evidence. It does not give the employee a write path to shared memory ([REQ-PR-09](01-requirements.md#req-pr-09)); the reviewed reconstruction still enters through [4].
- Generalization output is auditable: a Skill Candidate lists which steps map to native tools, which parameters were extracted from the single observed case, and which steps remain UI-bound.
- A Skill Candidate may run only in the evaluation sandbox ([REQ-EV-08](01-requirements.md#req-ev-08)) until the [REQ-ME-12](01-requirements.md#req-me-12) gates pass. Admitting the candidate as an Experience unit and promoting it to a Skill are two separate decisions with separate records.
- On-device redaction and employee controls are part of the adapter, not of a later release. Evidence travels only to organization-controlled ingest, and reconstruction runs only on approved endpoints ([REQ-IN-14](01-requirements.md#req-in-14)).
- Relative to public desktop-recorder reference implementations: the observe → reconstruct → review → generalize pattern is kept; sending recording data to a third-party cloud for analysis is not ([REQ-IN-14](01-requirements.md#req-in-14)); turning one approved run directly into an installed skill is not ([REQ-ME-12](01-requirements.md#req-me-12)).

## 5. Read path

Two modes. The **proactive** path is primary: memory arrives before the model reasons, triggered by harness lifecycle events. The **exploratory** path (MCP or API) lets a consumer inspect sources and expand evidence; it is never the only way memory reaches the model ([REQ-OUT-04](01-requirements.md#req-out-04), [REQ-OUT-07](01-requirements.md#req-out-07)). A third request type, **transcript pitfall review** ([§5.1](#51-transcript-pitfall-review)), runs on the same service and the same filters; it is a mode of the read path, not a third path.

```
   HARNESS                RETRIEVAL SERVICE (proactive read path)
   lifecycle hook
   SessionStart /         +---------------------------------------------------------+
   UserPromptSubmit       |  1. infer objective from prompt + session/task state    |
                          |  2. candidate recall: semantic . structural . temporal  |
   prompt + task/ ------->|     . scope signals                                     |
   session state          |  3. DETERMINISTIC FILTERS FIRST: identity, permissions, |
                          |     policy, validity, authority, explicit resource scope|
                          |  4. rank: relevance . confidence . freshness . utility  |
                          |  5. select smallest set that fits the context budget    |
                          |     (empty result is valid; unknown != permitted)       |
                          |  6. compose L0/L1 with provenance and confidence        |
                          |  7. log selection: units, query form, filters, reasons  |
                          +----------------------------+----------------------------+
                                                       |  context injected BEFORE
                                                       v  the model starts reasoning
                                        +------------------------------+
                                        | REASONING MODEL / AGENT      |
                                        | acts with injected context;  |
                                        | may call MCP tools to        |
                                        | inspect sources, expand L2,  |
                                        | report outcome               |
                                        +--------------+---------------+
                                                       |
                                                       v
                                        OUTCOME: helped / misled / ignored / corrected
                                        -> recorded as new evidence (feeds Input)
```

Steps (REQ-OUT-*):

1. Infer the current session objective and constraints from the prompt plus available task and session state (composition per [OD-01](01-requirements.md#od-01)).
2. Retrieve candidates by semantic, structural, temporal, and scope signals from [6].
3. Enforce identity, permissions, policy, validity, authority, explicit resource scope, per-request grounding eligibility, and integrity verification of high-impact records **before** ranking ([REQ-OUT-18](01-requirements.md#req-out-18), [REQ-SEC-16](01-requirements.md#req-sec-16)). A record that fails verification is quarantined and dropped ([§10.1](#101-record-integrity-and-attestation)). Units that may be retrieved but not used for grounding in this request stay in the set with a permitted-use label.
4. Rank by relevance, confidence, freshness, and demonstrated utility for this version, task class, and harness.
5. Select the smallest set that fits the declared context budget. An empty set is a legitimate answer.
6. Present L0 or L1 with source, confidence, usage boundaries, and grounding status, framed as advisory data ([REQ-OUT-16](01-requirements.md#req-out-16)). Units with a non-grounding label are marked so that neither a person nor a model cites them as fact.
7. Allow the consumer to expand to L2 and inspect the underlying evidence.
8. Record selection details, including each unit's per-request grounding decision, purpose, and label ([REQ-OUT-08](01-requirements.md#req-out-08)) and, later, downstream use and outcome signals ([REQ-EV-10](01-requirements.md#req-ev-10)).

Design rules:

- Interactive retrieval reads precomputed projections and L0/L1 representations. It never triggers expensive composition on every turn ([REQ-OUT-14](01-requirements.md#req-out-14)).
- Filters are deterministic and testable without a model in the loop. A ranking change can never widen what a caller is allowed to see.
- The retrieval service accepts the prompt and state; it does not require the harness to understand memory internals. Harness-specific logic lives in [8], not in [7].

### 5.1 Transcript pitfall review

Transcript pitfall review ([REQ-OUT-17](01-requirements.md#req-out-17)) changes the input of the read path, not its rules. Instead of a prompt and session state, the input is the full transcript of an agent conversation: live (a growing transcript passed at a checkpoint) or completed.

Steps:

1. [8] receives the transcript and the caller's identity and passes both to [7]. The transcript is untrusted input: special tokens are stripped or escaped before any model sees it ([REQ-SEC-06](01-requirements.md#req-sec-06)).
2. [7] derives the objectives, tools, claims, and decisions in the transcript. This replaces step 1 of the proactive path with inference from the whole conversation.
3. Candidate recall from [6] and deterministic filters run exactly as in steps 2 and 3 of the proactive path.
4. Each surviving item is classified as: an applicable lesson; a known pitfall the work is heading into or has hit; a contradiction between a claim or action in the transcript and admitted memory; or a policy conflict. A policy conflict is shown as a constraint with its human owner, never paraphrased as advice ([REQ-GOV-02](01-requirements.md#req-gov-02)).
5. [7] selects the smallest set that fits the budget and composes L0 or L1 with source, confidence, and usage bounds, framed as advisory data. An empty or fully filtered result is reported as unknown.
6. The selection is logged, and what the agent did next (changed course, ignored, corrected) is recorded as outcome signals for the evaluation loop.

Rules: no new component; the review never writes to [5]; a contradiction it finds is returned to the consumer and recorded as an evidence event, with owner routing in the contradiction triage flow (F3); a live transcript is reviewed at checkpoints, not on every turn ([REQ-OUT-14](01-requirements.md#req-out-14)).

## 6. Progressive disclosure

```
   +----------------------------+   +----------------------------+   +----------------------------+
   | L0  compact relevance      |   | L1  navigational overview  |   | L2  full managed unit      |
   |     signal                 |-->|     what exists, how it    |-->|     + supporting evidence  |
   |     very small, cheap      |   |     relates, confidence    |   |     + source trajectory    |
   |     used for selection     |   |     decide what to inspect |   |     on demand only         |
   +----------------------------+   +----------------------------+   +----------------------------+
   derived, rebuildable,            derived, rebuildable,            canonical, governed,
   linked to L2                     linked to L2                     versioned, audited
```

- L0 and L1 are derived views of canonical units and MUST be rebuildable from [5] and [2] at any time ([REQ-OUT-10](01-requirements.md#req-out-10)).
- Every L0 or L1 element carries the identity of its canonical unit so a consumer can always get to L2 and to the evidence.
- Every L0 or L1 element carries the grounding ceiling and permitted-use labels of its canonical unit and can never widen them ([REQ-OUT-18](01-requirements.md#req-out-18)).
- The context budget is spent on L0/L1 by default; L2 is pulled only on request or when the task class requires it.

## 7. Canonical memory and retrieval projections

```
   SOURCE EVIDENCE            CANDIDATE CLAIMS           ADMITTED MEMORY             RETRIEVAL PROJECTIONS
   what happened              probabilistic              governed record             optimized views
   +------------------+       +------------------+       +------------------+        +------------------+
   | events,          |       | extracted        |       | managed memory   |        | vector index     |
   | trajectories     |------>| entities,        |------>| units: versioned,|=======>| graph            |
   | tool results     |       | relations,       |  gate | owned, scoped,   | derive | full-text        |
   | outcomes         |       | summaries,       |       | linked, audited  |        | relational views |
   |                  |       | candidate lessons|       |                  |        |                  |
   +------------------+       +------------------+       +--------+---------+        +--------+---------+
                                                                  ^                           |
                                                                  |   every retrieved claim   |
                                                                  +----- resolves back to ----+
                                                                      its managed unit and its evidence

   writing a claim to a graph or index does NOT make it authoritative
   when a unit is weakened / superseded / archived / expired / deleted,
   every projection built from it is invalidated or updated
```

Rules:

- Four things stay distinct: source evidence, extracted candidate claims and relationships, admitted managed memory units, and retrieval projections.
- Probabilistically extracted entities, relationships, summaries, and lessons MUST NOT become authoritative memory merely because they were written to a graph or an index.
- Projections MAY use graph, vector, textual, relational, or hybrid representations. The representation never changes the governance status of the underlying unit.
- Every retrieved claim or relationship preserves a path to its managed memory unit and supporting evidence.
- Every projection entry inherits the retrieval and grounding restrictions of its canonical unit. A projection can never widen grounding rights; when the grounding ceiling or permitted uses change in [5], affected entries are invalidated like any other change ([REQ-OUT-18](01-requirements.md#req-out-18)).
- When a canonical unit is weakened, superseded, archived, expired, or deleted, every affected projection MUST be invalidated or updated. Projection staleness is a correctness bug, not an optimization problem.

Recommended hybrid store: immutable evidence log + text/vector store + graph + relational lifecycle store. The graph is a projection and a relations layer, never the authority.

### 7.1 Semantic identity and relationship typing

- Entity identity MUST NOT be inferred from semantic similarity alone. The system maintains stable canonical entity identifiers and explicitly typed relationships, while preserving source identifiers, provenance, confidence, and reversible entity-resolution decisions.
- Ontology constraints MAY validate candidate claims, but MUST NOT by themselves promote them to trusted memory.
- Ontologies and relationship types are versioned so that a semantic change never silently reinterprets existing memory.
- Code-graph entities and relations ([REQ-IN-15](01-requirements.md#req-in-15)) follow the same rules and carry repository, commit, and builder version so they can be re-derived and invalidated.

## 8. Memory lifecycle

```
                 admit              promote (wider scope)
   CANDIDATE ----------> ADMITTED ------------------------> PROMOTED
       |                    | ^                                |
       | reject             | | re-validate                    |
       v                    v |                                v
   DISCARDED             WEAKENED  <---------------------------+   harmful outcomes, contradiction,
   evidence kept            |                                      low utility, deprecation
                            | expire / supersede / archive
                            v
                         ARCHIVED  ---- governed deletion ---->  DELETED
                         not retrieved; history and evidence preserved
```

| Transition | Trigger | Who / what decides | Reversible |
|---|---|---|---|
| Candidate → Admitted | Passes validation, sensitivity, ownership, and policy gates | [4], per [OD-03](01-requirements.md#od-03) thresholds | Yes (rollback) |
| Candidate → Discarded | Fails admission | [4] | Evidence retained; can be re-proposed |
| Admitted → Promoted | Meets scope gate ([01-requirements.md](01-requirements.md) [§5.6](01-requirements.md#56-scope-and-promotion-component-4)) | Owner at target scope via [4] | Yes (demotion) |
| Admitted / Promoted → Weakened | Harmful or ignored outcomes, contradiction, low utility | [10], owner | Yes (re-validation) |
| Any → Superseded | A newer version replaces the unit | [4] | Old version archived, not overwritten |
| Weakened / Superseded → Archived | Expiry, retention, or owner action | [9], owner | Yes, within retention |
| Archived → Deleted | End of retention or governed deletion request | [9] only | No; secure deletion or anonymization ([REQ-SEC-15](01-requirements.md#req-sec-15)) |
| Admitted / Promoted / Weakened → Weakened or Archived (deprecation) | Evidence that a scope or source is deprecated ([REQ-ME-13](01-requirements.md#req-me-13)) | Deprecation detection flow (F4) proposes with evidence; [4] and [9] apply; owners notified | Yes, by re-validation or rollback; history and evidence preserved |
| Skill Candidate → Skill | All [REQ-ME-12](01-requirements.md#req-me-12) gates pass | Owner via [4], with evaluation evidence from [10] | Yes (disable or roll back; candidate and evidence kept) |

Every transition is versioned, attributed, and auditable. Forgetting means a unit stops influencing current work; its history stays until a governed deletion removes it.

### 8.1 Memory maintenance orchestrator

The orchestrator turns lifecycle rules into a running process ([REQ-ME-26](01-requirements.md#req-me-26)). It is a subservice spanning [5], [6], [9], and [10], not a new component. It runs under its own service identity, reads signals, and either runs an approved low-risk action or files a proposal. It never commits a change to canonical memory itself.

```
   TRIGGERS
     schedule . threshold crossed . event (source, tool, schema, model change;
     review date due; working-memory expiry)
          |
          v
   MONITOR (REQ-ME-27)
     growth . latency . duplicates . contradictions . staleness . expiry
     harmful / ignored units [10] . skill drift . changed dependencies
     due reviews . projection vs canonical [5]/[6]
          |
          v
   DEPENDENCY WALK   derived_from / supported_by / compiled_from links
          |          find every unit, skill, and projection entry affected
          v
   CLASSIFY the needed action
          |
          +--> on the owner-approved low-risk list (reversible) -----------+
          |      e.g. rebuild stale projection entries [6];                |
          |      expire working memory past its expiry [5]                 |
          |                                                                v
          |                                       run it; log trigger, prior version,
          |                                       rollback path; on failure roll back
          |                                       and notify the owner
          |
          +--> anything else ----> PROPOSAL to the owner queue
                 review . dedupe . weaken . supersede . archive . expire
                 invalidate . recompute . retire
                 decided by an owner; committed by [4] or [9]
```

Rules:

- No silent rewrite or deletion of evidence in [2] or canonical memory in [5]. Changes to [5] are committed only by [4] or [9] after an owner decision, or by an approved low-risk action ([REQ-ME-29](01-requirements.md#req-me-29), [REQ-PR-09](01-requirements.md#req-pr-09)).
- The low-risk list is owned by memory owners, governed by [9], versioned, and contains only reversible operations ([REQ-ME-30](01-requirements.md#req-me-30)). Adding to it is a governed change.
- Every action and proposal records trigger, evidence, owner, acting identity, prior version, reason, time, and rollback path.
- A failed automatic action is rolled back and shown to the owner on the dashboard ([§12.1](#121-management-dashboard)).
- Dependency-driven repair follows derived-state links: when a source, tool, or schema changes, every derived unit and projection entry is flagged, and recomputation is proposed rather than performed on canonical memory ([REQ-ME-13](01-requirements.md#req-me-13), [§7](#7-canonical-memory-and-retrieval-projections)).
- Existing flows keep their roles: contradiction triage (F3) and deprecation detection (F4) can be started by the orchestrator, and they still route to owners.
- Maintenance is incremental. It never rebuilds accumulated memory as one rewrite ([REQ-ME-09](01-requirements.md#req-me-09)).

Scope deprecation is a trigger on a whole population of units, not on one unit. The affected units are listed in one versioned and audited operation. A unit also supported by a non-deprecated source is weakened and routed to its owner for re-validation, not expired. Detection can be wrong (a repository archived by mistake, a procedure superseded only for one product), so the transition is reversible and the deprecation evidence is kept with it.

## 9. Scope and promotion

Scopes: personal → team → product → organization. A unit may start at any scope. Promotion widens the audience and is gated; repetition alone never promotes.

Visibility and relevance are separate dimensions enforced at different points:

- **Visibility** is enforced deterministically in step 3 of the read path and in [4] at admission and promotion. It answers "may this identity access this unit?"
- **Relevance** is computed in step 4 of the read path. It answers "should this consumer receive this unit for this task?"

A unit that is visible but irrelevant is noise and is not delivered. A unit that is relevant but not visible is never delivered, and its absence is reported as unknown.

## 10. Governance control plane

| Function | What [9] enforces |
|---|---|
| Input | What may be captured, from which sources, under which identity; classification at capture |
| Processing | What may be inferred; which model endpoints may be used ([REQ-SEC-08](01-requirements.md#req-sec-08)); sensitivity handling before admission |
| Memory | What may be remembered, for how long, at which scope; owner-controlled scope changes; expiry; deletion; rollback; deprecation transitions; which maintenance actions may run automatically |
| Output | Who may see and who should receive; policy precedence over local approval; fail closed when policy or permission data is missing |
| Policy | Human ownership; authenticated changes; deterministic precedence across every interface and execution environment |
| Integrity | Which records must be attested; managed signing identities; trust roots, rotation, revocation, expiry; quarantine on failed verification ([§10.1](#101-record-integrity-and-attestation)) |

Precedence is deterministic and environment-independent: a local setting, a user preference, a personal-scope memory, or a locally approved agent action can never weaken an organizational policy ([REQ-GOV-02](01-requirements.md#req-gov-02), [AS-04](01-requirements.md#as-04)).

### 10.1 Record integrity and attestation

High-impact records are tamper-evident and attributable ([REQ-SEC-16…18](01-requirements.md#req-sec-16)). This is a control in [9] applied at [2], [4], [5], [7], and [10]; it is not a new component.

```
   WRITE   admission [4] . outcome verifier [10] . policy owner
     record version --> sign with managed identity --> attestation
                        (key from org trust root [9])       |
                                                             v
   STORE   [2] / [5]: record + attestation + verification status
                                                             |
   USE     read path [7] . promotion [4] / [10]              v
     verify against trust root --+-- pass --> continue to grounding and
                                 |            promotion checks (truth is
                                 |            decided there, not here)
                                 |
                                 +-- fail / revoked / unknown key -->
                                     quarantine: no grounding, no
                                     promotion; audit; notify owner
```

| Object | Holds | Kept in |
|---|---|---|
| Integrity attestation | Record ID and version, content digest, signing identity (user or service), source identity, time, trust-root reference | With the record in [2] or [5] |
| Verification status | Verified, failed, unverified, or signer revoked; last check time; reason | [5]; audited in [9] |
| Trust root | Organization-managed keys and authorities, rotation schedule, revocation list, expiry | [9], backed by the approved vault ([REQ-SEC-03](01-requirements.md#req-sec-03)) |

Rules:

- **Covered records**, in proportion to risk: shared and organization-scope memory units, policy references, Skill versions, admission decisions, and verified outcome records ([§2.1](#21-decision-lineage)). Personal and working memory may be exempt by policy.
- **Verify before use.** Verification is deterministic and runs before grounding (read path step 3, [§5](#5-read-path)) and before promotion ([§11.3](#113-shadow-evaluation-and-promotion)).
- **Fail closed.** A record that fails verification, has an unknown key, or was signed only by a local self-signed agent identity is treated as unverified: quarantined, excluded from grounding and promotion, and reported to its owner as an audited incident. It stays in history and returns to use only after the owner re-verifies it.
- **Revocation.** Records presented under a revoked identity or key do not inherit trust. Records it signed earlier are flagged and routed to owners as maintenance proposals ([§8.1](#81-memory-maintenance-orchestrator)).
- **Integrity is not truth.** A valid attestation proves origin and that the record is unchanged. It does not make a claim grounding-eligible; that is still decided by [REQ-ME-16](01-requirements.md#req-me-16).

## 11. Evaluation loop

For every run the loop records: the complete set of units supplied (ids and versions), the query form, the filters, and the reasons for inclusion and omission ([REQ-OUT-08](01-requirements.md#req-out-08), [REQ-EV-06](01-requirements.md#req-ev-06)); the consumer's model, harness, task class, and scope; outcome signals (helped, misled, ignored, corrected), implicit and explicit.

What the loop does:

- updates utility history on each supplied unit version, scoped to task class, audience, model, harness, and context ([REQ-EV-03](01-requirements.md#req-ev-03))
- applies attribution rather than crediting or blaming every supplied unit for the whole outcome ([REQ-EV-06](01-requirements.md#req-ev-06)); where a decision-lineage chain exists, follows it from the Outcome through the Execution and Decision to the Proposal, and treats the Proposal's `informed_by` units as exposure only ([§2.1](#21-decision-lineage)). Credit or blame needs evidence of influence; a causal claim needs a controlled comparison (A/B, shadow, or replay). Each attribution records its level (exposure, influence, causal) and confidence; an outcome with no evidence of influence stays unattributed
- verifies an outcome only against an explicit acceptance criterion and records the evaluator and uncertainty; a positive signal without a criterion stays an unverified observation and counts as weaker evidence ([REQ-ME-14](01-requirements.md#req-me-14))
- triggers weakening or suspension when harmful signals accumulate against a version ([AS-03](01-requirements.md#as-03))
- never lets a new or materially changed version inherit its predecessor's utility ([REQ-EV-04](01-requirements.md#req-ev-04))
- runs evaluation of unvalidated units only in safe environments, canaries, or low-risk tasks, and never weakens mandatory policy or security controls to do so ([REQ-EV-08](01-requirements.md#req-ev-08), [REQ-EV-09](01-requirements.md#req-ev-09))
- runs side-effect-free shadow evaluation for candidate skills, rules, procedures, and material memory-policy changes on a shared comparison basis, and produces the promotion evidence record ([§11.3](#113-shadow-evaluation-and-promotion))

Attribution with several units supplied at once, delayed outcomes, and partly subjective signals is a core design problem, not an afterthought. Its design is a Phase 0 deliverable.

### 11.1 Net value accounting

The loop also answers: is the Brain worth what it costs? ([REQ-NFR-10](01-requirements.md#req-nfr-10))

| Side | What is counted |
|---|---|
| Cost | Tokens and latency of the write path (reflection, extraction, curation, admission); tokens of injected context (L0, L1, L2 as delivered); tokens and latency of retrieval calls; tokens of the evaluation runs themselves |
| Benefit | Shorter sessions; fewer retries; less re-research; fewer repeated mistakes; the change in outcome quality against the same work without Brain-supplied context |

How it is measured: an A/B or paired protocol runs comparable work with and without Brain-supplied advisory context. Withholding applies only to advisory memory; policy and authoritative memory are never withheld. Results are broken down by scope and by memory version. Cost and benefit are put on a common scale by owner-set weights. Weights, minimum sample size, and reporting period are Phase 0 deliverables. Net value accounting is a computation over selection logs, outcome records, and cost records the system already keeps; it is not an agentic flow and consumes the results of flows F6 and F7.

### 11.2 Research-agent baseline

[REQ-EV-12](01-requirements.md#req-ev-12) asks whether Brain-supplied context beats the alternative an organization would otherwise use: a research agent building context from scratch.

```
   defined task class + the same objective for both arms
          |                                   |
   BRAIN ARM                            BASELINE ARM
   [7] supplies context within budget   from-scratch research agent builds its own report;
          |                             no Brain access; approved endpoints only
   agent works with it                  agent works with it
          |                                   |
          +----------------+------------------+
                           v
   JUDGE: quality against the actual outcome . tokens . latency
          (not on how convincing the report reads)
          |
          +--> Brain matches or beats on quality AND costs less --> class passes; keep measuring
          +--> otherwise --> supplied units become candidates for weakening; evidence kept
```

Rules: the baseline agent gets the same objective, approved endpoints, and safe environment, and has no read access to [5], [6], or [2]; quality is judged against what actually happened (tests passed, review accepted, defects found later); the judging method is fixed before the run; a failing result marks units as weakening candidates and deletes nothing; the protocol is re-run on a schedule per task class because memory grows and baseline models improve.

### 11.3 Shadow evaluation and promotion

Shadow evaluation tests a candidate skill, rule, procedure, or material memory-policy change on real work without letting it control anything ([REQ-EV-13](01-requirements.md#req-ev-13)).

```
   task (live or replayed)
          |
   COMPARISON BASIS (same for every arm, EV-14)
   task identity . authorization boundary . source snapshot
   evidence universe . external-state snapshot . filters (OUT-06 / OUT-18)
          |                     |                            |
   ACTIVE ARM             SHADOW ARM                  BASELINE ARM
   controls production    skill/rule: same selection  no candidate
                          policy: own selection       (where feasible)
                          tools: READ-ONLY FACADE
                          mutating call -> recorded,
                          never executed (EV-18)
          |                     |                            |
          +---------------------+----------------------------+
                                v
   [10] compare, per arm: selection logged . conditions recorded (EV-19)
        applicability . abstention . agreement . execution validity
        verified outcome . negative transfer . uncertainty
        conditions differ in a way that could explain the result -> flagged, not counted
                                |
                                v
   PROMOTION EVIDENCE RECORD (per candidate version)
        distinct contexts counted . threshold and decision rule applied and why
        protected-task results . aggregate result vs active and baseline
                                |
        +--> protected regression -----------------> blocked; stays a candidate
        +--> evidence below threshold --------------> hold; keep evaluating
        +--> evidence meets threshold, no regression -> recommendation to owner via [4]
```

Rules:

- All arms share one comparison basis; the shadow arm gets no more access than the active path and runs under its own scoped identity ([REQ-EV-14](01-requirements.md#req-ev-14)).
- **Skill shadow versus policy shadow.** A candidate skill, rule, or procedure gets the active path's selected context, so only the candidate differs. A candidate retrieval or memory policy selects its own context from the same evidence universe, so the selection is what is tested. Every arm's selection is logged ([REQ-OUT-08](01-requirements.md#req-out-08)).
- **No side effects.** Shadow arms reach tools only through read-only façades. Mutating calls are recorded as intended actions and never executed; an attempt around the façade is blocked, recorded as a violation, and stops the run ([REQ-EV-18](01-requirements.md#req-ev-18)).
- **Controlled conditions.** Each arm records its time reference, tool results, model and harness versions, and randomness settings. Replays reuse recorded tool results and snapshots; live comparisons record differences. A run whose conditions could explain the result is flagged and not counted ([REQ-EV-19](01-requirements.md#req-ev-19)).
- Candidate, active, and baseline outputs are recorded against the same comparison basis, so comparisons are like for like ([REQ-EV-15](01-requirements.md#req-ev-15)).
- Evidence is counted by distinct representative contexts. Repeated runs on one context or one replay count once. Each comparison carries its uncertainty, and the recommendation follows the class's minimum-evidence decision rule ([REQ-EV-16](01-requirements.md#req-ev-16)).
- A regression on a protected task blocks promotion whatever the aggregate shows ([REQ-EV-17](01-requirements.md#req-ev-17)).
- The threshold is set per capability class, domain, evidence strength and diversity, risk, and blast radius; the record states which threshold applied and why ([REQ-ME-12](01-requirements.md#req-me-12), [OD-08](01-requirements.md#od-08)). No count is universal.
- [10] only recommends. Promotion is an owner decision committed through [4]. A changed version starts its evidence again ([REQ-EV-04](01-requirements.md#req-ev-04)).

Shadow run record (one per arm per run):

| Field | Holds |
|---|---|
| Comparison basis | Task identity, authorization boundary, source and external-state snapshot references, evidence universe, filters |
| Selection | Units and versions selected by this arm, with reasons |
| Conditions | Time reference, tool results or replay reference, model and harness versions, randomness settings |
| Intended actions | Mutating calls the arm would have made, as recorded by the façade |
| Results | Applicability, abstention, agreement, execution validity, verified outcome, negative transfer, uncertainty |
| Flags | Side-effect violation; condition mismatch (not counted) |

## 12. Integration surfaces

| Surface | Purpose | Notes |
|---|---|---|
| Harness lifecycle hooks | Proactive retrieval on `SessionStart` and `UserPromptSubmit` equivalents; outcome capture on session end or task completion; continuity checkpoint on session end, handoff, reset, or compaction ([§2.3](#23-working-memory-and-continuity)) | The primary read path. Hook handlers live in [8]; they pass prompt and state to [7] and inject the composed context. |
| MCP server | Exploratory access: inspect a unit's source, expand to L2, list related units, report an outcome, review a transcript | Illustrative tools: `memory.explore`, `memory.inspect_source`, `memory.expand_evidence`, `memory.report_outcome`, `memory.review_transcript`. Names are not normative. Never the only delivery path. |
| Source adapters | Pull or receive evidence from systems of record and work surfaces | The only components that touch external systems; run under scoped identities ([REQ-SEC-10](01-requirements.md#req-sec-10)). |
| Code-graph adapter | Ingest code graphs next to the code | Records repository, commit, and builder version on every entity and relation ([REQ-IN-15](01-requirements.md#req-in-15)); invalidates derived units when the code changes. |
| Work-observation recorder | Observe real work on the workstation and emit evidence events | On-device redaction; visible recording with pause, stop, discard; evidence goes only to organization-controlled ingest. |
| Memory import and export | Move memory in and out in portable structured formats | Import goes through a source adapter and produces candidates only. Export is filtered by the requester's permissions and written only to organization-controlled destinations. |
| Output adapters for applications and automations | Deliver context to non-interactive consumers through APIs | Same retrieval service, filters, and logging. |
| Transcript pitfall review | Return applicable lessons, pitfalls, contradictions, and policy conflicts for a full transcript | A request type on the same retrieval service, exposed as API, MCP tool, and harness hook at a task boundary. |
| Owner and administrator UI | Admission queue, contradiction review, scope changes, policy administration, audit, rollback, management dashboard | Human-in-the-loop surface for [4] and [9]. |

The retrieval and admission interfaces are versioned contracts that do not depend on any harness's internals ([REQ-NFR-06](01-requirements.md#req-nfr-06)).

### 12.1 Management dashboard

The dashboard ([REQ-NFR-11](01-requirements.md#req-nfr-11)) is a read-only view over data the system already keeps. It adds no store of its own and no write path to [5].

| Panel | Shows | Reads from | Requirements |
|---|---|---|---|
| [A] Users and agents | Active users and agents, scopes held, capture and injection activity | Identity records and audit in [9]; evidence counts from [2]; selection logs from [7] | [REQ-NFR-07](01-requirements.md#req-nfr-07), [REQ-GOV-05](01-requirements.md#req-gov-05) |
| [B] Memories | Counts by class, category, scope, and lifecycle state; admission queue; pending promotions; weakened, archived, expired | [5] metadata; queues of [4] | [REQ-ME-02](01-requirements.md#req-me-02), [REQ-ME-04](01-requirements.md#req-me-04), [REQ-ME-06](01-requirements.md#req-me-06) |
| [C] Relations | Typed relationships with contradictions, supersessions, cross-scope links, path to evidence | Relationship records in [5]; graph projection in [6] for navigation only | [REQ-ME-05](01-requirements.md#req-me-05), [REQ-GOV-06](01-requirements.md#req-gov-06), [§7](#7-canonical-memory-and-retrieval-projections) |
| [D] Statistics | Capture volume, admission rate, retrieval latency, selection size against budget, outcome signals | Observability data; [10] outcome records | [REQ-NFR-01](01-requirements.md#req-nfr-01), [REQ-NFR-04](01-requirements.md#req-nfr-04), [REQ-EV-05](01-requirements.md#req-ev-05) |
| [E] Net value | Cost, benefit, and net value per task class, per scope and version; baseline results | [10] ([§11.1](#111-net-value-accounting), [§11.2](#112-research-agent-baseline)) | [REQ-NFR-10](01-requirements.md#req-nfr-10), [REQ-EV-12](01-requirements.md#req-ev-12) |
| [F] Health and actions | Adapter status, projection staleness, policy conflicts, deprecation notices, open contradictions, pending owner actions, maintenance proposals, automatic actions and rollbacks | Adapters [1]; projections [6]; [9]; flows F3 and F4; maintenance orchestrator ([§8.1](#81-memory-maintenance-orchestrator)) | [REQ-NFR-02](01-requirements.md#req-nfr-02), [REQ-ME-13](01-requirements.md#req-me-13), [REQ-GOV-06](01-requirements.md#req-gov-06) |

Rules: every request authenticated and authorized server-side; each panel computed from what the viewer may see, with out-of-scope items absent or shown as unknown, never as a hidden zero; identifiers and aggregates only, with raw content only on a permitted L2 drill-down; actions open the matching workflow in [4] or [9] under the viewer's identity; queries read projections and precomputed aggregates, not live scans of [2] or [5].

## 13. Agentic flows catalog

An agentic flow is a place where a model decides something. Flows are not new components; each runs on components [1]–[10]. The functional view is in [02-functionality.md](02-functionality.md) [§7](02-functionality.md#7-agentic-flows).

| # | Flow | Trigger | Components | Requirements | May write |
|---|---|---|---|---|---|
| F1 | Capture and reflection | Harness events, session end, boundary in [2] | [1] [2] [3] | [REQ-IN-02](01-requirements.md#req-in-02), [REQ-IN-03](01-requirements.md#req-in-03), [REQ-PR-01…08](01-requirements.md#req-pr-01), [REQ-PR-11](01-requirements.md#req-pr-11) | Evidence to [2]; candidates to [4] queue |
| F2 | Admission and promotion review | Candidate arrives; owner request; scheduled batch | [4] [9] | [REQ-PR-10](01-requirements.md#req-pr-10), [REQ-ME-04](01-requirements.md#req-me-04), [REQ-ME-10](01-requirements.md#req-me-10), [REQ-SC-01…06](01-requirements.md#req-sc-01), [REQ-GOV-02…04](01-requirements.md#req-gov-02), [REQ-GOV-10](01-requirements.md#req-gov-10) | Recommendation with reasons; [4] commits after approval |
| F3 | Contradiction triage | [3] flags at candidate time; periodic scan; transcript review | [3] [4] [5] | [REQ-PR-06](01-requirements.md#req-pr-06), [REQ-ME-07](01-requirements.md#req-me-07), [REQ-GOV-06](01-requirements.md#req-gov-06), [AS-02](01-requirements.md#as-02) | Contradiction record and routing notice; never resolves |
| F4 | Deprecation detection | Periodic source-status scan; adapter status change | [1] [3] [4] [9] | [REQ-ME-13](01-requirements.md#req-me-13), [REQ-ME-06](01-requirements.md#req-me-06), [REQ-GOV-07](01-requirements.md#req-gov-07), [REQ-IN-07](01-requirements.md#req-in-07) | Deprecation evidence and proposal; [4] and [9] apply |
| F5 | Transcript pitfall review | Consumer, on checkpoint or completed transcript | [6] [7] [8] | [REQ-OUT-17](01-requirements.md#req-out-17), [REQ-OUT-01](01-requirements.md#req-out-01), [REQ-OUT-06](01-requirements.md#req-out-06), [REQ-OUT-08](01-requirements.md#req-out-08), [REQ-OUT-15](01-requirements.md#req-out-15), [REQ-OUT-16](01-requirements.md#req-out-16) | Selection logs and outcome events only |
| F6 | Research-agent baseline comparison | Scheduled per task class; on demand; after large memory change | [7] [8] [10] | [REQ-EV-12](01-requirements.md#req-ev-12), [REQ-EV-08](01-requirements.md#req-ev-08), [REQ-EV-09](01-requirements.md#req-ev-09), [REQ-NFR-10](01-requirements.md#req-nfr-10), [AS-08](01-requirements.md#as-08) | Evaluation records in [10]; weakening proposal |
| F7 | Candidate evaluation in canaries and shadow | Unvalidated admission or new version; candidate enters shadow; scheduled canary review | [4] [7] [10] | [REQ-EV-02…06](01-requirements.md#req-ev-02), [REQ-EV-08](01-requirements.md#req-ev-08), [REQ-EV-09](01-requirements.md#req-ev-09), [REQ-EV-13…16](01-requirements.md#req-ev-13), [REQ-EV-18…19](01-requirements.md#req-ev-18), [REQ-ME-10](01-requirements.md#req-me-10), [AS-03](01-requirements.md#as-03) | Utility records in [10]; promote / hold / weaken recommendation |
| F8 | Skill-candidate gating | Owner requests promotion; gate result arrives | [3] [4] [10] | [REQ-ME-12](01-requirements.md#req-me-12), [REQ-EV-16](01-requirements.md#req-ev-16), [REQ-EV-17](01-requirements.md#req-ev-17), [REQ-PR-13](01-requirements.md#req-pr-13), [REQ-PR-14](01-requirements.md#req-pr-14), [REQ-EV-08](01-requirements.md#req-ev-08), [OD-08](01-requirements.md#od-08), [AS-06](01-requirements.md#as-06) | Gate results and promotion evidence record on the candidate; sandbox and shadow runs only |

Common rules: no flow writes shared memory directly; changes to [5] are committed by [4], [9], or [10] under their own identities ([REQ-PR-09](01-requirements.md#req-pr-09), [REQ-GOV-04](01-requirements.md#req-gov-04)). Every flow runs under a scoped, least-privilege identity with default-deny egress and approved model endpoints only ([REQ-SEC-08](01-requirements.md#req-sec-08), [REQ-SEC-10](01-requirements.md#req-sec-10)). Evidence and memory read by a flow are untrusted input ([REQ-SEC-06](01-requirements.md#req-sec-06)). Flows are idempotent. Every run is logged with inputs, outputs, and identity ([REQ-GOV-05](01-requirements.md#req-gov-05), [REQ-NFR-03](01-requirements.md#req-nfr-03)). A flow may propose a flagged policy candidate but never authors policy ([REQ-GOV-11](01-requirements.md#req-gov-11)).

Phase mapping: Phase 1 needs F1, F2, and F5 in basic form. F3 and F4 come with governance in Phase 2. F6, F7, and F8 come with the evaluation loop in Phase 3.

## 14. Deployment and operations

- All components, stores, and model endpoints run inside organization-controlled environments ([REQ-SEC-09](01-requirements.md#req-sec-09)). Development uses synthetic or de-identified data only.
- Identity and authorization integrate with the enterprise identity provider; every component and agent runs under a scoped, least-privilege identity with default-deny egress ([REQ-SEC-10](01-requirements.md#req-sec-10)).
- Components communicate over mutually authenticated TLS ([REQ-SEC-02](01-requirements.md#req-sec-02)). Data at rest is AES-256-GCM encrypted with vault-managed keys ([REQ-SEC-03](01-requirements.md#req-sec-03)).
- The evidence log is the recovery root: projections and L0/L1 are rebuildable from [5] and [2]; [5] is restorable from its versioned history and the log.
- Observability covers capture volume, admission rate, retrieval latency, selection size against budget, outcome signal rate, and correction rate ([REQ-NFR-07](01-requirements.md#req-nfr-07)), and feeds net value accounting and the dashboard.
- Retention and expiry run as governed jobs with audit records ([REQ-GOV-07](01-requirements.md#req-gov-07), [REQ-SEC-15](01-requirements.md#req-sec-15)). Sizing and retention defaults are set in Phase 0 ([OD-07](01-requirements.md#od-07), [REQ-NFR-09](01-requirements.md#req-nfr-09)).

### 14.1 Topology position (OD-10)

Target: **a single logical memory**. Every unit carries scope attributes (personal, team, product, organization) and an owner, and permission filtering decides who sees what ([REQ-OUT-06](01-requirements.md#req-out-06), [REQ-SC-04](01-requirements.md#req-sc-04)).

Isolated per-group stores are acceptable only as Phase 1 containment (for example one pilot team's store while governance for wider scopes is built). They must not become the architecture: they block cross-team experience transfer ([AS-01](01-requirements.md#as-01), [AS-02](01-requirements.md#as-02)), since a lesson in one store reaches another only by manual copying, which is the documentation work [REQ-IN-05](01-requirements.md#req-in-05) rules out, and they split lessons, operations, and governance into parts kept in agreement by hand.

What "logical" means: one object model, one identity model, one set of filters, and one lifecycle for all units. It does not mean one physical store. Personal-scope units MAY sit in separate physical storage with separate keys so that private and shared experience stay separate in storage, access, and processing ([REQ-GOV-09](01-requirements.md#req-gov-09), [REQ-SEC-03](01-requirements.md#req-sec-03)), as long as the logical model stays single.

A containment store uses the same unit schema, identifiers, and interfaces as the target, so joining it to the one logical memory needs no re-extraction and no change of unit identity. The exit from containment is planned in Phase 0 and done by the end of Phase 2.

One user, many scopes: deterministic filters evaluate the union of what the identity may access; relevance ranking runs inside that set and never widens it; a unit's scope stays visible in its provenance. The detailed design (membership resolution; a user acting for a team versus alone) is part of [OD-10](01-requirements.md#od-10) and a Phase 0 deliverable.

Federation with controlled flows is not ruled out for specific data that must stay in a separate system for legal or classification reasons. It is an exception handled through [REQ-GOV-10](01-requirements.md#req-gov-10) and [OD-04](01-requirements.md#od-04), not the topology.

## 15. Proposed delivery phases — **Proposal**

Exit criteria are tied to acceptance scenarios.

```
   PHASE 0  discovery and design (short, fixed scope)
   +----------------------------------------------------------------------------+
   | close the blocking owner decisions (OD-04 to OD-07, OD-09, OD-10)          |
   | experience-unit schema v1 (OD-02) . threat model . evaluation design       |
   | attribution design . net value weights . detailed design approved          |
   +----------------------------------------------------------------------------+
        | exit: owner approval of the design; no code depends on an open decision
        v
   PHASE 1  thin vertical slice (one harness, pilot teams, personal + team scope)
   +----------------------------------------------------------------------------+
   | capture via harness hooks -> immutable evidence log  [1] [2]               |
   | reflector + admission as separated roles  [3] [4]                          |
   | managed memory store with versions and rollback  [5]                       |
   | proactive injection on SessionStart / UserPromptSubmit  [7] [8]            |
   | L0 / L1 / L2 disclosure and absence semantics (unknown, fail closed)       |
   | outcome capture: helped / misled / ignored / corrected  [10]               |
   | flows F1, F2, F5 in basic form                                             |
   +----------------------------------------------------------------------------+
        | exit: AS-01 and AS-05 pass on pilot teams; read path meets OD-06 target
        v
   PHASE 2  governance and scale (product + organization scope)
   +----------------------------------------------------------------------------+
   | promotion gates, visibility vs relevance, owner workflows  [4] [9]         |
   | retrieval projections (graph / vector / text) with invalidation  [6]       |
   | MCP exploration tools: inspect source, expand evidence, report outcome     |
   | audit, expiry, deletion and rollback end to end  [9]                       |
   | contradiction triage and deprecation detection, flows F3 and F4            |
   | management dashboard and owner / admin UI                                  |
   | exit from per-group containment into one logical memory                    |
   +----------------------------------------------------------------------------+
        | exit: AS-02, AS-04 and AS-09 pass; org-wide rollout approved
        v
   PHASE 3  learning at scale
   +----------------------------------------------------------------------------+
   | evaluation loop with per-version attribution and safe canaries  [10]       |
   | skill candidates with validation and rollback (OD-08)                      |
   | additional adapters: tickets, BI, notebooks, query history (REQ-IN-08)     |
   | work-observation recorder adapter with on-device redaction (REQ-IN-09..14) |
   | net value accounting and research-agent baseline  [10]                     |
   | flows F6 to F8: baseline, canaries, shadow, skill gating                   |
   +----------------------------------------------------------------------------+
          exit: AS-03, AS-06, AS-07 and AS-08 pass
```

Why this order: Phase 1 proves the loop end to end on a narrow scope where a wrong lesson has a small blast radius. Governance for wider scopes (Phase 2) is built before anything is promoted beyond a team. Phase 3 adds the parts whose design depends on real outcome data.

What Phase 1 deliberately excludes: cross-team promotion, projections beyond what the pilot needs, skills, and any adapter other than the first harness. Building those first produces a warehouse, not a memory. Phase 1 may run pilot teams in isolated stores as containment ([§14.1](#141-topology-position-od-10)); that is a temporary way to limit blast radius, not the target topology.

## 16. Design rationale

### 16.1 Why experience, not knowledge

Systems of record keep the result: the merged code, the closed ticket, the final document, the current runbook. The path to the result (what was tried first, why it failed, what was corrected, which exception turned up, what only worked in one context) is almost never written down, because writing it down is extra work with no immediate payoff. That path is where most reusable learning lives. A warehouse that indexes what the organization already wrote cannot surface it.

### 16.2 Why memory is not a warehouse

| Warehouse | Memory |
|---|---|
| Indexes documents | Admits experience units with provenance, confidence, scope, and usage bounds |
| Search box; the consumer asks | Proactive delivery before reasoning; the consumer may ask for more |
| More retrieval is better | Minimum sufficient context; returning nothing is a valid result |
| Content is true because it is stored | Validity and utility are measured separately and change over time |
| Stale content is a hygiene problem | Weakening, supersession, expiry, and rollback are first-class transitions |
| Access control on documents | Visibility and relevance as separate dimensions; policy precedence; fail closed |

### 16.3 Why the executing agent never writes shared memory

Two independent findings point the same way. First, agents that work from a shared context tend to copy what they most recently saw. Whoever writes first, or writes when nobody else is active, shapes every agent that follows. Shared memory becomes de facto behavioral policy for a fleet even when nobody intended it to be policy. Writes to it need authority, provenance, trust, expiry, and rollback. Second, letting a model rewrite accumulated memory wholesale destroys it: published work on agentic context engineering measured a "context collapse" where a monolithic rewrite compressed an accumulated context to a fraction of a percent of its size and dropped task accuracy. Memory updates must be incremental deltas against itemized units, with a reflector that proposes rather than overwrites.

### 16.4 Context layer versus memory

An alternative framing describes this system as a "context layer": a service that answers questions about the organization, resolves conflicts for the user, and aims for the perfect context for each task. This architecture takes a different position on four points:

- **Identify and route contradictions; do not auto-resolve them.** The system cannot know which side the organization wants. Auto-resolution hides disagreement and can turn one team's workaround into everyone's rule.
- **"Minimum sufficient, measured" instead of "perfect per task".** Perfect context cannot be defined before the work is done, and chasing it pushes the system to supply more. The Brain supplies the smallest set that fits a budget, then measures whether it helped and what it cost.
- **One memory with scopes instead of isolated per-group stores.** Separate stores are simple to secure but stop a lesson in one team from reaching another, which is the main value of an organizational memory.
- **Proactive injection instead of Q&A as the primary surface.** A question-and-answer tool works only if the person or agent knows what to ask. Most expensive mistakes come from not knowing that a lesson exists.

### 16.5 Known failure modes the design must avoid

| Failure mode | What it looks like | Guard |
|---|---|---|
| Context collapse | A rewrite compresses accumulated memory into a few lines and quietly loses most of it | Itemized units, incremental deltas, no monolithic rewrites ([REQ-ME-09](01-requirements.md#req-me-09)) |
| Memory as unintended policy | A frequently seen lesson starts steering every agent as if it were a rule | Separation of duties, policy precedence, advisory framing ([REQ-PR-09](01-requirements.md#req-pr-09), [REQ-GOV-02](01-requirements.md#req-gov-02), [REQ-OUT-16](01-requirements.md#req-out-16)) |
| Local mistake goes global | One team's wrong workaround is promoted because it was repeated often | Promotion gates on evidence, ownership, sensitivity, permission; never on frequency ([REQ-SC-02](01-requirements.md#req-sc-02), [REQ-SC-03](01-requirements.md#req-sc-03)) |
| Projection drift | The graph or index still returns a claim whose canonical unit was weakened or deleted | Mandatory invalidation on canonical change ([§7](#7-canonical-memory-and-retrieval-projections)) |
| Absence read as permission | Empty retrieval is taken to mean "no prior decision, no constraint" | Unknown semantics; fail closed for authority-requiring actions ([REQ-OUT-11](01-requirements.md#req-out-11), [REQ-OUT-13](01-requirements.md#req-out-13)) |
| Reputation inheritance | A materially changed lesson keeps the utility score of the version that earned it | Per-version utility; no inheritance ([REQ-EV-03](01-requirements.md#req-ev-03), [REQ-EV-04](01-requirements.md#req-ev-04)) |
| Retrieval that sounds relevant but does not help | High retrieval rates, no outcome improvement | Utility measured on outcomes, not citations ([REQ-EV-01](01-requirements.md#req-ev-01), [REQ-EV-02](01-requirements.md#req-ev-02)) |
| Leakage through derivation | A conclusion built partly from restricted sources is shown to a wider audience | Most-restrictive-source rule until an owner declassifies ([REQ-GOV-10](01-requirements.md#req-gov-10), [OD-04](01-requirements.md#od-04)) |
| Prompt injection via evidence | Captured content carries instructions that a downstream model follows | Evidence and memory are untrusted input; token stripping; no raw concatenation ([REQ-SEC-06](01-requirements.md#req-sec-06)) |
| Documentation as a second job | The system only learns if people write things up | Passive, trajectory-based capture ([REQ-IN-04](01-requirements.md#req-in-04), [REQ-IN-05](01-requirements.md#req-in-05)) |
| One example becomes a procedure | A single recorded run is installed as a skill and replayed everywhere | Skill Candidate gates ([REQ-ME-12](01-requirements.md#req-me-12)) |
| Click replay | Procedures stored as UI coordinates that break on any layout change | Generalize by intent toward native tools; mark UI-bound steps ([REQ-PR-13](01-requirements.md#req-pr-13)) |
| Reconstruction replaces evidence | The model's account of a session is kept and the raw events are discarded | Evidence and interpretation stored separately and linked; employee review ([REQ-IN-11](01-requirements.md#req-in-11), [REQ-IN-12](01-requirements.md#req-in-12)) |
| Memory that costs more than it saves | Tokens and latency are added to every task while the work gets no better | Net value accounting per task class; research-agent baseline; weakening of units that do not pay ([REQ-NFR-10](01-requirements.md#req-nfr-10), [REQ-EV-12](01-requirements.md#req-ev-12)) |
| A deprecated source keeps steering work | Lessons from an archived repository or retired procedure keep being supplied | Scope or source deprecation weakens or expires every derived unit ([REQ-ME-13](01-requirements.md#req-me-13)) |
| Imported memory admitted as truth | Memory from another system is trusted because it arrived in a clean format | Imports enter as candidates with provenance and pass admission ([REQ-IN-16](01-requirements.md#req-in-16), [REQ-IN-06](01-requirements.md#req-in-06)) |
| Crash loses or repeats work | The process dies before any end-of-session step; the resumed task redoes an irreversible action or forgets a recent constraint | Incremental checkpoints, action journal with idempotency keys, reconcile with external state on resume ([REQ-ME-31](01-requirements.md#req-me-31)) |
| Observation without notice or redaction | Screens and terminals are recorded silently, or secrets leave the workstation | Notice and employee control; on-device redaction; approved endpoints only ([REQ-IN-13](01-requirements.md#req-in-13), [REQ-IN-14](01-requirements.md#req-in-14)) |

### 16.6 Building blocks, not a brain

| Block | What it covers | What it does not cover |
|---|---|---|
| Connectors and RAG | Access to sources; basic retrieval | Learning from trajectories; lifecycle; governance of conclusions |
| Portable knowledge formats | Structured, portable representation of knowledge and metadata | Reflection; utility measurement; scope promotion |
| LLM-maintained wikis | Processing and upkeep of a knowledge base | Separation of duties; incremental, governed admission; outcome feedback |
| Reflector / curator patterns (agentic context engineering) | Lesson extraction; incremental memory deltas | Ownership, audiences, permissions, promotion |
| Experience → knowledge → skill pipelines | Skill proposal with validation and rollback | Organizational scopes and governance plane |
| Desktop work recorders | Observe → reconstruct → review → generalize for one user and one task | Multi-example comparison, policy check, outcome evaluation, owner approval, scopes |
| Knowledge graphs | Entities and typed relations; sub-graph answers | Capture from work trails, lesson extraction, validity, permissions, outcome feedback |
| Bandit-style skill evaluation | Allocating evaluation runs by expected utility; no score inheritance across versions | Attribution when several memories are supplied at once with delayed outcomes |
| Enterprise permission systems | Governance plane; authority enforcement with deterministic precedence | Everything about memory content |

Every block supplies part of the architecture. None supplies the full organizational loop.

### 16.7 Open research questions

- How to extract experience from sessions without collecting unnecessary or private material.
- The minimal schema of an experience unit that still supports provenance, bounds, and evaluation.
- How to tell a useful lesson from durable knowledge from a skill candidate.
- Who owns a conclusion formed from several sources with different permissions.
- How to measure that a retrieval improved an outcome rather than merely sounding relevant.
- When a lesson should widen its audience and when it should stay local.
- How to keep a fast read path without giving up provenance and evidence.
- How to put tokens, time, and outcome quality on one scale, so that net value can be compared across task classes.

## 17. Design constraints checklist

A proposal or implementation that violates any of these is non-compliant regardless of other merits:

- [ ] The executing agent has no write path to shared memory ([REQ-PR-09](01-requirements.md#req-pr-09)).
- [ ] No monolithic rewrite of accumulated memory, anywhere ([REQ-ME-09](01-requirements.md#req-me-09)).
- [ ] Deterministic identity, permission, policy, validity, authority, scope, and per-request grounding-eligibility filters run before ranking ([REQ-OUT-06](01-requirements.md#req-out-06), [REQ-OUT-18](01-requirements.md#req-out-18)).
- [ ] Memory reaches the model before it reasons; MCP is exploratory, not the only path ([REQ-OUT-04](01-requirements.md#req-out-04), [REQ-OUT-07](01-requirements.md#req-out-07)).
- [ ] Projections are derived, invalidated with canonical memory, and inherit its grounding restrictions; never authoritative ([§7](#7-canonical-memory-and-retrieval-projections), [REQ-OUT-18](01-requirements.md#req-out-18)).
- [ ] Empty, filtered, or unavailable results mean unknown; authority-requiring actions fail closed ([REQ-OUT-11](01-requirements.md#req-out-11), [REQ-OUT-13](01-requirements.md#req-out-13)).
- [ ] Policies are human-owned and take deterministic precedence everywhere ([REQ-GOV-02](01-requirements.md#req-gov-02), [REQ-GOV-11](01-requirements.md#req-gov-11)).
- [ ] Utility is measured per version and context; versions never inherit reputation; promotion needs evidence across distinct contexts and is blocked by a protected regression ([REQ-EV-03](01-requirements.md#req-ev-03), [REQ-EV-04](01-requirements.md#req-ev-04), [REQ-EV-16](01-requirements.md#req-ev-16), [REQ-EV-17](01-requirements.md#req-ev-17)).
- [ ] Forgetting preserves history; deletion is a separate governed operation ([REQ-ME-08](01-requirements.md#req-me-08)).
- [ ] Interfaces are harness-neutral ([REQ-NFR-06](01-requirements.md#req-nfr-06)).
- [ ] Reconstructions of observed work are stored beside raw evidence, never instead of it, and are reviewed by the observed employee before they become candidates ([REQ-IN-11](01-requirements.md#req-in-11), [REQ-IN-12](01-requirements.md#req-in-12)).
- [ ] Maintenance never silently rewrites or deletes evidence or canonical memory; only owner-approved low-risk, reversible actions run automatically ([REQ-ME-29](01-requirements.md#req-me-29), [REQ-ME-30](01-requirements.md#req-me-30)).
- [ ] High-impact records are verified before grounding or promotion; a failed check quarantines the record, and a valid signature never makes a claim true ([REQ-SEC-16](01-requirements.md#req-sec-16), [REQ-SEC-17](01-requirements.md#req-sec-17), [REQ-SEC-18](01-requirements.md#req-sec-18)).
- [ ] A continuity checkpoint resumes its own task; it is never organizational knowledge and never becomes durable memory by itself ([REQ-ME-21](01-requirements.md#req-me-21)).
- [ ] Checkpoints are kept current during work and irreversible actions are journaled, so a crash loses at most the recovery point and never repeats an action blindly ([REQ-ME-31](01-requirements.md#req-me-31)).
- [ ] Artifacts from observed work enter as Experience units or Skill Candidates; no Skill Candidate executes in production before the [REQ-ME-12](01-requirements.md#req-me-12) gates pass ([REQ-PR-14](01-requirements.md#req-pr-14), [REQ-ME-12](01-requirements.md#req-me-12)).
- [ ] Imported memory enters as candidates with provenance preserved; exports are permission-filtered and stay in organization-controlled destinations ([REQ-IN-16](01-requirements.md#req-in-16)).
- [ ] Deprecation of a scope or source weakens or expires every unit derived from it, with history preserved and owners notified ([REQ-ME-13](01-requirements.md#req-me-13)).
- [ ] Net value is measured per task class with an A/B or paired protocol, per scope and version; token savings alone are not the metric ([REQ-NFR-10](01-requirements.md#req-nfr-10)).
- [ ] Brain-supplied context beats or matches the research-agent baseline on quality at lower cost, or its units become weakening candidates ([REQ-EV-12](01-requirements.md#req-ev-12)).
- [ ] Transcript pitfall review is a read-path mode with the same filters, budget, provenance, and logging; it never writes memory ([REQ-OUT-17](01-requirements.md#req-out-17)).
- [ ] No agentic flow writes shared memory directly ([§13](#13-agentic-flows-catalog)).
- [ ] One logical memory with scopes is the target; isolated per-group stores are Phase 1 containment only ([§14.1](#141-topology-position-od-10), [OD-10](01-requirements.md#od-10)).
- [ ] The management dashboard shows only what the viewer may see, as identifiers and aggregates, and commits nothing by itself ([REQ-NFR-11](01-requirements.md#req-nfr-11)).
- [ ] Security baseline met in full ([01-requirements.md](01-requirements.md) [§7](01-requirements.md#7-security-and-compliance-baseline)).
