# Structured Reading Skill — Implementation Plan v0.1

## Status

- Architecture: complete for implementation review.
- Implementation: not started.
- Product source: [human requirements v0.2](../human/requirements-v0.2.md).
- Technical source: [architecture v0.1](architecture-v0.1.md).

The milestones below are vertical slices with observable exit conditions. A milestone is not complete merely because files exist.

## Milestone 1 — Skill and runtime skeleton

Deliver:

- root `SKILL.md` with discriminating trigger boundaries;
- `agents/openai.yaml` with implicit invocation enabled;
- progressive-disclosure reference files;
- Node runtime and exact package lock;
- CLI entrypoint and safe path utilities;
- unit-test and browser-test commands.

Exit conditions:

- the official Skill validator passes;
- a clean install from the lockfile succeeds;
- tests run without undeclared host-global packages;
- Skill instructions route each stage to only its relevant reference.

## Milestone 2 — Evidence and page reconstruction substrate

Deliver:

- canonical schemas for run, evidence, and logical pages;
- recursive raster discovery;
- byte-preserving evidence copies and SHA-256;
- normalized inspection previews;
- exact duplicate detection;
- safe new-bundle and explicit-resume behavior;
- page-reconstruction assignment/shard/decision flow;
- deterministic `apply-pages` and validation.

Exit conditions:

- duplicate filenames in different subfolders remain distinct;
- duplicate bytes are detected without losing either occurrence;
- all stable IDs match fixed cross-platform golden vectors for path normalization, region keys, and hash domains;
- a spread produces two stable logical pages linked to one evidence item;
- file order is never silently promoted to reading order;
- Evidence hashes fail validation after any copied-byte modification;
- input files remain byte-for-byte and name-for-name unchanged.
- moving the input root without changing relative paths preserves Evidence IDs, while an internal rename creates a new occurrence with the same content identity.

## Milestone 3 — Source layer

Deliver:

- Raw OCR, Reading Text, and correction-log schemas;
- OCR worker assignment and shard contracts;
- blind-review contract for low-confidence regions;
- immutable raw attempts;
- append-only correction validation;
- block types, formatting ranges, uncertainty ranges, and page boundaries;
- exact cross-page continuity checks;
- post-delivery correction revision through Codex and `begin-revision`;
- `apply-ocr` and source coverage reporting.

Exit conditions:

- a changed OCR reading cannot erase the first attempt;
- a Reading Text difference without a correction event fails validation;
- `candidate`, `illegible`, and `outside_photo` remain distinct;
- meaningful layout survives while photographic line wraps are removed;
- every non-duplicate in-scope page has text or an explicit illegibility marker;
- no image is required to read the normal Reading Text output.
- Unicode code-point offsets round-trip correctly through Node validation and browser DOM selection for BMP, supplementary, and combining characters.

## Milestone 4 — Semantic structure and orchestration

Deliver:

- Semantic Span and Knowledge Graph schemas;
- required role, relation, epistemic, confidence, and speaker enums;
- separate document and semantic hierarchies;
- concept sense/evolution model;
- argument step model;
- structural worker and global-agent shard contracts;
- conflict-resolution decision format;
- run ledger with model and effort;
- read-only Integrity Auditor protocol with mechanical and semantic passes;
- audit finding schema, severity, rework routing, and closure ledger;
- canonical merge and cross-layer validation.

Exit conditions:

- every AI node and relation resolves to exact Reading Text ranges;
- a stale range fails after the Reading Text revision changes;
- unsupported AI nodes and broken graph references fail validation;
- no two workers can merge ownership of the same canonical range;
- a complete run records at least two distinct model IDs;
- a one-model or no-subagent environment stops before ingest;
- partial input cannot be labeled as whole-document analysis.
- Luna roles use only their specified `xhigh`/`max` effort and Sol roles use only their specified `medium`/`xhigh` effort;
- an audit finding cannot disappear or change severity without a recorded Root decision;
- open blocker or major findings prevent `complete`.
- only a fresh verification auditor plus the required Sol triage/adjudication record can produce `verified_closed`;
- fresh/blind agents have an empty inherited-turn set and a ledger of exactly which artifacts they received;
- OCR and Structural assignments obey deterministic size, context, and ownership limits.

## Milestone 5 — Reading Atlas

Deliver:

- single-file offline HTML build;
- Atlas, Position, Logic, Concept, and Text views;
- shared selection state and bidirectional navigation;
- Semantic Zoom and Focus Mode;
- Semantic Lens controls;
- Argument Playback;
- browser-local working-memory shelf;
- explicit Audit Mode;
- applicable adaptive visualizations;
- deterministic static SVG exports;
- accessible labels, keyboard operation, legends, and non-color encodings.

Exit conditions:

- the principal artifact opens directly from disk with network disabled;
- normal mode contains no visible or loaded evidence image;
- selecting a graph node reaches exact text and selecting text reveals graph nodes;
- Focus Mode does not render the entire graph;
- the five required views work at narrow and desktop viewport sizes;
- adaptive views appear only for supported source relations;
- generated views can be deleted and rebuilt without changing canonical data.
- adaptive-view eligibility is tested separately for every minimum Graph pattern and for inferred-only negative cases;
- the pinned Playwright Chromium revision passes the full local-file suite.

## Milestone 6 — Integrity, acceptance, and forward evaluation

Deliver:

- complete integrity metric aggregation;
- HTML integrity report;
- full-page fidelity, high-risk OCR, boundary, grounding, speaker/status, structure, scope, conflict, and reading-experience audit evidence;
- unit, contract, corruption, interruption, and browser tests;
- a small synthetic photographed-page fixture with known expected structure;
- an independent end-to-end Skill evaluation in an isolated temporary directory;
- a requirements-to-test evidence record.

Exit conditions:

- all AC-01 through AC-18 have an automated test or a documented behavioral evaluation;
- hash, ID, range, reference, correction, and stale-view corruption are detected;
- interruption and resume preserve completed canonical work;
- the evaluator can use the Skill from a realistic user request without hidden setup knowledge;
- a partial run is never reported complete;
- no generated artifact requires the source images in normal reading mode.
- every in-scope logical page and every Graph node/relation appears in the semantic audit coverage ledger;
- affected audit passes rerun after correction and preserve the previous finding history.

## Milestone 7 — Deployment replacement

Deliver only after implementation acceptance:

- install or link the validated repository Skill into the official local Skill location;
- verify discovery and one realistic invocation from a fresh Codex session;
- remove the obsolete installed prototype after confirming the new Skill is active.

Exit conditions:

- the installed Skill resolves to repository-controlled content;
- there is only one active `structured-reading` Skill identity;
- the obsolete Markdown-only contract and missing-template path are gone;
- repository tests still pass after deployment.

This milestone changes user-level installed files and is intentionally separate from repository implementation.

## Acceptance-criteria traceability

| Requirement | Primary implementation evidence |
|---|---|
| AC-01 Evidence Preservation | Milestone 2 hash, copy, and no-input-mutation tests |
| AC-02 Text-first UX | Milestones 3 and 5 browser assertions |
| AC-03 Page Integrity | Milestone 2 reconstruction and uncertainty tests |
| AC-04 Layer Separation | Canonical schemas and forbidden cross-layer fields |
| AC-05 Full-text Coverage | Source coverage and explicit illegibility tests |
| AC-06 Correction Trace | Append-only correction-chain validation |
| AC-07 Traceability | Graph-to-span-to-text-to-OCR-to-evidence resolver tests |
| AC-08 Structural Coverage | Knowledge Graph schema and representative fixture |
| AC-09 Relation Visibility | Logic/Concept view browser tests |
| AC-10 Graph Canonicality | Delete-and-rebuild projection test |
| AC-11 Multi-resolution Visualization | Five-view and Semantic Zoom browser tests |
| AC-12 Bidirectional Navigation | Shared-selection browser tests |
| AC-13 Adaptive Visualization | Relation-pattern eligibility tests |
| AC-14 Uncertainty | Schema, line-style, label, and lens tests |
| AC-15 Multi-agent Integrity | Assignment ownership and run-ledger validation |
| AC-16 Mechanical Validation | Corruption matrix tests |
| AC-17 Scope Honesty | Partial-input behavioral evaluation |
| AC-18 Offline Availability | Network-disabled local-file browser test |

## Test strategy

### Unit tests

Use Node's built-in test runner for pure functions: IDs, paths, hashing, ordering, offsets, correction chains, graph references, report counts, layout selection, and safe HTML data escaping.

### Contract tests

Run the CLI against deterministic fixtures and compare semantic invariants, not generated prose or incidental formatting. Tests must assert schema-valid canonical files and exact failure codes for corrupted bundles.

### Browser tests

Use pinned Playwright with network requests denied. Open the generated `file://` artifact and exercise navigation, lenses, focus, playback, shelf persistence, Audit Mode, keyboard controls, and responsive layouts.

### Behavioral Skill evaluation

After the implementation is complete, run the Skill in an isolated temporary workspace on a small representative photo set. The evaluator receives the realistic user request and Skill but not the intended result or suspected failures. Review the generated canonical data, report, and UI before changing instructions.

## Corruption matrix

At minimum, automated tests deliberately introduce:

- changed Evidence bytes;
- duplicate stable IDs;
- duplicate reading orders;
- a page pointing to an unknown Evidence ID;
- an invalid confidence value;
- a Reading Text edit without correction history;
- a source range beyond block length;
- a Semantic Span on an old source revision;
- a Graph node without Source Spans;
- an edge pointing to a missing node;
- an overlapping worker ownership assignment;
- an unrecorded agent model;
- an effort that violates the role's Luna/Sol profile;
- an open blocker or major audit finding in a purported complete run;
- an audit finding removed or downgraded without a decision record;
- UTF-16 indices incorrectly submitted as canonical Unicode code-point offsets;
- a blind-review assignment containing a prior candidate or inherited conversation turn;
- a batch over its stage cap or with more than one neighboring context page;
- a generated view with a stale canonical hash;
- a pending page in a purported complete run.

Every case must produce a stable machine-readable error and a nonzero strict-validation exit code.

## Implementation rules

- Do not edit the installed prototype while repository implementation is in progress.
- Do not add compatibility output for its Markdown bundle.
- Do not allow workers to write canonical files.
- Do not weaken validation to make a fixture pass; correct the model, merger, or fixture.
- Do not add an external OCR or online fallback without a new product decision.
- Do not implement Perspective Lens Chat in MVP code paths.
- Keep human documentation in Japanese under `docs/human/` and agent documentation in English under `docs/agent/`.
