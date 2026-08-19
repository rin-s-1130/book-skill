# Structured Reading Skill — Third-party Adoption Plan v0.1

## Status and authority

- Status: approved technical redesign input; implementation has not started.
- Product authority remains [human requirements v0.2](../human/requirements-v0.2.md).
- The system architecture remains [architecture v0.1](architecture-v0.1.md).
- This document is authoritative for third-party selection, license boundaries, and the vendor-to-canonical normalization boundary.

The project reuses permissively licensed components where they improve fidelity or interaction quality. It does not delegate canonical data ownership to a third-party format, graph database, OCR service, or hosted reader.

## 1. Selection rules

A third-party component is adopted only when all of the following hold:

1. It materially improves an MVP acceptance criterion.
2. Its code and required model licenses permit local use and distribution of this project.
3. Its exact package and model versions can be pinned and hash-verified.
4. It works without network access after an explicit setup step.
5. Its output can be preserved verbatim and normalized into repository-owned schemas.
6. Failure is explicit; the workflow never silently switches engines or weakens the result.
7. Removal does not change the meaning of an already generated canonical bundle.

`THIRD_PARTY_NOTICES.md` and a machine-readable dependency/model lock are required implementation artifacts. PP-OCR model copyright/license terms are attached to the exact inner wheel files rather than inferred only from Python package metadata. No AGPL or license-unknown code is copied, vendored, linked, or adapted into the runtime.

## 2. Adopted runtime components

### 2.1 RapidOCR with ONNX Runtime

Adopt [RapidOCR](https://github.com/RapidAI/RapidOCR) directly as the local machine-perception sidecar and [ONNX Runtime](https://github.com/microsoft/onnxruntime) as its only inference backend. RapidOCR is Apache-2.0, ONNX Runtime is MIT, and the selected PP-OCRv6 models originate from the Apache-2.0 [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) project. Exact model files and upstream notices remain separately recorded because model copyright must not be inferred only from Python package metadata.

Use one fixed OCR path:

```text
RapidOCR 3.9.2
  `-- ONNX Runtime 1.29.0 CPUExecutionProvider
       |-- bundled PP-OCRv6 small detector/recognizer
       |    `-- language route: japan
       `-- bundled PP-OCRv4 orientation classifier
```

The adapter uses the pinned RapidOCR 3.9.2 Python API:

```text
RapidOCR(config_path=<repository config>)
engine(
  <logical-page image>,
  return_word_box=True,
  return_single_char_box=True,
  text_score=0.0
)
```

The repository configuration fixes:

- `Det.engine_type=onnxruntime`, `Det.ocr_version=PP-OCRv6`, `Det.lang_type=multi`, `Det.model_type=small`;
- `Rec.engine_type=onnxruntime`, `Rec.ocr_version=PP-OCRv6`, `Rec.lang_type=japan`, `Rec.model_type=small`;
- `Cls.engine_type=onnxruntime`, `Cls.ocr_version=PP-OCRv4`, `Cls.lang_type=ch`, `Cls.model_type=mobile`; this legacy classifier is intentionally distinct from the PP-OCRv6 detector/recognizer and only classifies orientation;
- CPUExecutionProvider only for the portable baseline;
- word and single-character boxes enabled;
- recognition filtering disabled (`text_score=0.0`) so low-confidence text is preserved;
- automatic model download and every alternate backend disabled;
- network access during a reading run disabled.

#### Locked machine-perception artifacts

The design baseline is fixed as follows:

| Artifact | Version/file | SHA-256 / integrity | License |
|---|---|---|---|
| Python | `3.13.15` | Python release manifest/Sigstore verified during setup | PSF-2.0 |
| RapidOCR wheel | `rapidocr-3.9.2-py3-none-any.whl` | `04d6b8d151f823d930bd91910555f57bea897c0c44fa6794267b94cf9c1ef9a0` | Apache-2.0 |
| ONNX Runtime Windows x64 wheel | `onnxruntime-1.29.0-cp313-cp313-win_amd64.whl` | `2945e1f82f81f27e88decea88c7861f45baea23818950d467bf3909aa303119e` | MIT |
| detector | `PP-OCRv6_det_small.onnx` inside the RapidOCR wheel | `090f04abcd9d9a7498bc4ebf677e4cb9bdce1fe4197ddb7e529f1ef44e1ff94f` | Apache-2.0 upstream + attribution |
| recognizer | `PP-OCRv6_rec_small.onnx` inside the RapidOCR wheel | `6f327246b50388f3c176ae304bd95767ea6dc0c9ae92153ef8cbe210b3c14884` | Apache-2.0 upstream + attribution |
| orientation classifier | `ch_ppocr_mobile_v2.0_cls_mobile.onnx` inside the RapidOCR wheel | `e47acedf663230f8863ff1ab0e64dd2d82b838fceb5957146dab185a89d6215c` | Apache-2.0 upstream + attribution |

The RapidOCR wheel is the only permitted source for those three model files in MVP. Setup verifies the outer wheel and inner file hashes before installation/use. A platform other than Windows x64 requires a separately reviewed ONNX Runtime wheel entry but must keep the same RapidOCR and model hashes.

#### Ownership boundary

RapidOCR output is an observation, not the project schema:

1. The adapter serializes the complete unmodified `RapidOCROutput` into schema-versioned vendor JSON and records its SHA-256.
2. A deterministic normalizer converts it to repository-owned Machine Raw OCR and Layout Element records.
3. Detection polygons, recognized text, confidence, word/character boxes, order, preprocessing metadata, and package/model/config identities are preserved.
4. Repository code groups primitive boxes into page/block/line/token hierarchy; Luna may propose a different semantic layout but cannot rewrite Machine Raw OCR.
5. The Python sidecar writes only its assigned work path. Only the Root invoking the Node CLI may promote validated records to canonical Source data.

#### Why RapidOCR directly rather than Docling or PaddleOCR runtime

[Docling](https://github.com/docling-project/docling) remains a design reference for document hierarchy, lossless serialization, bounding boxes, and provenance, but its runtime would add layout/table models and download/lock surfaces that the machine Raw OCR layer does not require. Direct PaddleOCR would duplicate the same PP-OCR role with a heavier runtime. One direct RapidOCR path provides reproducible boxes/text/confidence while Luna handles source-faithful layout interpretation. Neither Docling nor PaddleOCR is an MVP runtime fallback.

#### Qualification gate

Before Milestone 3 is accepted, the pinned RapidOCR configuration must pass a repository fixture containing Japanese horizontal and vertical text, ruby, a two-page spread, footnotes, a table, a figure caption, emphasis, numerals, negation, and a formula.

The gate requires:

- every input and logical page produces a vendor record or an explicit engine error;
- every recognized token has a valid logical-page region and confidence;
- low-confidence tokens are retained;
- the adapter is deterministic on repeated runs with the same package/model locks;
- exact machine/model disagreements are detected by the reconciliation stage;
- no known meaning-changing gold-fixture error reaches final Reading Text.

Failure blocks the milestone and requires a reviewed architecture change. It does not activate another OCR engine.

### 2.2 Cytoscape.js

Adopt [Cytoscape.js](https://github.com/cytoscape/cytoscape.js) as the interactive graph renderer. It is MIT licensed and provides graph collections, filtering, selection events, pan/zoom, styling, and a `preset` layout that respects positions supplied by the application.

The repository retains semantic layout authority:

- projection code selects the visible subgraph;
- projection code computes deterministic positions and lanes;
- Cytoscape uses only the `preset` layout for canonical Reading Atlas views;
- no force-directed layout may create apparent semantic proximity;
- Focus Mode, Semantic Lens, bidirectional selection, and keyboard actions use Cytoscape events but repository-owned IDs/state;
- a DOM text/table alternative remains authoritative for accessibility;
- static SVG exports remain deterministic repository-generated projections rather than screenshots of the Cytoscape canvas.

The exact Cytoscape package is pinned in `package-lock.json` and bundled into `atlas.html`; generated artifacts make no CDN or network request.

## 3. Adopted design patterns without runtime dependency

### 3.1 Fine-grained source addressability

Borrow the useful traceability pattern demonstrated by [Aletheia](https://github.com/ayanalamMOON/aletheia) and Docling, without copying their schemas as canonical truth or depending on either runtime.

Repository Source addresses use:

```text
Evidence image/region
  -> logical page
  -> block
  -> line
  -> token
  -> Reading Text code-point range
```

Machine tokens retain polygons, confidence, reading order, and vendor references. Model-only corrections retain an explicit Evidence region even when no machine token matches.

### 3.2 W3C PROV profile

Use the [W3C PROV Data Model](https://www.w3.org/TR/prov-dm/) concepts `Entity`, `Activity`, and `Agent`, with the relations `used`, `wasGeneratedBy`, `wasDerivedFrom`, `wasAttributedTo`, `wasAssociatedWith`, and `wasInvalidatedBy`.

The project stores a repository-owned append-only provenance ledger. It covers Evidence ingest, machine OCR, model OCR, reconciliation, correction, Source revision, Semantic Span extraction, Graph merge, audit, and view generation. Correction history and the run manifest reference provenance activity IDs rather than duplicating lineage authority.

W3C PROV export is a derived interoperability view. The internal ledger remains the canonical representation because it also carries repository-specific stable IDs and validation fields.

### 3.3 Revision-aware Graph history

Borrow the useful temporal/provenance separation from [Graphiti](https://github.com/getzep/graphiti), but do not depend on Graphiti, Neo4j, embeddings, its LLM calls, or its graph schema.

Every semantic Graph element records two independent dimensions:

- processing history: `created_in_run`, `asserted_in_source_revision`, `invalidated_in_run`, `invalidated_in_source_revision`, and `supersedes`;
- authored-document scope when applicable: `source_valid_from_span` and `source_valid_to_span` for concept meanings or claims that change across the document.

Processing revision is never presented as an authored historical date. Active projections hide invalidated elements by default; Audit Mode can display their append-only history.

### 3.4 Argument-map semantics

Borrow the premise/conclusion and support/attack clarity of [Argdown](https://github.com/argdown/argdown), which is MIT licensed. Do not make Argdown syntax or packages canonical.

The Graph remains authoritative. Logic View projections map:

- `supports`, `has_premise`, and `has_evidence` to support paths;
- `refutes` and counterexamples to attack paths;
- objections and responses to explicit lanes;
- argument premises and conclusions to ordered argument steps.

An Argdown-compatible static export is post-MVP unless it can be validated by a pinned official parser without adding a second argument authority.

## 4. Components deliberately not adopted

- Lumenfolio and pi-tree: AGPL runtime/code is excluded. Their product ideas may be observed, but no code or schema is copied.
- `codex-book-to-skill`: license was not established and its source-reduction goal conflicts with full-text preservation.
- Graphiti and LightRAG runtimes: retrieval, embeddings, databases, and additional LLM calls do not improve canonical Source traceability enough to justify their complexity.
- Docling runtime: valuable design reference, but unnecessary additional models and vendor schema above the selected primitive Machine OCR role.
- Microsoft GraphRAG: retrieval-oriented and in maintenance mode; not a canonical graph dependency.
- MinerU: custom license adds avoidable uncertainty.
- Surya/Marker model stacks: code/model license combinations require additional review and duplicate the chosen machine-perception role.
- Readium/epub.js: optimized for EPUB publication rendering, while MVP renders repository-owned Reading Text and Graph views.
- automatic Cytoscape layouts: they can imply unsupported semantic relationships.

## 5. Runtime and lock boundary

The project has two pinned local runtimes:

- Node.js 22 for the CLI, schemas, merge, validation, projection, bundling, and browser code;
- Python 3.13.15 for the isolated RapidOCR machine-perception sidecar.

Required repository declarations are:

- `.nvmrc`, `package.json`, and `package-lock.json`;
- `.python-version`, `pyproject.toml`, and a fully resolved hash-locked Python dependency file;
- `models.lock.json` containing model name, source, license identifier, expected files, SHA-256, and local cache path;
- `THIRD_PARTY_NOTICES.md` covering directly shipped packages and model artifacts.

Model/setup download is a distinct installation step. Normal reading runs are offline and fail when locked artifacts are absent or mismatched.

The Node process starts Python without a shell, passes explicit argument arrays, requires schema-versioned JSON on stdout or a designated shard path, captures stderr as diagnostics, and treats nonzero exit or unexpected output as a stage failure. The sidecar receives only resolved paths inside the bundle and read-only Evidence paths.

The Node direct-version baseline is also fixed for the first implementation lock:

| Component | Version | License |
|---|---:|---|
| Node.js | `22.20.0` | MIT core plus the official release's bundled third-party notices |
| Ajv | `8.20.0` | MIT |
| Sharp | `0.35.3` | Apache-2.0 |
| esbuild | `0.28.2` | MIT |
| Cytoscape.js | `3.34.1` | MIT |
| `@playwright/test` | `1.62.1` | Apache-2.0 |

`package-lock.json` is the authority for all transitive and platform packages; the registry integrity for every direct/transitive package is retained. A version change requires rerunning contract, browser, license, and offline tests.

## 6. License compliance tests

License inventory scope is exact:

- every `package-lock.json.packages` entry, including optional platform packages actually bundled or installed;
- every Python distribution in the hash-locked requirements file and `importlib.metadata`, including native libraries and their bundled notices;
- Python itself and the Node runtime used to build/release;
- the RapidOCR outer wheel and its three locked inner model files;
- every file listed by the esbuild metafile for `atlas.html`;
- every copied font, icon, fixture, template, browser asset, and static export dependency;
- model/source URLs, hashes, and license/attribution documents in `models.lock.json`.

`license-policy.json` permits only `0BSD`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `MIT`, `PSF-2.0`, `Python-2.0`, `Unicode-3.0`, and `Zlib` for automatic acceptance. AGPL/GPL/SSPL/BUSL, unknown, missing, custom, or unparseable expressions fail. A new license requires an explicit reviewed policy/documentation change; it is never silently accepted.

Release validation must verify:

- every direct/transitive package appears in the applicable lockfile;
- the installed package inventory equals the lock inventory and the bundle metafile is a subset of it;
- every installed/embedded model matches `models.lock.json` and has a recorded license;
- `THIRD_PARTY_NOTICES.md` includes all distributed attribution text;
- no forbidden AGPL or license-unknown package enters dependency graphs or bundled assets;
- `atlas.html` contains no remote imports;
- archived vendor JSON records exact RapidOCR/ONNX Runtime/model/config versions.
