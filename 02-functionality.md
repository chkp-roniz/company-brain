# Company Brain — Functionality

This document describes **what the Company Brain does** from the point of view of the people and agents who use it. It explains the four functions, the capabilities inside each, the user and agent journeys, and the behaviors that must hold at every surface. Requirement IDs (`REQ-…`) refer to [01-requirements.md](01-requirements.md). Component numbers `[n]` refer to [03-architecture.md](03-architecture.md) [§3](03-architecture.md#3-logical-architecture).

## 1. What the system is for

The Company Brain captures the **experience** created during real work (what was tried, what failed, what was corrected, why a decision changed), converts it into **governed memory**, and delivers the **minimum sufficient context** to the right person or agent at the moment of decision or action.

It is not a search layer over documents. Systems of record already hold what the organization knows or decided. The Brain also remembers what the organization learned on the way.

Three principles shape every capability:

1. **Experience, not documents.** The unit of capture is the trajectory; the unit of memory is the experience unit; delivery is a budgeted selection, not a search result.
2. **Separation of duties.** The agent that does the work never writes shared memory. Reflection proposes, admission decides, memory stores, evaluation corrects.
3. **Governance first, semantics second.** Identity, permissions, policy, validity, and authority filters run before any ranking. Projections are never authoritative. Absence means unknown.

## 2. The four functions

```
   INPUT ---> PROCESSING ---> MEMORY ---> OUTPUT ---> (what happened next) ---+
     ^                                                                        |
     +------------------------------------------------------------------------+
   governance (identity, permissions, policy precedence, audit) applies to every arrow
```

| Function | What the user or agent experiences |
|---|---|
| **Input** | Work is captured as it happens, inside the tools already in use. Nobody writes documentation for the Brain. |
| **Processing** | Lessons, decisions, and contextual solutions are extracted from the work and linked to the evidence that supports them. |
| **Memory** | Lessons are kept with owner, scope, confidence, and version. They are weakened, superseded, or expired when they stop being trustworthy. |
| **Output** | The smallest useful set of memory arrives before the model reasons, with provenance, inside the tool. Nothing arrives when nothing applies. |
| **Governance** (cross-cutting) | Policy wins everywhere. Who may capture, infer, store, see, and promote is decided per identity and scope. Every change is audited and reversible. |
| **Evaluation** (the feedback loop) | Whether delivered memory helped, misled, was ignored, or was corrected is recorded and changes what is delivered next. |

## 3. Capabilities by function

### 3.1 Input: capture

**Official knowledge.** Documents, databases, code, tickets, procedures, and policies are ingested from systems of record ([REQ-IN-01](01-requirements.md#req-in-01)). Code graphs (symbols, call and dependency relations, ownership, version history) are ingested beside the code and tagged with repository, commit, and builder version so they can be re-derived when the code changes ([REQ-IN-15](01-requirements.md#req-in-15)).

**Work trails.** Agent and coding-tool sessions, conversations, corrections, tool calls and results, code reviews, decisions, failed attempts, retries, and outcomes are captured as trajectories ([REQ-IN-02](01-requirements.md#req-in-02), [REQ-IN-03](01-requirements.md#req-in-03)). Capture is passive, a by-product of the tools in use ([REQ-IN-04](01-requirements.md#req-in-04)). The system never depends on employees documenting, tagging, or maintaining a second repository ([REQ-IN-05](01-requirements.md#req-in-05)).

**Observed work.** Where a surface can be observed (an agent session, a screen, a terminal, narration), the observation itself is evidence ([REQ-IN-09](01-requirements.md#req-in-09)). The observed person always sees the recording and can pause, stop, or discard it. Secrets and personal data are redacted on the device before anything leaves it ([REQ-IN-13](01-requirements.md#req-in-13)). The system reconstructs intent and ordered steps from several streams, links every step to its evidence, marks unsupported steps as inferred, and asks the observed person to review before anything becomes a candidate ([REQ-IN-10…12](01-requirements.md#req-in-10)).

**Analytical surfaces.** Adapters extend capture to query history, notebooks, BI tools, dashboard changes, and research tools, so experience from ad hoc analytical work is not lost ([REQ-IN-08](01-requirements.md#req-in-08)).

**Import.** Memory from other systems in portable structured formats (for example structured Markdown memory files) can be imported. Imported items keep their original provenance and enter as candidates through admission; they are never trusted because they arrived in a clean format ([REQ-IN-16](01-requirements.md#req-in-16)).

What capture never does: it never creates memory directly. Everything captured is immutable evidence until processing and admission have run ([REQ-IN-06](01-requirements.md#req-in-06)).

### 3.2 Processing: reflection and extraction

Processing turns raw activity into **candidate experience units** ([REQ-PR-01](01-requirements.md#req-pr-01)). It runs when a meaningful boundary is detected (completion, failure, correction, decision), not on every event ([REQ-PR-11](01-requirements.md#req-pr-11)).

What it extracts ([REQ-PR-02](01-requirements.md#req-pr-02)):

- new facts
- decisions together with their reasons
- mistakes, corrections, and candidate lessons
- recurring patterns
- solutions that hold only in a specific context

Each output is classified as one of: evidence, interpretation, lesson, durable knowledge, skill candidate, policy reference, authoritative reference ([REQ-PR-03](01-requirements.md#req-pr-03)). Each candidate carries a link to the trajectory and evidence it came from ([REQ-PR-04](01-requirements.md#req-pr-04)), the conditions under which it applies ([REQ-PR-07](01-requirements.md#req-pr-07)), and a confidence ([REQ-PR-08](01-requirements.md#req-pr-08)).

Processing also:

- finds duplicates and merges them without losing contextual differences ([REQ-PR-05](01-requirements.md#req-pr-05))
- finds contradictions with existing memory and routes them to owners instead of resolving them ([REQ-PR-06](01-requirements.md#req-pr-06))
- normalizes and classifies sensitive content before anything is considered for admission ([REQ-PR-10](01-requirements.md#req-pr-10))
- generalizes recorded procedures by intent toward native tools (CLIs, APIs, connectors) instead of click replay, and marks the steps that stay UI-bound ([REQ-PR-13](01-requirements.md#req-pr-13))
- classifies every artifact from observed work as an Experience unit or a Skill Candidate, never as a Skill, durable knowledge, or policy ([REQ-PR-14](01-requirements.md#req-pr-14))
- links what was proposed, what was decided, what was executed, and what resulted into a decision-lineage chain, one stage at a time and only as far as the evidence goes ([REQ-PR-15](01-requirements.md#req-pr-15))

The executing agent proposes nothing to shared memory and writes nothing to it. Reflection, curation, and admission are separate roles with separate identities ([REQ-PR-09](01-requirements.md#req-pr-09)).

### 3.3 Memory: governed retention

Memory decides what to keep, in what form, for how long, for whom, and when to stop trusting it ([REQ-ME-01](01-requirements.md#req-me-01)).

**Four categories** ([REQ-ME-02](01-requirements.md#req-me-02)):

| Category | Holds | Typical scope |
|---|---|---|
| Working memory | Current objective, state, constraints, unresolved decisions | A session or task |
| Episodic memory | What happened in a specific task or session | Personal or team |
| Durable knowledge | Facts and lessons distilled across experiences | Team to organization |
| Personal context | One person's preferences and working patterns | Personal |

**Working-memory lifecycle** ([REQ-ME-19…25](01-requirements.md#req-me-19), [REQ-ME-31](01-requirements.md#req-me-31)): working memory belongs to one session, task, or handoff and expires with it. Before the session ends, the task is handed off, the context is reset, or it is compacted, the Brain writes a **continuity checkpoint**: objective, done and pending work, open decisions, constraints, dependencies, evidence and authority boundaries, and next actions. Compaction first removes obsolete and duplicate items by rule, then summarizes what is left, and records what it kept, removed, changed, and is unsure about. A checkpoint lets the task resume; it is not organizational knowledge and is never promoted by itself. What should last goes through the normal write path. Resume tests check that nothing is invented, repeated, or lost. A crash gives no warning, so the checkpoint is also updated during the task, after each important change and on a timer. Every action that cannot be undone or that others can see is written to an **action journal** before it runs and again with its result. After a crash, the task resumes from the latest checkpoint and the journal; an action with no recorded result is checked, not run again.

**Background maintenance** ([REQ-ME-26…35](01-requirements.md#req-me-26)): the Brain checks itself on a schedule, when a measure crosses a threshold, and when something changes (a source, tool, schema, or model). It looks for growth, slow retrieval, duplicates, contradictions, stale or expired units, harmful or ignored units, skill drift, changed dependencies, due reviews, and indexes that disagree with canonical memory. A short owner-approved list of low-risk, reversible actions runs automatically, such as rebuilding a stale index. Everything else becomes a proposal in the owner's queue. Maintenance never silently rewrites or deletes evidence or canonical memory, and every action can be traced and rolled back. Each action is based on a known version of the unit; if someone changed the unit in the meantime, the action is rejected and looked at again, and a retry never applies twice. A legal hold, then retention and deletion rules, then owner decisions all come before maintenance. Each kind of action has an owner. A merged or replaced unit stays as a redirect, so nothing that points to it breaks.

**Separate object classes, linked not merged** ([REQ-ME-03](01-requirements.md#req-me-03)): memories and lessons advise; skills execute; policies constrain and stay human-owned; authoritative references point to systems of record.

**What every unit carries** ([REQ-ME-04](01-requirements.md#req-me-04)): identity, source and evidence, owner, creation context, confidence, visibility scope, relevance scope, version, freshness and validity, typed relationships (`derived_from`, `supported_by`, `contradicts`, `supersedes`, `relevant_to`, `promoted_from`), usage and outcome history, source class, its retention and retrieval decisions, and its grounding ceiling.

**Kept, found, and trusted are different decisions** ([REQ-ME-15](01-requirements.md#req-me-15), [REQ-ME-16](01-requirements.md#req-me-16)): a unit can be kept (retention) and returned (retrieval) yet still not be allowed to support a factual claim, a recommendation, or a consequential action (grounding). Grounding eligibility is decided by rules from source class, evidence, authority, validity, freshness, visibility, purpose, task context, and policy, never by how often the unit is seen. The unit stores a **grounding ceiling**, the most it may ever support. Each request then gets its own decision for its purpose: a rough figure may support an informal explanation but not a payment approval. That decision is logged with the selection and never changes the unit.

**Autobiographical memory** ([REQ-ME-17](01-requirements.md#req-me-17), [REQ-ME-18](01-requirements.md#req-me-18)): what an assistant itself said, believed, summarized, or judged about its own work. It is kept so a conversation or task can continue, but it is not evidence that what was said is true. Saying it again, summarizing it, or copying it into another unit does not make it trusted. It becomes grounding-eligible only when independent evidence (a tool result, a verified outcome, an authoritative reference, or a person's confirmation) supports it.

**Lifecycle behaviors**:

- Conflicting lessons coexist when each is valid in its own context. No single global truth is forced ([REQ-ME-07](01-requirements.md#req-me-07)).
- Forgetting means a unit stops influencing work. Its history stays. Deletion is a separate governed operation ([REQ-ME-08](01-requirements.md#req-me-08)).
- Updates are incremental deltas on the affected units. There is never a wholesale rewrite of accumulated memory ([REQ-ME-09](01-requirements.md#req-me-09)).
- Every version is kept and any change can be rolled back ([REQ-ME-11](01-requirements.md#req-me-11)).
- When a source or scope is deprecated (archived repository, retired product, superseded procedure), every unit derived from it is weakened or expired in one audited operation, owners are notified, and the transition is reversible ([REQ-ME-13](01-requirements.md#req-me-13)).
- A Skill Candidate stays a candidate until it has passed all gates: more examples, policy check, outcome evaluation in a safe environment, owner approval, versioning, reversibility ([REQ-ME-12](01-requirements.md#req-me-12)).

### 3.4 Output: context delivery

**Proactive delivery is the primary surface.** Memory arrives before the model reasons, triggered by harness lifecycle events equivalent to `SessionStart` and `UserPromptSubmit` ([REQ-OUT-04](01-requirements.md#req-out-04), [REQ-OUT-05](01-requirements.md#req-out-05)). A question-and-answer tool works only if the person knows what to ask; most expensive mistakes come from not knowing that a lesson exists.

**Minimum sufficient context** ([REQ-OUT-01](01-requirements.md#req-out-01)): the smallest set that fits an explicit budget, to the correct identity, at the moment of decision, with provenance and confidence, inside the tool. Returning nothing is a valid and often desirable result. Results are never padded ([REQ-OUT-03](01-requirements.md#req-out-03)).

**Session-aware** ([REQ-OUT-02](01-requirements.md#req-out-02)): retrieval can use the current objective and trajectory, not only the latest prompt.

**Deterministic filters first** ([REQ-OUT-06](01-requirements.md#req-out-06)): identity, permissions, policy, validity, authority, and explicit resource scope are enforced before any ranking. A ranking change can never widen what a caller may see.

**Progressive disclosure** ([REQ-OUT-09](01-requirements.md#req-out-09), [REQ-OUT-10](01-requirements.md#req-out-10)):

```
   L0  compact relevance signal   -->  L1  navigational overview   -->  L2  full unit + evidence
       very small; for selection          what exists, how it relates,      canonical, on demand
       derived, rebuildable               confidence; decide what to open   governed, audited
```

The budget is spent on L0 and L1 by default. L2 is pulled on request or when the task class requires it. Every L0 or L1 element carries the identity of its canonical unit, so the consumer can always reach the evidence ([REQ-OUT-15](01-requirements.md#req-out-15)).

**Advisory framing** ([REQ-OUT-16](01-requirements.md#req-out-16)): delivered memory is marked as advisory data with provenance and bounds. Neither humans nor models should treat it as policy or as a command.

**Permitted uses** ([REQ-OUT-18](01-requirements.md#req-out-18)): a unit that may be retrieved but not used for grounding in this request is delivered, if relevant, with a permitted-use label: `continuity` (what was said or done before), `hypothesis` (a lead to check), `disputed` (one side of an open contradiction), `audit` (context for a review), or `reflection` (input to learning). It can explain, suggest, or show disagreement; it cannot support a claim or an action. An action that needs grounding and finds only labeled non-grounding memory treats the grounding as unknown and fails closed. L0 and L1 views carry the same ceiling and labels as their canonical unit.

**Absence means unknown** ([REQ-OUT-11](01-requirements.md#req-out-11), [REQ-OUT-13](01-requirements.md#req-out-13)): an empty, filtered, or unavailable result is reported as unknown. It is never read as approval, as "no constraints apply", or as "no prior decision exists". An action that requires authoritative memory or policy stops when that authority cannot be retrieved. The reason for absence may be withheld if revealing it would disclose restricted information ([REQ-OUT-12](01-requirements.md#req-out-12)).

**Exploration tools** ([REQ-OUT-07](01-requirements.md#req-out-07)): MCP or API tools let a consumer inspect a unit's source, expand to L2, list related units, and report an outcome. They are never the only path by which memory reaches the model.

**Transcript pitfall review** ([REQ-OUT-17](01-requirements.md#req-out-17)): given a full agent transcript, live at a checkpoint or completed, the Brain returns what applies to that work:

- lessons that apply
- known pitfalls the work is heading into or has hit
- contradictions between claims or actions in the transcript and admitted memory
- policy conflicts, shown as constraints with their human owner, never paraphrased as advice

Same filters, budget, provenance, advisory framing, and logging as any delivery. The review never writes to memory.

**Export** ([REQ-IN-16](01-requirements.md#req-in-16)): memory can be exported in portable structured formats. Only what the requester may see is exported, each item carries provenance, scope, version, and classification, and exports stay inside organization-controlled destinations.

### 3.5 Governance: the control plane

Governance applies to every function ([REQ-GOV-01](01-requirements.md#req-gov-01)):

| Function | What governance decides |
|---|---|
| Input | What may be captured, from which sources, under which identity; classification at capture |
| Processing | What may be inferred; which model endpoints may be used; sensitivity handling before admission |
| Memory | What may be remembered, for how long, at which scope; owner-controlled scope changes; expiry; deletion; rollback |
| Output | Who may see and who should receive; policy precedence over local approval; fail closed when permission or policy data is missing |
| Policy | Human ownership; authenticated changes; deterministic precedence across every interface and execution environment |
| Integrity | Which records must be attested; managed signing identities; trust roots, rotation, revocation, expiry, and compromise times; trusted time source; algorithm retirement dates; quarantine on failed verification |

Key behaviors:

- Organizational policy has deterministic precedence over any local setting, user preference, personal memory, or locally approved agent action, in every interface and environment ([REQ-GOV-02](01-requirements.md#req-gov-02)).
- Capture, inference, storage, retrieval, and promotion are authorized separately, per identity and scope ([REQ-GOV-03](01-requirements.md#req-gov-03)).
- Only the unit's owner, or an authorized owner at the target scope, can change a unit's scope ([REQ-GOV-04](01-requirements.md#req-gov-04)).
- Maintenance acts automatically only from the owner-approved low-risk list; every other maintenance change is a proposal an owner decides, and every action keeps its prior version and rollback path ([REQ-ME-29](01-requirements.md#req-me-29), [REQ-ME-30](01-requirements.md#req-me-30)).
- Contradictions are recorded with both sides, their owners, and their evidence, and routed to owners ([REQ-GOV-06](01-requirements.md#req-gov-06)).
- A unit derived from restricted sources is never visible more widely than its most restrictive source until an owner approves wider visibility ([REQ-GOV-10](01-requirements.md#req-gov-10)).
- High-impact records (shared memory, policy references, Skill versions, admission decisions, verified outcomes) carry integrity attestations bound to managed identities and organization-managed keys. A record that fails verification is quarantined and never used for grounding or promotion. A valid signature shows who wrote a record and that it is unchanged; it does not make the claim true. Signatures stay checkable over time: a routine key change does not cast doubt on older records, and only records signed after a key was compromised lose trust. Each signature has a trusted time, and records are signed again before an old algorithm is retired ([REQ-SEC-16…19](01-requirements.md#req-sec-16)).
- The system never authors or changes policy. It may propose a clearly flagged policy candidate for human approval ([REQ-GOV-11](01-requirements.md#req-gov-11)).

### 3.6 Evaluation: the feedback loop

The evaluation loop is what makes the Brain a memory rather than a warehouse. For every run it records the complete set of units supplied (ids and versions), the query form, the filters, the reasons for inclusion and omission, the consumer's model, harness, task class, and scope, and the outcome signals: helped, misled, ignored, corrected ([REQ-EV-05](01-requirements.md#req-ev-05), [REQ-EV-06](01-requirements.md#req-ev-06), [REQ-EV-10](01-requirements.md#req-ev-10)). Signals are captured implicitly (later corrections, retries, reverts, failed checks) and explicitly (consumer feedback in the tool) ([REQ-EV-11](01-requirements.md#req-ev-11)).

What the loop does:

- measures validity and utility separately ([REQ-EV-01](01-requirements.md#req-ev-01))
- treats every unit as a hypothesis proven by outcomes, not by confidence, retrieval frequency, or popularity ([REQ-EV-02](01-requirements.md#req-ev-02))
- scopes utility to version, task class, audience, model, harness, and context ([REQ-EV-03](01-requirements.md#req-ev-03))
- never lets a new version inherit the reputation of the previous one ([REQ-EV-04](01-requirements.md#req-ev-04))
- attributes outcomes rather than crediting or blaming every supplied unit ([REQ-EV-06](01-requirements.md#req-ev-06)), following the decision-lineage chain where one exists. It keeps three things apart: memory that was **available** (exposure), memory that was **used** (influence, shown by evidence), and memory that **changed the result** (causal evidence from a controlled comparison). Only used memory earns credit or blame; an outcome with no evidence of influence stays unattributed
- counts an outcome as verified only when an explicit acceptance criterion was checked by a named evaluator; a successful execution alone is not a verified outcome ([REQ-ME-14](01-requirements.md#req-me-14))
- weakens or suspends a version when harmful signals accumulate ([AS-03](01-requirements.md#as-03))
- tests unvalidated units only in safe environments or canaries and never weakens policy or security controls to do so ([REQ-EV-08](01-requirements.md#req-ev-08), [REQ-EV-09](01-requirements.md#req-ev-09))
- runs **shadow evaluation** for candidate skills, rules, procedures, and memory-policy changes: every arm works from the same task, permissions, and snapshots of sources and external state. A candidate skill gets the same context as the active path; a candidate retrieval policy picks its own context from the same evidence, and both picks are logged. Outputs are compared with the active path and a no-candidate baseline. The candidate never controls production and never changes anything: its tools are read-only, and an action it would take is recorded, not done ([REQ-EV-13…19](01-requirements.md#req-ev-13))
- counts promotion evidence by distinct contexts, not by repeated runs, and blocks promotion on any protected-task regression; the evidence threshold is set by risk and blast radius ([REQ-EV-16](01-requirements.md#req-ev-16), [REQ-EV-17](01-requirements.md#req-ev-17), [REQ-ME-12](01-requirements.md#req-me-12))
- computes **net value** per task class: outcome benefit minus the cost of the write path, injected context, and retrieval ([REQ-NFR-10](01-requirements.md#req-nfr-10))
- runs a **research-agent baseline**: for each task class, Brain-supplied context must match or beat a from-scratch research agent on quality while costing less in tokens and latency ([REQ-EV-12](01-requirements.md#req-ev-12))

Measurement definitions, protocols, and targets are in [04-kpi-and-measurements.md](04-kpi-and-measurements.md).

## 4. Scope and promotion

Scopes: personal → team → product → organization. A unit may begin at any scope. Most experience starts personal.

```
  PERSONAL ==[gate]==> TEAM ==[gate]==> PRODUCT ==[gate]==> ORGANIZATION

  each gate requires ALL of:
    [ ] sufficient evidence and demonstrated utility at the current scope
    [ ] sensitive information removed or protected for the wider audience
    [ ] a named owner at the new scope who accepts the unit
    [ ] permission to widen visibility
  repetition alone never promotes; demotion and rollback are always possible
```

**Visibility and relevance are different questions** ([REQ-SC-04](01-requirements.md#req-sc-04)). Visibility: may this identity access this unit? Relevance: should this consumer receive it for this task? A unit that is visible but irrelevant is noise and is not delivered. A unit that is relevant but not visible is never delivered, and its absence is reported as unknown.

**One user, many scopes.** A person normally belongs to a personal scope, one or more teams, a product, and the organization. Filters evaluate the union of what that identity may access; ranking runs inside that set and never widens it. The scope that made a unit visible is shown in its provenance.

## 5. Memory lifecycle (functional view)

```
                 admit              promote (wider scope)
   CANDIDATE ----------> ADMITTED ------------------------> PROMOTED
       |                    | ^                                |
       | reject             | | re-validate                    |
       v                    v |                                v
   DISCARDED             WEAKENED  <---------------------------+   harmful outcomes, contradiction,
   evidence kept            |                                      low utility, source deprecation
                            | expire / supersede / archive
                            v
                         ARCHIVED  ---- governed deletion ---->  DELETED
                         not retrieved; history and evidence preserved
```

| What happens | When | Who decides | Reversible |
|---|---|---|---|
| Candidate is admitted | Passes validation, sensitivity, ownership, and policy gates | Admission service, per configured thresholds | Yes (rollback) |
| Candidate is discarded | Fails admission | Admission service | Evidence kept; can be re-proposed |
| Unit is promoted | Meets the scope gate | Owner at target scope | Yes (demotion) |
| Unit is weakened | Harmful or ignored outcomes, contradiction, low utility | Evaluation loop and owner | Yes (re-validation) |
| Unit is superseded | A newer version replaces it | Admission service | Old version archived, not overwritten |
| Unit is archived | Expiry, retention, or owner action | Governance plane, owner | Yes, within retention |
| Unit is deleted | End of retention or governed deletion request | Governance plane only | No; secure deletion or anonymization |
| Population is weakened or expired | A scope or source is deprecated | Deprecation detection proposes; admission and governance apply; owners notified | Yes, by re-validation or rollback |
| Skill Candidate becomes a Skill | All gates pass: more examples, policy check, safe-environment evaluation, owner approval, version, reversibility | Owner, with evaluation evidence | Yes (disable or roll back) |

## 6. Journeys

### 6.1 Worker: a lesson travels without anyone writing it down (AS-01)

1. An engineer's first attempt at a task fails. A second attempt succeeds. Nothing is documented.
2. The harness emitted the session as evidence. The Brain detected the correction boundary, reflected, and extracted a candidate lesson with usage bounds and evidence links.
3. Admission validated the candidate, checked sensitivity, assigned an owner at team scope, and admitted it as a versioned unit.
4. Weeks later, a teammate starts a similar task. On `SessionStart` the Brain injects an L0/L1 lesson, within budget, marked advisory, with provenance.
5. The teammate expands to L2 and sees the original evidence. The task succeeds first time. The outcome is recorded as "helped" against that unit version.

### 6.2 Two teams, two truths (AS-02)

1. Two teams learn conflicting lessons, each valid in its own context.
2. Processing flags the contradiction. Both units are kept with their owners and evidence; a `contradicts` relation links them; owners are notified.
3. A member of either team retrieves the unit that matches their context. No global truth is forced. Owners can see the contradiction on the dashboard and decide if it needs resolution.

### 6.3 A good lesson goes bad (AS-03)

1. A lesson that used to help starts producing worse outcomes after an environment change.
2. Outcome signals (reverts, failed checks, explicit "misled" feedback) accumulate against the current version.
3. The evaluation loop weakens the unit. It stops being delivered. History and evidence are preserved. The owner is notified and can re-validate, supersede, or roll back.

### 6.4 Policy wins everywhere (AS-04)

1. An agent runs with a locally approved setting that conflicts with organizational policy.
2. Whatever interface or environment it uses, the policy prevails. The conflict is written to audit.

### 6.5 Nothing applies (AS-05)

1. A task has no reliable memory.
2. The Brain returns nothing and reports unknown. No plausible lesson is manufactured. If the task requires authoritative memory or a policy decision, it stops and escalates.

### 6.6 Observed work becomes a Skill Candidate (AS-06)

1. An employee records one run of a repetitive task with notice and full control of the recording.
2. Secrets and personal data are redacted on the workstation. Raw evidence is stored immutably.
3. The Brain reconstructs intent and ordered steps and links each step to evidence. The employee reviews, corrects, redacts, or rejects. The review is itself evidence.
4. The reconstruction is generalized by intent toward native tools. The artifact is classified as a Skill Candidate.
5. The candidate runs only in the evaluation sandbox until all gates pass. It becomes a Skill only by owner decision, versioned and reversible.

### 6.7 An agent asks for a pitfall review (REQ-OUT-17)

1. At a task boundary, an agent or a person sends a full transcript for review.
2. The Brain derives objectives, tools, claims, and decisions from the whole conversation, runs the same filters as any retrieval, and returns applicable lessons, pitfalls, contradictions with memory, and policy conflicts, within budget, with provenance.
3. What the agent does next (changes course, ignores, corrects) is recorded as outcome signals. Contradictions found are recorded as evidence and routed to owners through contradiction triage.

### 6.8 Owner: reviewing and governing

1. An owner opens the management dashboard and sees only what their identity may see: admission queue, pending promotions, open contradictions, deprecation notices, net value per task class.
2. They approve a promotion, resolve a contradiction, or roll back a change. Each action opens the matching admission or governance workflow under their identity and is audited. The dashboard itself commits nothing.

### 6.9 A suggestion, a different decision, a verified result (AS-10)

1. An agent, using a delivered lesson, proposes fix A. The engineer reviews it and selects fix B instead.
2. Fix B is deployed. The deployment succeeds, but nothing has yet checked that the problem is gone. This is an Execution, not a verified outcome.
3. The next day the test that defines "fixed" passes. The Outcome is recorded as verified, with the evaluator, the acceptance criterion, the time, and the uncertainty.
4. The chain is linked: Proposal A (proposed by the agent; the lesson is linked as `informed_by`, which records that it was available, not that it caused anything) ← Decision B (decided by the engineer, under their authority) ← Execution (implements Decision B) → verified Outcome. Proposal A stays in history and is never shown as the decision taken.
5. Evaluation credits the outcome to Decision B. The lesson behind Proposal A gets no credit for a result it did not produce; the override is kept as a signal for its owner.
6. A week later a regression shows fix B was incomplete. The correction is linked to the original Outcome, which is kept, not overwritten.

### 6.10 A better average, a protected regression (AS-12)

1. A candidate rule is created from repeated work. It enters shadow evaluation: on each matching task it gets the same task, permissions, snapshots, and context as the active path, works through read-only tools, and records what it would do. The active path still decides.
2. Over the following weeks [10] compares the candidate, the active path, and a no-candidate baseline on the same tasks. It records applicability, abstentions, agreement, execution validity, verified outcomes, and negative transfer. Ten replays of one task count as one context.
3. The candidate is better on average across many distinct contexts, but it fails one task the owner marked as protected.
4. The owner opens the promotion request. The record shows the aggregate gain, the contexts counted, the threshold for this capability class and risk, and the protected regression. Promotion is blocked.
5. The candidate is revised. The new version starts its evidence again; it does not inherit the old version's results.

### 6.11 A handoff that does not lose or invent state (AS-13)

1. An agent is halfway through a long task. It has run a database migration (irreversible), a reviewer has said "do not touch the public API", and the choice between two rollout plans is still open.
2. The context fills up and is compacted, and the task is handed to another agent instance.
3. Before compaction, obsolete and duplicate items are removed by rule. The summary that follows keeps who said what, the reviewer's constraint, the open choice, and the evidence links. The compaction record lists what was removed and marks one test result as uncertain.
4. The checkpoint states: objective, migration done, remaining steps, the API constraint, the open rollout decision, and next actions.
5. The new instance resumes from the checkpoint. It does not run the migration again, keeps the API constraint, and asks for the rollout decision instead of assuming one. It re-checks the uncertain test.
6. Other people searching the Brain do not see the checkpoint. If the session produced a lesson worth keeping, it reaches shared memory only through capture and admission.

### 6.12 A tool changes under a skill (AS-14)

1. A team changes the interface of an internal tool. The change arrives as an event.
2. Maintenance finds a Skill and four lessons that depend on the old interface, and index entries that point to them.
3. Rebuilding the affected index entries is on the low-risk list, so it runs automatically. The action is logged with its trigger, prior state, and rollback path.
4. Changing the Skill or the lessons is not on the list. Maintenance sends the owner a proposal: weaken the Skill until it is re-validated, and review the lessons. The proposal shows the event and which units depend on the tool.
5. The owner accepts weakening the Skill and rejects one lesson change. Both decisions go through admission and are audited. Nothing was rewritten before the owner decided.

### 6.13 A crash in the middle of work (AS-18)

1. An agent has run a database migration and is about to send a release notice. A reviewer added a constraint a few minutes ago.
2. The checkpoint was updated after the migration and after the constraint. The migration is in the action journal with its result; the notice is in the journal as intended.
3. The agent process is killed. No end-of-session step runs.
4. A new instance resumes from the latest checkpoint and the journal. It does not run the migration again and keeps the constraint.
5. The notice has no recorded result, so the instance checks whether it was sent before doing anything. It was not, so it sends it once, using the same idempotency key.
6. The checkpoint's age at the crash is recorded and compared with the recovery point for this task class.

## 7. Agentic flows

An agentic flow is a place where a model decides something. Flows run on the existing components. No flow writes shared memory directly; the model-driven step writes only evidence, candidates, recommendations, or proposals, and deterministic services commit changes.

| # | Flow | Trigger | May write (never shared memory directly) |
|---|---|---|---|
| F1 | Capture and reflection: detect a boundary, reflect, extract candidate lessons with evidence links and usage bounds | Harness events, session end, boundary detected | Evidence events; candidates to the admission queue |
| F2 | Admission and promotion review: check a candidate or promotion request against validation, sensitivity, ownership, scope, and policy gates | Candidate arrives; owner requests promotion; scheduled batch review | A recommendation with reasons; the admission service commits after rule or owner approval |
| F3 | Contradiction triage: find a contradiction, preserve both sides, propose the `contradicts` relation, route to owners | Candidate time; periodic scan; contradiction found in a transcript review | A contradiction record and a routing notice; never resolves |
| F4 | Deprecation detection: watch sources and scopes for deprecation and list derived units | Periodic source-status scan; adapter reports a status change | A deprecation evidence event and a proposal listing affected units |
| F5 | Transcript pitfall review | A consumer, on a live checkpoint or completed transcript | Selection logs and outcome events only |
| F6 | Research-agent baseline comparison: run the Brain arm and the from-scratch arm for a task class and judge both | Scheduled per task class; on demand; after a large memory change | Evaluation records; a weakening proposal when the class fails |
| F7 | Candidate evaluation in canaries and shadow: run an unvalidated or changed unit in a safe canary, or a candidate in shadow beside the active path, and record outcomes per version | A unit is admitted as unvalidated or a new version is created; scheduled canary review | Utility records for the version; a promote, hold, or weaken recommendation |
| F8 | Skill-candidate gating: track and run the gates for a Skill Candidate | Owner requests promotion; a gate result arrives | Gate results on the candidate; sandbox runs only; a Skill is created only by owner decision |

Common rules: every flow runs under its own scoped, least-privilege identity with default-deny egress and approved model endpoints only; evidence and memory it reads are untrusted input and never instructions; flows are idempotent; every run is logged; a flow may propose a flagged policy candidate but never changes policy.

F5 and F6 are the only flows on the interactive read path, so both stay inside latency and budget rules.

## 8. Management dashboard

A read-only view over data the system already keeps. It adds no store and no write path to memory.

```
   MANAGEMENT DASHBOARD   scope [ all my scopes ]   period [ 30 days ]   viewer: <identity>
   read-only; every panel shows only what the viewer may see; identifiers and aggregates only

   [A] USERS AND AGENTS      active users / agents . scopes held . captures and injections per day
   [B] MEMORIES              counts by class / scope / state . admission queue . promotions pending
                             weakened / archived
   [C] RELATIONS             unit --contradicts--> unit . derived_from --> evidence . supersedes
                             cross-scope links
   [D] STATISTICS            capture volume . admission rate . latency p50 / p95 . selection vs budget
                             helped / misled / ignored / corrected
   [E] NET VALUE             per task class: net . baseline pass / fail
                             cost: write path, injected context, retrieval
                             benefit: shorter sessions, fewer retries, less re-research, quality delta
   [F] HEALTH AND ACTIONS    adapters up / total . projection staleness . policy conflicts
                             deprecation notices . open contradictions . pending owner actions
                             maintenance proposals . automatic actions and rollbacks
                             maintenance backlog . stale-action rejections . legal-hold conflicts
                             [approve] [promote] [resolve]  -> run through admission / governance
   [G] TRUST AND LEARNING    grounding: ceiling coverage . per-request decisions . ceiling breaches
                             continuity: resume / crash-resume failures . checkpoint age
                             shadow: candidates . promotion decisions and evidence . side effects
                             integrity: failed checks . quarantined . compromise . unverifiable

   drill-down: panel > unit > L1 > L2 evidence, only where the viewer is permitted
```

Rules: every request is authenticated and authorized server-side; items outside the viewer's permission are absent or shown as unknown, never as a zero that hides restricted units; panels show identifiers and aggregates, with raw content only on a permitted drill-down to L2; actions open the matching workflow under the viewer's identity; queries read projections and precomputed aggregates, not live scans of the evidence log or the store.

Review behavior: each item in [F] and [G] links to the owner review that decides it, for example the protected-task set of a capability class, the low-risk maintenance action list and its action-class owners, or trust-root health. A red item with no owner is itself shown as a pending owner action.

## 9. Integration surfaces

| Surface | Purpose | Notes |
|---|---|---|
| Harness lifecycle hooks | Proactive retrieval on `SessionStart` and `UserPromptSubmit` equivalents; outcome capture on session end or task completion; continuity checkpoint on session end, handoff, reset, or compaction | The primary read path |
| MCP server | Exploration: inspect a unit's source, expand to L2, list related units, report an outcome | Illustrative tools: `memory.explore`, `memory.inspect_source`, `memory.expand_evidence`, `memory.report_outcome`, `memory.review_transcript`. Never the only delivery path |
| Source adapters | Pull or receive evidence from systems of record and work surfaces | The only components that touch external systems; run under scoped identities |
| Code-graph adapter | Ingest code graphs beside the code | Records repository, commit, and builder version; invalidates derived units when the code changes |
| Work-observation recorder | Observe real work on the workstation and emit evidence | On-device redaction; visible recording with pause, stop, discard; evidence goes only to organization-controlled ingest |
| Memory import and export | Move memory in and out in portable structured formats | Import produces candidates only; export is permission-filtered and stays in organization-controlled destinations |
| Output adapters for applications and automations | Deliver context to pipelines, bots, and internal apps through APIs | Same retrieval service, filters, and logging |
| Transcript pitfall review | Review a full agent transcript | API call, MCP tool, or harness hook at a task boundary |
| Owner and administrator UI | Admission queue, contradiction review, scope changes, policy administration, audit, rollback, management dashboard | Human-in-the-loop surface for admission and governance |

## 10. Behaviors that hold everywhere

These hold at every surface and in every phase. An implementation that breaks one is non-compliant.

- The executing agent has no write path to shared memory.
- No monolithic rewrite of accumulated memory, anywhere.
- Deterministic identity, permission, policy, validity, authority, scope, and per-request grounding-eligibility filters run before ranking.
- Memory reaches the model before it reasons; exploration tools are never the only path.
- Projections are derived, invalidated with canonical memory, and inherit its grounding restrictions; never authoritative.
- Empty, filtered, or unavailable results mean unknown; authority-requiring actions fail closed.
- Policies are human-owned and take deterministic precedence everywhere.
- Utility is measured per version and context; versions never inherit reputation; promotion needs evidence across distinct contexts and is blocked by a protected regression.
- Forgetting preserves history; deletion is a separate governed operation.
- Interfaces are harness-neutral.
- Reconstructions of observed work are stored beside raw evidence and reviewed by the observed person before they become candidates.
- Maintenance never silently rewrites or deletes evidence or canonical memory; only owner-approved low-risk, reversible actions run automatically.
- Maintenance acts only on the version it was based on, never overrides a legal hold or retention obligation, and leaves a redirect behind every merge.
- High-impact records are verified before grounding or promotion; a failed check quarantines the record, and a valid signature never makes a claim true.
- A continuity checkpoint resumes its own task; it is never organizational knowledge and never becomes durable memory by itself.
- A crash loses at most the task class's recovery point; an irreversible action with no recorded result is checked, never repeated blindly.
- Artifacts from observed work enter as Experience units or Skill Candidates; no Skill Candidate executes in production before the gates pass.
- Imported memory enters as candidates with provenance preserved; exports are permission-filtered and stay in organization-controlled destinations.
- Deprecation of a scope or source weakens or expires every derived unit, with history preserved and owners notified.
- Net value is measured per task class with an A/B or paired protocol; token savings alone are not the metric.
- Brain-supplied context beats or matches the research-agent baseline on quality at lower cost, or its units become weakening candidates.
- Transcript pitfall review is a read-path mode with the same filters, budget, provenance, and logging; it never writes memory.
- No agentic flow writes shared memory directly.
- One logical memory with scopes is the target; isolated per-group stores are Phase 1 containment only.
- The management dashboard shows only what the viewer may see, as identifiers and aggregates, and commits nothing by itself.
- The security baseline ([01-requirements.md](01-requirements.md) [§7](01-requirements.md#7-security-and-compliance-baseline)) is met in full.
