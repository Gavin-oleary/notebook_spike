# Document Layer — shared parser

**Status:** draft spec
**Date:** 2026-11-19
**Companion:** `FEATURES.md`, `CODE_BIBLE.md`

The desktop assistant currently has no document handling. `.pdf`, `.docx`, `.pptx`, `.xlsx` are listed as non-renderable in `apps/backend/app/api/files.py`, so the agent cannot read them at all.

The notebook layer needs the same parsing. This document defines the shared layer so we build it once.

---

## 1. The decision

**One parser layer, three consumers.** Not one pipeline.

| | Agent tool | Notebook ingestion | File viewer |
|---|---|---|---|
| Trigger | user asks about a file | user adds sources | user opens a file |
| Lifetime | one turn | persistent | one view |
| Input | path | path, then many | path |
| Output | markdown for the model | chunks + vectors + spans | rendered blocks |
| Needs offsets | no | **yes** | no |
| Needs index | no | yes | no |
| Needs citations | no | yes | no |

They share the parse. They share nothing after it.

**Why not merge the pipelines:** an ad-hoc "read this PDF" must not embed and index a single file, and notebook ingestion must not be built on a parser that discards offsets. The parser is the only thing that has to satisfy both, and it must satisfy the stricter one.

---

## 2. The contract

```python
# apps/backend/app/documents/models.py
from typing import Any, Literal

from pydantic import BaseModel, Field


class Block(BaseModel):
    kind: Literal[
        "heading", "paragraph", "list", "table",
        "code", "image", "caption", "footnote",
    ]
    text: str
    start: int                      # offset into ParsedDocument.text
    end: int
    page: int | None = None
    level: int | None = None        # heading level, 1-6
    bbox: tuple[float, float, float, float] | None = None


class PageInfo(BaseModel):
    index: int
    width: float | None = None
    height: float | None = None


class DocumentMeta(BaseModel):
    parser: str                     # "pymupdf" | "python-docx" | ...
    parser_version: str
    source_sha256: str
    page_count: int | None = None
    word_count: int
    language: str | None = None
    warnings: list[str] = Field(default_factory=list)


class ParsedDocument(BaseModel):
    text: str                       # normalized, canonical
    blocks: list[Block]
    pages: list[PageInfo]
    meta: DocumentMeta
```

### Invariants

1. **`text[block.start:block.end] == block.text`** for every block. Property test over a corpus.
2. **Blocks are ordered and non-overlapping.** Gaps are allowed (whitespace, page breaks); overlaps are not.
3. **`text` is the canonical string.** The viewer, the chunker, and the citation resolver all read it. Nothing re-parses the original file.
4. **`meta.parser` + `meta.parser_version` are recorded.** A parser upgrade invalidates cached parses.

---

## 3. The three consumers

### 3.1 Agent tool — `document_read`

```python
async def document_read(
    ctx: RunContext[AppDeps],
    path: str,
    pages: str | None = None,       # "1-5", "3", None = all
) -> str:
    """Read a document and return it as markdown."""
```

- Resolves `path` through the existing workspace sandbox. Never escapes the root.
- Parses (or reads from cache), renders blocks to markdown.
- Truncates to a token budget with an explicit `[truncated: pages 6-40 omitted]` marker.
- Returns markdown, not `ParsedDocument` — the model does not need bboxes.

This is the capability gap that exists today. It is also the smallest possible slice: one tool, one parser, no schema changes.

### 3.2 Notebook ingestion

- `ParsedDocument.blocks` are the chunk boundaries. The chunker never splits mid-block unless a block exceeds the token ceiling.
- `block.start` / `block.end` become `chunk.start_offset` / `chunk.end_offset` directly.
- `block.page` becomes `chunk.page`; the heading stack becomes `chunk.heading_path`.

This is why the block model matters: it turns "chunk on structure" from an aspiration into a loop.

**The parser records provenance, not policy.** `meta.parser` and `meta.parser_version` are recorded so an ingest is reproducible and auditable. Whether a source may be sent to a given tier is decided by the injected `EgressPolicy` (see `DEPLOYMENT_PROFILE.md` §3) — not by anything the parser produces.

### 3.3 File viewer

- Renders `blocks` as readable text with page markers.
- Replaces the current "this file type cannot be displayed" path for PDF/DOCX/PPTX.
- Read-only, same as today.

---

## 4. Parser registry

```python
# apps/backend/app/documents/registry.py
Parser = Callable[[Path], ParsedDocument]

_REGISTRY: dict[str, Parser] = {}


def register(parser_kind: str) -> Callable[[Parser], Parser]:
    def deco(fn: Parser) -> Parser:
        if parser_kind in _REGISTRY:
            raise RuntimeError(f"duplicate parser for {parser_kind!r}")
        _REGISTRY[parser_kind] = fn
        return fn
    return deco


def get(parser_kind: str) -> Parser:
    try:
        return _REGISTRY[parser_kind]
    except KeyError:
        raise UnsupportedSourceKind(parser_kind) from None
```

Rules:

- **Registration is import-time.** An unknown `parser_kind` is a startup error, not a runtime 500.
- **`parser_kind` is derived from content, not extension.** A `.pdf` that is actually HTML should be detected, or at minimum warned about in `meta.warnings`.
- **One parser per kind.** No fallback chains inside the registry; a parser may call a secondary library internally.

---

## 5. Format tiers

| Tier | Formats | When |
|---|---|---|
| **1** | pdf, docx, pptx, md, txt, csv, html | P0 — covers the real corpus |
| **2** | xlsx, audio (transcript), youtube | P1 |
| **3** | image (OCR), epub, email (.eml/.msg) | P2 |

Do not build tier 3 before tier 1 is solid. Format breadth is the classic way this layer becomes a permanent project.

---

## 6. Library choices

Per-format, not one universal parser. A universal parser means every DOCX goes through an ML layout model.

| Format | Candidate | Why | Caveat |
|---|---|---|---|
| PDF | `pymupdf` | fast, per-span bbox, precise offsets | weak on multi-column and tables |
| PDF (complex) | `docling` | layout model, reading order, tables | slow; use as a fallback, not the default |
| DOCX | `python-docx` | precise, fast, native styles | no layout, which is fine |
| PPTX | `python-pptx` | slide + shape text | speaker notes need explicit handling |
| XLSX | `openpyxl` | cell-level, formulas | large sheets need a row cap |
| HTML | `trafilatura` | boilerplate removal | verify current API |
| MD / TXT | stdlib | trivial | — |
| Audio | `faster-whisper` | word-level timestamps → spans | model download on first use |
| Image | `tesseract` or a vision model | OCR | quality varies wildly |

**Verify current APIs and versions before implementing** — these move. Per the repo rule, check against the installed release rather than memory.

**Do not adopt Docling as the single parser.** It is excellent for complex PDFs and unnecessary overhead for a DOCX.

---

## 7. Caching

Parse results are cached by `sha256` of the source bytes, keyed also by `parser` + `parser_version`.

```
cache_key = sha256(source_bytes) + ":" + parser + ":" + parser_version
```

- Re-reading the same PDF in a later turn is instant.
- A parser upgrade invalidates cleanly.
- The cache is content-addressed, so it is path-independent — the same file at two paths parses once.

Store as JSON next to the app DB, not in the workspace.

---

## 8. Why not Docling MCP

MCP is a process boundary. That is the wrong shape here:

1. **Batch throughput.** Ingestion parses hundreds of documents. MCP round-trips per document add latency and failure modes for no benefit.
2. **Fidelity.** `bbox` and offsets would have to survive serialization across a socket. They can, but you gain nothing and lose the ability to keep the parsed document in memory.
3. **It is the wrong abstraction.** MCP exposes *tools to an agent*. Parsing is an *internal pipeline stage*. The agent-facing tool is `document_read`; that tool calls the library.

**Use the library in-process.** If we later want other clients to reach document parsing, add an MCP server as an *additional* surface that wraps the same registry. It should never be the primary path.

---

## 9. What this changes in the plan

| Before | After |
|---|---|
| `NB-201` parser registry (2d) | `DH-101` shared registry (2d) |
| `NB-202` chunking with offsets (2d) | `DH-102` block model + offsets (3d), `NB-202` chunker (1d) |
| - | `DH-103` `document_read` tool (1d) |
| — | `DH-104` viewer rendering (1.5d) |
| — | `DH-105` parse cache (0.5d) |
| — | `DH-106` format tier 2 (3d) |

Net: **~4–6 days saved** versus building both separately, and the offset retrofit is avoided entirely.

The module lives at `apps/backend/app/documents/`, not under `notebook/`, because it is not notebook-specific.

---

## 10. Anti-patterns

| Anti-pattern | Why it fails |
|---|---|
| MCP in the ingestion hot path | per-document round-trips; offsets cross a socket for no gain |
| One universal parser | every DOCX pays for an ML layout model |
| Parse to markdown, discard structure | offsets are unrecoverable; notebook layer must be rewritten |
| Re-parse the original file to render the viewer | viewer can disagree with what was indexed |
| Extension-based `parser_kind` | a mislabelled file silently produces garbage |
| Format breadth before format depth | tier 3 becomes a permanent project |
| Cache by path | the same file at two paths parses twice; a moved file re-parses |
| Return `ParsedDocument` to the model | bboxes and offsets are noise in the context window |

---

## 11. Sequencing

1. **`DH-101` + `DH-102`** — registry and block model, PDF + DOCX + MD only.
2. **`DH-103`** — `document_read` tool. This alone closes the capability gap and is independently shippable.
3. **`DH-104`** — viewer rendering. Visible win, no new backend surface.
4. **`NB-202`** — chunker consuming blocks. Now the notebook layer has its foundation.
5. **`DH-106`** — tier 2 formats, once tier 1 is proven on a real corpus.

Steps 1–3 are ~7 days and deliver a standalone feature. Step 4 is where the merge pays off.
