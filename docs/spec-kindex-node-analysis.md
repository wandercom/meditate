# Spec: Kindex additive-node analysis target class

Status: **draft — awaiting founder ratification.** Agent-authored 2026-08-24.
Derived authority: the founder-reviewed kindex PRD "Lineage Provenance, Grounded
Claims, Merge Receipts, Graph Metrics" (2026-08-24), Review outcome item 3 and
R2b, which assigns Meditate's Analyst the standing-conflict-sweep role over
kindex's additive nodes. Nothing in this document is canonical until a human
ratifies it; where it conflicts with that PRD's review outcome, the PRD wins.

## Purpose

Kindex today serves two contradicting additive nodes to recall identically; a
stale architectural decision misleads every downstream agent the same way. The
PRD's chosen remedy is deliberately not a second detector: Meditate v0.5 already
ships an evidence-grounded, read-only semantic Analyst with local citation
validation and a nomination-only boundary, and kindex is already one of its
required evidence sources. This spec defines kindex's additive nodes as a new
**analysis target class** for that existing Analyst, with three hard properties:

1. **One nomination authority.** Meditate's Analyst becomes the single
   nomination path for contradiction/supersession/staleness findings over
   kindex additive nodes. No second independent demotion authority is created,
   so two systems can never disagree about a node's trust state.
2. **Quarantine, never mutation.** Every finding lands as a kindex quarantine
   *candidate* (the existing `candidate` review flow). Meditate never mutates a
   node, never changes trust state, and never reviews its own findings.
3. **Same fail-closed boundaries.** The secret-sanitization boundary (kindex
   constraint `5e48daf20c91`) and the `kindex_required_failed` /
   `kindex_unavailable` semantics apply to node content exactly as they apply
   to every other evidence source today.

## Vocabulary mapping

The external labels (PRD R2b) map onto the Analyst's existing bounded candidate
classes. **No new candidate class is introduced.**

| External finding | Analyst class | Notes |
| --- | --- | --- |
| contradiction | `contradiction` | Two additive nodes in the same semantic domain cannot both govern the same case. |
| supersession | `temporal_supersession` | Newer admitted evidence (or a newer node) explicitly revises an older node's claim. |
| staleness | `temporal_supersession` | Evidence-grounded staleness is the supersession case where the node's claim has been overtaken. Computable referent staleness (PRD R0: re-hash the referent, compare `true_of`) is kindex's job and is **out of scope** here — Meditate nominates only what evidence supports, never what a digest comparison already proves. |

Over node subjects, **only `contradiction` and `temporal_supersession` are
eligible to become quarantine candidates.** Any other class the Analyst
nominates against a node subject (`underspecified`, `overspecified`,
`wrong_scope`, `enforcement_candidate`, `missing_rule`) is report-only: visible
in Meditate's plan report, never submitted to kindex.

## Target class definition

### Input: read-only enumeration

- The class covers kindex **additive** nodes: types `decision`, `constraint`,
  and `directive`, status `active`. Other node types (concept, watch,
  checkpoint, task, question, skill) are out of scope for v1.
- Enumeration is read-only through the configured kin CLI: one
  `kin list --type <t> --status active --json` per configured type, then one
  `kin show <id> --json` per unique ID, under the same minimal subprocess
  environment and timeouts as the existing kindex evidence reader.
- Enumeration is gated by configuration (`[kindex] analyze_nodes`, default
  `false`). With the flag off, behavior is byte-identical to v0.5.
- When the flag is on and `kin` is absent, the run reports
  `kindex_unavailable` and performs no node analysis; it never pretends the
  sweep ran. When `kin` is installed, any enumeration failure — subprocess
  failure, invalid JSON, or a node ID that fails the local well-formedness
  check — aborts with `kindex_required_failed` before any model call.
- Node volume is bounded (`[kindex] max_nodes`, default 200). Truncation is
  deterministic (stable node-ID order) and reported, never silent.

### Subject projection

- Each enumerated node is projected into the Analyst packet as an **immutable
  analysis subject**, the same trust posture as imported Claude documents
  (`mutable=false`): nominatable, never a disposition target, never writable.
- Node subjects are *not* pre-image directives. They are excluded from the
  total-disposition contract: no keep/replace/remove/relocate/escalate
  disposition exists or is required for a node subject.
- Subject IDs are minted locally from the packet schema version, the node ID,
  and the sanitized-content SHA-256, exactly as directive IDs are minted. The
  model references only IDs published in `allowed_source_ids`; ID-shaped
  strings inside node content remain data. The model never mints an ID.
- Each subject record carries: node ID, node type, tags, created/updated
  timestamps, sanitized bounded excerpt, and content SHA-256. The kindex node
  ID appears as metadata (`kindex:<id>` locator), giving every finding an exact
  node-ID citation that kindex can resolve.

### Analysis and local validation

- The Analyst receives node subjects alongside the existing directive set and
  evidence, in one packet. It may nominate only its existing seven classes;
  eligibility for candidate submission is restricted per the mapping above.
- Every existing local validation gate applies unchanged: unknown or unlisted
  source IDs reject the nomination; behavioral intent must share at least three
  meaningful terms with cited sources and evidence; `temporal_supersession`
  requires explicit reversal evidence; `contradiction` requires at least two
  cited sources/evidence records; invalid nominations are rejected and counted
  without discarding valid siblings; an all-rejected response is
  `semantic_analysis_inconclusive`, never a clean result.
- A `contradiction` over node subjects must cite at least two exact node IDs
  (or one node ID plus admitted external evidence). A `temporal_supersession`
  must cite the superseded node ID and the newer evidence or node that revises
  it. Citations resolve to exact enumerated node IDs; a finding citing a
  non-enumerated node fails validation locally.
- **Cross-scope findings are report-only.** A finding that spans a node subject
  and a configured instruction-file directive (cross-plane), or that spans
  scopes kindex cannot resolve to a single quarantine decision, appears in
  Meditate's report and is never submitted as a candidate. This extends the
  existing cross-target/cross-heading report-only rule.
- Packet inclusion of node subjects requires an Analyst prompt/parser version
  bump; older cached analyses fail closed under the existing content-addressed
  cache rather than being reinterpreted.

### Sanitization boundary

- Node content passes the same local secret detection and redaction pipeline as
  every other evidence source before entering any packet. A high-confidence
  secret match excludes the node wholesale with a warning naming only the node
  ID, never the content. A high-confidence match surviving redaction anywhere
  in the assembled packet blocks the packet before the first model call.
- This applies even though curated nodes are lower-risk than raw history
  (kindex constraint `5e48daf20c91` carries over verbatim). Candidate payloads
  (Phase 3) contain only sanitized text that already passed this boundary, plus
  IDs and hashes.

### Finding disposition: quarantine candidates

- Every locally validated, eligible finding lands in kindex's existing
  quarantine as a **pending candidate** — the same `candidate` flow whose
  review verbs are `list`/`show`/`accept`/`reject`/`prune`/`erase`.
- **Named prerequisite (kindex-side):** the kin CLI currently exposes only
  review verbs; there is no submission verb. Before Phase 3, kindex must add a
  bounded external-nomination surface (e.g. `kin candidate submit` accepting a
  structured JSON payload) that records provenance — nominating tool, Meditate
  run ID, nomination fingerprint, Analyst prompt/parser versions, cited node
  IDs — and lands the finding as an ordinary pending candidate. Meditate does
  not invent this surface, and Meditate's spec does not design kindex's schema;
  it names the dependency.
- Submission is idempotent per nomination fingerprint: resubmitting the same
  finding (same class, same cited node set, same evidence fingerprint) is a
  no-op, not a duplicate candidate.
- A submission failure is a visible fail-closed error, never a silent drop.
- Acceptance, rejection, ranking, and any resulting supersession or
  invalidation of a node happen **inside kindex, under existing human review**.
  Kindex's job is to store and rank the conflict; Meditate's job is to find it.

### Authority boundary

- The Analyst remains nomination-only: no drafting, no destination choice, no
  authority assignment, no self-answered collisions, no writes. The Drafter
  never receives node subjects; there is nothing to compile because Meditate
  cannot write kindex content.
- The only kindex-write surface reachable from Meditate code is candidate
  submission through the named prerequisite surface (Phase 3). Meditate never
  invokes `add`, `edit`, `supersede`, `invalidate`, `set-state`, `link`,
  `verify`, or any `candidate` review verb. This is testable: every kin
  invocation Meditate constructs is asserted against a read-only allowlist
  (plus `candidate submit` in Phase 3), with denial probes feeding the
  forbidden verbs end-to-end and asserting the run does not perform them.
- Standing-sweep cadence comes from Meditate's existing cron surface (dry-run
  by default). No new daemon, scheduler, or kindex-side hook is created.

## Phasing

**Phase 1 — read-only enumeration (this repo, this release cycle).** A
config-gated entry point enumerates additive nodes via the kin CLI and produces
sanitized, bounded, deterministic node records. No packet wiring, no model
call, no write path, no candidate submission. Fixture-tested.

**Phase 2 — packet projection and report-only findings.** Node subjects join
the Analyst packet behind the same flag; prompt/parser versions bump; local
validation extends to node citations; all node findings appear in plan reports
as report-only. A run with node findings is never `stable_noop`.

**Phase 3 — candidate submission.** Blocked on the kindex `candidate submit`
surface. Eligible findings (contradiction, temporal_supersession over node
subjects only) are submitted with provenance, idempotently, fail-closed.

## Acceptance criteria

Phase 1:

- With `analyze_nodes=false` (the default), no enumeration subprocess runs and
  every existing test remains green unchanged.
- With the flag on and `kin` absent, the result is empty plus
  `kindex_unavailable`; with `kin` present, exactly one `list` call per
  configured additive type (`--status active --json`) and one `show` per
  unique node ID, deduplicated across types, output sorted by node ID.
- Subprocess failure, invalid JSON, or a malformed node ID aborts with
  `kindex_required_failed`.
- A node whose content matches a high-confidence secret shape is excluded
  wholesale, counted, and named by ID only.
- The `max_nodes` ceiling truncates deterministically with an explicit warning.
- Enumeration constructs only `list` and `show` invocations; a test asserts the
  argv allowlist and a denial probe proves a non-allowlisted verb cannot be
  reached from this path.

Phase 2:

- Node subjects appear in the packet as immutable subjects with locally minted
  IDs in `allowed_source_ids`; a nomination citing a non-enumerated node ID is
  rejected locally; node subjects require and receive no disposition.
- A synthetic fixture graph with two contradicting decision nodes yields a
  validated `contradiction` finding citing both exact node IDs; a fixture with
  an older decision and newer explicit reversal evidence yields
  `temporal_supersession`; a well-formed fixture graph yields no node findings
  and byte-identical targets.
- A nomination pairing a node subject with an instruction-file directive is
  classified cross-scope and never enters any candidate set (report-only), and
  no node finding enters the Drafter packet.

Phase 3:

- Round-trip on a fixture: a validated finding produces exactly one pending
  kindex candidate with correct class, cited node IDs, and provenance; exact
  re-run produces zero new candidates (idempotent); submission failure is a
  visible fail-closed error.
- Denial probes: an ineligible-class finding, a cross-scope finding, and a
  finding citing an unknown node ID are each fed end-to-end and each fails to
  produce a candidate; no kin write verb outside `candidate submit` is ever
  constructed.

## Out of scope

- **No auto-accept.** Meditate never invokes `candidate accept`/`reject`/
  `prune`/`erase`, on its own findings or anyone else's.
- **No direct node mutation.** No `add`, `edit`, `supersede`, `invalidate`,
  `set-state`, `link`, or `verify` invocation exists in Meditate, in any phase.
- **No new demotion authority besides this path.** Node trust state changes
  only through kindex's existing quarantine review after a human accepts a
  candidate. Meditate findings have no effect on recall until then.
- No new Analyst candidate classes; no relaxation of existing validation gates.
- No R0 referent hashing, `true_of` clocks, or digest-staleness computation in
  Meditate (kindex-side per the PRD).
- No drafting, rewriting, or compilation of kindex node content; no writes to
  `.kin/` exports; no new daemon or scheduler.
- No analysis of non-additive node types (concept, watch, checkpoint, task,
  question, skill) in v1.

## Open questions (founder decision)

1. Shape of the kindex candidate-submission surface: CLI verb and payload
   schema, and which provenance fields kindex records as first-class columns
   versus opaque payload. Kindex-side design; blocks Phase 3 only.
2. Whether the node target class should later widen to `watch`/`checkpoint`
   nodes, which are additive but operational.
3. Whether node analysis shares one packet with instruction-file analysis
   (proposed here, because cross-plane contradictions are findable and safely
   report-only) or runs as a separate invocation mode with its own budget.
4. Default `max_nodes` (proposed 200) and its interaction with the existing
   token budgets on graphs with large additive-node populations.
