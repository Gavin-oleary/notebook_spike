# Notebook Layer — Code Bible

**Status:** draft engineering reference
**Date:** 2026-11-19
**Companion:** `FEATURES.md` (what to build), `../notebooklm-architecture-extraction.md` (why)

This is the document you read before touching notebook code, and the one you cite in review when someone proposes a shortcut. It is prescriptive on purpose.

---

## 0. Scope

The notebook layer adds source-grounded research to the existing chat app:

- a **notebook** owns sources, a chat, and artifacts
- **retrieval** is a service that returns ranked chunks with character spans
- **answers** are structured documents with citations, not markdown
- **artifacts** are async jobs that re-render the same sources into other formats

It does **not** add: enterprise ACLs, multi-tenant sync, or a company-wide knowledge base. Those are a different product (see `../notebooklm-replica-brief-and-plan.md` §6).

---

## 1. Non-negotiable invariants

These are the rules that are expensive to fix later. If a change violates one, the change is wrong.

### I-1 · Offsets are sacred

Every chunk carries `start_offset` and `end_offset` into the **normalized** source text.

```python
assert normalized_text[chunk.start_offset : chunk.end_offset] == chunk.text
```

This is a property test over a corpus, not a comment. If a parser cannot produce offsets, it does not ship.

### I-2 · Citations are written, never reconstructed

The model emits citation objects as part of its structured output. We store them. We never string-match an answer against the corpus afterwards to "find" citations.

Reconstruction fails on paraphrase, on reordering, and on any source refresh. It is the single most common way this feature gets built wrong.

### I-3 · Retrieval is a service

Chat calls `retrieve()`. Chat never touches the index, never builds SQL, never ranks.

This is what makes recall testable without a model, and what lets us swap the ranker without touching the chat path.

### I-4 · The wire format is `AnswerDoc`

Markdown is a **render target**, not the transport. The stream carries typed elements and annotation ranges.

If you find yourself streaming a string and planning to "add citations later", stop. That plan does not work.

### I-5 · Artifacts are jobs

No artifact generation inside a request handler. Every artifact is a row with a status, picked up by the worker.

A 30-second slide deck inside a POST handler is a timeout waiting to happen, and it cannot be cancelled, retried, or costed.

### I-6 · Egress fails closed

A policy that raises, returns `None`, or is missing is a **deny**. There is no allow-by-default path.

This is asserted by a test that fails on any HTTP client invocation when the policy denies — not by code review.

### I-7 · Two source type axes

`origin_kind` (where it came from) and `parser_kind` (what to do with it) are separate columns.

A PDF uploaded from Drive is `origin_kind=drive, parser_kind=pdf`. Collapsing these into one enum causes parser-selection bugs that only appear for one combination.

### I-8 · Source filter is applied inside the query

Filtering after ranking silently truncates top-k. If you ask for 12 chunks from 3 sources and filter afterwards, you may return 4.

### I-9 · Migrations are append-only

`MIGRATIONS` in `app/db/schema.py` is append-only. Never edit a released entry. This is already the repo rule; it applies doubly here because notebook tables will be migrated often.

### I-10 · Never write into workspace folders

Notebook storage lives in Electron `userData`. The workspace is the user's code, not our scratch space.

### I-11 · One parser, three consumers

`app/documents/` is shared by the agent tool, notebook ingestion, and the file viewer. It is not notebook-specific and must not live under `notebook/`.

The parser is designed for the **strictest** consumer. That means it always produces offsets and blocks, even though the agent tool and the viewer do not need them. Building the general handler first and retrofitting offsets later is the same mistake as I-1, one layer up.

Corollary: the parser is a **library**, not an MCP server. MCP is a process boundary; ingestion parses hundreds of documents in a batch and needs the parsed document in memory.

### I-12 · Policy is injected, not defined here

The notebook layer does not own a data taxonomy. It passes source/chunk identity and the destination tier to an injected `EgressPolicy` and records the decision.

Whatever classification, ACL, or handling-caveat system the deployment already has is the policy's business. The layer's job is to call it, fail closed, and produce a record.

Corollary: the gate never inspects content to decide. It passes identity, not text.

### I-13 · The notebook is an agent capability, not only a UI feature

Every notebook operation is a tool with a stable prefix (`document_*`, `notebook_*`, `artifact_*`) so `ToolProfile.tool_prefixes` can allowlist a group.

The UI and the agent call the **same handlers**. There is no second implementation for the agent path.

Read-only profiles get the read tools and none of the write tools — that falls out of the existing allowlist, with no new gating code.

---

## 2. Architecture

```
┌──────────────────────────── renderer ────────────────────────────┐
│  NotebookView                                                    │
│   ├─ SourcePanel      (list, status, selection)                  │
│   ├─ ChatPanel        (AnswerDoc renderer + citations)           │
│   └─ StudioPanel      (artifacts, status, export)                │
└───────────────┬──────────────────────────────────────────────────┘
                │ HTTP + SSE (existing loopback + bearer token)
┌───────────────▼──────────────────────────────────────────────────┐
│  FastAPI sidecar                                                 │
│                                                                  │
│  api/notebooks.py   api/sources.py   api/retrieve.py             │
│  api/notebook_chat.py                api/artifacts.py            │
│                                                                  │
│  documents/   parse → blocks (+ offsets)                         │
│  notebook/                                                       │
│    ingest/    chunk → embed → index                              │
│    retrieve/  hybrid rank → spans                                │
│    chat/      retrieve → generate AnswerDoc → stream             │
│    artifacts/ queue → worker → kind handlers                     │
│    policy/    egress gate + log                                  │
└───────────────┬──────────────────────────────────────────────────┘
                │
        ┌───────▼────────┐
        │  app.db        │  notebooks, sources, chunks, vectors,
        │  (sqlite)      │  answer_docs, citations, artifacts
        └────────────────┘
```

**Boundary rule:** the renderer never reads files, never queries the index, and never assembles context. It renders what the backend sends.

---

## 3. Module layout

```
apps/backend/app/
  api/
    notebooks.py          # CRUD
    sources.py            # add/list/delete/refresh
    retrieve.py           # POST /notebooks/{id}/retrieve
    notebook_chat.py      # SSE chat
    artifacts.py          # create/list/get/cancel/export
  documents/              # shared parser — NOT notebook-specific
    __init__.py
    models.py             # ParsedDocument, Block, PageInfo, DocumentMeta
    registry.py           # parser_kind -> parser
    cache.py              # content-addressed parse cache
    render.py             # blocks -> markdown (agent tool, viewer)
    parsers/
      pdf.py docx.py pptx.py markdown.py html.py text.py
      xlsx.py audio.py image.py youtube.py
  notebook/
    __init__.py
    models.py             # Pydantic contracts (this document's §5)
    store.py              # sqlite access, no business logic
    ingest/
      chunker.py          # blocks -> chunks with offsets
      embedder.py
      index.py
    retrieve/
      hybrid.py           # FTS5 + vector + RRF
      rerank.py
      assemble.py         # context assembly + token budget
    chat/
      runner.py           # pydantic-ai agent + doc emitter
      prompts.py
      refusal.py
    artifacts/
      worker.py
      kinds/
        mind_map.py flashcards.py quiz.py report.py
        slides.py infographic.py audio.py
    policy/
      egress.py
      log.py
```

```
apps/desktop/src/renderer/src/features/notebook/
  NotebookView.tsx
  SourcePanel.tsx
  ChatPanel.tsx
  StudioPanel.tsx
  answer/
    AnswerDoc.tsx
    Element.tsx
    CitationMark.tsx
    CitationPopover.tsx
  artifacts/
    MindMap.tsx
    ArtifactCard.tsx
```

```
packages/shared/src/notebook.ts   # TS mirror of models.py
```

**Rule:** `models.py` and `notebook.ts` are generated from one source of truth or reviewed together in the same PR. They never drift.

---

## 4. Data model

```sql
-- Notebook container
CREATE TABLE notebooks (
    id            TEXT PRIMARY KEY,
    workspace_id  TEXT NOT NULL REFERENCES workspaces (id) ON DELETE CASCADE,
    title         TEXT NOT NULL,
    created_at    INTEGER NOT NULL,
    updated_at    INTEGER NOT NULL
);

-- A source is a snapshot, not a live view
CREATE TABLE sources (
    id            TEXT PRIMARY KEY,
    notebook_id   TEXT NOT NULL REFERENCES notebooks (id) ON DELETE CASCADE,
    origin_kind   TEXT NOT NULL,   -- upload | url | drive | youtube | note
    parser_kind   TEXT NOT NULL,   -- pdf | docx | pptx | markdown | html | text | audio | image
    uri           TEXT,
    title         TEXT NOT NULL,
    sha256        TEXT NOT NULL,
    bytes         INTEGER NOT NULL,
    words         INTEGER NOT NULL,
    status        TEXT NOT NULL,   -- queued | parsing | chunking | embedding | ready | failed
    error         TEXT,
    added_at      INTEGER NOT NULL,
    synced_at     INTEGER
);

-- Normalized text is stored once; chunks reference it by offset
CREATE TABLE source_texts (
    source_id     TEXT PRIMARY KEY REFERENCES sources (id) ON DELETE CASCADE,
    text          TEXT NOT NULL
);

CREATE TABLE source_chunks (
    id            TEXT PRIMARY KEY,
    source_id     TEXT NOT NULL REFERENCES sources (id) ON DELETE CASCADE,
    ordinal       INTEGER NOT NULL,
    text          TEXT NOT NULL,
    start_offset  INTEGER NOT NULL,
    end_offset    INTEGER NOT NULL,
    page          INTEGER,
    heading_path  TEXT,
    token_count   INTEGER NOT NULL,
    embedding_model TEXT NOT NULL
);

CREATE VIRTUAL TABLE chunk_fts USING fts5 (text, content='source_chunks', content_rowid='rowid');
CREATE VIRTUAL TABLE chunk_vec USING vec0 (chunk_id TEXT PRIMARY KEY, embedding FLOAT[1024]);

-- Answers are documents, not strings
CREATE TABLE answer_docs (
    id            TEXT PRIMARY KEY,
    notebook_id   TEXT NOT NULL REFERENCES notebooks (id) ON DELETE CASCADE,
    message_id    TEXT NOT NULL,
    status        TEXT NOT NULL,   -- streaming | finished | cancelled | failed
    body_json     TEXT NOT NULL,   -- Element[]
    annotations_json TEXT NOT NULL,
    created_at    INTEGER NOT NULL
);

-- Queryable projection of citations
CREATE TABLE citations (
    id            TEXT PRIMARY KEY,
    answer_doc_id TEXT NOT NULL REFERENCES answer_docs (id) ON DELETE CASCADE,
    object_id     TEXT NOT NULL,
    source_id     TEXT NOT NULL,
    chunk_id      TEXT NOT NULL,
    start_offset  INTEGER NOT NULL,
    end_offset    INTEGER NOT NULL,
    quote         TEXT NOT NULL
);

-- Artifacts are jobs
CREATE TABLE artifacts (
    id            TEXT PRIMARY KEY,
    notebook_id   TEXT NOT NULL REFERENCES notebooks (id) ON DELETE CASCADE,
    kind          TEXT NOT NULL,
    status        TEXT NOT NULL,
    options_json  TEXT NOT NULL,
    output_path   TEXT,
    error         TEXT,
    model         TEXT,
    cost_json     TEXT,
    created_at    INTEGER NOT NULL,
    started_at    INTEGER,
    completed_at  INTEGER
);

-- Append-only egress audit
CREATE TABLE egress_log (
    id            TEXT PRIMARY KEY,
    notebook_id   TEXT NOT NULL,
    model         TEXT NOT NULL,
    tier          TEXT NOT NULL,
    purpose       TEXT NOT NULL,
    chunk_ids_json TEXT NOT NULL,
    decision      TEXT NOT NULL,
    reason        TEXT NOT NULL,
    created_at    INTEGER NOT NULL
);
```

**No policy columns.** There is no `classification` on `sources` and no `policy` on `notebooks`. The policy is injected at runtime and resolves whatever it needs from its own store. Storing a taxonomy here would make the notebook layer the author of a policy it does not own.

**Why `source_texts` is separate:** the viewer needs the full text, retrieval needs chunks, and citations need offsets into the full text. Storing the text once and referencing it by offset keeps all three consistent.

---

## 5. Contracts

### 5.1 Retrieval

```python
# apps/backend/app/notebook/models.py
from enum import StrEnum

from pydantic import BaseModel, Field


class RetrieveMode(StrEnum):
    FAST = "fast"
    DEEP = "deep"


class ChunkSpan(BaseModel):
    start: int
    end: int


class RetrievedChunk(BaseModel):
    chunk_id: str
    source_id: str
    rank: int
    score: float
    spans: list[ChunkSpan]
    text: str


class SourceChunks(BaseModel):
    source_id: str
    chunks: list[RetrievedChunk]


class RetrieveRequest(BaseModel):
    query: str = Field(min_length=1, max_length=4000)
    source_ids: list[str] | None = None
    top_k: int = Field(default=12, ge=1, le=50)
    mode: RetrieveMode = RetrieveMode.FAST


class RetrieveResponse(BaseModel):
    sources: list[SourceChunks]
```

Rules:

- `source_ids=None` means all sources in the notebook.
- `source_ids=[]` is a **client bug** — reject with 422, do not silently return everything.
- `spans` is non-empty for every chunk. A chunk with no span cannot be cited.
- `rank` is 0-based and dense within a source.

### 5.2 AnswerDoc

```python
from typing import Annotated, Any, Literal, Union

from pydantic import BaseModel, Field


class Range(BaseModel):
    start: int
    end: int


class TextStyle(BaseModel):
    bold: bool = False
    italic: bool = False
    code: bool = False
    link: str | None = None


class TextRun(BaseModel):
    kind: Literal["text"] = "text"
    text: str
    style: TextStyle = Field(default_factory=TextStyle)


class Paragraph(BaseModel):
    kind: Literal["paragraph"] = "paragraph"
    id: str
    runs: list[TextRun]


class Table(BaseModel):
    kind: Literal["table"] = "table"
    id: str
    header: list[str]
    rows: list[list[str]]


class CodeBlock(BaseModel):
    kind: Literal["code_block"] = "code_block"
    id: str
    language: str | None = None
    code: str


class Thought(BaseModel):
    kind: Literal["thought"] = "thought"
    id: str
    text: str


class ToolCall(BaseModel):
    kind: Literal["tool_call"] = "tool_call"
    id: str
    call_id: str
    name: str
    args: dict[str, Any]


class ToolResult(BaseModel):
    kind: Literal["tool_result"] = "tool_result"
    id: str
    call_id: str
    ok: bool
    summary: str


Element = Annotated[
    Union[Paragraph, Table, CodeBlock, Thought, ToolCall, ToolResult],
    Field(discriminator="kind"),
]


class Citation(BaseModel):
    object_id: str
    source_id: str
    chunk_id: str
    ranges: list[Range]
    quote: str


class AnnotationRef(BaseModel):
    object_id: str
    range: Range


class AnswerDoc(BaseModel):
    schema_version: Literal[1] = 1
    doc_id: str
    elements: list[Element]
    annotations: dict[str, AnnotationRef]
    objects: dict[str, Citation]
```

**How the three pieces fit:**

- `elements` is the readable document.
- `annotations` maps a location key (e.g. `"p3:run1"`) to an object ID and a range **within that element's text**.
- `objects` maps object ID → `Citation`, which carries ranges into the **source**.

So a citation is a two-hop reference: element → annotation → citation → source span. That indirection is what lets one citation be referenced from several places without duplicating the quote.

**Validation rules (enforce in a model validator):**

1. Every `AnnotationRef.object_id` exists in `objects`.
2. Every `Citation.chunk_id` exists in `source_chunks`.
3. `quote == source_text[start:end]` for at least one range.
4. Element `id`s are unique within a document.

Rule 3 is the one that catches real bugs. Run it in dev on every generated document; in prod, log and drop the citation rather than failing the answer.

### 5.3 Streaming events

The stream rides the existing AI SDK UI message stream as a custom data part named `doc`.

```python
class DocElementEvent(BaseModel):
    type: Literal["element"] = "element"
    element: Element


class DocAnnotationEvent(BaseModel):
    type: Literal["annotation"] = "annotation"
    key: str
    ref: AnnotationRef


class DocCitationEvent(BaseModel):
    type: Literal["citation"] = "citation"
    citation: Citation


class DocStatusEvent(BaseModel):
    type: Literal["status"] = "status"
    status: Literal["started", "finished", "cancelled", "failed"]
    detail: str | None = None


DocEvent = Annotated[
    Union[DocElementEvent, DocAnnotationEvent, DocCitationEvent, DocStatusEvent],
    Field(discriminator="type"),
]
```

Wire shape: `data-doc` part with `{ "event": DocEvent }`.

**Ordering guarantees:**

- `status:started` is first.
- `element` events are append-only and ordered.
- An `annotation` may arrive before or after its `element`; the renderer buffers.
- A `citation` may arrive before its `annotation`; the renderer buffers.
- Exactly one terminal `status` event.

**Replay:** joining all events must reconstruct a valid `AnswerDoc`. This is a test, not an aspiration.

### 5.4 Artifacts

```python
class ArtifactKind(StrEnum):
    MIND_MAP = "mind_map"
    FLASHCARDS = "flashcards"
    QUIZ = "quiz"
    REPORT = "report"
    SLIDES = "slides"
    INFOGRAPHIC = "infographic"
    AUDIO_OVERVIEW = "audio_overview"


class ArtifactStatus(StrEnum):
    INITIALIZED = "initialized"
    PROCESSING = "processing"
    READY = "ready"
    FAILED = "failed"
    PENDING_REVIEW = "pending_review"
    SCHEDULED = "scheduled"
```

**State machine:**

```
INITIALIZED ──► PROCESSING ──► READY
     │              │
     │              ├──► FAILED        (terminal, retryable)
     │              └──► PENDING_REVIEW ──► PROCESSING ──► READY
     └──► SCHEDULED ──► PROCESSING
```

Rules:

- `FAILED` is terminal for a given attempt; retry creates a new attempt, it does not mutate the old row.
- `PENDING_REVIEW` only exists for kinds that support an outline step (slides, video, report).
- `SCHEDULED` requires a `scheduled_for` timestamp in `options_json`.
- The worker claims jobs with `UPDATE ... WHERE status='initialized' RETURNING *` — never a read-then-write.

---

## 6. Ingestion rules

1. **Parse once, store normalized text.** The viewer, the chunker, and the citation resolver all read `source_texts.text`. Nothing re-parses the original file.
2. **Chunk on block boundaries.** `ParsedDocument.blocks` from `app/documents/` are the chunking unit. Fall back to a token window only when a single block exceeds the ceiling.
3. **Overlap is for retrieval, not for citation.** A chunk may overlap its neighbour; a citation range is always within one chunk.
4. **Record the embedding model per chunk.** Mixing embedding spaces is silent and catastrophic.
5. **Index in the same transaction as the chunk insert.** A crash must not leave a chunk without its vector.
6. **Parser registration is a document-layer concern.** See `DOCUMENT_LAYER.md` §4 — an unknown `parser_kind` is a startup error, not a runtime 500.
7. **Limits are checked before parsing.** Rejecting a 2 GB PDF after parsing it is a denial-of-service vector.

---

## 7. Chat rules

1. **Retrieve, then generate.** The model never sees the corpus, only the assembled context.
2. **Refusal is a feature.** If the top score is below threshold, return a refusal with `empty_answer_reason`. Do not let the model improvise.
3. **The model emits structured output**, not prose. Use `pydantic-ai` structured output with `AnswerDoc` as the result type, or a tool that appends elements.
4. **Citations come from the model's structured output**, validated against the retrieved chunks. A citation to a chunk that was not retrieved is a bug — drop it and log.
5. **Style and length affect the prompt only.** They must not affect retrieval or citation behaviour.
6. **Cancellation is cooperative.** Check the cancellation token between elements, not between tokens.

---

## 8. Artifact rules

1. **One worker, one queue.** No Celery, no Redis. An asyncio task in the sidecar is enough at this scale.
2. **Every kind is a pure function** `(sources, options) -> output`. It does not read the DB directly; it receives its inputs.
3. **Options are validated before enqueue.** A bad `aspect_ratio` should fail at the API, not 40 seconds into a render.
4. **Cost is recorded per artifact.** Model, tokens, estimated USD. Without this, the first enthusiastic user is a budget incident.
5. **Outputs go to app storage**, never the workspace.
6. **Export is a separate step** from generation. Generating a deck and downloading it are different operations with different failure modes.

---

## 9. Policy & egress

Full model in `DEPLOYMENT_PROFILE.md`. Summary:

```python
class TrustTier(StrEnum):
    T0_LOCAL = "t0_local"        # Ollama, on-device
    T1_INTERNAL = "t1_internal"  # internal Azure API
    T2_EXTERNAL = "t2_external"  # public provider
```

The tiers are a fact about the deployment. The **policy** is injected — the notebook layer does not define a data taxonomy.

```python
class EgressRequest(BaseModel):
    notebook_id: str
    source_ids: list[str]
    chunk_ids: list[str]
    tier: TrustTier
    model: str
    purpose: Literal["chat", "artifact", "embedding"]


class EgressDecision(BaseModel):
    allowed: bool
    reason: str
    obligations: list[str] = Field(default_factory=list)


class EgressPolicy(Protocol):
    async def decide(self, request: EgressRequest) -> EgressDecision: ...


async def decide_egress(
    *,
    policy: EgressPolicy,
    request: EgressRequest,
) -> EgressDecision:
    """The only place that decides whether data may leave the device."""
```

Rules:

1. **One gate.** Every outbound model call goes through `decide_egress`. There is no second path.
2. **Fail closed.** A policy that raises, returns `None`, or is missing is a deny.
3. **The gate never inspects content.** It passes identity (source IDs, chunk IDs, tier) and records the answer.
4. **Log every decision**, including denials.
5. **A denial is a normal outcome**, not an exception. The chat path falls back to T0.

**The test that matters:**

```python
async def test_denied_policy_makes_no_outbound_calls(monkeypatch):
    calls = []
    monkeypatch.setattr(httpx.AsyncClient, "send", lambda *a, **k: calls.append(a))
    await run_chat(notebook=nb, policy=DenyAllPolicy(), tier=TrustTier.T1_INTERNAL)
    assert calls == []
```

---

## 10. Error taxonomy

Use typed errors, not bare `HTTPException` with a string.

| Error | HTTP | Meaning |
|---|---|---|
| `NotebookNotFound` | 404 | |
| `SourceNotFound` | 404 | |
| `SourceTooLarge` | 413 | names the limit and the actual size |
| `UnsupportedSourceKind` | 415 | `parser_kind` has no parser |
| `SourceNotReady` | 409 | chat against a source still ingesting |
| `EmptySourceSelection` | 422 | `source_ids=[]` |
| `RetrievalUnavailable` | 503 | index missing or stale |
| `PolicyDenied` | 403 | egress gate refused |
| `ArtifactKindUnsupported` | 415 | |
| `ArtifactBudgetExceeded` | 402 | cost ceiling hit |
| `ArtifactNotReady` | 409 | export before `READY` |

Every error carries a machine-readable `code` and a human `message`. The renderer switches on `code`, never on `message`.

---

## 11. Naming

- **Tables:** plural snake_case (`notebooks`, `source_chunks`).
- **Columns:** snake_case; timestamps end in `_at` and are Unix milliseconds.
- **Enums:** `StrEnum`, values snake_case, stored as text.
- **Pydantic models:** PascalCase, no `Model` suffix.
- **TS types:** mirror the Python names exactly. `AnswerDoc` is `AnswerDoc` on both sides.
- **Event types:** `doc.element`, `doc.annotation`, `doc.citation`, `doc.status` — dotted, lowercase.
- **Artifact kinds:** snake_case, matching the enum value.

Do not invent a second name for a concept that already has one. `AnswerDoc` is not "response", "message body", or "rich text".

---

## 12. Testing

### Must exist before the feature is "done"

| Test | Asserts |
|---|---|
| `test_chunk_offsets_roundtrip` | `normalized[start:end] == chunk.text` over a corpus |
| `test_citation_quote_matches_source` | every generated citation's quote equals the source slice |
| `test_refusal_on_absent_answer` | 20 adversarial questions → ≥18 refusals, 0 fabricated citations |
| `test_denied_policy_no_egress` | zero T1/T2 calls |
| `test_source_filter_applied_in_query` | deselected source contributes no chunks |
| `test_stream_replay_reconstructs_doc` | joined events == original `AnswerDoc` |
| `test_artifact_state_machine` | illegal transitions rejected |
| `test_worker_resumes_processing_jobs` | restart does not strand jobs |
| `test_retrieval_recall_benchmark` | recall@10 ≥ committed baseline |

### Retrieval benchmark

Commit a `retrieval_bench.json` with 50 queries and expected source/chunk hits. `bun run eval:retrieval` prints recall@k and MRR. CI fails on regression beyond a set threshold.

This is the only way to know whether a chunking or ranking change helped. Without it, every retrieval change is a vibe.

### What not to test

- Exact model output. It is non-deterministic; test the invariants instead.
- Prompt strings. They change constantly and the test becomes a change-detector.

---

## 13. Performance budgets

| Operation | Budget | Notes |
|---|---|---|
| Retrieve (fast) | < 150 ms p50, < 400 ms p95 | local index |
| Retrieve (deep) | < 1.5 s p95 | includes rerank |
| First streamed element | < 2.5 s p95 | retrieval + first model token |
| Source viewer open | < 100 ms | reads stored text, no parsing |
| Ingestion, 100-page PDF | < 20 s | background |
| Mind map artifact | < 30 s | |
| Slide deck artifact | < 3 min | |
| Audio overview | < 10 min | background, cancellable |

If a budget is exceeded, the fix is architectural, not a bigger timeout.

---

## 14. Anti-patterns

Each of these has a specific failure mode. They are listed because they are all tempting.

| Anti-pattern | Why it fails |
|---|---|
| Stream markdown, add citations later | Citation ranges cannot be recovered from rendered text. Rewrite required. |
| Store citations as markdown links | Loses the source span; hover-quote becomes impossible. |
| Put retrieval inside the chat handler | Recall becomes untestable; ranker swaps touch chat code. |
| Generate artifacts synchronously | Timeouts, no cancellation, no retry, no cost accounting. |
| One source type enum | Parser selection breaks for one origin/format combination. |
| Filter sources after top-k | Silent truncation; users see "no results" with sources selected. |
| Re-parse the original file to render the viewer | Slow, and the viewer can disagree with what was indexed. |
| Use the workspace folder for storage | Violates the repo rule; pollutes the user's repo. |
| Mix embedding spaces | Retrieval quality degrades silently and unevenly. |
| Silently downgrade policy | A source the policy would deny that quietly reaches T1 is a data incident. |
| Reconstruct citations by string matching | Fails on paraphrase and on any source refresh. |
| One giant `notebook.py` | The layer has five distinct concerns; keep them in separate modules. |

---

## 15. Decision log

**ADR-001 · Retrieval is a service**
*Context:* NotebookLM exposes `RetrieveRelevantChunks` as its own RPC.
*Decision:* Mirror it. `POST /notebooks/{id}/retrieve`.
*Consequence:* Recall is testable without a model; the ranker is swappable.

**ADR-002 · `AnswerDoc` is the wire format**
*Context:* NotebookLM streams a `TailwindDoc` with an annotation map, not markdown.
*Decision:* Adopt a typed document model with annotation ranges.
*Consequence:* More work up front; citations, inline thoughts, and inline tool calls all become possible.

**ADR-003 · Citations at generation time**
*Context:* Post-hoc matching is the common shortcut.
*Decision:* The model emits citation objects; we validate and store them.
*Consequence:* Requires structured output and a validation step; eliminates a whole class of bugs.

**ADR-004 · Artifacts are jobs**
*Context:* NotebookLM has `ArtifactStatus` with `PENDING_REVIEW` and `SCHEDULED`.
*Decision:* One table, one asyncio worker, explicit state machine.
*Consequence:* Cancellation, retry, and cost accounting are free.

**ADR-005 · Two source type axes**
*Context:* NotebookLM has both `OriginalSourceContentType` and `TextContentType`.
*Decision:* `origin_kind` + `parser_kind`.
*Consequence:* Slightly more schema; correct parser selection for every combination.

**ADR-006 · Egress at the retrieval boundary**
*Context:* Naive hybrid sends whole documents to the cloud or degrades to a local model.
*Decision:* Retrieve locally, send only retrieved spans, gate per chunk.
*Consequence:* Frontier quality on sensitive corpora; requires a real gate and an audit log.

**ADR-007 · No `batchexecute`-style envelope**
*Context:* Google's envelope exists for their frontend architecture.
*Decision:* Plain JSON + SSE over the existing loopback transport.
*Consequence:* Nothing to reverse-engineer, nothing to break.

**ADR-008 · Custom data parts, not a second SSE channel**
*Context:* The app already streams via the AI SDK UI message stream.
*Decision:* Emit `data-doc` parts on the existing stream.
*Consequence:* One connection, one cancellation path, one replay mechanism.

**ADR-009 · Shared parser, separate pipelines**
*Context:* The app has no document handling, and the notebook layer needs parsing. Building both separately duplicates the expensive part.
*Decision:* One `app/documents/` library with a `ParsedDocument { text, blocks, pages, meta }` contract, consumed by three thin callers: the `document_read` agent tool, notebook ingestion, and the file viewer.
*Consequence:* ~4–6 days saved and the offset retrofit avoided. The parser must satisfy the strictest consumer (offsets + blocks) even though two of three callers ignore them. See `DOCUMENT_LAYER.md`.

---

## 16. Migration from the current app

The notebook layer is additive. Nothing in the existing chat path changes.

**Step 0 — document layer (7d).** `app/documents/` with the block model, PDF + DOCX + MD parsers, the `document_read` agent tool, and viewer rendering. This ships on its own: the assistant can finally read a PDF. No notebook tables, no retrieval.

**Step 1 — schema only (1d).** Append notebook tables to `MIGRATIONS`. No code reads them yet. Ship it.

**Step 2 — ingestion (3d).** `POST /notebooks/{id}/sources`, chunker over blocks, embedder, index, source viewer. No retrieval, no chat.

**Step 3 — retrieval (3d).** `POST /notebooks/{id}/retrieve` + the eval harness. Still no chat. Now recall is measurable.

**Step 4 — chat + AnswerDoc (6d).** The first end-to-end slice. This is where the document model earns its cost.

**Step 5 — artifacts (4d).** Mind map first — highest value per unit effort.

**Step 6 — policy (3d).** Egress gate + log. Do this before any sensitive corpus is loaded.

**Sequencing rule:** do not start Step 4 until Step 3 has a committed recall number. Otherwise you cannot tell whether a bad answer is a retrieval problem or a generation problem, and you will spend weeks tuning prompts to fix a chunking bug.

**Why Step 0 comes first:** it is the only step that is independently shippable, it closes a real capability gap today, and it produces the block model that Step 2 chunks on. Building Step 2 first would mean writing a parser under notebook pressure and then discovering the agent tool needs a different shape.

---

## Appendix · Reference

- `docs/notebooklm-architecture-extraction.md` — the extracted contract this derives from
- `docs/notebooklm-replica-brief-and-plan.md` — product framing and use-case fit
- `docs/notebook-layer/FEATURES.md` — the build list
- `plans/app-vision.md` — locked architecture decisions for the host app
