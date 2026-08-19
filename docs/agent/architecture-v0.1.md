# Structured Reading Skill — Technical Architecture v0.1

## Status and authority

- Status: implementation-ready design; code has not been started.
- Product baseline: [human requirements v0.2](../human/requirements-v0.2.md).
- Scope: MVP v0.1 only. Perspective Lens Chat remains outside this design.
- Authority rule: the human requirements define product behavior. This document defines the technical realization and must be changed when an implementation decision changes.

This design intentionally replaces the previously installed, Markdown-oriented prototype. The repository is the only implementation source of truth. An installed copy is a deployment artifact, never an independently edited source.

## 1. Architectural decisions

### 1.1 The repository root is the skill root

The repository will be a standalone Codex Skill, with `SKILL.md` at its root. This keeps authoring, validation, tests, assets, and the installed Skill identical. It also avoids a second nested copy that could drift.

The project will initially remain a standalone Skill rather than a plugin. A plugin can be introduced later if public distribution or bundled connectors become a requirement.

### 1.2 Codex orchestrates AI work; repository scripts own deterministic work

The Skill will not call the OpenAI API directly and will not require a second API key. Codex supplies image understanding, semantic judgment, and subagent orchestration. Repository scripts perform operations that must be deterministic:

- image discovery, hashing, evidence copying, and preview creation;
- stable ID generation;
- schema and reference validation;
- canonical merge operations;
- report aggregation;
- derived projection and Reading Atlas generation.

This boundary avoids hiding model calls inside an opaque program while ensuring that mechanical integrity does not depend on free-form model output.

### 1.3 Node.js is the sole project runtime

The implementation will target Node.js 22, matching the current host and allowing one language for the CLI, validation, rendering, and browser code.

- The exact runtime is declared in `.nvmrc` and `package.json#engines`.
- Every runtime and development package is pinned exactly in `package.json` and `package-lock.json`.
- Runtime packages are limited to image normalization and JSON Schema validation.
- The generated Reading Atlas has no runtime package, network, CDN, font, or server dependency.
- Browser behavior tests use a pinned Playwright development dependency.

The implementation phase must check current official package documentation before selecting exact versions. No package is assumed to exist globally.

### 1.4 Canonical data is separate from work products and views

Canonical layers are immutable or append-only wherever practical:

1. Evidence inventory and copied image bytes.
2. Raw OCR attempts.
3. Reading Text plus append-only correction events.
4. Semantic Spans.
5. Knowledge Graph.

Everything under `views/` is derived and can be deleted and regenerated. Agent shards under `work/` are proposals, not canonical data.

### 1.5 The browser artifact is a single offline HTML entrypoint

`views/atlas.html` embeds its styles, scripts, and validated data. This avoids `file://` fetch restrictions and allows the principal reader to work by opening one local file. Static SVG exports are generated beside it.

Normal reading mode does not create image or thumbnail elements. Audit Mode creates evidence image elements only after an explicit user action.

### 1.6 Explicit MVP decisions and runtime assumptions

The following are design decisions, not hidden implementation assumptions:

- **OCR implementation:** GPT-5.6 Luna reads the page images directly. There is no Tesseract, cloud OCR service, or silent OCR fallback in MVP. The Raw OCR engine identity is the exact model ID, effort, agent ID, prompt/reference revision, and run timestamp.
- **AI execution host:** the Skill runs in a Codex environment that provides local image inspection, subagent creation, explicit child-model selection, and explicit reasoning-effort selection. Missing capability is a preflight failure.
- **Required models:** both `gpt-5.6-luna` and `gpt-5.6-sol` must be selectable. Neither Terra nor a same-model effort variation substitutes for the two-model requirement.
- **Semantic canonicality:** Semantic Spans are a canonical traceability index over a frozen Reading Text revision. The Knowledge Graph is the sole canonical semantic interpretation. Concept, role, argument, relation, and visualization files are projections from the Graph.
- **MVP presentation:** single-file offline HTML and static SVG are the selected MVP implementation, not a permanent product-level technology constraint.
- **Editing and correction:** the browser is read-only with respect to canonical data. In MVP, a user requests a correction through Codex and identifies the page/block/span or quoted text. The Root creates a revision request, runs the normal OCR/review/merge path for the affected range, appends correction provenance, invalidates downstream hashes, and rebuilds affected structure and views. There is no browser edit form or public API in MVP.
- **Output language:** source text remains in its source language. UI labels, structural annotations, and integrity reports default to Japanese unless the user explicitly requests another language.
- **Document scope:** preflight records `complete_document`, `excerpt`, or `unknown`. When the user has not established completeness, the default is `unknown`; the system describes only the observed input and never infers whole-book completeness.
- **Legibility:** every input page is processed even when unreadable. An unreadable page receives an explicit marker and issue record; it is not omitted. `processing_status=complete` means every in-scope page and required audit was processed; it does not mean every character was legible. Legibility is reported separately as `fully_legible`, `legibility_issues`, or `unreadable`.
- **Concurrency:** the Root uses at most `min(host child-thread capacity, ready independent assignments)` child threads and keeps canonical writes serialized. OCR ownership targets eight logical pages and never exceeds twelve; Structural Analysis targets sixteen logical pages and never exceeds twenty-four. A visible section boundary may shorten a batch. Each worker may read exactly one preceding and one following logical page as non-owned context. Ready batches beyond the thread limit queue in reading order.
- **Storage:** preflight estimates exact evidence-copy bytes plus preview/work overhead and verifies that the destination is writable. If available capacity cannot be established or is insufficient, it stops before copying.
- **Local viewing:** generation does not require a browser. The normative browser target is the Chromium revision installed by the pinned Playwright lockfile, using `file://`, JavaScript, CSS Grid, inline SVG, and local storage. Equivalent or newer Chromium-based Chrome/Edge releases are supported; Firefox and Safari are not MVP acceptance targets. Browser incompatibility does not authorize an online fallback.
- **Independent context:** `fresh` and `blind` mean a newly spawned subagent thread with no inherited conversation turns, a unique agent ID, and only the assignment, required schema/reference instructions, Evidence region, and allowed canonical context. It receives no previous candidate, worker reasoning, suspected-error list, or unneeded shards. The run ledger records the empty-fork policy and supplied artifact IDs. If the host cannot provide this isolation, blind review and independent audit cannot be claimed.
- **Evidence identity across runs:** `content_sha256` identifies identical bytes. `evidence_id` identifies one input occurrence using the exact ID algorithm in Section 8. Moving the whole input root while preserving normalized relative paths preserves IDs; renaming or moving a file within the root creates a new occurrence ID while retaining the same content identity. Resume uses the copied Evidence and stored IDs and never silently rebinds a changed source tree.

## 2. System context

```text
User page-photo folder
        |
        v
Deterministic inventory and evidence copy
        |
        v
Codex Root Orchestrator
  |-- Page Reconstruction Agent
  |-- OCR Workers
  |-- Independent OCR Reviewers
  |-- Structural Analysis Workers
  |-- Global Structure Agent
  `-- Integrity Auditor (read-only)
        |
        v
Validated canonical bundle
        |
        +-- Integrity report
        +-- Single-file offline Reading Atlas
        `-- Static SVG projections
```

Codex subagents are the execution substrate, not a hidden application service. The Skill must verify that subagent creation, image inspection, explicit model selection, and at least two distinct suitable models are available before processing starts.

## Defined terms

- **Reading Atlas:** the generated family of synchronized views over full Reading Text and the Knowledge Graph; it is not a summary.
- **Evidence:** the exact copied bytes and metadata of each input image occurrence.
- **Raw OCR:** an immutable model reading attempt before correction.
- **Reading Text:** the source-faithful, human-readable transcription selected from Raw OCR plus explicit correction events.
- **Source:** the complete evidence-backed chain for a claim: an exact Reading Text range in a named Source revision, its Raw OCR provenance, and its Evidence image/region. A **Source Reference** is the Reading Text revision, page ID, block ID, and code-point range that resolves through that chain.
- **Semantic Span:** one semantic unit linked to exact character ranges in a frozen Reading Text revision.
- **Knowledge Graph:** the canonical semantic nodes, relations, hierarchies, concepts, and arguments grounded in Semantic Spans.
- **Canonical/正本:** validated data that later stages depend on and that no worker may overwrite directly.
- **Shard:** one agent's schema-constrained proposal in its uniquely owned work path; a shard is not canonical until the Root accepts and applies it.
- **Stable ID:** an identifier derived from stable source identity rather than display order or a printed page label.
- **Source revision:** the immutable identity of one Reading Text state and its upstream OCR/correction chain.
- **Stale:** derived data whose recorded upstream hash no longer matches the current canonical revision.
- **Epistemic status:** `explicit`, `strongly_inferred`, `inferred`, or `uncertain`, describing how directly the Source supports an interpretation.
- **Confidence:** a numeric value in `[0,1]` for uncertainty within a stated decision; it never converts unsupported content into supported content.
- **Speaker attribution:** whether a statement belongs to the author, a quoted person, an editor, or AI interpretation.
- **Blind review:** an independent re-reading that receives the evidence and task but not the earlier proposed answer.
- **Logical page:** one ordered reading unit reconstructed from all or part of an Evidence image; a spread normally creates two logical pages.
- **Page kind:** a classified layout role such as cover, contents, body, notes, bibliography, index, or blank.
- **Adaptive view:** a visualization emitted only when the validated Graph contains the relation pattern required by that view.
- **Suitable model:** an explicitly available Luna or Sol configuration that supports the required image/tool inputs and the effort assigned in Section 6.
- **Global synthesis:** reasoning across multiple sections or the whole observed input, including concept identity, long-range references, and argument integration.

## 3. Planned repository layout

```text
book-skill/
|-- README.md
|-- SKILL.md
|-- agents/
|   `-- openai.yaml
|-- references/
|   |-- orchestration.md
|   |-- page-reconstruction.md
|   |-- transcription.md
|   |-- structural-analysis.md
|   `-- output-contract.md
|-- scripts/
|   `-- book-atlas.mjs
|-- src/
|   |-- cli/
|   |-- evidence/
|   |-- ids/
|   |-- merge/
|   |-- schema/
|   |-- validation/
|   |-- projections/
|   `-- rendering/
|-- schemas/
|   |-- common.schema.json
|   |-- run-manifest.schema.json
|   |-- evidence-inventory.schema.json
|   |-- page-index.schema.json
|   |-- raw-ocr.schema.json
|   |-- reading-text.schema.json
|   |-- correction-log.schema.json
|   |-- semantic-spans.schema.json
|   |-- knowledge-graph.schema.json
|   `-- agent-shard.schema.json
|-- assets/
|   `-- reader/
|       |-- atlas-template.html
|       |-- atlas.css
|       `-- atlas.js
|-- test/
|   |-- unit/
|   |-- contract/
|   |-- browser/
|   `-- fixtures/
|-- docs/
|   |-- human/
|   `-- agent/
|-- package.json
|-- package-lock.json
`-- .nvmrc
```

`SKILL.md` contains only shared invariants, routing, preflight, the stage sequence, and completion rules. Detailed prompts and data contracts are loaded from `references/` only when their stage is active.

## 4. Generated book bundle

Each run creates a new bundle or explicitly resumes an existing bundle. Initialization never silently overwrites a non-empty destination.

```text
book/
|-- book.json
|-- evidence/
|   |-- images/
|   |-- previews/
|   |-- inventory.json
|   `-- page-index.json
|-- source/
|   |-- raw-ocr/
|   |-- reading-text/
|   `-- correction-history/
|-- structure/
|   |-- semantic-spans.json
|   `-- knowledge-graph.json
|-- work/
|   `-- runs/<run-id>/
|       |-- run-manifest.json
|       |-- assignments/
|       |-- shards/
|       `-- decisions/
|-- views/
|   |-- atlas.html
|   `-- static/
|       |-- atlas.svg
|       |-- position.svg
|       |-- logic.svg
|       `-- concepts.svg
`-- report/
    |-- integrity.json
    `-- integrity.html
```

Evidence previews are derived convenience files for agent inspection. They never replace the exact copied evidence bytes and are not shown in normal reading mode.

## 5. End-to-end execution protocol

### Stage 0 — Preflight

Before any output bundle is created, the Root Orchestrator verifies:

- the input folder exists and is readable;
- subagents and local image inspection are available;
- explicit child model and reasoning-effort selection are available;
- at least two distinct suitable model IDs are available;
- the destination is new, or the user explicitly requested resume;
- Node and repository packages match the lockfile.

If multi-agent or model diversity requirements fail, processing stops before ingest. The Skill must report the missing capability and must not silently substitute a single agent or one-model topology.

### Stage 1 — Evidence ingest

The CLI recursively discovers supported raster images, calculates SHA-256, copies the exact bytes, records metadata, creates normalized inspection previews, and identifies exact duplicates. Inputs are never renamed, modified, moved, or deleted.

The preliminary inventory order is only an ingest order. It is not treated as reading order.

### Stage 2 — Page reconstruction

One Page Reconstruction Agent inspects every input image and proposes:

- orientation;
- full page versus spread;
- normalized page regions;
- visible printed page labels;
- page kind;
- likely reading order;
- exact and content duplicate candidates;
- blur, crop, occlusion, and missing-neighbor warnings;
- confidence and evidence for every uncertain decision.

The proposal is written only to that agent's shard. The Root Orchestrator resolves ambiguity into a decision file and is the only agent permitted to apply the canonical `page-index.json` update.

Page Reconstruction is one Luna `xhigh` logical agent, not parallel owners. It inspects at most twenty Evidence images per persisted pass, with at most two provisional-ingest neighbors on each side for continuity, then performs one global ordering pass over all proposed page labels, regions, hashes, content cues, and confidences. The persisted passes are checkpoints of one agent role, not separately mergeable canonical shards.

### Stage 3 — Raw OCR and Reading Text

Logical pages are divided into contiguous ownership ranges. Each OCR Worker receives its owned pages plus read-only neighboring pages for continuity. A worker writes only its own shard.

The deterministic assignment builder creates batches using the Section 1.6 limits after page reconstruction. It never splits a logical page, never gives ownership overlap, records the one-page read-only context on each side when present, and queues excess batches rather than enlarging them because concurrency is low.

For every page, the worker emits:

- an immutable first OCR attempt;
- layout blocks and confidence;
- a source-faithful Reading Text candidate;
- uncertainty ranges and candidates;
- proposed corrections that explicitly link before and after text.

Regions with confidence below `0.85`, any explicit uncertainty marker, every correction, suspicious negation/numeral/proper noun/comparison/logical operator, and every cross-page discontinuity are sent to a fresh blind reviewer using Luna at `max`. The reviewer receives the evidence region and task question, but not the primary worker's proposed reading. Sol is used only when independent Luna readings remain materially inconsistent or the ambiguity changes the document's logic. The Root records the selected result and correction reason.

The workflow continues with explicit uncertainty markers when no reading can be justified. It never requires the user to inspect an image in order to obtain the text and structural output.

Raw OCR is never rewritten. Reading Text is frozen for the structural-analysis revision. A later Reading Text change creates a new source revision and marks all dependent Semantic Spans and Graph views stale until rebuilt.

### Stage 4 — Local structural analysis

Structural workers own non-overlapping Reading Text ranges and may read adjacent ranges. They propose Semantic Spans, role nodes, local concepts, local relations, document hierarchy observations, semantic hierarchy observations, and local argument fragments.

The deterministic assignment builder prefers visible section boundaries, targets sixteen logical pages, caps a batch at twenty-four, and provides exactly one adjacent logical page on each available side as read-only continuity context. A section longer than twenty-four pages is split at the nearest paragraph boundary before the cap.

Workers must not infer whole-document conclusions from a fragment. Every proposed AI node and relation includes source span IDs, epistemic status, confidence, and speaker attribution.

### Stage 5 — Global structure

The Global Structure Agent reads validated local shards and resolves:

- concept identity and meaning changes;
- long-distance references;
- cross-section arguments;
- global document and semantic hierarchies;
- adaptive-view candidates;
- scope limitations for partial inputs.

It produces a proposal, not a canonical write. The Root Orchestrator resolves conflicts and applies the canonical Knowledge Graph merge through the CLI.

Global Structure is one Sol `xhigh` logical agent. It may process one document section at a time into persisted private notes, but it must finish with a single pass over every section-level result and all boundary/long-range candidates before proposing the global shard. Section notes are not independently mergeable Graph shards.

### Stage 6 — Independent integrity audit

Audit has two distinct parts. Passing one never implies passing the other.

#### 6A. Mechanical audit

The deterministic CLI checks every record without model judgment:

1. Evidence bytes still match their SHA-256 hashes.
2. Schemas, stable IDs, page orders, ownership ranges, and revision hashes are valid and unique.
3. Every in-scope logical page has Raw OCR and Reading Text or an explicit unreadable marker.
4. Every Reading Text change has a continuous correction chain.
5. Every Semantic Span resolves to valid character offsets in the current Reading Text revision.
6. Every Graph node, relation, hierarchy link, concept sense, and argument step resolves through a Span to Source and Evidence.
7. No required model, effort, assignment, conflict decision, or agent result is absent from the run ledger.
8. Derived views match the current canonical hash and normal reader mode does not preload Evidence images.

Mechanical failure blocks semantic audit promotion and rendering, but the report still records all discoverable failures.

#### 6B. Semantic integrity audit

A fresh Luna `max` Integrity Auditor is read-only and receives canonical data, Evidence, assignments, and unresolved-decision records, but not the workers' reasoning or the Root's suspected-error list. It performs these concrete passes:

1. **Page fidelity:** re-inspect every logical page against its Reading Text, with exact attention to headings, paragraph boundaries, omissions, emphasis, footnotes, tables, figures, formulas, poetry/code line breaks, and explicit uncertainty markers.
2. **High-risk OCR:** re-read every correction, low-confidence range, illegible marker, negation, numeral, proper noun, comparison term, and logical operator. Confirm that uncertain text was not silently promoted to certain text.
3. **Boundary continuity:** compare every adjacent page boundary and every worker-range boundary for missing clauses, duplicated text, detached footnotes, and false section breaks.
4. **Source grounding:** inspect every AI node and relation against its Source Spans; reject unsupported paraphrases, reversed edge direction, invented connections, and claims stronger than the cited text.
5. **Speaker and epistemic status:** verify author/quoted-person/editor/AI attribution and whether `explicit`, `strongly_inferred`, `inferred`, or `uncertain` matches the evidence.
6. **Structural coherence:** check role classification, separate document/semantic hierarchies, concept sense identity and evolution, argument step order, objection/response pairing, and long-distance references.
7. **Scope honesty:** verify that excerpt or unknown input is not described as a complete book and that unresolved missing-page candidates remain visible.
8. **Conflict closure:** verify that every worker disagreement and blind-review result has a recorded decision, rationale, deciding agent, model, and effort.
9. **Reading experience:** verify that each required view exposes the intended Source links, uncertainty, confidence, and focus information without substituting a summary for full text.

Semantic audit is one Luna `max` logical agent. It persists page-fidelity/high-risk-OCR findings in the same page batches used by OCR, then performs one global pass over all boundary, Graph, scope, conflict, and reading-experience findings. Persisted audit batches are checkpoints, not separate auditors, and do not independently close findings.

The Auditor proposes findings and severity. A fresh Sol `medium` Audit Finding Triage agent checks each finding against the fixed rubric and establishes its initial severity and owning rework stage. The Root records that triage decision but does not unilaterally weaken it. A later severity change requires a new Sol `medium` triage record; if the change depends on whole-document meaning, Sol `xhigh` adjudicates it. Root may escalate any finding but may not downgrade or close one without the required independent record.

The finding ledger stores: finding ID, proposed severity, triaged severity, audit pass, affected layer and IDs, Source/Evidence references, observed problem, expected invariant, confidence, owning rework stage, and status. Status is one of `open`, `in_rework`, `corrected_pending_verification`, `verified_closed`, or `accepted_minor`. Only a triaged `minor` finding may become `accepted_minor` without a content change.

Severity and completion rules are fixed:

- `blocker`: missing/fabricated Source, Evidence mismatch, broken traceability, or a false complete status;
- `major`: meaning-changing OCR, unsupported author attribution, materially wrong argument/concept/relation, or hidden uncertainty;
- `minor`: a localized presentation or labeling issue that does not change Source or semantic meaning.

A run cannot be `complete` with an open `blocker` or `major`. Open `minor` findings remain visible in the final report. Severity may change only through a recorded Root decision; deleting or downgrading a finding to obtain completion is invalid.

Closure is equally constrained. After canonical rework, the CLI sets affected findings to `corrected_pending_verification` and reruns the affected mechanical checks. A newly spawned fresh Luna `max` verification auditor reruns the named semantic audit pass without receiving the previous proposed fix or reasoning. If both checks pass, a fresh Sol `medium` Triage agent may issue a `verified_closed` decision; a whole-document/cross-section finding instead requires Sol `xhigh`. Root only records the signed decision. Failed verification returns the same finding to `open` with a new attempt record. Finding history is append-only.

Rework routes are explicit: page-order findings return to Page Reconstruction Luna `xhigh`; OCR findings return to OCR Luna `max`; local-structure findings return to Structural Luna `xhigh`; bounded semantic disputes go to Sol `medium`; whole-document or cross-section disputes go to Sol `xhigh`. After rework, mechanical validation and the affected semantic audit passes run again. The auditor never edits canonical files.

### Stage 7 — Final validation and rendering

After all rework, the CLI reruns the complete Stage 6A mechanical audit and checks the Stage 6B finding ledger. Rendering is refused when validation errors or open blocker/major findings remain. A bundle with pending or unprocessed pages is labeled `partial`, never `complete`.

On success, the CLI generates projections, `atlas.html`, SVG exports, and the final integrity report. Derived files record the canonical content hash from which they were built.

## 6. Model and reasoning allocation

The current default profile uses Luna as the primary workhorse and Sol for bounded or global adjudication. Luna uses only `xhigh` and `max`; Sol uses only `medium` and `xhigh`. Terra is not part of the default topology. Exact model IDs and supported efforts are checked during preflight because availability can change.

| Role | Current preferred model | Effort | Reason |
|---|---|---:|---|
| Root Orchestrator | Prefer `gpt-5.6-sol`; otherwise delegate semantic decisions to the listed Sol agents | medium | Scheduling, assignment, ledger updates, and validated merge coordination are bounded orchestration tasks |
| Page Reconstruction | `gpt-5.6-luna` | xhigh | Visual inventory and page ordering require careful analysis but do not require exhaustive character-level transcription |
| Primary OCR Worker | `gpt-5.6-luna` | max | Luna supports image input; `max` is the default because transcription errors propagate into every downstream layer |
| Blind OCR Reviewer | fresh `gpt-5.6-luna` | max | A context-independent second reading avoids anchoring while keeping page-scale work on Luna |
| OCR Adjudicator | `gpt-5.6-sol` | medium | Used only for bounded Luna/Luna disagreement about a specific high-impact reading |
| Structural Analysis Worker | `gpt-5.6-luna` | xhigh | Local span, role, concept, relation, and argument extraction is bounded to an owned range |
| Global Structure Agent | `gpt-5.6-sol` | xhigh | Whole-document argument synthesis, concept identity, and long-range dependency resolution require deep global judgment |
| Integrity Auditor | fresh `gpt-5.6-luna` | max | Full page/source reinspection and all semantic audit passes prioritize exhaustive coverage |
| Audit Finding Triage | `gpt-5.6-sol` | medium | Classifies a bounded finding and chooses the owning rework stage without rebuilding global meaning |
| Complex Audit Adjudicator | `gpt-5.6-sol` | xhigh | Resolves findings that change whole-document arguments, cross-section concepts, or scope claims |

Mechanical work does not use a model. Luna `xhigh` is for bounded visual/structural analysis; Luna `max` is for exact transcription and exhaustive audit. Sol `medium` is for a bounded, well-evidenced decision; Sol `xhigh` is for cross-section or whole-document synthesis. Production runs do not use another effort for these roles without a reviewed design change.

Runtime rules:

1. Record exact model ID and effort for every agent in `run-manifest.json`.
2. Require both Luna and Sol model IDs in a completed run; separate Luna agents alone do not satisfy model diversity.
3. Use no more child threads than the host allows, and reserve the main thread for orchestration.
4. Prefer contiguous page batches; never have two workers own the same canonical range.
5. If Luna or Sol is unavailable, stop and ask the user before changing the topology. Do not introduce Terra implicitly, and do not treat a different effort on the same model as model diversity.

## 7. Canonical data contracts

All canonical JSON files include `schema_version`, `book_id`, `run_id`, and an ISO-8601 creation timestamp. Schemas reject unknown fields in stable records so misspellings do not silently become data.

### 7.1 Evidence inventory

Each input occurrence has a stable evidence ID, normalized input-root-relative path, original display path, copied relative path, exact byte hash, byte length, media type, dimensions when readable, ingest order, and optional exact-duplicate reference.

Evidence IDs follow the exact Section 8 algorithm. Duplicate occurrences therefore remain individually addressable while sharing the same `content_sha256`.

### 7.2 Logical page index

A logical page references one evidence ID and a normalized rectangular region. It records reading order separately from printed page label, plus orientation, page kind, status, quality flags, ordering confidence, page-number confidence, duplicate reference, and decision evidence.

One evidence image may map to two logical pages. Reordering pages changes only `reading_order`; it does not change page IDs.

### 7.3 Raw OCR

A raw OCR record is append-only and contains:

- attempt ID, page ID, region, model/agent identity, and timestamp;
- exact model ID, reasoning effort, and run/instruction revision;
- original recognized text;
- ordered layout blocks;
- block and attempt confidence;
- detected layout roles such as heading, paragraph, footnote, caption, figure, table, code, formula, or poetry;
- legible figure/table labels and a source-grounded non-text description when applicable.

The first attempt is always retained even when a later attempt is better.

### 7.4 Reading Text and correction history

Reading Text stores ordered blocks. Each block contains plain Unicode text, its source OCR block references, meaningful layout type, formatting ranges, and uncertainty ranges. Semantic source ranges always address this plain text, not rendered markup.

Correction events are append-only and include source attempt, target block/range, before text, after text, reason, confidence, deciding agent, and review provenance. A text difference without a matching correction chain is a validation error.

Uncertainty kinds distinguish `candidate`, `illegible`, and `outside_photo`. Low confidence never forces the reader to open evidence.

### 7.5 Semantic Spans

A Semantic Span has a stable ID and one or more exact Reading Text references: Source revision, page ID, block ID, start offset, and end offset. Offsets are zero-based half-open ranges `[start,end)` counted in Unicode code points in the exact stored string. Canonical text is not Unicode-normalized after OCR selection. JavaScript code must explicitly convert code-point offsets to UTF-16 DOM indices; raw JavaScript string indices are not canonical offsets. Spans may cross blocks or pages by containing multiple ordered references. The validator confirms every offset and quoted slice against the frozen Reading Text revision.

### 7.6 Knowledge Graph

The graph is the sole canonical semantic structure. It contains:

- document-unit, semantic-unit, concept, entity, and argument nodes; an entity uses `entity_kind=actor` when it owns actions in a swimlane;
- the required role enum on semantic-unit nodes;
- the human-required relation enum on directed edges plus exactly `precedes`, `causes`, `performed_by`, and `evolves_to` for adaptive-view semantics; these extensions have the same Source, epistemic, confidence, and speaker requirements as every other edge;
- separate `document` and `semantic` hierarchy collections;
- argument step records for question, premise, evidence/example, inference, conclusion, objection, and response;
- concept original terms and translations, senses, definitions, first occurrence, change events, contrasts, broader/narrower links, related claims, and observed-input scope.

Every AI-generated node, edge, hierarchy link, concept sense, and argument step has at least one source span ID, `epistemic_status`, numeric confidence in `[0,1]`, and speaker attribution (`author`, `quoted_person`, `editor`, or `ai`).

Concept, argument, role, and relation files used by views are generated projections. Agents never edit them separately.

## 8. Stable IDs and revisions

Every hash-derived ID uses one function:

```text
hash_id(prefix, domain, parts) =
  prefix + "-" + lowercase_hex(
    SHA-256(UTF-8(JSON.stringify([domain, ...parts])))
  )
```

`parts` may contain only JSON strings, integers, booleans, null, or arrays of those primitives. Object key ordering is therefore never an ID input. Full 64-character SHA-256 hex is retained. Domain strings include a version and are fixed below.

Path normalization for ID input is also fixed:

1. Resolve the input root and file, reject traversal outside the root, and compute the relative path.
2. Normalize every path segment to Unicode NFC without changing case.
3. Join segments with `/`; do not include drive letter, volume, or input-root path.
4. Encode the resulting string as UTF-8 through `JSON.stringify` above.
5. Reject two discovered files whose normalized relative paths are identical.

Canonical IDs are:

- Book ID: `hash_id("book", "book-v1", sort(evidence_ids))`.
- Evidence ID: `hash_id("evidence", "evidence-v1", [content_sha256, normalized_relative_path])`. A path change creates a new occurrence ID; `content_sha256` remains the cross-run byte-identity key.
- Logical Page ID: `hash_id("page", "logical-page-v1", [evidence_id, region_key])`.
- Raw OCR attempt ID: `hash_id("ocr", "raw-ocr-v1", [page_id, agent_id, attempt_ordinal, raw_text_sha256])`.
- Raw OCR block ID: `hash_id("ocr-block", "raw-ocr-block-v1", [ocr_attempt_id, block_ordinal])`.
- Source revision: `hash_id("source-revision", "source-revision-v1", [parent_revision_or_null, ordered_reading_content_hashes, ordered_correction_event_hashes])`.
- Reading Text block ID: `hash_id("text-block", "reading-block-v1", [source_revision, page_id, block_ordinal])`.
- Semantic Span ID: `hash_id("span", "semantic-span-v1", [source_revision, ordered_source_reference_tuples])`.
- Graph node/edge ID: `hash_id("graph", "graph-element-v1", [element_type, ordered_source_span_ids, local_discriminator])`.

`region_key` is independent of reading order and mutable crop coordinates: `full` for one page occupying the image, `left`/`right` for a horizontal spread, `top`/`bottom` for a vertical split, or `region-NN` for any other layout. Other regions are sorted by `(top, left, height, width)` in the original unrotated Evidence coordinate system and numbered from `01`. Coordinates use a top-left origin and integer millionths in `[0,1000000]`; a region covers `[x,x+width) × [y,y+height)`, requires positive width/height, and must remain inside that range. Coordinate refinements do not change a Page ID while its `region_key` is unchanged. Changing between full/split layouts creates new logical Page IDs and a supersession record.

Canonical writes are atomic: write a sibling temporary file, validate it, then replace the destination. The old valid file remains intact if validation fails.

Revisions are explicit. Changing an upstream canonical hash marks downstream artifacts stale; the renderer refuses to present stale views as current.

## 9. Write ownership and conflict prevention

| Area | Writer |
|---|---|
| `evidence/images`, initial inventory | deterministic CLI |
| `work/.../shards/<agent-id>` | that one assigned agent only |
| `work/.../decisions` | Root Orchestrator only |
| canonical Evidence/Page Index | Root invoking deterministic CLI |
| canonical Source/Reading Text | Root invoking deterministic CLI |
| canonical Knowledge Graph | Root invoking deterministic CLI |
| Integrity Auditor findings | returned read-only result; Root records it |
| views and mechanical report | deterministic CLI |

Workers receive an assignment file listing owned IDs, readable context IDs, output path, schema path, and stopping condition. A shard outside its assignment fails merge. Overlapping ownership fails merge rather than applying last-writer-wins behavior.

## 10. Reading Atlas design

The viewer has one shared selection state, so all views navigate to the same node/span without duplicating semantic data.

### Atlas View

Shows document scope, chapter purposes, central questions, major claims, concepts, and arguments. It starts collapsed and never renders the full graph at once.

### Position View

Shows breadcrumb, document tree, current topic, current question, and overall progress. Document and semantic hierarchies remain visually distinct.

### Logic View

Uses deterministic layered lanes for question, premise, evidence/example, inference, conclusion, objection, and response. It supports one-step Argument Playback.

### Concept View

Shows definition, source-grounded senses, first occurrence, contrasts, broader/narrower concepts, related claims, and definition changes. It does not merge distinct senses merely because labels match.

### Text Reader

Shows every Reading Text block in order, page boundaries, uncertainties, footnotes, and semantic highlights. It contains no evidence image in normal mode.

### Shared interaction

- Semantic Zoom moves between document, chapter, argument, structure-with-text, and full text without treating an upper level as a substitute.
- Focus Mode shows the selected node, immediate neighbors, ancestors, source text, long-distance links, epistemic status, and confidence.
- Bidirectional navigation uses graph/span IDs and stable DOM anchors.
- Semantic Lens toggles role, concept, and uncertainty highlights.
- The working-memory shelf pins current question, claim, concepts, premises, objections, and user-selected items in browser-local state.
- Audit Mode is a deliberate toggle and is the only route that loads evidence images.

Color is never the only carrier of meaning. Shape, line style, label, legend, keyboard focus, and a text alternative accompany every graph.

The shared visual grammar maps explicit relations to solid lines, inferred relations to dashed lines, lower confidence to reduced line emphasis, long-distance references to curved lines, and roles to labeled shapes/icons. Concept or relation colors always have a redundant text/shape encoding.

Confidence is always available numerically. Display bands are fixed at `confidence >= 0.85` (normal emphasis), `0.60 <= confidence < 0.85` (reduced emphasis plus a confidence label), and `confidence < 0.60` (warning treatment plus a confidence label). Epistemic status controls explicit-versus-inferred line style independently of confidence; a high confidence score never upgrades `inferred` to `explicit`.

### Adaptive views

The renderer selects a view only when the Graph meets one of these deterministic minimum patterns. Eligibility uses `explicit` or `strongly_inferred` records; merely `inferred` or `uncertain` records may appear with the required styling but cannot make an otherwise ineligible view appear.

- Timeline: at least two event nodes connected by a sourced temporal-order relation or carrying sourced ordered time values.
- Flowchart: at least two procedure-step nodes connected by sourced `precedes` or `depends_on` relations.
- Swimlane: at least two attributed actors and at least two temporally/procedurally ordered actions distributed across them.
- Causal diagram: at least two nodes connected by a sourced causal relation; generic support is not re-labeled as causation.
- Taxonomy/treemap: at least one semantic-hierarchy branch with one parent and two children.
- Comparison matrix: at least two compared entities and two common sourced comparison dimensions.
- Arc/backlink view: at least one sourced `refers_to` relation whose endpoints are separated by another logical page or document section.
- Concept evolution: one concept with at least two source-ordered definition/sense states.
- Distribution heatmap: at least two document sections and two concepts with counted sourced occurrences.
- Objection/response lanes: at least one objection linked to a claim and at least one response linked to that objection.

The Graph schema uses `precedes` for sourced temporal/procedural order, `causes` only for sourced causation, `performed_by` from an action to an actor entity, and `evolves_to` between source-ordered concept states. Empty or unsupported view types are omitted rather than filled with speculative structure.

## 11. Deterministic CLI surface

The thin entrypoint `scripts/book-atlas.mjs` exposes bounded commands:

- `init`: create a new evidence bundle safely;
- `status`: report current stage, completed IDs, and remaining IDs;
- `validate-shard`: validate an agent proposal without canonical writes;
- `apply-pages`: apply a Root page-reconstruction decision;
- `apply-ocr`: merge owned OCR shards and correction decisions;
- `apply-structure`: merge local/global structure decisions;
- `begin-revision`: create an explicit correction revision and affected-range assignment from a Root-authored request;
- `validate`: run schema and cross-layer integrity checks;
- `build`: regenerate projections, reports, HTML, and SVG;
- `verify`: run strict completion checks and return nonzero unless complete.

Commands that modify canonical files accept only paths inside the resolved bundle root. They refuse unresolved traversal, non-empty initialization destinations, schema-version mismatches, overlapping ownership, and stale upstream hashes.

A `begin-revision` request contains a revision-request ID, user request text, one or more target selectors (page/block/span ID and optional quoted context), reason, optional proposed reading, request timestamp, and parent Source revision. The command resolves every selector before creating assignments and fails rather than guessing when quoted context is absent or non-unique.

## 12. Integrity report

The report includes every metric required by the human baseline plus:

- bundle and canonical revision hashes;
- status (`invalid`, `partial`, or `complete`);
- separate legibility status (`fully_legible`, `legibility_issues`, or `unreadable`);
- per-stage completion counts;
- stale derived artifact count;
- exact validation errors with locations;
- agent ID, role, model, effort, assignment, and completion status;
- conflict and blind-review decision ledger.

`complete` is mechanically impossible while any in-scope page is pending, any required reference is broken, any correction is untracked, any AI node lacks a source span, or model diversity is absent.

CLI exit codes are stable: `0` for processing-complete bundles (including disclosed legibility issues), `1` for invalid bundles, and `2` for partial/interrupted bundles. Legibility never changes an invalid or partial bundle into complete and is always shown beside processing status.

## 13. Security, privacy, and failure behavior

- No network request is made by generated artifacts.
- The Skill does not browse or use external sources unless the user separately requests an isolated extension layer.
- No credential is accepted into, copied to, or embedded in a bundle.
- Input images are read-only from the workflow's perspective.
- Initializing an existing non-empty destination fails.
- Resume requires a matching book ID and schema version.
- Interrupted stages preserve validated canonical work and leave proposals in the run directory.
- Partial processing is reported with exact completed and remaining page IDs.

## 14. Deployment and replacement

Implementation and tests occur only in this repository. After acceptance tests pass, the repository can be linked or installed into the current official user Skill location. The existing `C:\Users\sugiy\.codex\skills\structured-reading` prototype is not a source and must not be merged back.

Replacing or removing the installed prototype is a separate deployment step after the user accepts the implementation. No compatibility layer will preserve the obsolete Markdown-only output contract.

## 15. Explicit non-goals

- Direct OpenAI API integration or a background AI service.
- PDF, handwriting, audio, ebook, translation, fact-checking, or cloud sync.
- Perspective Lens Chat.
- Editing canonical source layers from the browser.
- Generic force-directed layouts that imply unsupported relationships.
- Silent single-agent, same-model, OCR-engine, or online fallbacks.
