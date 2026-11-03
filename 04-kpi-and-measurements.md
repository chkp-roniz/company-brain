# Company Brain — KPIs and Measurements

This document defines **how a Company Brain is measured**: what is counted, how each measure is computed, where the data comes from, how often it is reported, and which thresholds trigger action. It fulfils [REQ-NFR-07](01-requirements.md#req-nfr-07) (observability), [REQ-NFR-10](01-requirements.md#req-nfr-10) (net value accounting), [REQ-EV-12](01-requirements.md#req-ev-12) (research-agent baseline), and [REQ-NFR-11](01-requirements.md#req-nfr-11) (management dashboard) from [01-requirements.md](01-requirements.md). Component numbers `[n]` refer to [03-architecture.md](03-architecture.md) [§3](03-architecture.md#3-logical-architecture).

Numeric targets marked **Proposal** are starting points for Phase 0 calibration, not approved requirements. Targets governed by an open decision (for example latency, [OD-06](01-requirements.md#od-06)) are stated as "set in Phase 0".

## 1. Measurement principles

1. **Outcomes, not activity.** The Brain exists to change what happens to the work. Retrieval counts, memory size, and token savings are inputs to analysis, never the goal ([REQ-EV-02](01-requirements.md#req-ev-02)).
2. **Validity and utility are separate.** A unit can be well supported by evidence and still help nobody. Both are measured, and neither is inferred from the other ([REQ-EV-01](01-requirements.md#req-ev-01)).
3. **Per version, per context.** Every utility measure is scoped to memory version, task class, audience, model, harness, and scope. A new version starts with no history ([REQ-EV-03](01-requirements.md#req-ev-03), [REQ-EV-04](01-requirements.md#req-ev-04)).
4. **Attribution, not blanket credit.** When several units are supplied to one run, the outcome is not assigned in full to each of them ([REQ-EV-06](01-requirements.md#req-ev-06)).
5. **Net value, not savings.** Benefit minus cost per task class. A class that saves tokens but lowers quality is negative ([REQ-NFR-10](01-requirements.md#req-nfr-10)).
6. **Identifiers and aggregates only.** Metrics pipelines carry unit ids, versions, counts, durations, and signal labels. They never carry evidence or memory content ([REQ-SEC-07](01-requirements.md#req-sec-07)).
7. **Permission-filtered reporting.** A viewer sees metrics only over what their identity may see. Out-of-scope items are absent or unknown, never a zero that hides restricted units ([REQ-OUT-11](01-requirements.md#req-out-11), [REQ-SC-04](01-requirements.md#req-sc-04)).
8. **Measurement never weakens controls.** Withholding context for an experiment applies only to advisory memory. Policy and authoritative memory are never withheld ([REQ-EV-09](01-requirements.md#req-ev-09), [REQ-OUT-13](01-requirements.md#req-out-13)).

## 2. The measurement model

Metrics are organized along the loop. Each layer answers one question.

```
   ADOPTION      CAPTURE       PROCESSING     MEMORY        DELIVERY      OUTCOME        NET VALUE
   who uses it?  what comes    what becomes   what is kept  what is       did it help?   is it worth
                 in?           a candidate?   and trusted?  delivered?                   the cost?
      |             |              |              |             |              |              |
      v             v              v              v             v              v              v
   [8] [9]        [1] [2]        [3] [4]        [5] [6]       [7] [8]        [10]           [10]
                                                                                             |
   GOVERNANCE AND SECURITY ----- every layer ----- audit completeness, policy precedence, leakage, fail-closed
   OPERATIONAL HEALTH ---------- every layer ----- adapter status, projection staleness, queue age, error rates
```

## 3. KPI catalog

Each table lists: the KPI, how it is computed, the data source, the granularity it is broken down by, the reporting cadence, the accountable role, and the target or trigger.

### 3.1 Adoption and coverage

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Active users | Distinct human identities with at least one capture or injection in the period | [9] identity, [2], [7] logs | scope, team | Weekly | Platform admin | Growth to pilot population in Phase 1; organization in Phase 2 |
| Active agents | Distinct agent identities with at least one capture or injection in the period | same | scope, harness | Weekly | Platform admin | Tracks with active users |
| Harness coverage | Share of in-scope harnesses or tools with a working hook or adapter | [1], [8] registry | harness | Monthly | Platform admin | Phase 1: first harness; Phase 3: all in-scope surfaces |
| Session coverage | Share of agent sessions on covered harnesses that emitted evidence | [2] vs harness session counts | harness, team | Weekly | Platform admin | **Proposal:** ≥ 90% on covered harnesses |
| Scopes held per user | Mean and distribution of scopes an identity can access | [9] | team | Monthly | Platform admin | Informational; supports [OD-10](01-requirements.md#od-10) design |
| Proactive injection coverage | Share of `SessionStart` / `UserPromptSubmit` events that reached [7] | [8] hook logs vs [7] request logs | harness | Weekly | Platform admin | **Proposal:** ≥ 99% of hook events reach the retrieval service |

### 3.2 Capture (Input)

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Capture volume | Evidence events and trajectories ingested per day | [2] | source adapter, scope | Daily | Platform admin | Sizing baseline set in Phase 0 |
| Trajectory completeness | Share of closed trajectories with problem, attempt, and outcome events present | [2] | adapter | Weekly | Platform admin | **Proposal:** ≥ 80% |
| Boundary detection rate | Share of closed trajectories where at least one boundary (completion, failure, correction, decision) was detected | [3] | adapter | Weekly | Platform admin | Informational; low values mean reflection rarely triggers |
| Capture authorization compliance | Share of events with a recorded identity, source, and authorization context | [2] | adapter | Daily | Security reviewer | 100%; any gap is an incident |
| On-device redaction rate | Share of observed-work sessions where redaction ran before upload; count of redactions | Recorder adapter telemetry (counts only) | adapter version | Weekly | Security reviewer | 100% ran; redaction counts informational |
| Observed-session discard rate | Share of observed-work sessions paused, stopped, or discarded by the employee | Recorder adapter telemetry | team | Monthly | Privacy / compliance | Informational; a sudden rise signals trust or noise problems |
| Import provenance completeness | Share of imported items carrying source system, file, author, time, and original id | [1] import adapter | source format | Per import | Platform admin | 100% |

### 3.3 Processing and admission

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Candidate yield | Candidates produced per closed trajectory | [3] | task class, adapter | Weekly | Memory owner | Informational; calibrates [OD-03](01-requirements.md#od-03) |
| Evidence link completeness | Share of candidates with at least one `derived_from` or `supported_by` link to evidence | [3], [4] | — | Daily | Memory owner | 100% ([REQ-PR-04](01-requirements.md#req-pr-04)) |
| Classification completeness | Share of candidates with a class (evidence, interpretation, lesson, durable knowledge, skill candidate, policy ref, authoritative ref) and usage bounds | [3] | — | Daily | Memory owner | 100% ([REQ-PR-03](01-requirements.md#req-pr-03), [REQ-PR-07](01-requirements.md#req-pr-07)) |
| Admission rate | Admitted / (admitted + discarded + deferred) in the period | [4] | scope, class | Weekly | Memory owner | Set by [OD-03](01-requirements.md#od-03); track drift |
| Admission queue age | Median and p95 time from candidate creation to admission decision | [4] | scope | Daily | Memory owner | **Proposal:** p95 < 5 working days for team scope |
| Duplicate merge rate | Share of candidates merged into an existing unit | [3] | scope | Weekly | Memory owner | Informational |
| Contradiction detection rate | Contradictions recorded per 100 admitted units | [3], F3 | scope | Weekly | Memory owner | Informational; zero over a long period is suspicious |
| Contradiction resolution time | Median time from contradiction record to owner decision (resolve, keep both, supersede) | F3, [9] | scope | Weekly | Memory owner | **Proposal:** median < 10 working days |
| Sensitivity classification coverage | Share of candidates that passed sensitive-content classification before admission | [4] | — | Daily | Security reviewer | 100% ([REQ-PR-10](01-requirements.md#req-pr-10)) |
| Write-path cost | Tokens and latency spent by reflection, extraction, curation, and admission per candidate | [3], [4] cost records | task class | Weekly | Platform admin | Feeds net value; budget set in Phase 0 |

### 3.4 Memory quality and lifecycle

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Units by class, category, scope, state | Counts of managed units | [5] | class, category, scope, lifecycle state | Daily | Memory owner | Informational (dashboard panel B) |
| Metadata completeness | Share of units carrying every [REQ-ME-04](01-requirements.md#req-me-04) attribute | [5] | scope | Weekly | Memory owner | 100% |
| Ownership coverage | Share of units with a named, active owner | [5], [9] | scope | Weekly | Memory owner | 100%; orphaned units are routed for ownership |
| Version depth | Mean versions per unit; share of units rolled back at least once | [5] | scope | Monthly | Memory owner | Informational |
| Freshness | Share of units whose validity or last re-validation is within the retention window for their category | [5] | category, scope | Weekly | Memory owner | Window set by [OD-07](01-requirements.md#od-07) |
| Weakened share | Weakened units / admitted units | [5], [10] | scope, reason (harmful, ignored, contradiction, deprecation) | Weekly | Memory owner | Informational; a rising trend prompts admission threshold review |
| Promotion rate | Promotions per period; share of promotion requests approved | [4] | from-scope → to-scope | Monthly | Memory owner | Informational |
| Promotion gate compliance | Share of promotions with all four gate records (evidence, sensitivity, owner, permission) | [4] | — | Monthly | Security reviewer | 100% ([REQ-SC-03](01-requirements.md#req-sc-03)) |
| Demotion and rollback count | Scope demotions and memory rollbacks per period | [4], [9] | scope | Monthly | Memory owner | Informational; each rollback is reviewed |
| Deprecation response | Time from deprecation evidence to weakening or expiry of derived units; units affected | F4, [9] | source | Per event | Memory owner | **Proposal:** < 24 hours from detection to transition |
| Projection staleness | Time between a canonical change in [5] and projection update in [6]; count of stale projections | [6] change-feed lag | projection type | Continuous | Platform admin | **Proposal:** p95 < 5 minutes; any stale read is a correctness bug |
| Rebuild success | Share of scheduled projection and L0/L1 rebuild drills that complete from [5] and [2] | [6] | — | Quarterly | Platform admin | 100% |
| Checkpoint completeness | Share of continuity checkpoints carrying every [REQ-ME-20](01-requirements.md#req-me-20) field | [5], [8] | trigger (session end, handoff, reset, compaction) | Weekly | Memory owner | 100% |
| Resume success | Share of resume tests passed (no invented state, no repeated work, no lost constraint) | [10] | trigger, task class | Weekly | Memory owner | **Proposal:** ≥ 95%; each failure reviewed ([REQ-ME-24](01-requirements.md#req-me-24)) |
| Repeated-work rate after resume | Share of resumed tasks that repeat an already completed operation | [10], [2] | task class, operation type | Weekly | Memory owner | Zero for irreversible operations; each case reviewed |
| Invented-state failures | Resume tests where the resumed agent asserts state found in neither the checkpoint nor the evidence | [10] | task class | Weekly | Memory owner | Target zero; each case reviewed |
| Lost-constraint failures | Resume tests where a material constraint present before the transition is missing after it | [10] | task class | Weekly | Memory owner | Target zero; each case reviewed |
| Deterministic-cleanup share | Share of compactions where rule-based cleanup ran before semantic summarization; share of removed items removed by rule | [2] compaction records | trigger | Weekly | Platform admin | 100% ran first ([REQ-ME-22](01-requirements.md#req-me-22)); share informational |
| Checkpoint age | Age of the latest checkpoint at each abrupt end; p95 per task class | [5], [8] | task class | Weekly | Memory owner | **Proposal:** within the task class's recovery point ([REQ-ME-31](01-requirements.md#req-me-31)) |
| Recovery point achieved | Share of abrupt ends where the lost state (work after the latest checkpoint) stayed within the recovery point | [5], [2] | task class | Weekly | Memory owner | **Proposal:** ≥ 99% |
| Crash-resume success | Share of crash-resume tests passed (no repeated irreversible action, no invented state, no lost constraint) | [10] | task class | Weekly | Memory owner | **Proposal:** ≥ 95%; each failure reviewed ([REQ-ME-24](01-requirements.md#req-me-24)) |
| Duplicate irreversible actions | Irreversible actions executed more than once for one journal idempotency key | [5] journal, [2] | task class, action type | Continuous | Memory owner | Target zero; each case reviewed |
| Maintenance backlog and lag | Open maintenance proposals; age of the oldest; time from trigger to proposal or action | Orchestrator, [9] | trigger type, owner | Daily | Memory owner | Informational; aging items escalate |
| Due-review completion | Share of units with a due review date reviewed on time | Orchestrator, [5] | scope | Monthly | Memory owner | **Proposal:** ≥ 90% |
| Skill-drift detection | Time from a dependency change (tool, schema, model) to a flag on every affected Skill | Orchestrator, [10] | dependency type | Per event | Memory owner | **Proposal:** < 24 hours ([AS-14](01-requirements.md#as-14)) |
| Projection repair time | Time from a detected projection inconsistency to repair | Orchestrator, [6] | projection type | Per event | Platform admin | **Proposal:** < 1 hour |
| Automatic-action rollback rate | Share of automatic maintenance actions rolled back (failed or reversed by an owner) | Orchestrator, [9] | action type | Monthly | Memory owner | Informational; a rising rate puts the action's place on the low-risk list under review |
| Canonical-rewrite violations | Maintenance changes to [2] or [5] made without an owner decision and outside the low-risk list | [9] audit | — | Continuous | Security reviewer | Zero ([REQ-ME-29](01-requirements.md#req-me-29)) |
| Stale-action rejections | Maintenance actions and proposals rejected because a unit changed after their expected version; share later re-proposed | Orchestrator, [9] | action class | Weekly | Memory owner | Informational; every rejection is re-evaluated, never forced ([REQ-ME-32](01-requirements.md#req-me-32)) |
| Maintenance conflicts | Conflicting actions on one unit, duplicate applications of one idempotency key, and class-owner versus unit-owner disagreements | Orchestrator, [9] audit | action class | Weekly | Memory owner | Duplicate applications: zero; conflicts informational ([REQ-ME-32](01-requirements.md#req-me-32), [REQ-ME-34](01-requirements.md#req-me-34)) |
| Legal-hold violations | Records under legal hold changed, merged away, expired, or deleted by maintenance | [9] audit | — | Continuous | Compliance | Zero ([REQ-ME-33](01-requirements.md#req-me-33)) |
| Broken-reference count | References (links, dependents, selection logs, evaluation records) that no longer resolve after a merge, supersession, or archival | Orchestrator, [5] | action class | Weekly | Memory owner | Zero ([REQ-ME-35](01-requirements.md#req-me-35)) |

### 3.5 Delivery (read path)

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Retrieval latency | p50 and p95 time from hook event to context injected | [7], [8] | harness, scope | Continuous | Platform admin | Target set by [OD-06](01-requirements.md#od-06) (Phase 0) |
| Filter-before-rank compliance | Share of retrievals where deterministic filters ran before ranking (asserted by tests and logs) | [7] | — | Continuous | Security reviewer | 100% ([REQ-OUT-06](01-requirements.md#req-out-06)) |
| Selection size vs budget | Tokens delivered / declared context budget | [7] | harness, task class | Weekly | Platform admin | **Proposal:** median ≤ 60%; never > 100% |
| Empty result rate | Share of retrievals that returned nothing (unknown) | [7] | task class | Weekly | Memory owner | Informational; both very high and near-zero values need review |
| Units per injection | Mean and distribution of units delivered per retrieval | [7] | task class | Weekly | Memory owner | Informational; supports budget tuning |
| L2 expansion rate | Share of injections where the consumer expanded at least one unit to L2 | [8] | task class | Weekly | Memory owner | Informational; signal of trust and inspection |
| Selection log completeness | Share of retrievals with units, versions, query form, filters, and inclusion/omission reasons logged | [7] | — | Daily | Security reviewer | 100% ([REQ-OUT-08](01-requirements.md#req-out-08)) |
| Grounding-ceiling coverage | Share of admitted units with recorded retention and retrieval decisions and a grounding ceiling with a reason | [4], [5] | source class | Weekly | Memory owner | 100% ([REQ-ME-15](01-requirements.md#req-me-15)) |
| Per-request grounding coverage | Share of delivered units whose selection log records the per-request grounding decision, purpose, and permitted-use label | [7] logs | purpose class | Weekly | Platform admin | 100% ([REQ-ME-16](01-requirements.md#req-me-16), [REQ-OUT-08](01-requirements.md#req-out-08)) |
| Ceiling breaches | Per-request grounding decisions that exceeded the unit's grounding ceiling | [7], [9] | purpose class | Continuous | Security reviewer | Zero ([REQ-ME-16](01-requirements.md#req-me-16)) |
| Ineligible-grounding attempts blocked | Count of times a non-grounding unit was offered as support for a claim or an authority-requiring action; share blocked | [7], [9] | source class, cause | Weekly | Security reviewer | 100% blocked; every unblocked case reviewed ([REQ-OUT-18](01-requirements.md#req-out-18)) |
| Projection-policy parity | Share of sampled projection and L0/L1 entries whose retrieval rules, grounding ceiling, and permitted-use labels match the canonical unit | [6], [7] | projection type | Daily | Platform admin | 100% ([REQ-OUT-18](01-requirements.md#req-out-18)) |
| Advisory framing compliance | Share of injections carrying advisory status, provenance, and bounds | [8] | harness | Daily | Security reviewer | 100% ([REQ-OUT-16](01-requirements.md#req-out-16)) |
| Fail-closed events | Count of authority-requiring actions stopped because policy or authoritative memory could not be retrieved | [7], [9] | cause | Weekly | Security reviewer | Each event reviewed; trend tracked |
| Transcript reviews | Count of transcript pitfall reviews; items returned by class (lesson, pitfall, contradiction, policy conflict) | F5 | harness | Weekly | Memory owner | Informational |
| Exploration tool usage | MCP or API calls by tool | [8] | tool | Weekly | Platform admin | Informational; must never be the only path to memory |

### 3.6 Outcome and utility

Outcome signals are the core of the loop. Each delivered unit version accumulates signals in four classes.

| Signal | Implicit evidence | Explicit evidence |
|---|---|---|
| Helped | Task completed without retry; checks passed; no later correction on the point the unit addressed | Consumer marks the unit as helpful |
| Misled | Revert, failed check, or correction directly on the point the unit addressed | Consumer marks the unit as wrong or harmful |
| Ignored | Unit delivered, no reference to it in subsequent actions | Consumer dismisses the unit |
| Corrected | Consumer edits or contradicts the unit's content during the task | Consumer submits a correction |

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Outcome signal coverage | Share of injections with at least one recorded outcome signal within the attribution window | [10] | harness, task class | Weekly | Memory owner | **Proposal:** ≥ 70% |
| Helped rate | Helped / (helped + misled + ignored + corrected) per unit version and task class | [10] | unit version, task class, scope | Weekly | Memory owner | Informational per unit; portfolio trend upward |
| Misled rate | Misled / all signals per unit version | [10] | unit version, task class | Weekly | Memory owner | **Proposal:** weakening review when misled > 20% over ≥ 10 signals |
| Ignored rate | Ignored / all signals per unit version | [10] | unit version, task class | Weekly | Memory owner | **Proposal:** relevance review when ignored > 60% over ≥ 20 signals |
| Corrected rate | Corrected / all signals per unit version | [10] | unit version | Weekly | Memory owner | Each correction becomes new evidence; trend informational |
| Time to weakening | Time from the first harmful signal crossing threshold to the weakening transition | [10], [5] | scope | Per event | Memory owner | **Proposal:** < 48 hours |
| Harmful exposure | Number of injections of a unit version after its harmful threshold was crossed and before weakening | [10], [7] | unit version | Per event | Memory owner | Target zero; each occurrence reviewed ([AS-03](01-requirements.md#as-03)) |
| Repeat-mistake rate | Share of trajectories whose failure matches an already admitted lesson that was available but not delivered or not followed | [3], [10] | task class | Monthly | Memory owner | Downward trend; distinguishes delivery gaps from adoption gaps |
| Validity score | Per unit: evidence count, source diversity, recency, contradiction status | [5] | unit | Weekly | Memory owner | Informational; reported beside utility, never substituted for it |
| Canary evaluation throughput | Unvalidated or changed unit versions evaluated in canaries per period; share with a decision (promote, hold, weaken) | F7, [10] | scope | Weekly | Memory owner | No unvalidated unit exposed outside canaries ([REQ-EV-08](01-requirements.md#req-ev-08)) |
| Shadow coverage | Share of candidate skills, rules, procedures, and material memory-policy changes evaluated in shadow before a promotion decision | F7, [10] | capability class | Monthly | Memory owner | 100% for classes the owner marks as shadow-required ([REQ-EV-13](01-requirements.md#req-ev-13)) |
| Distinct-context coverage | Per candidate version: distinct representative contexts evaluated, against the threshold for its capability class and risk | [10] | candidate, capability class | Weekly | Memory owner | Informational; repeats of one context count once ([REQ-EV-16](01-requirements.md#req-ev-16)) |
| Protected-regression count | Protected-task regressions found in shadow; share that blocked promotion | [10], [4] | capability class | Per event | Memory owner | 100% blocked ([REQ-EV-17](01-requirements.md#req-ev-17)) |
| Candidate disagreement rate | Share of shadow runs where the candidate's output differs from the active path; split by which was right where verified | [10] | candidate, task class | Weekly | Memory owner | Informational; high disagreement with no verified outcomes means more evidence is needed |
| Promotion evidence sufficiency | Share of promotions whose record states the threshold applied, the reason it fits the risk, the distinct contexts counted, and the protected-task results | [4], [10] | capability class | Monthly | Security reviewer | 100% ([REQ-ME-12](01-requirements.md#req-me-12)) |
| Shadow side-effect violations | Attempts by a shadow arm to reach a mutating path outside the read-only façade | [10], [9] | candidate, tool | Continuous | Security reviewer | Zero executed; every attempt reviewed ([REQ-EV-18](01-requirements.md#req-ev-18)) |
| Snapshot parity | Share of shadow runs where every arm shares the same comparison basis and recorded conditions | [10] | capability class | Weekly | Memory owner | **Proposal:** ≥ 95%; mismatched runs are not counted ([REQ-EV-14](01-requirements.md#req-ev-14), [REQ-EV-19](01-requirements.md#req-ev-19)) |
| Selection divergence | For policy candidates, how far the candidate's selected context differs from the active path's (for example, share of units not in common) | [10], [7] logs | candidate | Weekly | Memory owner | Informational; near-zero divergence means the policy change is not being exercised |
| Comparison uncertainty | Uncertainty (for example, a confidence interval) on the candidate-versus-active difference at the time of each recommendation | [10] | candidate, capability class | Per decision | Memory owner | Recommendation only when the class's minimum-evidence decision rule is met ([REQ-EV-16](01-requirements.md#req-ev-16)) |
| Evidence-supported stage coverage | Per stage, share of decision boundaries where the stage is linked to supporting evidence; stages without evidence are counted as unknown, not as gaps to fill | [3], [5] | task class, stage | Weekly | Memory owner | Informational; a low share for one stage points to a capture gap |
| Inferred-stage violations | Lineage records created without supporting evidence for their stage | [3], [5] audit sample | stage | Monthly | Memory owner | Zero ([REQ-PR-15](01-requirements.md#req-pr-15)) |
| Attribution confidence | Distribution of attribution level (exposure, influence, causal) and confidence across attributed outcomes | [10] | task class | Monthly | Memory owner | Informational; a rising share of exposure-only attribution points to weak evidence of use ([REQ-EV-06](01-requirements.md#req-ev-06)) |
| Unattributed-outcome share | Share of outcomes with no evidence of influence from any supplied unit | [10] | task class | Monthly | Memory owner | Informational; reported, never forced to zero |
| Verified-outcome coverage | Share of Outcome records that are verified (explicit acceptance criterion applied by a named evaluator) | [10] | task class | Weekly | Memory owner | **Proposal:** ≥ 60% for task classes with a defined acceptance check |
| Outcome uncertainty distribution | Distribution of recorded uncertainty on Outcome records; share with uncertainty missing | [10] | task class, evaluator type | Monthly | Memory owner | Missing uncertainty is 0%; distribution informational |
| Skill Candidate gate status | For each Skill Candidate: gates passed of six; sandbox outcomes; time in candidate state | F8, [10] | candidate | Monthly | Memory owner | No candidate executes in production before all six gates pass |

### 3.7 Net value

Net value answers "is the Brain worth what it costs?" per task class ([REQ-NFR-10](01-requirements.md#req-nfr-10)).

```
   NET VALUE (task class, scope, version, period)
     = weighted BENEFIT - weighted COST

   COST                                        BENEFIT (vs comparable work without advisory context)
   . write-path tokens and latency             . session length reduction
     (reflection, extraction, curation,        . retries avoided
      admission)                               . re-research avoided
   . injected context tokens (L0, L1, L2)      . repeated mistakes avoided
   . retrieval call tokens and latency         . outcome quality delta
   . evaluation run tokens                       (tests passed, review accepted, defects later)

   weights, minimum sample size, and reporting period: owner-set, fixed in Phase 0
```

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Cost per task class | Sum of write-path, injection, retrieval, and evaluation cost attributed to the class | [3], [4], [7], [10] cost records | task class, scope, version | Monthly | Platform admin | Reported separately from benefit, never netted silently |
| Benefit per task class | Measured differences in session length, retries, re-research, repeated mistakes, and quality between arms | A/B or paired protocol ([§4.2](#42-net-value-ab-or-paired-protocol)) | task class, scope, version | Monthly | Memory owner | Reported per component |
| Net value per task class | Weighted benefit minus weighted cost | [§4.2](#42-net-value-ab-or-paired-protocol) | task class, scope, version | Monthly | Memory owner | Non-positive → class flagged to owner; supplied units become weakening candidates ([AS-07](01-requirements.md#as-07)) |
| Quality-adjusted result | Net value with the quality delta term shown separately | [§4.2](#42-net-value-ab-or-paired-protocol) | task class | Monthly | Memory owner | A class that saves tokens but lowers quality is reported negative |
| Research-agent baseline result | Pass or fail per task class: Brain matches or beats quality AND costs less in tokens and latency than a from-scratch research agent | [§4.3](#43-research-agent-baseline), F6 | task class | Per schedule (Proposal: quarterly, and after any large memory change) | Memory owner | Fail → supplied units become weakening candidates; comparison kept as evidence ([AS-08](01-requirements.md#as-08)) |
| Share of task classes with positive net value | Classes with positive net value / classes measured | [10] | scope | Quarterly | Product owner | Upward trend; the headline value KPI |
| Brain cost share | Brain cost / total model spend of covered work | Cost records, harness telemetry | harness | Monthly | Platform admin | Informational; a ceiling set in Phase 0 |

### 3.8 Governance and security

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Audit completeness | Share of derived items and changes with actor, time, evidence link, and rule recorded | [9] | component | Daily | Security reviewer | 100% ([REQ-GOV-05](01-requirements.md#req-gov-05), [REQ-NFR-03](01-requirements.md#req-nfr-03)) |
| Policy precedence violations | Count of cases where a local setting or approval overrode organizational policy | [9], [AS-04](01-requirements.md#as-04) tests | interface | Continuous | Policy owner | Zero; any case is an incident |
| Policy change integrity | Share of policy changes bound to an authenticated policy owner identity | [9] | — | Per change | Policy owner | 100% ([REQ-GOV-12](01-requirements.md#req-gov-12)) |
| Integrity verification coverage | Share of high-impact records (shared memory, policy references, Skill versions, admission decisions, verified outcomes) with a valid attestation from a managed identity | [9], [5] | record type | Daily | Security reviewer | **Proposal:** 100% for shared and organization scope ([REQ-SEC-16](01-requirements.md#req-sec-16)) |
| Failed-verification count | Records that failed integrity verification, and how many of them reached grounding or promotion | [7], [4], [9] | record type, cause | Continuous | Security reviewer | Reaching grounding or promotion: zero ([AS-15](01-requirements.md#as-15)) |
| Key-rotation compliance | Share of signing keys and trust roots rotated within policy and not past expiry | Key infrastructure, [9] | key type | Monthly | Security reviewer | 100% ([REQ-SEC-18](01-requirements.md#req-sec-18)) |
| Compromise exposure | Records signed after a key's compromise time, or with no trusted timestamp before it, that were treated as trusted | [7], [9] | identity class | Continuous | Security reviewer | Zero ([REQ-SEC-18](01-requirements.md#req-sec-18)) |
| Revoked-identity exposure | Records signed by an administratively revoked identity after its revocation that were treated as trusted | [7], [9] | identity class | Continuous | Security reviewer | Zero ([REQ-SEC-17](01-requirements.md#req-sec-17), [REQ-SEC-18](01-requirements.md#req-sec-18)) |
| Rotation false flags | Records signed validly before a routine rotation, expiry, or administrative revocation that were quarantined or flagged anyway | [7], [9] | key event type | Monthly | Security reviewer | Zero; routine key events must not cast doubt on history ([REQ-SEC-18](01-requirements.md#req-sec-18)) |
| Unverifiable history | Retained high-impact records whose attestation can no longer be verified (missing trust-chain entry, missing trusted timestamp, unsupported algorithm) | [9], [5] | record type, cause | Monthly | Security reviewer | Zero ([REQ-SEC-19](01-requirements.md#req-sec-19)) |
| Algorithm-migration coverage | Share of records on an algorithm scheduled for retirement that were re-attested before the retirement date | [9], [5] | algorithm | Monthly | Security reviewer | 100% by the retirement date ([REQ-SEC-19](01-requirements.md#req-sec-19)) |
| Time to quarantine | Time from a failed verification or a revocation to exclusion of the affected records from grounding and promotion | [9] | record type | Per event | Security reviewer | **Proposal:** < 15 minutes |
| Cross-scope leakage events | Units visible more widely than their most restrictive source without an owner's approval | [4], [9] audit | scope | Continuous | Security reviewer | Zero ([REQ-GOV-10](01-requirements.md#req-gov-10)) |
| Separation of duties | Executing-agent identities holding any write credential to [5] | [9] | — | Continuous | Security reviewer | Zero ([REQ-PR-09](01-requirements.md#req-pr-09)) |
| Direct-write attempts to shared memory | Rejected attempts by non-admission identities to write [5] | [5], [9] | identity class | Daily | Security reviewer | Each attempt investigated |
| Prompt-injection detections | Evidence or memory items where special tokens or instruction patterns were stripped before model use | [3], [7] sanitizers | source | Weekly | Security reviewer | Informational; zero over time is suspicious |
| Retention compliance | Share of expired records securely deleted or anonymized within the retention SLA | [9] retention jobs | category | Monthly | Compliance | 100% ([REQ-GOV-07](01-requirements.md#req-gov-07), [REQ-SEC-15](01-requirements.md#req-sec-15)) |
| Encryption and TLS posture | Stores and channels verified as AES-256-GCM at rest and mutual TLS 1.2+ in transit | Infrastructure scans | component | Monthly | Security reviewer | 100% |
| Least-privilege posture | Agent and MCP components running under scoped identities with default-deny egress | [9], runtime config | component | Monthly | Security reviewer | 100% ([REQ-SEC-10](01-requirements.md#req-sec-10)) |
| Security review gate | Components connected to production data or LLM infrastructure without a passed security review | Review records | component | Per release | Security reviewer | Zero ([REQ-SEC-12](01-requirements.md#req-sec-12)) |
| Dashboard visibility compliance | [AS-09](01-requirements.md#as-09) test outcomes: no out-of-scope counts or content shown | [AS-09](01-requirements.md#as-09) automated tests | role | Per release | Security reviewer | Pass |
| Observed-work consent compliance | Observed-work sessions recorded without a notice event | Recorder adapter | — | Continuous | Privacy / compliance | Zero ([REQ-IN-13](01-requirements.md#req-in-13)) |

### 3.9 Operational health

| KPI | Computation | Source | Breakdown | Cadence | Owner | Target / trigger |
|---|---|---|---|---|---|---|
| Adapter availability | Adapters up / total; time since last successful event per adapter | [1] | adapter | Continuous | Platform admin | Alert on any adapter silent beyond its expected interval |
| Ingest error rate | Rejected or failed ingest calls / total | [2] | adapter | Daily | Platform admin | **Proposal:** < 0.1% |
| Processing backlog | Closed trajectories awaiting reflection; age of the oldest | [3] | — | Daily | Platform admin | Write path is asynchronous; backlog bounded by a Phase 0 limit |
| Retrieval availability | Successful retrieval responses / requests | [7] | harness | Continuous | Platform admin | Degraded responses must report unknown, never fabricate ([REQ-NFR-02](01-requirements.md#req-nfr-02)) |
| Degraded-mode rate | Share of retrievals served while a dependency was unavailable | [7] | dependency | Daily | Platform admin | Informational; each incident reviewed |
| Idempotency violations | Duplicate candidate identities produced from the same trajectory | [3], [4] | — | Weekly | Platform admin | Zero |
| Flow run success | Agentic flow runs completed / started, per flow F1–F8 | Flow logs | flow | Daily | Platform admin | Informational; failures reviewed |
| Pending owner actions | Open items in admission, promotion, contradiction, and deprecation queues; age distribution | [4], [9] | owner, scope | Daily | Memory owner | Dashboard panel F; aging items escalated |
| Recovery drill result | Time to rebuild [6] and L0/L1 from [5] and [2]; time to restore [5] from history and log | Drill records | — | Quarterly | Platform admin | Within the recovery objectives set in Phase 0 |

## 4. Measurement protocols

### 4.1 Outcome signal capture and attribution

1. Every retrieval logs the full set of unit ids and versions supplied, the query form, the filters, and the inclusion and omission reasons ([REQ-OUT-08](01-requirements.md#req-out-08)).
2. Each run is tagged with consumer model, harness, task class, and scope.
3. Implicit signals are derived from later evidence in the same trajectory: retries, reverts, failed checks, corrections. Explicit signals come from the tool interface ([REQ-EV-11](01-requirements.md#req-ev-11)).
4. An **attribution window** per task class (Phase 0 deliverable) bounds how long after injection a signal may be linked to it.
5. Attribution assigns signals to specific units, not to the whole set. The initial method should at least: link a correction or revert to the unit whose bounds cover the corrected point; treat units never referenced as ignored; record outcomes with no evidence of influence as unattributed, giving supplied units at most weak exposure-level evidence, never full credit to each ([REQ-EV-06](01-requirements.md#req-ev-06)). Every attribution records its level (exposure, influence, causal) and confidence; causal claims need a controlled comparison ([§4.2](#42-net-value-ab-or-paired-protocol), [§4.5](#45-shadow-evaluation)). The attribution design is a Phase 0 deliverable and a known hard problem.
6. Where a decision-lineage chain exists ([REQ-ME-14](01-requirements.md#req-me-14)), attribution follows it: a signal on an Outcome is linked through the Execution and the Decision to the Proposal. The Proposal's `informed_by` units are exposure only: they earn credit or blame only with evidence that they influenced the Proposal or the Decision. A proposal that was not selected gets no credit for the outcome of the decision taken. A verified outcome is stronger evidence than an unverified observation.
7. Signals update the utility history of the exact unit version only ([REQ-EV-03](01-requirements.md#req-ev-03), [REQ-EV-04](01-requirements.md#req-ev-04)).

### 4.2 Net value: A/B or paired protocol

1. Define task classes and the quality judgment for each (for example: tests pass, review accepted, no defect within N days).
2. Assign comparable work to two arms: with Brain-supplied advisory context, and without. Use random assignment where volume allows, or pair matched tasks.
3. Policy and authoritative memory are never withheld from either arm. Only advisory memory is withheld ([REQ-EV-09](01-requirements.md#req-ev-09), [REQ-OUT-13](01-requirements.md#req-out-13)). Runs on unvalidated units stay in safe environments ([REQ-EV-08](01-requirements.md#req-ev-08)).
4. Record cost components and benefit components separately per arm.
5. Apply owner-set weights to put tokens, time, and quality on one scale. Report the quality delta separately so it cannot be hidden by token savings.
6. Report per task class, scope, and memory version. Flag non-positive classes to their owner. Supplied units for those classes become weakening candidates.
7. Minimum sample size and reporting period are fixed in Phase 0 and reviewed quarterly.

### 4.3 Research-agent baseline

1. Per task class, both arms receive the same objective in the same safe environment on approved endpoints. The baseline agent has no read access to [5], [6], or [2].
2. The Brain arm receives budgeted context from [7]. The baseline arm builds its own research report from scratch.
3. A judge fixed before the run scores quality against what actually happened to the work, not against how complete or convincing the report reads.
4. Compare quality, tokens, and latency. The Brain's cost is its supply cost as defined in [§3.7](#37-net-value); the baseline's cost is its full research run.
5. Pass: Brain matches or beats quality and costs less. Fail: supplied units become weakening candidates; the comparison is kept as evidence. Nothing is deleted.
6. Re-run on a schedule per task class, because memory grows and baseline models improve.

### 4.4 Canary evaluation of unvalidated units

1. A unit admitted as unvalidated, or a materially changed version, is exposed only to canary identities or low-risk task classes ([REQ-EV-08](01-requirements.md#req-ev-08)).
2. Outcomes accumulate against the version until the minimum sample for the task class is reached.
3. The result is a promote, hold, or weaken recommendation to [4] and the owner. The flow never commits the change itself.

### 4.5 Shadow evaluation

1. A candidate skill, rule, procedure, or material memory-policy change enters shadow under its own scoped identity ([REQ-EV-13](01-requirements.md#req-ev-13), [REQ-EV-14](01-requirements.md#req-ev-14)).
2. On each matching task, the candidate, the active path, and, where feasible, a no-candidate baseline share one comparison basis: task identity, authorization boundary, source snapshot, evidence universe, external-state snapshot, and deterministic filters. A skill candidate gets the active path's selection; a policy candidate makes its own, and both selections are logged. Only the active path controls the result ([REQ-EV-14](01-requirements.md#req-ev-14), [REQ-EV-15](01-requirements.md#req-ev-15)).
3. Shadow arms use read-only tool façades; intended mutating calls are recorded, never executed ([REQ-EV-18](01-requirements.md#req-ev-18)). Each arm records time reference, tool results, model and harness versions, and randomness settings; runs whose conditions differ are flagged and not counted ([REQ-EV-19](01-requirements.md#req-ev-19)).
4. Each run records applicability, abstention, agreement, execution validity, verified outcome where available, negative transfer, and uncertainty. Runs are grouped by distinct representative context; repeats count once ([REQ-EV-16](01-requirements.md#req-ev-16)).
5. Protected tasks for the capability class are always run. Any regression blocks promotion ([REQ-EV-17](01-requirements.md#req-ev-17)).
6. **Minimum-evidence decision rule** (**Proposal**, per capability class): recommend promotion only when the distinct-context count meets the threshold, the uncertainty interval on the difference from the active path excludes a loss larger than the owner-set tolerance, and no protected task regressed. Otherwise hold or reject.
7. When distinct-context evidence meets the threshold for the capability class, domain, risk, and blast radius, [10] sends a recommendation to [4] and the owner with the full evidence record. The owner decides; [10] never commits the change ([REQ-ME-12](01-requirements.md#req-me-12)).
8. Threshold values and decision rules per capability class are **Proposal** until set under [OD-08](01-requirements.md#od-08).

### 4.6 Acceptance evidence

Each acceptance scenario in [01-requirements.md](01-requirements.md) [§9](01-requirements.md#9-acceptance-scenarios) is backed by automated or scripted evidence and by the metrics below.

| Scenario | Primary evidence | Supporting KPIs |
|---|---|---|
| [AS-01](01-requirements.md#as-01) Basic experience transfer | End-to-end test: failure → lesson → proactive injection with evidence links | Candidate yield, evidence link completeness, retrieval latency, L2 expansion rate, helped rate |
| [AS-02](01-requirements.md#as-02) Contextual contradiction | Both units retained and context-matched retrieval | Contradiction detection rate, contradiction resolution time |
| [AS-03](01-requirements.md#as-03) Harmful memory recovery | Signal accumulation triggers weakening with history kept | Misled rate, time to weakening, harmful exposure |
| [AS-04](01-requirements.md#as-04) Policy precedence | Policy prevails on every interface; conflict audited | Policy precedence violations (zero), audit completeness |
| [AS-05](01-requirements.md#as-05) Safe absence | Unknown returned; authority actions fail closed | Empty result rate, fail-closed events |
| [AS-06](01-requirements.md#as-06) Observed work to Skill Candidate | Redaction on device; reconstruction beside evidence; candidate gated | On-device redaction rate, observed-work consent compliance, Skill Candidate gate status |
| [AS-07](01-requirements.md#as-07) Net value accounting | Cost and benefit reported separately per class, scope, version | Net value per task class, quality-adjusted result |
| [AS-08](01-requirements.md#as-08) Research-agent baseline | Pass or fail per class with evidence kept | Research-agent baseline result |
| [AS-09](01-requirements.md#as-09) Dashboard visibility | Role-based tests show only permitted panels, counts, relations | Dashboard visibility compliance |
| [AS-10](01-requirements.md#as-10) Decision lineage | Scripted trajectory yields a chain linked with the corrected edges, verified; the unselected proposal, the exposure-only lesson, and the unverified execution are handled correctly | Evidence-supported stage coverage, inferred-stage violations, attribution confidence, verified-outcome coverage |
| [AS-11](01-requirements.md#as-11) Self-reinforcing model claim | Repeated and summarized assistant claim stays labeled `continuity`; grounding blocked until independent evidence | Ineligible-grounding attempts blocked, projection-policy parity |
| [AS-12](01-requirements.md#as-12) Protected regression | Scripted candidate with better average and one protected failure is blocked; shadow output never reaches production | Protected-regression count, promotion evidence sufficiency, shadow coverage |
| [AS-13](01-requirements.md#as-13) Session continuity | Scripted compaction and handoff; resumed agent neither repeats the irreversible step nor loses the constraint; checkpoint absent from organizational retrieval | Resume success, repeated-work rate after resume, lost-constraint failures, checkpoint completeness |
| [AS-14](01-requirements.md#as-14) Dependency change | Scripted tool change yields owner proposals for the Skill and units, automatic projection repair, and a rollback path for each action | Skill-drift detection, projection repair time, canonical-rewrite violations |
| [AS-15](01-requirements.md#as-15) Tampered record | Scripted alteration fails verification before grounding and promotion; the record is quarantined and the incident audited; new records under a revoked identity are untrusted; a signed but unsupported claim stays ineligible | Failed-verification count, revoked-identity exposure, time to quarantine |
| [AS-16](01-requirements.md#as-16) Purpose-dependent grounding | Same unit grounds the explanation and is labeled `hypothesis` for the approval; approval fails closed; both decisions logged; no new unit version | Per-request grounding coverage, ceiling breaches, ineligible-grounding attempts blocked |
| [AS-17](01-requirements.md#as-17) Retrieval-policy shadow | Arms share one comparison basis; selections logged and divergence measured; the message is recorded, not sent; a bypass attempt is blocked and stops the run; condition mismatch is flagged and not counted | Shadow side-effect violations, snapshot parity, selection divergence |
| [AS-18](01-requirements.md#as-18) Crash mid-task | Process killed by script between journaled intent and result; resumed agent does not repeat the completed action, checks the unfinished one, keeps the constraint; checkpoint age recorded | Checkpoint age, crash-resume success, duplicate irreversible actions |
| [AS-19](01-requirements.md#as-19) Concurrent maintenance under legal hold | Scripted owner edit between proposal and commit, retried expiry, and a held duplicate; stale proposal rejected, retry applied once, held unit untouched, redirect resolves | Stale-action rejections, maintenance conflicts, legal-hold violations, broken-reference count |
| [AS-20](01-requirements.md#as-20) Key events over time | Scripted rotation, revocation, compromise with a compromise time, and algorithm retirement on a signed history; only records after the compromise or revocation lose trust; retiring-algorithm records re-attested | Compromise exposure, rotation false flags, unverifiable history, algorithm-migration coverage |

## 5. Phase gates

Which measurements must be live before each phase can exit.

| Phase | Measurements that must be live at exit |
|---|---|
| Phase 0 | Task classes defined; quality judgment per class; attribution window and method; net value weights, sample size, period; latency target ([OD-06](01-requirements.md#od-06)); retention windows ([OD-07](01-requirements.md#od-07)); admission thresholds ([OD-03](01-requirements.md#od-03)) initial values; purpose classes for grounding; recovery point per task class; action-class owners and legal-hold process; trust roots and trusted time source |
| Phase 1 | Capture volume, trajectory completeness, candidate yield, evidence link completeness, admission rate and queue age, retrieval latency vs [OD-06](01-requirements.md#od-06), selection size vs budget, empty result rate, selection log completeness, advisory framing compliance, outcome signal coverage, helped / misled / ignored / corrected rates, audit completeness, separation of duties, adapter availability, grounding-ceiling coverage, per-request grounding coverage, ceiling breaches (zero), ineligible-grounding attempts blocked, checkpoint completeness, resume success, checkpoint age, crash-resume success, duplicate irreversible actions (zero) |
| Phase 2 | Promotion rate and gate compliance, cross-scope leakage (zero), policy precedence violations (zero), legal-hold violations (zero), broken-reference count (zero), contradiction detection and resolution time, deprecation response, projection staleness, pending owner actions, dashboard visibility compliance, retention compliance, maintenance backlog and lag, stale-action rejections, canonical-rewrite violations (zero), integrity verification coverage, failed-verification count, time to quarantine, compromise exposure (zero), unverifiable history (zero) |
| Phase 3 | Time to weakening, harmful exposure, canary throughput, shadow coverage, shadow side-effect violations at zero, snapshot parity, protected-regression count, promotion evidence sufficiency, Skill Candidate gate status, net value per task class, research-agent baseline result, share of classes with positive net value, repeat-mistake rate, observed-work consent and redaction compliance, selection divergence, comparison uncertainty, evidence-supported stage coverage, inferred-stage violations (zero) |

## 6. Dashboard mapping

The management dashboard ([REQ-NFR-11](01-requirements.md#req-nfr-11)) renders a subset of this catalog. Everything shown is permission-filtered, identifiers and aggregates only.

| Panel | KPIs from this catalog |
|---|---|
| [A] Users and agents | Active users, active agents, scopes held per user, capture volume, injections per day |
| [B] Memories | Units by class, category, scope, state; admission queue age; promotion rate; weakened share |
| [C] Relations | Contradiction detection rate, open contradictions, supersessions, cross-scope links, path to evidence |
| [D] Statistics | Capture volume, admission rate, retrieval latency p50 / p95, selection size vs budget, helped / misled / ignored / corrected rates |
| [E] Net value | Cost, benefit, and net value per task class, scope, version; research-agent baseline pass / fail |
| [F] Health and actions | Adapter availability, projection staleness, policy precedence violations, deprecation response, open contradictions, pending owner actions, maintenance backlog and lag, stale-action rejections, legal-hold violations, broken-reference count |
| [G] Trust and learning | Grounding-ceiling coverage, per-request grounding coverage, ceiling breaches; resume success, crash-resume success, checkpoint age; shadow coverage, promotion evidence sufficiency, shadow side-effect violations; failed-verification count, time to quarantine, compromise exposure, unverifiable history |

## 7. Reporting cadence and reviews

| Cadence | Audience | Content |
|---|---|---|
| Continuous | Platform admin, security reviewer, compliance | Availability, latency, staleness, fail-closed events, leakage and precedence alerts, ceiling breaches, shadow side-effect violations, legal-hold violations, failed verifications, compromise exposure |
| Daily | Memory owners, platform admin | Queues and backlogs, admission activity, completeness checks |
| Weekly | Memory owners | Outcome signals per unit version, weakening candidates, contradiction and deprecation items, capture and delivery trends, resume and crash-resume results, maintenance backlog and stale-action rejections, shadow decisions pending |
| Monthly | Memory owners, product owner, security reviewer | Net value per task class, cost share, promotion and lifecycle summary, governance and retention compliance, automatic-action rollback rate, key rotation and algorithm migration |
| Quarterly | Product owner, security and compliance, policy owners | Research-agent baseline results, share of classes with positive net value, recovery drills, threshold and weight review, acceptance scenario re-runs, protected-task sets per capability class, low-risk maintenance action list and action-class owners, recovery points, trust-root health (rotation, compromise drills, unverifiable history) |

Every threshold marked Proposal is reviewed at the quarterly review against observed distributions and adjusted by the owner.

## 8. Anti-metrics

These are tracked as inputs but must never be used as goals or success claims.

| Anti-metric | Why it misleads | Use instead |
|---|---|---|
| Memory size (units stored) | A warehouse grows; a memory curates. Size says nothing about value | Share of classes with positive net value; helped rate |
| Retrieval or injection count | Delivery is cheap; relevance and help are not | Helped rate, ignored rate, selection size vs budget |
| Retrieval frequency or popularity of a unit | Frequency is not evidence and never justifies promotion ([REQ-SC-02](01-requirements.md#req-sc-02)) | Per-version utility and promotion gate records |
| Token savings alone | A system can save tokens while the work gets worse ([REQ-NFR-10](01-requirements.md#req-nfr-10)) | Net value with the quality delta shown separately |
| Model confidence or unit confidence score | Confidence is a hypothesis, not an outcome ([REQ-EV-02](01-requirements.md#req-ev-02)) | Outcome signals per version |
| Citation or "sounds relevant" ratings | Relevance ratings do not measure outcome improvement | Quality judged against what happened to the work |
| Inherited utility across versions | A changed lesson must earn its own record ([REQ-EV-04](01-requirements.md#req-ev-04)) | Per-version utility from zero |
| Zero empty results | Padding to look useful is a defect ([REQ-OUT-03](01-requirements.md#req-out-03)) | Empty result rate reviewed in context |

## 9. Data handling for metrics

- Metrics pipelines carry identifiers, versions, counts, durations, and signal labels. They never carry evidence payloads or memory content ([REQ-SEC-07](01-requirements.md#req-sec-07)).
- Metrics stores are encrypted at rest and reached over mutual TLS like every other store ([REQ-SEC-02](01-requirements.md#req-sec-02), [REQ-SEC-03](01-requirements.md#req-sec-03)).
- Metrics are computed from projections and precomputed aggregates, not live scans of the evidence log or the managed store.
- Every metric view is filtered by the viewer's identity. Restricted items are absent or unknown, never zero ([REQ-OUT-11](01-requirements.md#req-out-11)).
- Per-person metrics (for example observed-session discard rate or outcome signals by individual) are reported only as aggregates at team scope or wider, and are subject to the organization's privacy review for employee monitoring before any individual-level view exists.
- Evaluation, A/B, and baseline runs stay inside organization-controlled systems on approved endpoints ([REQ-SEC-08](01-requirements.md#req-sec-08), [REQ-SEC-09](01-requirements.md#req-sec-09)).
