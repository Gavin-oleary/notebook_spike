# NotebookLM → "Gemini Notebook": capability brief + replication plan

**Status:** research brief, not yet an approved epic
**Date:** 2026-11-19
**Scope:** understand what NotebookLM does today, decide which parts are worth replicating for our work tasking, and lay out a hybrid (local + cloud) build plan.

---

## 0. TL;DR

- NotebookLM was **rebranded to "Gemini Notebook"** (`notebook.google.com`). The feature set grew well past "upload PDFs and chat": it is now a **source-grounded research engine + an artifact factory** (audio, video, slides, reports, mind maps, flashcards, quizzes, infographics, interactive sites).
- The product is three layers, and people usually only notice the first:
  1. **Grounded retrieval + chat** — answers only from your sources, with inline citations.
  2. **Provenance** — hover a citation to see the exact quote, click to jump to it in context. This is the actual moat.
  3. **Artifact factory** — the same source set re-rendered into ~10 output formats.
- **~80% of the utility is replicable in 6–8 engineer-weeks** on top of the existing `pydantic-chat` app. The remaining 20% (Audio/Video Overview polish) is where effort explodes and where buying is the better call.
- **Not all five of your use cases fit the NotebookLM shape.** "Internal knowledge base Q&A" does *not* — it needs ACLs, incremental sync, and thousands of sources. That is a different architecture (enterprise RAG), and pretending otherwise is the main way this project fails.
- **Hybrid boundary should sit at retrieval, not at the document.** Index and retrieve locally; send only the retrieved spans to a cloud model when policy allows. That gives cloud-quality synthesis on sensitive corpora without shipping the corpus.

---

## 1. What it is now

| | |
|---|---|
| Product name | **Gemini Notebook** (formerly NotebookLM) |
| URL | `notebook.google.com` |
| Positioning | "AI research tool & thinking partner" — grounded in *your* sources |
| Languages | 80+ |
| Data policy | Org/school data not used for training; consumer data not used for training unless you give feedback |
| Limits model | **Compute-based**, not message-count. Refreshes every 5h, weekly cap. Paid tiers = 2×/4×/5–20× |

### Plans and hard limits

| | Free | AI Plus | AI Pro | Ultra |
|---|---|---|---|---|
| Notebooks / user | 100 | 200 | 500 | 500 |
| Sources / notebook | 50 | 100 | 300 | 500–600 |
| Notes / notebook | 500 | 500 | 500 | 600 |
| Per-source cap | 500k words **or** 200 MB | same | same | same |

The compute-based limit is the tell: **artifact generation is the expensive part**, not chat. Any replica needs a job queue and a budget model from day one, not a request/response endpoint.

---

## 2. Feature inventory

Difficulty is *replication* difficulty for a small team, not product quality.

### 2.1 Ingestion

| Source type | Notes | Difficulty |
|---|---|---|
| PDF, DOCX, PPTX, TXT, MD, CSV | core | Low |
| Images (png/jpg/webp/heic/tiff/…) | OCR + vision | Low–Med |
| Audio (mp3/wav) | transcription | Low |
| Web URLs | fetch + extract | Low |
| YouTube URLs (public) | transcript + metadata | Low |
| Google Docs / Slides / Sheets | API + sync | Med (auth) |
| ePub, Play Books | DRM-adjacent, skip | — |
| Pasted text, notes-as-sources | trivial | Low |
| Gemini Chats as sources | cross-product | skip |

Per-source ceiling: **500k words / 200 MB**. That is a real design constraint — it forces chunking + hierarchical summarisation rather than "stuff the context window".

### 2.2 Chat

- Answers **only** from selected sources; refuses creative/off-source asks.
- **Inline citations** with hover-quote and click-to-context.
- Per-notebook source selection (chat against a subset).
- **Chat styles:** Default / Learning Guide / Custom ("respond like a PhD student").
- **Response length** control.
- **Save to note** — pins a response (tables and live citations preserved) to the noteboard.
- **Agentic actions:** web search, run code, produce downloadable files, charts, images.
- **Live voice chat** (mobile, 18+).
- **Connected apps** (web only) — e.g. Drive, enabled per-notebook.

### 2.3 Studio artifacts

| Artifact | What it produces | Difficulty | Notes |
|---|---|---|---|
| **Audio Overview** | 2-host podcast. Formats: Deep Dive, Brief (<2 min, 1 host), Critique, Debate. Language, length, focus prompt. | **High** | Scripted dialogue + multi-voice TTS + turn-taking. The *banter* is the hard part, not the audio. |
| **Video Overview** | Cinematic / Explainer / Short (~60s). Producer-notes review step. Can take 30+ min. | **Very high** | Script → visuals → narration → render. MVP = narrated slide deck. |
| **Slide Deck** | Outline review → deck. Present in-app, export PDF. Per-slide revise. | Med | HTML/Reveal or pptxgenjs gets you 90%. |
| **Report** | Interactive (embedded artifacts) or document. Templates. | Low–Med | Markdown/HTML + template registry. |
| **Mind Map** | Branching diagram, expand/collapse, click node → ask about it. | **Low** | LLM → tree JSON → renderer. Highest value per unit effort. |
| **Flashcards** | Difficulty, count, "Got it / Missed it" progress, CSV export. | Low | |
| **Quiz** | Difficulty, hints, explanations, scoring. | Low | |
| **Infographic** | Concise/Standard/Detailed, orientation, visual style, PNG export. | Med | HTML/SVG render or image-gen. |
| **Sites** | Interactive dashboards/trackers from sources; state persists per user. | Med | LLM → single-file HTML app. |

### 2.4 Collaboration

- Share notebook (private / link / public), share individual artifacts.
- Public notebooks are **disabled for Workspace Enterprise/Education** — relevant if we're on Workspace.
- Featured public notebooks as onboarding.

---

## 3. Where the value actually is

Ranked by "what users would miss if it were gone":

1. **Citation fidelity.** Hover-to-quote and click-to-context is what makes the output *trustable*. A chat that says "according to your sources" without span-level provenance is a different, much weaker product. This is the single highest-leverage thing to get right.
2. **The artifact factory.** One source set → deck, audio, quiz, report. This is the actual time saving for training/onboarding work.
3. **Source scale + selection.** 300 sources in one notebook, chat against a subset. Requires real retrieval, not context stuffing.
4. **Refusal discipline.** "Not in your sources" is a feature. It is also the hardest thing to tune — too strict and it's useless, too loose and it hallucinates.

---

## 4. Commodity vs. hard

**Commodity (buy or use OSS):**
- Document parsing (PDF/DOCX/PPTX/HTML) — Docling, `unstructured`, `pymupdf`
- Chunking, embeddings, vector index — `sqlite-vec`, LanceDB, pgvector
- Hybrid retrieval (BM25 + vector + rerank) — standard
- Transcription — `faster-whisper`
- TTS — ElevenLabs, Gemini TTS (multi-speaker), OpenAI TTS; local: Kokoro, Piper
- Slide rendering — Reveal.js, Marp, `pptxgenjs`
- Image generation — any image model

**Genuinely hard:**
- **Span-level provenance through the whole pipeline.** Chunk → cite → render → hover → jump. Every stage must carry offsets. Retrofitting this is painful; design it in.
- **Audio Overview dialogue quality.** Two hosts that sound like they *read the material and are reacting to it* — not two narrators alternating paragraphs. This is a prompt-engineering + TTS-timing problem with a long tail.
- **Video Overview.** 30+ minute generation, cinematic pacing, visual consistency. Do not attempt in v1.
- **Retrieval quality at 300–600 sources.** Recall collapses without reranking and query decomposition.
- **Cost control.** Compute-based limits exist for a reason. Without a budget model, one enthusiastic user can burn a month of API spend in an afternoon.

---

## 5. Hybrid architecture

### 5.1 The key decision: put the boundary at retrieval

The naive hybrid is "sensitive docs → local model, everything else → cloud". That is wasteful: a local 8B model is much worse at synthesis than a frontier model, so sensitive notebooks get a bad experience.

Better: **index and retrieve locally, then send only the retrieved spans to the model.**

```
┌───────────────────────── LOCAL (never leaves the machine) ─────────────────────────┐
│                                                                                    │
│  ingest ──► parse ──► chunk (+ offsets) ──► embed ──► index (sqlite-vec + FTS5)     │
│                                                                                    │
│  query ──► hybrid retrieve ──► rerank ──► top-k spans ──┐                          │
│                                                          │                         │
│  ┌───────────────────────────────────────────────────────▼──────────────────────┐  │
│  │ EGRESS GATE: per-chunk classification + notebook policy                      │  │
│  │   local-only  → local model only                                             │  │
│  │   cloud-ok    → spans may go to cloud model                                  │  │
│  │   redact      → strip PII/identifiers, then cloud                            │  │
│  └───────────────────────────────────────────────────────┬──────────────────────┘  │
└──────────────────────────────────────────────────────────┼─────────────────────────┘
                                                           │
                        ┌──────────────────────────────────▼──────────────────────┐
                        │ CLOUD (only when policy allows)                         │
                        │  synthesis · long-context reasoning · TTS · image gen   │
                        └─────────────────────────────────────────────────────────┘
```

Why this wins:
- Sensitive corpora still get frontier-model synthesis, because only the *relevant spans* cross the boundary.
- Cost drops: you send 8k tokens of retrieved context, not 200k of document.
- The policy is auditable: every egress is a logged set of chunk IDs, not an opaque "we sent your file somewhere".

Caveat: this is only safe if the egress gate is real. A per-chunk classifier that misses one paragraph is a data incident. For genuinely regulated material, `local-only` must be a hard, non-overridable notebook setting.

### 5.2 Tiers

| Tier | Runs | Contents |
|---|---|---|
| **Local core** | Electron sidecar (existing FastAPI) | ingestion, parsing, chunking, embeddings, index, retrieval, rerank, notebook/source/artifact metadata, artifact storage |
| **Local model** | Ollama / llama.cpp | chat + artifact drafting for `local-only` notebooks |
| **Cloud model** | Anthropic / OpenAI / Google (already wired) | synthesis for `cloud-ok` notebooks |
| **Cloud media** | TTS, image gen, video render | Audio/Video/Infographic generation |

### 5.3 Data model (extends existing `app.db`)

```
notebooks(id, workspace_id, title, policy, created_at, updated_at)
sources(id, notebook_id, kind, uri, title, sha256, bytes, words,
        status, added_at, synced_at)
source_chunks(id, source_id, ordinal, text, start_offset, end_offset,
              page, heading_path, token_count)
chunk_vectors(chunk_id, embedding BLOB)          -- sqlite-vec virtual table
chunk_fts(chunk_id, text)                        -- FTS5
messages(id, notebook_id, role, content_json, created_at)   -- reuse conversation_store
citations(message_id, chunk_id, quote, start_offset, end_offset)
artifacts(id, notebook_id, kind, status, spec_json, output_path,
          model, cost_json, created_at, completed_at)
notes(id, notebook_id, title, body_md, source_message_id)
```

`citations` is the table that makes or breaks the product. It must be written at generation time, not reconstructed afterwards.

### 5.4 Job queue

Artifacts are minutes-to-tens-of-minutes jobs. Do not run them in the request handler.

- One `artifacts` row per job, `status ∈ queued|running|failed|done`.
- A single asyncio worker in the sidecar (v1) — no Celery/Redis needed at this scale.
- Cancellation, retry with backoff, and a per-notebook cost ceiling.
- Progress events over the existing SSE channel so the UI can show "generating…".

---

## 6. Which of your five use cases actually fit

| Use case | Fit | Why |
|---|---|---|
| **Research & synthesis over source docs** | ✅ Strong | This is exactly the product's shape. Build this first. |
| **Training / onboarding content** | ✅ Strong | The artifact factory is the whole value. Slides + quiz + audio from the same source set. |
| **Autobuild video / audio / pptx / reports** | ✅ Strong, 💰 expensive | Reports and decks are cheap. Audio is medium. Video is a project of its own. |
| **Competitive / market research** | ⚠️ Medium | Needs web capture + freshness + source provenance from the open web. NotebookLM's web search is bolted onto a source-grounded product; a replica needs a proper "capture this page as a source" flow. |
| **Internal knowledge base Q&A** | ❌ Weak | Needs per-user ACLs, incremental sync from Confluence/Drive/SharePoint, and 10k+ sources. NotebookLM has none of that. **This is a different product.** |

**Recommendation:** build the notebook-shaped tool for use cases 1–3, add web capture for 4, and treat 5 as a separate RAG project with its own ACL model. Do not try to make one system do both — the permission model alone will sink it.

---

## 7. Concrete plan

Effort in engineer-weeks (ew), one experienced dev. Halve for two devs working in parallel on independent phases.

### Phase 0 — Spike (1 ew) · *go/no-go*

**Goal:** prove citation fidelity on real documents before building anything.

- Ingest 20 real docs (mixed PDF/DOCX/PPTX/MD) from one actual work corpus.
- Chunk with offsets, embed, hybrid retrieve, answer with a frontier model.
- Render citations with hover-quote + click-to-context in a throwaway UI.
- Measure: does the quote actually match the source span? Does the model refuse when the answer isn't there?

**Exit criteria:** ≥95% of citations resolve to the correct span; refusal behaviour is sane on 20 adversarial questions.
**If this fails, stop.** Everything downstream depends on it.

### Phase 1 — Notebook core (2–3 ew)

- `notebooks` / `sources` / `chunks` / `citations` tables + migrations.
- Ingestion pipeline: PDF, DOCX, PPTX, MD, TXT, CSV, URL, pasted text. (Images/audio/YouTube in Phase 1.5.)
- Chunking with heading paths and page numbers preserved.
- Hybrid retrieval: FTS5 + `sqlite-vec`, reciprocal-rank fusion, optional reranker.
- Grounded chat with citation rendering, source selection, chat styles, response length.
- Source Guide (auto-summary per source).
- Notes: save-to-note, notes-as-sources.

**Exit criteria:** a real user can drop 30 docs in, ask 20 questions, and every answer is traceable.

### Phase 1.5 — Ingestion breadth (1 ew)

- Images (OCR + vision), audio (whisper), YouTube (transcript), Google Docs/Slides/Sheets (OAuth).
- Per-source status + re-sync.

### Phase 2 — Text artifacts (2 ew)

Order by value-per-effort:

1. **Mind Map** (2–3 days) — tree JSON → interactive renderer, click-node-to-ask.
2. **Flashcards + Quiz** (3–4 days) — difficulty, count, progress, CSV export.
3. **Report** (3 days) — template registry → markdown/HTML.
4. **Slide Deck** (4–5 days) — outline review step → Reveal.js/Marp HTML → PDF export. PPTX via `pptxgenjs` if required.

**Exit criteria:** one source set produces a deck, a quiz, and a mind map that a human would actually use.

### Phase 3 — Audio Overview (2–3 ew)

- Script generator with format presets (Deep Dive / Brief / Critique / Debate), length, focus prompt.
- Two-voice TTS with turn-taking; cloud TTS first (Gemini multi-speaker or ElevenLabs), local Kokoro as fallback.
- Player with transcript, click-to-jump, speed control.
- Background job + progress events.

**Exit criteria:** a 10-minute Deep Dive on a 20-doc notebook that a colleague will listen to without wincing. This is the bar; expect to iterate on the script prompt more than the audio.

### Phase 4 — Hybrid policy + egress gate (2 ew)

- Per-notebook policy: `local-only` / `cloud-ok` / `redact-then-cloud`.
- Per-chunk classification + egress logging (chunk IDs, model, timestamp).
- Local model path via Ollama for `local-only`.
- Cost accounting per artifact; per-notebook ceiling.

**Exit criteria:** a `local-only` notebook provably makes zero outbound model calls (test asserts on the HTTP client).

### Phase 5 — Optional (only if Phase 0–4 land well)

- **Infographic** (1 ew) — HTML/SVG render, or image-gen.
- **Sites** (1–2 ew) — LLM → single-file HTML app with persisted per-user state.
- **Video Overview** (4–6 ew) — start with "narrated slide deck", not cinematic.
- **Connected apps** (2 ew) — Drive/Confluence connectors.

### Build vs. buy

| Component | Verdict |
|---|---|
| Parsing, chunking, retrieval, citations | **Build.** This is the product. |
| Chat + artifact orchestration | **Build** on `pydantic-ai` (already in the app). |
| Embeddings | **Buy/use** — API or local `bge`/`nomic`. |
| TTS | **Buy** for quality, local fallback for `local-only`. |
| Video render | **Buy** or defer. |
| Slide rendering | **Use OSS** (Reveal/Marp/pptxgenjs). |
| Vector store | **Use** `sqlite-vec` — no new infra. |

### Sequencing note

Phase 0 and Phase 4 are the two that can invalidate the plan. Do Phase 0 first. Do a thin slice of Phase 4 (the policy enum + a no-op gate) in Phase 1 so the data model doesn't have to be retrofitted.

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| Citation offsets drift through parsing/chunking | Carry offsets end-to-end from day one; assert in tests. |
| Scope creep into "internal KB" | Explicitly out of scope. Separate project, separate ACL model. |
| Audio quality disappoints | Set the bar in Phase 3 exit criteria; be willing to ship text-only artifacts and buy audio. |
| Cost blowout on artifacts | Per-notebook ceiling + cost logging from the first artifact. |
| Egress gate false negative | `local-only` is a hard setting, not a heuristic. Log every egress. |
| Retrieval recall at scale | Reranker + query decomposition; benchmark at 100 and 300 sources. |

---

## 9. Recommendation

1. **Run Phase 0 this week.** It is one week and it decides everything.
2. **Build Phases 1–2** — that is the 80% of value for research synthesis and training content, in ~4–5 weeks.
3. **Buy audio, defer video.** Audio Overview is the most impressive-looking feature and the worst effort-to-value ratio to build.
4. **Do not build the internal KB.** If that need is real, it is a separate RAG project with ACLs and connectors, and it should be scoped on its own.
5. **Put the hybrid boundary at retrieval.** It is the only design that gives sensitive corpora frontier-model quality without shipping the corpus.

---

## Appendix — sources

- `notebook.google` (product page, Nov 2026)
- `support.google.com/gemininotebook` — Learn about, Sources, Chat, Audio Overview, Video Overviews, Slide Decks, Reports, Mind Maps, Flashcards/Quizzes, Infographic, Sites, Usage limits, Plans, Connected apps
