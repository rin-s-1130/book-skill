# Structured Reading Skill — Technical Architecture v0.1

## Status and authority

- Status: implementation-ready design; code has not been started.
- Product baseline: [human requirements v0.2](../human/requirements-v0.2.md).
- Third-party boundary: [third-party adoption plan v0.1](third-party-adoption-v0.1.md).
- Scope: MVP v0.1 only. Perspective Lens Chat remains outside this design.
- Authority rule: the human requirements define product behavior. This document defines the technical realization and must be changed when an implementation decision changes.

This design intentionally replaces the previously installed, Markdown-oriented prototype. The repository is the only implementation source of truth. An installed copy is a deployment artifact, never an independently edited source.

## 1. Architectural decisions

### 1.1 The repository root is the skill root

The repository will be a standalone Codex Skill, with `SKILL.md` at its root. This keeps authoring, validation, tests, assets, and the installed Skill identical. It also avoids a second nested copy that could drift.

The project will initially remain a standalone Skill rather than a plugin. A plugin can be introduced later if public distribution or bundled connectors become a requirement.

### 1.2 Codex orchestrates AI work; repository code owns deterministic and machine-perception work

The Skill will not call the OpenAI API directly and will not require a second API key. Codex supplies independent image understanding, semantic judgment, and subagent orchestration. Repository code performs deterministic work and invokes one pinned local RapidOCR/ONNX Runtime machine-perception sidecar:

- image discovery, hashing, evidence copying, and preview creation;
- Machine Raw OCR, layout hierarchy, line/token polygons, and recognition confidence;
- deterministic alignment of Machine and Model Raw OCR;
- stable ID generation;
- schema and reference validation;
- canonical merge operations;
- report aggregation;
- derived projection and Reading Atlas generation.

This boundary avoids hiding model calls inside an opaque program while giving Luna an independent machine observation to disagree with. Third-party output is archived verbatim and normalized into repository-owned schemas; it never becomes an external canonical authority.

### 1.3 Node.js is the core runtime; Python is an isolated OCR sidecar

The implementation uses two pinned local runtimes with a narrow process boundary.

- Node.js 22 owns the CLI, IDs, schemas, canonical merge, validation, projections, HTML bundling, and browser application. Its exact runtime and packages are declared in `.nvmrc`, `package.json`, and `package-lock.json`.
- Python 3.13.15 owns only the RapidOCR adapter. Its runtime and complete transitive dependency set are declared in `.python-version`, `pyproject.toml`, and a hash-locked requirements file.
- OCR model files are declared by name, source, license, and SHA-256 in `models.lock.json`; automatic model download is disabled during processing.
- Cytoscape.js is bundled as the offline interactive renderer. It never owns canonical Graph data or layout semantics.
- The generated Reading Atlas has no runtime package, network, CDN, font, or server dependency.
- Browser behavior tests use a pinned Playwright development dependency.

The implementation phase must check current official package documentation before selecting exact versions. No package, Python module, native executable, or model file is assumed to exist globally. The full selection and license boundary is defined in the [third-party adoption plan](third-party-adoption-v0.1.md).

### 1.4 Canonical data is separate from work products and views

Canonical layers are immutable or append-only wherever practical:

1. Evidence inventory and copied image bytes.
2. Machine and Model Raw OCR attempts, their archived vendor observation, and deterministic alignment.
3. Reading Text plus append-only correction events.
4. Semantic Spans.
5. Knowledge Graph.
6. Append-only provenance ledger.

Everything under `views/` is derived and can be deleted and regenerated. Agent shards under `work/` are proposals, not canonical data.

### 1.5 The browser artifact is a single offline HTML entrypoint

`views/atlas.html` embeds its styles, scripts, and validated data. This avoids `file://` fetch restrictions and allows the principal reader to work by opening one local file. Static SVG exports are generated beside it.

Normal reading mode does not create image or thumbnail elements. Audit Mode creates evidence image elements only after an explicit user action.

### 1.6 Explicit MVP decisions and runtime assumptions

The following are design decisions, not hidden implementation assumptions:

- **OCR implementation:** each logical page receives two independent observations. RapidOCR 3.9.2 with ONNX Runtime 1.29.0 uses PP-OCRv6 small detection/recognition through the Japanese route plus the bundled legacy PP-OCRv4 orientation classifier. These exact hash-locked components produce Machine Raw OCR with word/character boxes and confidence. Luna `max`, without seeing that result, produces Model Raw OCR. Deterministic alignment exposes every disagreement before Reading Text is selected. There is no cloud OCR service or silent OCR-engine fallback.
- **AI execution host:** the Skill runs in a Codex environment that provides local image inspection, subagent creation, explicit child-model selection, and explicit reasoning-effort selection. Missing capability is a preflight failure.
- **Required models:** both `gpt-5.6-luna` and `gpt-5.6-sol` must be selectable. Neither Terra nor a same-model effort variation substitutes for the two-model requirement.
- **Required machine perception:** Python 3.13.15 and the exact RapidOCR, ONNX Runtime, PP-OCRv6 detector/recognizer, and PP-OCRv4 orientation-classifier artifacts from the [adoption baseline](third-party-adoption-v0.1.md) must be present. Missing or mismatched artifacts stop preflight; Luna-only processing is not a fallback.
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
- **Third-party licensing:** only components listed as adopted in the [third-party adoption plan](third-party-adoption-v0.1.md) enter the runtime. Lockfiles, model hashes/licenses, and `THIRD_PARTY_NOTICES.md` are completion requirements.
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
  |-- Local RapidOCR/ONNX Runtime Sidecar
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
- **Machine Raw OCR:** the immutable, normalized result of the pinned local RapidOCR run, including detected polygons, word/character geometry, text, and confidence.
- **Model Raw OCR:** an immutable Luna image-reading attempt made without access to Machine Raw OCR.
- **Raw OCR:** the canonical collection of immutable Machine and Model attempts plus their deterministic alignment; it is never the corrected Reading Text.
- **Reading Text:** the source-faithful, human-readable transcription selected from Raw OCR plus explicit correction events.
- **Source:** the complete evidence-backed chain for a claim: an exact Reading Text range in a named Source revision, its Raw OCR provenance, and its Evidence image/region. A **Source Reference** is the Reading Text revision, page ID, block ID, and code-point range that resolves through that chain.
- **Layout element:** a repository-owned page, block, line, or token record with reading order and an Evidence-relative polygon/bounding box when available.
- **Provenance ledger:** the append-only Entity/Activity/Agent derivation record that links scripts, models, humans, revisions, audits, and generated artifacts.
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
|-- python/
|   |-- book_atlas_ocr/
|   `-- tests/
|-- src/
|   |-- cli/
|   |-- evidence/
|   |-- ids/
|   |-- merge/
|   |-- ocr-alignment/
|   |-- provenance/
|   |-- schema/
|   |-- validation/
|   |-- projections/
|   `-- rendering/
|-- schemas/
|   |-- common.schema.json
|   |-- run-manifest.schema.json
|   |-- evidence-inventory.schema.json
|   |-- page-index.schema.json
|   |-- machine-raw-ocr.schema.json
|   |-- model-raw-ocr.schema.json
|   |-- ocr-alignment.schema.json
|   |-- layout-elements.schema.json
|   |-- reading-text.schema.json
|   |-- correction-log.schema.json
|   |-- semantic-spans.schema.json
|   |-- knowledge-graph.schema.json
|   |-- provenance-ledger.schema.json
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
|-- pyproject.toml
|-- requirements.lock
|-- models.lock.json
|-- THIRD_PARTY_NOTICES.md
|-- .python-version
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
|   |   |-- machine/
|   |   |-- model/
|   |   |-- alignment/
|   |   `-- vendor/
|   |-- layout-elements/
|   |-- reading-text/
|   `-- correction-history/
|-- structure/
|   |-- semantic-spans.json
|   `-- knowledge-graph.json
|-- provenance/
|   `-- ledger.jsonl
|-- work/
|   `-- runs/<run-id>/
|       |-- run-manifest.json
|       |-- assignments/
|       |-- shards/
|       `-- decisions/
|-- views/
|   |-- atlas.html
|   |-- THIRD_PARTY_NOTICES.txt
|   `-- static/
|       |-- atlas.svg
|       |-- position.svg
|       |-- logic.svg
|       |-- concepts.svg
|       `-- provenance.provn
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
- Node and repository packages match `package-lock.json`;
- Python and sidecar packages match `.python-version` and `requirements.lock`;
- every OCR model file matches `models.lock.json`, and its license is recorded;
- the RapidOCR adapter loads only the exact fixed API/configuration and verifies the outer wheel plus inner model hashes without automatic download or fallback.

If multi-agent, model diversity, machine-perception, lock, or license requirements fail, processing stops before ingest. The Skill must report the missing capability and must not silently substitute a single agent, one-model topology, or another OCR engine.

Host capability is verified, not assumed. Milestone 1 creates `test/fixtures/capability/luna-vision.svg`, containing one black line `CODEX VISION OK 2048` on white, and uses pinned Sharp to deterministically build `test/fixtures/capability/luna-vision.png`; its dimensions and SHA-256 are fixed in `test/fixtures/capability/manifest.json`.

Before creating a book bundle, Root validates that PNG against the fixture manifest and runs four fresh no-history probe assignments:

| Probe | Input | Required result |
|---|---|---|
| `luna_xhigh` | nonce in text assignment | exact JSON `{"probe":"luna_xhigh","nonce":"<nonce>","ok":true}` |
| `luna_max_vision` | the PNG plus instruction to read its single line | exact JSON `{"probe":"luna_max_vision","text":"CODEX VISION OK 2048","ok":true}` |
| `sol_medium` | nonce in text assignment | exact JSON `{"probe":"sol_medium","nonce":"<nonce>","ok":true}` |
| `sol_xhigh` | nonce in text assignment | exact JSON `{"probe":"sol_xhigh","nonce":"<nonce>","ok":true}` |

Every spawn uses `fork_turns="none"` and the exact role model/effort. The successful spawn call with those explicit parameters is the host capability record; rejection, substitution reported by the host, tool absence, timeout, malformed JSON, nonce mismatch, or image-text mismatch fails preflight. An OS-temporary `preflight-run.json` records probe ID, role, requested model/effort, `fork_turns`, tool-call ID, fixture/assignment hash, start/end time, status, and response SHA-256. After successful bundle initialization, those metadata records—not prompts or responses—are copied into `run-manifest.json`; on failure, the temporary report path is returned for diagnosis. Probe content is never copied into the book bundle. Failure of any probe stops before ingest.

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

Stage 3 has four ordered sub-stages for every logical page.

#### 3A. Machine observation

The deterministic Node CLI invokes the pinned Python RapidOCR sidecar on the normalized logical-page crop. The sidecar writes only its assignment path and emits:

- complete schema-versioned serialization of the unmodified `RapidOCROutput` plus its hash;
- Machine Raw OCR page/block/line/token records;
- original recognized strings and token confidence without filtering;
- reading order, layout roles, polygons/bounding boxes, table cells, and figure/caption records when available;
- exact Python, RapidOCR, ONNX Runtime, embedded PP-OCR model, adapter, and configuration identities.

The adapter never corrects or rewrites machine text. A sidecar error becomes an explicit stage error; it does not cause Luna-only continuation.

#### 3B. Independent Model Raw OCR

The Primary OCR Worker uses Luna `max` to inspect the same Evidence region without receiving Machine Raw OCR, vendor JSON, alignment, or machine confidence. It emits immutable Model Raw OCR, layout-block candidates, confidence, uncertainty ranges, and a source-faithful Reading Text candidate.

#### 3C. Deterministic alignment and review

The CLI aligns Machine and Model OCR at block/line/token and Unicode code-point levels without changing either attempt. A reconciliation item is the smallest contiguous aligned range sharing one Evidence region and one difference class. Items are never merged across block, line, table-cell, formula, code, footnote, or page boundaries.

Comparison-only normalization converts CRLF/CR to LF. For prose, it may also classify a difference consisting only of photographic line wrapping as `layout_only`; it does not normalize Unicode, spelling, punctuation, digits, or other whitespace. A `layout_only` prose item is selected mechanically according to the Reading Text reflow rule. The same difference in poetry, code, formulas, tables, lists, or any ambiguous layout is not automatic.

Every content insertion, deletion, substitution, reorder, unmatched polygon, meaningful-layout disagreement, confidence below `0.85`, explicit uncertainty, correction, or cross-page discontinuity requires blind review. A versioned risk detector also marks an item `meaning_critical` when it contains a numeral/date, mathematical or logical symbol, negation, comparison, quantifier, proper-name candidate, citation, heading/section number, footnote marker, quotation boundary, table cell, formula, or code token. Its Japanese/English lexicons and symbol sets are repository data covered by tests and the run's instruction revision.

A fresh blind Luna `max` reviewer receives only the Evidence region and narrow transcription/layout question. Selection then follows this fixed table:

| Machine / Primary Luna / Blind Luna result | `meaning_critical` | Decision |
|---|---:|---|
| Machine and Primary are exact and no review trigger exists | no | Select the shared text; no Blind Luna call |
| all three are exact | either | Select the shared text |
| exactly two are exact | no | Select the exact majority |
| exactly two are exact | yes | Sol `medium` adjudication |
| all three differ, no exact majority, or any required region is unmatched | either | Sol `medium` adjudication |
| evidence remains unreadable after Sol | either | Select an explicit candidate/illegible/outside-photo marker; never guess |

An "exact" comparison uses canonical code points after only the comparison normalization above. "Material disagreement" means any table row that routes to Sol; the phrase is not an additional subjective test. Sol receives the Evidence region, the three immutable candidates, confidence/provenance, and the risk flags. It may select a candidate, or emit a new reading only when it identifies the exact Evidence region and confidence; otherwise it must select explicit uncertainty. The Root records the selected reading, rejected alternatives, rule/table row, rationale, and provenance activity.

#### 3D. Reading Text selection

The Root invokes deterministic merge with the decisions. Each reconciliation item has exactly one decision record and exactly one resulting Reading Text range. That decision contains zero or one correction edge for each Machine, Primary Luna, and Blind Luna slice whose exact text differs from the selected text. Correction edges are keyed by `(reconciliation_item_id, source_attempt_id)` and cannot be duplicated or merged with another item. Alignment describes differences; correction edges describe selection; the provenance ledger describes who/what generated the decision.

Final Reading Text keeps ordered block/line/token correspondence where available and explicit uncertainty where no reading is justified. A model-only character or range must still reference an explicit Evidence polygon/region and review decision.

The workflow continues with explicit uncertainty markers when no reading can be justified. It never requires the user to inspect an image in order to obtain the text and structural output.

Machine and Model Raw OCR and vendor JSON are never rewritten. Reading Text is frozen for the structural-analysis revision. A later Reading Text change creates a new Source revision, append-only provenance, and invalidation records for dependent Semantic Spans, Graph elements, and views before they are rebuilt.

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
3. Every in-scope logical page has hash-valid vendor JSON, Machine Raw OCR, Model Raw OCR, reconciliation output, and Reading Text or an explicit failed/unreadable record.
4. Every Reading Text change has a continuous correction chain.
5. Every Semantic Span resolves to valid character offsets in the current Reading Text revision.
6. Every Machine layout element has valid containment, order, polygon bounds, vendor provenance, and model/configuration identity; every model-only correction has an Evidence region and decision.
7. Every Graph node, relation, hierarchy link, concept sense, and argument step resolves through a Span, Reading Text, Raw OCR alignment, and Evidence.
8. Every canonical/generated entity has a valid provenance generation/derivation chain and no append-only history was deleted.
9. No required model, effort, assignment, conflict decision, OCR artifact, package/model lock, license record, or agent result is absent from the run ledger.
10. Derived views match the current canonical hash, bundled third-party code matches the lock/notice, and normal reader mode does not preload Evidence images or make network requests.

Mechanical failure blocks semantic audit promotion and rendering, but the report still records all discoverable failures.

#### 6B. Semantic integrity audit

A fresh Luna `max` Integrity Auditor is read-only and receives canonical data, Evidence, assignments, and unresolved-decision records, but not the workers' reasoning or the Root's suspected-error list. It performs these concrete passes:

1. **Page fidelity:** re-inspect every logical page against Machine Raw OCR, Model Raw OCR, reconciliation output, and Reading Text, with exact attention to headings, paragraph boundaries, omissions, emphasis, footnotes, tables, figures, formulas, poetry/code line breaks, and explicit uncertainty markers.
2. **High-risk OCR:** re-read every machine/model disagreement, unmatched token/polygon, correction, low-confidence range, illegible marker, negation, numeral, proper noun, comparison term, and logical operator. Confirm that machine/model uncertainty was not silently promoted to certain text.
3. **Boundary continuity:** compare every adjacent page boundary and every worker-range boundary for missing clauses, duplicated text, detached footnotes, and false section breaks.
4. **Source grounding:** inspect every AI node and relation against its Source Spans and the underlying Reading Text/Raw OCR/Evidence path; reject unsupported paraphrases, reversed edge direction, invented connections, and claims stronger than the cited text.
5. **Speaker and epistemic status:** verify author/quoted-person/editor/AI attribution and whether `explicit`, `strongly_inferred`, `inferred`, or `uncertain` matches the evidence.
6. **Structural coherence:** check role classification, separate document/semantic hierarchies, concept sense identity and evolution, argument step order, objection/response pairing, and long-distance references.
7. **Scope honesty:** verify that excerpt or unknown input is not described as a complete book and that unresolved missing-page candidates remain visible.
8. **Conflict closure:** verify that every machine/model disagreement, worker disagreement, and blind-review result has a recorded decision, rationale, deciding agent, model, effort, and provenance activity.
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
| Machine OCR Sidecar | RapidOCR 3.9.2 / ONNX Runtime 1.29.0 / PP-OCRv6 small det+rec Japanese route / PP-OCRv4 orientation classifier | none | Produces reproducible word/character geometry, text, and confidence without AI correction |
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

Machine OCR and other mechanical work do not use a Codex model. Luna `xhigh` is for bounded visual/structural analysis; Luna `max` is for independent exact transcription and exhaustive audit. Sol `medium` is for a bounded, well-evidenced disagreement; Sol `xhigh` is for cross-section or whole-document synthesis. Production runs do not use another effort or OCR engine for these roles without a reviewed design change.

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

### 7.3 Layout elements and Raw OCR

Machine Raw OCR contains a repository-owned hierarchy of page, block, line, and token Layout Elements. Each record contains:

- attempt and element ID, parent ID, page ID, element kind, and reading order;
- original machine-recognized string and confidence;
- Evidence and logical-page polygons/bounding boxes in normalized integer-millionth coordinates;
- detected layout role such as heading, paragraph, line, token, footnote, caption, figure, table/cell, code, formula, or poetry;
- vendor record ID and archived vendor JSON/hash;
- exact Python, adapter, RapidOCR, ONNX Runtime, recognizer/model, configuration, and timestamp.

Model Raw OCR contains the immutable Luna attempt ID, page/Evidence region, original recognized text, ordered block candidates, model-reported confidence/uncertainty, exact model ID, reasoning effort, agent ID, instruction/reference revision, and timestamp.

The OCR Alignment record contains immutable links between Machine tokens/lines and Model code-point ranges, alignment operation (`match`, `substitute`, `insert`, `delete`, `reorder`, or `unmatched_region`), scores, and reconciliation-item IDs. It never changes either source attempt.

All attempts and the first vendor observation are retained even when a later reading is better.

### 7.4 Reading Text and correction history

Reading Text stores ordered blocks, lines, and token/range links. Each block contains plain Unicode text, Machine/Model Raw OCR and Alignment references, meaningful layout type, formatting ranges, Evidence polygons, and uncertainty ranges. Semantic Source ranges always address this plain text, not rendered markup. Machine-aligned characters can resolve to token polygons; model-only characters resolve to the explicit Evidence region recorded by their reconciliation decision.

Correction events are append-only and include every differing source attempt, target block/range, before text, after text, rejected alternatives, reason, confidence, deciding agent, and provenance activity. A text difference without a matching reconciliation/correction chain is a validation error.

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

Each Graph element also preserves processing history: `created_in_run`, `asserted_in_source_revision`, optional `invalidated_in_run`, optional `invalidated_in_source_revision`, and optional `supersedes`. When the authored text changes a concept or claim over document position, `source_valid_from_span` and `source_valid_to_span` record that authored scope separately. Processing revisions are never displayed as authored historical dates.

Concept, argument, role, and relation files used by views are generated projections. Agents never edit them separately.

### 7.7 Provenance ledger

The append-only ledger uses a repository profile of W3C PROV:

- Entity: Evidence, vendor observation, Raw OCR attempt, Alignment, Reading Text revision, correction, Span set, Graph revision, audit finding/decision, report, and view;
- Activity: ingest, machine OCR, model OCR, align, reconcile, correct, merge, audit, validate, and render;
- Agent: human requester, Codex agent/model configuration, deterministic Node command, and pinned Python sidecar;
- Relations: `used`, `wasGeneratedBy`, `wasDerivedFrom`, `wasAttributedTo`, `wasAssociatedWith`, and `wasInvalidatedBy`.

Every generated canonical or derived Entity has exactly one generation activity and one or more upstream derivations where applicable. The ledger is canonical; `provenance.provn` is a regenerated interoperability projection.

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
- Vendor observation ID: `hash_id("vendor-ocr", "rapidocr-observation-v1", [page_id, vendor_json_sha256, locked_configuration_id])`.
- Machine Raw OCR attempt ID: `hash_id("ocr-machine", "machine-raw-ocr-v1", [page_id, vendor_observation_id, exact_machine_text_sha256])`.
- Model Raw OCR attempt ID: `hash_id("ocr-model", "model-raw-ocr-v1", [page_id, agent_id, attempt_ordinal, raw_text_sha256])`.
- Layout Element ID: `hash_id("layout", "layout-element-v1", [machine_attempt_id, element_kind, parent_id_or_null, element_ordinal])`.
- OCR Alignment ID: `hash_id("ocr-align", "ocr-alignment-v1", [machine_attempt_id, model_attempt_id, aligner_version])`.
- Source revision: `hash_id("source-revision", "source-revision-v1", [parent_revision_or_null, ordered_reading_content_hashes, ordered_correction_event_hashes])`.
- Reading Text block ID: `hash_id("text-block", "reading-block-v1", [source_revision, page_id, block_ordinal])`.
- Semantic Span ID: `hash_id("span", "semantic-span-v1", [source_revision, ordered_source_reference_tuples])`.
- Graph node/edge ID: `hash_id("graph", "graph-element-v1", [element_type, ordered_source_span_ids, local_discriminator])`.
- Provenance record ID: `hash_id("prov", "provenance-record-v1", [record_kind, subject_id, activity_or_relation_identity])`.

`region_key` is independent of reading order and mutable crop coordinates: `full` for one page occupying the image, `left`/`right` for a horizontal spread, `top`/`bottom` for a vertical split, or `region-NN` for any other layout. Other regions are sorted by `(top, left, height, width)` in the original unrotated Evidence coordinate system and numbered from `01`. Coordinates use a top-left origin and integer millionths in `[0,1000000]`; a region covers `[x,x+width) × [y,y+height)`, requires positive width/height, and must remain inside that range. Coordinate refinements do not change a Page ID while its `region_key` is unchanged. Changing between full/split layouts creates new logical Page IDs and a supersession record.

Canonical writes are atomic: write a sibling temporary file, validate it, then replace the destination. The old valid file remains intact if validation fails.

Revisions are explicit. Changing an upstream canonical hash marks downstream artifacts stale; the renderer refuses to present stale views as current.

## 9. Write ownership and conflict prevention

| Area | Writer |
|---|---|
| `evidence/images`, initial inventory | deterministic CLI |
| RapidOCR vendor JSON and Machine OCR proposal | pinned Python sidecar in its assigned work path |
| `work/.../shards/<agent-id>` | that one assigned agent only |
| `work/.../decisions` | Root Orchestrator only |
| canonical Evidence/Page Index | Root invoking deterministic CLI |
| canonical Source/Reading Text | Root invoking deterministic CLI |
| canonical Knowledge Graph | Root invoking deterministic CLI |
| canonical provenance ledger | append-only deterministic CLI |
| Integrity Auditor findings | returned read-only result; Root records it |
| views and mechanical report | deterministic CLI |

Workers receive an assignment file listing owned IDs, readable context IDs, output path, schema path, and stopping condition. A shard outside its assignment fails merge. Overlapping ownership fails merge rather than applying last-writer-wins behavior.

## 10. Reading Atlas design

The viewer has one shared selection state, so all views navigate to the same node/span without duplicating semantic data.

Cytoscape.js renders interactive Graph canvases from repository-generated elements and positions. All canonical views use Cytoscape's `preset` layout; force-directed or automatic semantic layouts are forbidden. Cytoscape owns only rendering and interaction events. The projection JSON, shared selection state, Source resolver, accessibility tree, and deterministic coordinates remain repository-owned.

### Atlas View

Shows document scope, chapter purposes, central questions, major claims, concepts, and arguments. It starts collapsed and never renders the full graph at once.

### Position View

Shows breadcrumb, document tree, current topic, current question, and overall progress. Document and semantic hierarchies remain visually distinct.

### Logic View

Uses deterministic layered lanes for question, premise, evidence/example, inference, conclusion, objection, and response. It supports one-step Argument Playback.

Its support/attack and premise/conclusion presentation follows the clarity of Argdown argument maps, but the Knowledge Graph—not Argdown text—is authoritative.

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
- Audit Mode can traverse Graph history and provenance activities, including invalidated prior interpretations, machine/model OCR disagreement, and the exact decision that selected Reading Text.

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
- `machine-ocr`: invoke the locked RapidOCR sidecar for assigned logical pages and validate/archive its output;
- `align-ocr`: align immutable Machine and Model attempts and emit reconciliation items;
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
- conflict and blind-review decision ledger;
- Machine/Model OCR agreement and disagreement counts, unmatched Layout Elements, and adjudication counts;
- exact Python/RapidOCR/ONNX Runtime/model lock identities and license-verification status;
- provenance entities/activities/agents, broken provenance relations, and invalidated Graph element counts.

`complete` is mechanically impossible while any in-scope page is pending, any required reference/provenance link is broken, any correction or machine/model disagreement is untracked, any AI node lacks a source span, model diversity is absent, or required package/model/license locks are unverified.

CLI exit codes are stable: `0` for processing-complete bundles (including disclosed legibility issues), `1` for invalid bundles, and `2` for partial/interrupted bundles. Legibility never changes an invalid or partial bundle into complete and is always shown beside processing status.

## 13. Security, privacy, and failure behavior

- No network request is made by generated artifacts or normal reading runs. Package/model acquisition is a separate explicit setup operation that verifies repository locks before installation.
- The Skill does not browse or use external sources unless the user separately requests an isolated extension layer.
- No credential is accepted into, copied to, or embedded in a bundle.
- Input images are read-only from the workflow's perspective.
- Initializing an existing non-empty destination fails.
- Resume requires a matching book ID and schema version.
- Interrupted stages preserve validated canonical work and leave proposals in the run directory.
- Partial processing is reported with exact completed and remaining page IDs.
- The Python sidecar is invoked without a shell, receives only resolved in-bundle paths and read-only Evidence, and cannot write canonical locations.
- Missing/mismatched OCR packages or models fail preflight; the system does not continue with Luna-only OCR.
- Bundled third-party code and model artifacts require lockfile and license-notice coverage.

## 14. Deployment and replacement

Implementation and tests occur only in this repository. After acceptance tests pass, the repository can be linked or installed into the current official user Skill location. The existing `C:\Users\sugiy\.codex\skills\structured-reading` prototype is not a source and must not be merged back.

Replacing or removing the installed prototype is a separate deployment step after the user accepts the implementation. No compatibility layer will preserve the obsolete Markdown-only output contract.

## 15. Explicit non-goals

- Direct OpenAI API integration or a background AI service.
- PDF, handwriting, audio, ebook, translation, fact-checking, or cloud sync.
- Perspective Lens Chat.
- Editing canonical source layers from the browser.
- Generic force-directed layouts that imply unsupported relationships.
- Graphiti/LightRAG/GraphRAG as canonical Graph stores or hidden LLM pipelines.
- Docling, PaddleOCR, Tesseract, EasyOCR, or another OCR engine as a runtime fallback alongside the selected RapidOCR/ONNX Runtime path.
- Silent single-agent, same-model, OCR-engine, package, model-file, or online fallbacks.
