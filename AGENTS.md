# AGENTS.md

Rules for anyone, human or agent, who edits this repository. The repository is a set of organization-neutral foundation documents for a Company Brain. Read [README.md](README.md) first for what each document covers and the reading order.

## Document roles

Each document has one job and one type. Do not move content between roles.

| Document | Type | Owns |
|---|---|---|
| `01-requirements.md` | Normative | Every requirement, acceptance scenario, open decision, and the glossary |
| `02-functionality.md` | Descriptive | Behavior from the user's and agent's view; journeys; agentic flows F1–F8 |
| `03-architecture.md` | Normative where RFC 2119 words appear | Object model, components `[1]`–`[10]`, paths, lifecycle, phases, rationale, constraints checklist |
| `04-kpi-and-measurements.md` | Normative for the catalog | KPI catalog, measurement protocols, phase gates, anti-metrics |
| `05-source-map.md` | Informative | The only list of external sources and their traceability |
| `README.md` | Index | Document table, conventions, the ten components, the ten rules |

If a document is added or renamed, update the README table, the reading order, and the "Related documents" lines at the top of the affected documents.

## Identifiers

- Formats: `REQ-<AREA>-<nn>`, `AS-<nn>`, `OD-<nn>`, `SRC-<Cat><nn>`, flows `F1`–`F8`, components `[1]`–`[10]`. Areas are `OBJ IN PR ME EV OUT SC GOV SEC NFR`.
- IDs are stable. Never renumber, reuse, or delete one. To remove, keep the row and mark it retired with a reason.
- A new ID takes the next free number in its area or category.
- Every `REQ`, `AS`, and `OD` row carries an HTML anchor whose id is the lowercase ID, for example `<a id="req-out-17"></a>REQ-OUT-17`.
- Requirements are defined only in `01-requirements.md`. Other documents reference them; they never restate or redefine them.

## Cross-references

- Link with relative paths and anchors: `[REQ-ME-04](01-requirements.md#req-me-04)`, `[§3](03-architecture.md#3-logical-architecture)`.
- Section anchors are GitHub heading slugs. If a heading changes, fix every link to it.
- Inside tables the `REQ-` and `SRC-` prefixes may be dropped to save width, as the existing tables do.
- Every link must resolve. Check before finishing (see below).

## Changing content

- **New requirement**: add the row and anchor in `01`, trace it from any acceptance scenario it serves, and record it in `05` §3 (if it has a source) or §5 (if it does not).
- **New source**: follow `05` §7 exactly. `03` §16 must not cite a source missing from `05`.
- **New KPI**: use the seven-column catalog table in `04`. Unapproved numeric targets are marked **Proposal**.
- **New open decision or acceptance scenario**: add to the table in `01` and reference it from `02`, `03`, or `04` where it is exercised.
- Anything that changes one of the ten rules in the README must change the constraints checklist in `03` §17 and the requirement it cites.

## Language and style

- Normative statements use RFC 2119 keywords in capitals. Descriptive text does not.
- Mark recommended approaches or starting thresholds awaiting owner approval with **Proposal**.
- Organization-neutral: no company names, people, hosts, IP addresses, internal URLs, internal service names, or policy titles. Public sources only. Never paste internal or customer material.
- Diagrams are plain ASCII in fenced code blocks. No Mermaid, no images.
- Numbered headings: `## n. Title`, `### n.m Title`. Catalogs are tables. Plain, terse English.
- Keep this file slim. It states rules, not structure. Do not add a folder map or a per-section summary.

## Checks before finishing

1. Every relative link and `#anchor` resolves to an existing file, heading, or `<a id>`.
2. No duplicate IDs; no anchor without its ID and no ID without its anchor in `01` and `05`.
3. A scan for company names, email addresses, IP addresses, and internal hostnames finds nothing.
4. The README document table still matches the files on disk.
