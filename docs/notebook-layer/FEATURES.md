# Notebook Layer — Feature List

**Status:** draft spec, not yet an approved epic
**Date:** 2026-11-19
**Derived from:** `docs/notebooklm-architecture-extraction.md`
**Target:** `apps/backend` + `apps/desktop` in this repo

---

## How to read this

- **IDs are stable.** Never renumber. Retire with `~~NB-xxx~~` and a reason.
- **P0** = v1 must ship. **P1** = next. **P2** = later / optional.
- **Mirrors** names the NotebookLM concept the feature derives from. Where we deliberately diverge, it says so.
- **Acceptance** is testable. If it cannot be asserted in a test, it is not acceptance criteria.
- Effort is engineer-days for one experienced dev, rough.

---

## Summary

| ID | Feature | P | Depends | Effort |
|---|---|---|---|---|
| DH-101 | Parser registry | P0 | — | 2 |
| DH-102 | Block model + offsets | P0 | DH-101 | 3 |
| DH-103 | `document_read` tool | P0 | DH-102 | 1 |
| DH-104 | Viewer rendering | P0 | DH-102 | 1.5 |
| DH-105 | Parse cache | P1 | DH-102 | 0.5 |
| DH-106 | Format tier 2 | P1 | DH-102 | 3 |
| NB-101 | Notebook CRUD | P0 | — | 1 |
| NB-102 | Add source: file upload | P0 | DH-102 | 2 |
| NB-103 | Add source: URL | P0 | DH-102 | 1 |
| NB-104 | Add source: pasted text | P0 | DH-102 | 0.5 |
| NB-105 | Source list + status | P0 | NB-102 | 1 |
| NB-106 | Source viewer | P0 | NB-105 | 1 |
| NB-107 | Source selection for chat | P0 | NB-105 | 1 |
| NB-108 | Source guide | P1 | NB-301 | 1 |
| NB-109 | Source freshness + refresh | P1 | NB-102 | 1.5 |
| NB-110 | Add source: YouTube | P1 | DH-102 | 1 |
| NB-111 | Add source: audio | P1 | DH-102 | 1.5 |
| NB-112 | Add source: image | P2 | DH-102 | 1.5 |
| NB-113 | Source labels | P2 | NB-105 | 2 |
| NB-202 | Chunker over blocks | P0 | DH-102 | 1 |
| NB-203 | Embedding | P0 | NB-202 | 1 |
| NB-204 | Hybrid index (FTS5 + vector) | P0 | NB-203 | 2 |
| NB-205 | Ingestion job status | P0 | DH-102 | 1 |
| NB-206 | Per-source limits | P0 | DH-102 | 0.5 |
| NB-207 | Dedup by content hash | P1 | DH-102 | 0.5 |
| NB-301 | Retrieve endpoint | P0 | NB-204 | 2 |
| NB-302 | Hybrid ranking (RRF) | P0 | NB-301 | 1.5 |
| NB-303 | Reranker | P1 | NB-302 | 2 |
| NB-304 | Source filter | P0 | NB-301 | 0.5 |
| NB-305 | Retrieval eval harness | P0 | NB-301 | 2 |
| NB-306 | Context assembly | P0 | NB-301 | 1 |
| NB-401 | Grounded chat (SSE) | P0 | NB-301 | 3 |
| NB-402 | Refusal when not in sources | P0 | NB-401 | 1 |
| NB-403 | Chat styles | P1 | NB-401 | 1 |
| NB-404 | Response length control | P1 | NB-401 | 0.5 |
| NB-405 | Conversation history | P0 | NB-401 | 1 |
| NB-406 | Cancel generation | P0 | NB-401 | 1 |
| NB-407 | Save answer to note | P1 | NB-501 | 1 |
| NB-408 | Notes as sources | P1 | NB-407 | 1 |
| NB-409 | Query origin instrumentation | P2 | NB-401 | 0.5 |
| NB-410 | Suggested prompts | P1 | NB-701 | 1 |
| NB-501 | AnswerDoc schema | P0 | — | 2 |
| NB-502 | Structural elements | P0 | NB-501 | 2 |
| NB-503 | Annotation map | P0 | NB-501 | 1 |
| NB-504 | Citation objects | P0 | NB-503 | 1.5 |
| NB-505 | Streaming doc events | P0 | NB-501 | 2 |
| NB-506 | AnswerDoc renderer | P0 | NB-502 | 3 |
| NB-507 | Hover-quote + click-to-context | P0 | NB-504 | 2 |
| NB-508 | Citation persistence | P0 | NB-504 | 1 |
| NB-601 | Artifact table + status | P0 | — | 1 |
| NB-602 | Artifact worker | P0 | NB-601 | 2 |
| NB-603 | Mind map artifact | P0 | NB-602 | 2 |
| NB-604 | Flashcards artifact | P1 | NB-602 | 1.5 |
| NB-605 | Quiz artifact | P1 | NB-602 | 1.5 |
| NB-606 | Report artifact | P1 | NB-602 | 2 |
| NB-607 | Slide deck artifact | P1 | NB-602 | 3 |
| NB-608 | Infographic artifact | P2 | NB-602 | 3 |
| NB-609 | Audio overview artifact | P2 | NB-602 | 6 |
| NB-610 | Customization choices | P1 | NB-601 | 1 |
| NB-611 | Outline review (PENDING_REVIEW) | P1 | NB-602 | 2 |
| NB-612 | Scheduled generation | P2 | NB-602 | 1.5 |
| NB-613 | Artifact export | P1 | NB-602 | 1.5 |
| NB-614 | Artifact cost accounting | P1 | NB-602 | 1 |
| NB-701 | Notebook guide | P1 | NB-301 | 1 |
| NB-702 | Next-step suggestions | P2 | NB-701 | 1 |
| NB-801 | Policy provider interface | P0 | — | 1 |
| NB-802 | Egress gate | P0 | NB-801 | 2 |
| NB-803 | Egress log | P0 | NB-802 | 1 |
| NB-804 | Local model path | P1 | NB-802 | 2 |
| NB-805 | Redaction | P2 | NB-802 | 2 |
| NB-806 | Artifact release gate | P1 | NB-601 | 1.5 |
| NB-807 | Provenance marking | P1 | NB-806 | 1 |
| NB-808 | Retention & hard delete | P1 | NB-101 | 1.5 |
| NB-901 | Per-notebook cost ceiling | P1 | NB-614 | 1 |
| NB-902 | Usage metering | P2 | NB-901 | 2 |
| NB-903 | Job retry + cancellation | P0 | NB-602 | 1 |
| NB-904 | Notebook observability | P1 | NB-401 | 1.5 |
| NB-1001 | Notebook tools | P0 | NB-301 | 2 |
| NB-1002 | `document-handling` skill | P0 | DH-103 | 0.5 |
| NB-1003 | `notebook-research` skill | P1 | NB-1001 | 0.5 |
| NB-1004 | MCP server | P2 | NB-1001 | 2 |
| NB-1005 | A2A agent | P2 | NB-1001 | 4 |

**P0 total: ~60.5 engineer-days.** P1 adds ~39, P2 ~25.5. The P0 number is the honest cost of a usable v1.

---

## 1. Notebook & sources

### NB-101 — Notebook CRUD · P0 · 1d
Create, rename, delete a notebook scoped to a workspace.
**Mirrors:** `Project`.
**Acceptance:** notebook survives app restart; deleting cascades sources, chunks, artifacts, and answer docs; a notebook cannot reference a workspace that does not exist.

### NB-102 — Add source: file upload · P0 · 2d
Upload PDF, DOCX, PPTX, MD, TXT, CSV.
**Mirrors:** `AddSources` + `UserContent`.
**Acceptance:** file is copied into app storage (never the workspace folder); `sha256` recorded; ingestion job created; source appears with `status=processing` then `ready` or `failed`.

### NB-103 — Add source: URL · P0 · 1d
Fetch a public URL, extract readable text.
**Acceptance:** redirects followed; non-HTML content rejected with a typed error; extracted text preserves heading structure where available.

### NB-104 — Add source: pasted text · P0 · 0.5d
Create a source from a textarea, with an editable title.
**Acceptance:** source behaves identically to an uploaded `.md` for retrieval and citation.

### NB-105 — Source list + status · P0 · 1d
Left panel listing sources with kind icon, title, word count, status.
**Mirrors:** source panel.
**Acceptance:** status transitions render without a full refresh; failed sources show a reason and a retry action.

### NB-106 — Source viewer · P0 · 1d
Open a source and read it in the right panel.
**Mirrors:** `LoadSource → PlainTextSourceContent { header, body, markdown_string }`.
**Acceptance:** viewer renders from the stored normalized text, never by re-parsing the original file; opening a source is O(1) in file size.

### NB-107 — Source selection for chat · P0 · 1d
Checkbox subset of sources; chat runs against the selection.
**Mirrors:** `InputSourceOptions.use_all_sources` + `SourceIdFilter`.
**Acceptance:** selection is a **retrieval filter**, not a prompt instruction; deselecting a source provably removes its chunks from the candidate set (assert on the retrieve call).

### NB-108 — Source guide · P1 · 1d
Auto-generated per-source summary + main ideas.
**Mirrors:** `GenerateDocumentGuides → DocumentGuide { snippet, main_ideas }`.
**Acceptance:** generated once per source revision, cached, invalidated on refresh.

### NB-109 — Source freshness + refresh · P1 · 1.5d
Detect that a URL/Drive source changed; re-ingest.
**Mirrors:** `CheckSourceFreshness` / `RefreshSource`.
**Acceptance:** refresh replaces chunks atomically; existing answer docs keep their original citation text (they are historical records, not live views).

### NB-110 — Add source: YouTube · P1 · 1d
Public video → transcript + metadata.
**Acceptance:** transcript stored with timestamps as spans; citation click seeks the player.

### NB-111 — Add source: audio · P1 · 1.5d
Upload audio → transcript.
**Acceptance:** transcript chunked with time offsets; source viewer shows transcript with timestamps.

### NB-112 — Add source: image · P2 · 1.5d
OCR + vision description.
**Acceptance:** extracted text is citable; the image itself is viewable in the source panel.

### NB-113 — Source labels · P2 · 2d
AI-generated or manual topic labels; filter sources by label.
**Mirrors:** source labels.

---

## 2. Document layer

Shared parser used by the agent tool, notebook ingestion, and the file viewer. Full contract in `DOCUMENT_LAYER.md`.

### DH-101 — Parser registry · P0 · 2d
Map `parser_kind` → parser implementation, registered at import time.
**Acceptance:** duplicate registration is a startup error; an unknown `parser_kind` raises `UnsupportedSourceKind`, never a 500.

### DH-102 — Block model + offsets · P0 · 3d
`ParsedDocument { text, blocks[], pages[], meta }` where `text[block.start:block.end] == block.text`.
**Acceptance:** a property test over a 20-document corpus asserts the offset invariant for every block; blocks are ordered and non-overlapping.

### DH-103 — `document_read` tool · P0 · 1d
pydantic-ai tool: path → markdown, sandboxed to the workspace root.
**Acceptance:** the agent answers a question about a PDF it has never seen; a path outside the root is rejected; truncation is explicit, never silent.

### DH-104 — Viewer rendering · P0 · 1.5d
Render `blocks` in the right panel for PDF/DOCX/PPTX.
**Acceptance:** opening a PDF shows readable text with page markers instead of "cannot display"; render time is independent of file size after parse.

### DH-105 — Parse cache · P1 · 0.5d
Content-addressed cache keyed by `sha256 + parser + parser_version`.
**Acceptance:** re-reading the same file at a different path is a cache hit; bumping `parser_version` is a miss.

### DH-106 — Format tier 2 · P1 · 3d
xlsx, audio transcript, YouTube.
**Acceptance:** each new format has a parser test and a `document_read` smoke test.

---

## 3. Ingestion

### NB-202 — Chunker over blocks · P0 · 1d
Turn `ParsedDocument.blocks` into chunks carrying `start_offset`, `end_offset`, `page`, `heading_path`.
**Mirrors:** `TextContentType` enum (via `DH-101`).
**Acceptance:** chunks break on block boundaries, never mid-block unless a single block exceeds the token ceiling; `text[start:end] == chunk.text` for every chunk; a property test over 100 documents asserts this.

### NB-203 — Embedding · P0 · 1d
Embed chunks; store vectors.
**Acceptance:** embedding model identity is recorded per chunk; changing the model marks the index stale rather than silently mixing spaces.

### NB-204 — Hybrid index · P0 · 2d
FTS5 + vector index over chunks.
**Acceptance:** both indexes are updated in the same transaction as the chunk insert; a crash mid-ingest leaves no orphan vectors.

### NB-205 — Ingestion job status · P0 · 1d
`queued → parsing → chunking → embedding → ready | failed`.
**Acceptance:** progress is visible in the UI; a failed job records the stage and the error.

### NB-206 — Per-source limits · P0 · 0.5d
Reject sources over the configured word/byte ceiling with a typed error.
**Mirrors:** 500k words / 200 MB.
**Acceptance:** limit is configurable; the error names the limit and the actual size.

### NB-207 — Dedup by content hash · P1 · 0.5d
Adding an identical file to the same notebook reuses the existing source.

---

## 4. Retrieval

### NB-301 — Retrieve endpoint · P0 · 2d
`POST /notebooks/{id}/retrieve` → ranked chunks with spans.
**Mirrors:** `RetrieveRelevantChunks`.
**Acceptance:** returns `SourceRelevantChunks[]`; every chunk carries at least one span; the endpoint is callable without invoking a model.

### NB-302 — Hybrid ranking · P0 · 1.5d
BM25 + vector, fused with reciprocal rank fusion.
**Acceptance:** a fixed 50-query benchmark reports recall@10; the number is committed to the repo and regressions fail CI.

### NB-303 — Reranker · P1 · 2d
Cross-encoder or LLM rerank over the fused top-N.
**Acceptance:** reranker is behind a flag; enabling it must not reduce recall@10 on the benchmark.

### NB-304 — Source filter · P0 · 0.5d
Restrict retrieval to a source ID set.
**Acceptance:** filter is applied inside the query, not after ranking (otherwise top-k is silently truncated).

### NB-305 — Retrieval eval harness · P0 · 2d
A committed query set with expected source/chunk hits.
**Acceptance:** `bun run eval:retrieval` prints recall@k and MRR; CI fails on regression beyond a set threshold.

### NB-306 — Context assembly · P0 · 1d
Turn retrieved chunks into a model context with stable ordering and token budget.
**Acceptance:** assembly is deterministic for a given retrieval result; budget overflow drops lowest-ranked chunks, never truncates mid-chunk.

---

## 5. Grounded chat

### NB-401 — Grounded chat · P0 · 3d
SSE endpoint that retrieves, then generates an `AnswerDoc`.
**Mirrors:** `GenerateFreeFormStreamed`.
**Acceptance:** the model receives only retrieved context; the response streams as `AnswerDoc` events (NB-505), never as raw markdown.

### NB-402 — Refusal when not in sources · P0 · 1d
If retrieval confidence is below threshold, answer "not in your sources" instead of guessing.
**Acceptance:** a 20-question adversarial set (answers deliberately absent) produces ≥18 refusals and 0 fabricated citations.

### NB-403 — Chat styles · P1 · 1d
Default / Learning Guide / Custom persona.
**Mirrors:** `Configure Chat`.
**Acceptance:** style changes the system prompt only; it cannot change retrieval or citation behaviour.

### NB-404 — Response length control · P1 · 0.5d
Short / default / long.

### NB-405 — Conversation history · P0 · 1d
Multi-turn within a notebook, persisted.
**Acceptance:** history is stored as `ModelMessage` rows via the existing `ConversationStore`; resuming a notebook restores the exact turn chain.

### NB-406 — Cancel generation · P0 · 1d
**Mirrors:** `CancelGeneration`.
**Acceptance:** cancelling mid-stream persists the partial `AnswerDoc` with `status=cancelled`; no orphan job remains.

### NB-407 — Save answer to note · P1 · 1d
Pin an answer to the noteboard, preserving structure and live citations.
**Acceptance:** the saved note renders identically to the original answer, including citation hover.

### NB-408 — Notes as sources · P1 · 1d
A note can be added to the retrieval corpus.
**Acceptance:** note chunks are indistinguishable from document chunks at retrieval time.

### NB-409 — Query origin instrumentation · P2 · 0.5d
Record which entry point produced a query.
**Mirrors:** `QueryOrigin`.
**Acceptance:** origin is stored per message; no PII in the origin value.

### NB-410 — Suggested prompts · P1 · 1d
**Mirrors:** `GeneratePromptSuggestions`.

---

## 6. Answer document

### NB-501 — AnswerDoc schema · P0 · 2d
Pydantic model + TS mirror in `packages/shared`.
**Acceptance:** schema is versioned; the renderer rejects unknown element kinds rather than crashing.

### NB-502 — Structural elements · P0 · 2d
`paragraph | table | code_block | thought | tool_call | tool_result`.
**Mirrors:** `StructuralElement`.
**Acceptance:** each element round-trips through JSON unchanged; `thought` and `tool_call` render inline, not in a side panel.

### NB-503 — Annotation map · P0 · 1d
Map from location key → `{ object_id, range }`.
**Mirrors:** `Body.inline_object_locations`.
**Acceptance:** every annotation resolves to an existing object; a dangling annotation is a hard error in dev, a dropped annotation in prod.

### NB-504 — Citation objects · P0 · 1.5d
`Citation { object_id, source_id, chunk_id, ranges[], quote }`.
**Mirrors:** `Citation { ranges, fragment, source_attribution }`.
**Acceptance:** `quote` is byte-identical to `source_text[range]`; asserted in a test over every generated citation.

### NB-505 — Streaming doc events · P0 · 2d
Emit `AnswerDoc` incrementally over the existing AI SDK stream as custom data parts.
**Acceptance:** a client that joins all events reconstructs a valid `AnswerDoc`; replaying a recorded stream produces a byte-identical document.

### NB-506 — AnswerDoc renderer · P0 · 3d
React renderer for all element kinds.
**Acceptance:** renders 10k-word documents without layout thrash; no `any` in the component props.

### NB-507 — Hover-quote + click-to-context · P0 · 2d
Hover a citation → show the exact quote; click → open the source at the span.
**Acceptance:** the highlighted span in the source viewer matches the citation range exactly, including after a source refresh (historical docs use the stored quote).

### NB-508 — Citation persistence · P0 · 1d
Write citations to a queryable table at generation time.
**Acceptance:** "which sources did this answer use" is a single SQL query; no reconstruction from text.

---

## 7. Artifacts

### NB-601 — Artifact table + status · P0 · 1d
`initialized | processing | ready | failed | pending_review | scheduled`.
**Mirrors:** `ArtifactStatus`.

### NB-602 — Artifact worker · P0 · 2d
Single asyncio worker consuming the artifact queue.
**Acceptance:** no artifact generation runs inside a request handler; worker restart resumes `processing` jobs or marks them `failed` with a reason.

### NB-603 — Mind map · P0 · 2d
Tree JSON → interactive renderer; click a node to ask about it.
**Acceptance:** node click seeds a chat query with the node's topic and the notebook's source filter.

### NB-604 — Flashcards · P1 · 1.5d
Difficulty, count, progress, CSV export.

### NB-605 — Quiz · P1 · 1.5d
Difficulty, hints, explanations, scoring.

### NB-606 — Report · P1 · 2d
Template registry → markdown/HTML.

### NB-607 — Slide deck · P1 · 3d
Outline → HTML deck → PDF export.

### NB-608 — Infographic · P2 · 3d
HTML/SVG render or image generation.

### NB-609 — Audio overview · P2 · 6d
Script with format presets → TTS → player with transcript sync.
**Note:** this is the highest effort-to-value ratio in the list. Buy the TTS; do not build it.

### NB-610 — Customization choices · P1 · 1d
Server-driven option lists per artifact kind.
**Mirrors:** `GetArtifactCustomizationChoices`.
**Acceptance:** adding a new format requires no client release.

### NB-611 — Outline review · P1 · 2d
Generation pauses at `pending_review` for human approval before the expensive render.
**Mirrors:** producer notes / outline review.

### NB-612 — Scheduled generation · P2 · 1.5d
Defer a job when over budget.

### NB-613 — Artifact export · P1 · 1.5d
Download as the native format (PDF, PNG, CSV, JSON, MP3).

### NB-614 — Artifact cost accounting · P1 · 1d
Record model, tokens, and estimated cost per artifact.

---

## 8. Notebook intelligence

### NB-701 — Notebook guide · P1 · 1d
Summary of the notebook + what it can do.
**Mirrors:** `GenerateNotebookGuide`.

### NB-702 — Next-step suggestions · P2 · 1d
**Mirrors:** `NextStepSuggestions`.

---

## 9. Trust tiers & egress

Full model in `DEPLOYMENT_PROFILE.md`. Three tiers (T0 local / T1 internal / T2 external) and an **injected** policy. The notebook layer does not define a data taxonomy.

### NB-801 — Policy provider interface · P0 · 1d
`EgressPolicy` protocol: `decide(EgressRequest) -> EgressDecision`, plus a default `TierCeilingPolicy` that allows T0/T1 and denies T2.
**Acceptance:** the policy is injected at composition time; swapping it requires no change to the gate or the chat path; the default denies T2.

### NB-802 — Egress gate · P0 · 2d
Every outbound model call passes through one function that calls the policy and records the answer.
**Acceptance:** a denying policy produces **zero** outbound calls — asserted by a test that fails on any HTTP client invocation; a policy that raises or returns `None` denies.

### NB-803 — Egress log · P0 · 1d
Record notebook, source IDs, chunk IDs, tier, model, purpose, decision, reason, and timestamp per egress.
**Acceptance:** the log is append-only and queryable per notebook; every decision, including denials, is present.

### NB-804 — Local model path · P1 · 2d
Ollama for T0, including the fallback when a decision denies T1/T2.

### NB-805 — Redaction · P2 · 2d
Strip identifiers after an allow, before T1/T2 egress.

### NB-806 — Artifact release gate · P1 · 1.5d
Optional: an artifact cannot be exported or shared until it passes review.
**Acceptance:** when enabled, `artifact_export` on an unreviewed artifact returns `ArtifactNotReleased`; the reviewer identity and timestamp are recorded. When disabled, export is unrestricted.

### NB-807 — Provenance marking · P1 · 1d
Optional: generated artifacts carry an embedded marker (AI-generated, source count, model, date).
**Acceptance:** when enabled, a downloaded PDF/PNG still carries the marker after leaving the app.

### NB-808 — Retention & hard delete · P1 · 1.5d
Configurable retention for sources and artifacts; hard delete removes text, chunks, vectors, and cached parses.
**Acceptance:** after hard delete, no chunk text is recoverable from the DB or the parse cache.

---

## 10. Operations

### NB-901 — Per-notebook cost ceiling · P1 · 1d
Hard stop when a notebook exceeds its budget.

### NB-902 — Usage metering · P2 · 2d
**Mirrors:** `MeteringService` / `QuotaService`.

### NB-903 — Job retry + cancellation · P0 · 1d
Bounded retries with backoff; cancellation is cooperative.

### NB-904 — Notebook observability · P1 · 1.5d
Per-stage latency, retrieval hit rate, refusal rate, artifact failure rate.

---

## Definition of done (all P0)

- [ ] Feature has a test that fails without it.
- [ ] No `any` in new TypeScript; `ruff check` and `ruff format --check` pass.
- [ ] New tables have a migration appended to `MIGRATIONS` (never edit a released one).
- [ ] New API routes are covered by the OpenAPI snapshot.
- [ ] `bun run verify` passes.
- [ ] If it touches retrieval or citations, the eval harness was run and the number recorded.

---

## 11. Agent surface

The notebook is reachable by the agent, not only by the UI. Full rationale in `DEPLOYMENT_PROFILE.md` §5.

### NB-1001 — Notebook tools · P0 · 2d
`document_read`, `document_list`, `document_extract_tables`, `notebook_create`, `notebook_add_source`, `notebook_list_sources`, `notebook_retrieve`, `notebook_ask`, `artifact_generate`, `artifact_get`.
**Acceptance:** every tool is prefixed so `ToolProfile.tool_prefixes` can allowlist a group; the UI and the agent call the same handlers; `load_capability` stays exempt from allowlists.

### NB-1002 — `document-handling` skill · P0 · 0.5d
Teaches when to read a file directly vs. create a notebook.
**Acceptance:** the skill loads via `load_capability`; a one-file question uses `document_read`; a many-file question creates a notebook.

### NB-1003 — `notebook-research` skill · P1 · 0.5d
Retrieval discipline: cite or refuse, prefer `notebook_retrieve` when the user wants to see the evidence.

### NB-1004 — MCP server · P2 · 2d
Expose the same handlers to external clients.
**Acceptance:** the MCP adapter calls the same functions; no logic is duplicated.

### NB-1005 — A2A agent · P2 · 4d
Capability card + task lifecycle wrapping `notebook_*`.
**Acceptance:** deferred until the notebook is stable; the agent wraps tools, it does not reimplement them.
