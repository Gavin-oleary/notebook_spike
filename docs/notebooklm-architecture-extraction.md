# NotebookLM / Gemini Notebook — extracted architecture

**Date:** 2026-11-19
**Purpose:** mirror the architecture (not the code) so we can decide what to adopt in our desktop assistant.
**Evidence basis:** shipped protobuf service definitions + reverse-engineered wire protocol from public clients. See §10 for confidence levels and caveats.

---

## 0. How this was obtained, and what it is

NotebookLM has no public API. But three things leak the architecture:

1. **The Android client ships protobuf definitions.** The service is `google.internal.labs.tailwind.orchestration.v1.*` — `Project`, `Source`, `Artifact`, `Chat`, `Research`, `Notes`, `Metering`, `Quota`. These are the *server's own message contracts*, not a guess.
2. **The web client uses Google's `batchexecute` RPC envelope**, which several open-source clients have mapped method-by-method.
3. **The chat path is a separate streaming endpoint**, not `batchexecute`.

So we have the **service contract** with high confidence, and the **internal implementation** (how retrieval actually ranks, how audio is synthesised) only by inference.

| Layer | Confidence | Basis |
|---|---|---|
| Message/field model | **Observed** | shipped `.proto` descriptors |
| RPC method IDs | **Observed** | captured `batchexecute` traffic |
| Transport framing | **Observed** | captured request/response bodies |
| Retrieval algorithm | **Inferred** | contract shape only |
| Audio/video synthesis | **Inferred** | output artifacts only |

---

## 1. System shape

```
┌──────────────────────────────────────────────────────────────────────┐
│ CLIENT (web bundle / Android)                                        │
│  • notebook shell, source panel, chat panel, Studio panel            │
│  • renders TailwindDoc (structured answer), not markdown             │
└───────────────┬──────────────────────────────┬───────────────────────┘
                │ batchexecute (RPC)           │ streaming (chat)
                ▼                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ ORCHESTRATION SERVICE  google.internal.labs.tailwind.orchestration.v1│
│                                                                      │
│  ProjectService   SourceService   ArtifactService   ChatService      │
│  ResearchService  NotesService    MeteringService   QuotaService     │
└───────┬──────────────────┬───────────────────┬───────────────────────┘
        │                  │                   │
        ▼                  ▼                   ▼
   ┌─────────┐      ┌─────────────┐     ┌──────────────┐
   │ INGEST  │      │  RETRIEVAL  │     │  GENERATION  │
   │ parse   │      │  chunk+rank │     │  chat        │
   │ chunk   │      │  spans      │     │  audio/video │
   │ index   │      │             │     │  slides/etc  │
   └─────────┘      └─────────────┘     └──────────────┘
```

The important structural fact: **retrieval is its own service with its own RPC**, sitting between ingestion and generation. It is not a helper inside the chat handler.

---

## 2. Transport layer

### 2.1 `batchexecute` (everything except chat)

```
POST https://notebooklm.google.com/_/LabsTailwindUi/data/batchexecute
     ?rpcids=<6-char-id>&source-path=<path>&hl=en&rt=c&f.sid=<session-id>
Content-Type: application/x-www-form-urlencoded

f.req=<urlencoded JSON>&at=<csrf-token>
```

Request envelope — triple-nested, params as a **JSON string inside JSON**:

```json
[[["wXbhsf", "{\"params\":...}", null, "generic"]]]
```

Response — anti-XSSI prefix, then alternating byte-count / JSON lines:

```
)]}'

1234
[["wrb.fr","wXbhsf","{\"inner\":\"json\"}",null,null,null,"generic"]]
```

The payload at `[0][2]` is itself a JSON-encoded string that needs a second parse. Errors arrive as `["er", rpcID, message]`.

Auth: session cookies + `at` (CSRF, scraped from page HTML as `SNlM0e`) + `f.sid` (session id, `FdrFJe`).

### 2.2 Chat streaming

Chat does **not** use `batchexecute`. It posts to:

```
/_/LabsTailwindUi/data/google.internal.labs.tailwind.orchestration.v1.LabsTailwindOrchestrationService/GenerateFreeFormStreamed
```

with an incrementing `_reqid`, and streams `GenerateFreeFormStreamedResponse` frames carrying `answer` + `is_final_response`.

### 2.3 Android

gRPC + protobuf, bearer auth from a long-lived master token minting short-lived tokens. Same service, different encoding. This is why the protos exist in the wild.

### 2.4 RPC method map (observed)

| Surface | Method | ID |
|---|---|---|
| Notebooks | list / create / get / rename / delete | `wXbhsf` `CCqFvf` `rLM1Ne` `s0tc2d` `WWINqb` |
| Sources | add / delete / get / refresh / update | `izAoDd` `tGMBJ` `hizoJc` `FLmJqe` `b7Wfje` |
| Summaries | summarize / source guide | `VfAZjd` `tr032e` |
| Artifacts | create / list / delete / export | `R7cb6c` `gArtLc` `V5N4be` `Krh3pd` |
| Conversation | last conv id / turns | `hPTbtc` `khqZz` |
| Notes / mind map | generate mind map / create note / get notes | `yyryJe` `CYK0Xb` `cFji9` |
| Research | start fast / start deep / poll / import | `Ljjv0c` `QA9ei` `e3bVqc` `LBwxtb` |
| Sharing | share / status | `QDyure` `JFMDGd` |
| Settings | get | `ZwVcOc` |

---

## 3. Domain model

### 3.1 `Project` (= notebook)

```
Project
  project_id
  sources[]              Source
  emoji
  metadata               ProjectMetadata
    user_role            OWNER | WRITER | READER | NOT_SHARED
    create_time
    is_public
    audio_overview_artifact_ids[]
  premium_feature_info   PremiumFeatureInfo
  project_tier_limits    TierLimits
  chat_sessions[]        ChatSession
```

Note `audio_overview_artifact_ids` is hoisted onto the project — audio is treated as a first-class project property, not just another artifact.

### 3.2 `Source`

```
Source
  source_id
  metadata               SourceMetadata
    oneof: webpage_metadata | google_docs_metadata
         | google_drive_metadata | expert_intelligence_source_metadata
    source_added_timestamp
    original_source_content_type
  settings               SourceSettings
```

`OriginalSourceContentType` enum — the full ingestion matrix:

```
GOOGLE_DOC, GOOGLE_SLIDES, PDF, TEXT, URL, POWERPOINT, GOOGLE_SHEET,
MARKDOWN, YOUTUBE_VIDEO, AUDIO, WORD, EXCEL, IMAGE, DRIVE, GMAIL,
CSV, EPUB, GEMINI_CHAT, AI_MODE_CHAT, EXPERT_INTELLIGENCE
```

`UserContent` (what you upload) is a separate oneof: `google_drive_content | text_content | web_content | video_content | expert_intelligence_content`.

`TextContentType` (what the parser sees): `PDF, TEXT, MARKDOWN, AUDIO_TRANSCRIPT, EPUB, GEMINI_CHAT, CSV`.

**Design note:** there are *two* type axes — "where it came from" and "what the parser should do with it". Collapsing them into one enum is a common mistake.

### 3.3 Source lifecycle

```
AddSources / AddTentativeSources   →  SourceAcknowledgement { status }
CheckSourceFreshness               →  SourceFreshness { is_fresh }
RefreshSource                      →  re-ingest
LoadSource                         →  PlainTextSourceContent { header, body, markdown_string }
GenerateDocumentGuides             →  DocumentGuide { source, snippet, main_ideas }
```

`LoadSource` returning `header` + `body` + `markdown_string` is the "source viewer" contract — the client never parses the original file.

---

## 4. The retrieval service — the core of the design

This is the piece worth copying.

```
RetrieveRelevantChunksRequest
  query            string
  options          RetrieveRelevantChunksOptions
    mode
    source_filter  SourceIdFilter
  source_filter    SourceIdFilter

RetrieveRelevantChunksResponse
  source_chunks[]  SourceRelevantChunks
    chunks[]       RelevantChunk
      rank         int
      spans[]      RelevantChunkSpan { start, end }
      content      RelevantChunkContent { text, parts }
```

Three things make this contract good:

1. **It is a standalone RPC.** Retrieval can be called, tested, benchmarked, and swapped independently of generation. You can evaluate recall without invoking a model.
2. **It returns spans, not just text.** `RelevantChunkSpan { start, end }` is character offsets into the source. That is what makes citation highlighting possible later.
3. **It is grouped by source.** `SourceRelevantChunks` means the caller can reason about "which documents contributed" without re-grouping.

The `source_filter` at both request and options level is how "chat against a subset of sources" works — it is a retrieval filter, not a prompt instruction.

**Inferred but not observable:** the ranking algorithm, chunk size, whether there is a reranker, and how `mode` changes behaviour. The contract tells us the shape, not the quality.

---

## 5. The answer document model — how citations actually work

The chat response is **not markdown**. It is a structured document:

```
AnswerResponse
  response              string
  conversation_turn_key ConversationTurnKey { session_id, conversation_id }
  empty_answer_reason   EmptyAnswerReason
  response_doc          TailwindDoc

TailwindDoc
  body                  Body
    inline_object_locations  map<string, AnnotationMapEntry>
      object_id
      content_range     Range { start_index, end_index }
  objects[]             DocumentObject
    citation            Citation
      ranges[]          Range
      fragment          TailwindDocFragment
      source_attribution CitationSource { ingested_source: SourceRevision { source: SourceId } }
```

`Body` is a list of `StructuralElement`, a oneof:

```
paragraph | table | code_block | a2ui_block | thought
| function_call | function_response | horizontal_rule
```

And `Paragraph` contains `ParagraphElement` = `text_run | image | resource`, where `TextRun` carries a full `TextStyle` (bold, italic, underline, url, font_info, text_color, background_color, code, strikethrough, math).

### Why this matters

Hover-to-quote is not a UI trick on top of markdown. It works because:

- the answer is a document with **object IDs**,
- the body maps each object ID to a **character range**,
- each citation object carries **ranges into the source fragment** plus a **source revision**.

So the client can resolve "this sentence → this exact span of this exact source version" without re-parsing anything.

Three more things fall out of this model for free:

- **`thought`** — thinking steps render inline in the answer, not in a separate panel.
- **`function_call` / `function_response`** — tool calls render inline in the answer document.
- **`a2ui_block`** — agent-to-UI: the model can emit a generative UI block as part of the answer.

### Chat request shape

```
ActOnSourcesRequest
  sources[]            InputSource
  options              ActOnSourcesOptions { citation_content_type, answer_content_type }
  free_form_action     FreeFormAction { user_query, conversation_history[], user_message_id }
  mind_map_action      ActOnSourcesMindMapAction
  source_options       InputSourceOptions { project_id, use_all_sources }
  request_context      RequestContext { chat_session_id, origin }
```

`QueryOrigin` enum: `CHAT_TEXT_BOX, SUGGESTION_CHIP, APP_VIEWER_EXPLAIN_BUTTON, SOURCE_KEY_TOPIC_CHIP, MINDMAP_TOPIC_CHIP, GETTING_STARTED_MESSAGE`.

That enum is a small but telling detail: **every entry point into chat is instrumented**, including "clicked a mind-map node" and "clicked explain on a source". The product is designed around many entry points into the same grounded engine.

Session control: `GetChatSessionStatus { generation_token }`, `CancelGeneration { agency_session_id }`, `ListChatTurns`, `DeleteChatTurns`.

---

## 6. The artifact pipeline

### 6.1 Model

```
Artifact
  artifact_id
  type            ArtifactType
  sources[]       ArtifactSource { source_id }
  status          ArtifactStatus
  oneof: audio_overview | tailored_report | explainer_video
       | app | infographic | slides | mindmap | table | file
  last_modified_timestamp
  artifact_user_state  ArtifactState
    audio_overview_state  { playback_position }
    video_overview_state
    app_artifact_state    Struct
    scheduled_notification_configs[]
```

`ArtifactStatus`: `UNKNOWN, INITIALIZED, PROCESSING, READY, FAILED, SUGGESTED, PENDING_REVIEW, SCHEDULED`.

`ArtifactType`: `AUDIO_OVERVIEW, TAILORED_REPORT, EXPLAINER_VIDEO, APP, MINDMAP, FANTASY_MAP, INFOGRAPHIC, SLIDES, TABLE, FILE`.

### 6.2 Lifecycle

```
CreateArtifact        → Artifact (INITIALIZED)
GenerateArtifact      → Artifact (PROCESSING)
GetArtifact / ListArtifacts  → poll status
ExportToDrive         → push to Drive
DeriveArtifact        → new version from edits (slide_edit_instructions)
CopyArtifactsAsync    → copy across projects
```

`PENDING_REVIEW` is the "producer notes" / "outline review" step — generation pauses for human approval before the expensive render. `SCHEDULED` is "generate later" when out of quota.

### 6.3 Generation options (per type)

```
AudioOverviewGenerationOptions
  episode_focus, episode_length (SHORT|MEDIUM|LONG), source_ids[]

ExplainerVideoGenerationOptions
  video_focus, style_prompt
  template_format (EXPLAINER|BRIEF|BREAKDOWN|SHORT|WHITEBOARD_ANIMATION)
  video_overview_style (AUTO_SELECT|CLASSIC|WHITEBOARD|HERITAGE
                        |PAPERCRAFT|WATERCOLOR|ANIME|RISOGRAPHIC|KAWAII)

SlidesGenerationOptions
  deck_type (READING|PRESENTATION)
  length (DYNAMIC|SHORT|MEDIUM|LONG)

InfographicGenerationOptions
  user_steering_prompt
  aspect_ratio (LANDSCAPE|PORTRAIT|SQUARE)
  style (AUTO|SKETCH_NOTE|PROFESSIONAL|BENTO_GRID|EDITORIAL|STORYBOARD
         |BRICKS|CLAYMATION|ANIME|KAWAII|SCIENTIFIC|ACADEMIC)

QuizGenerationOptions
  question_quantity (FEWER|STANDARD|MORE)
  quiz_difficulty (EASY|MEDIUM|HARD)
  image_option

FlashcardsGenerationOptions
  card_quantity, flashcards_difficulty

AppArtifactGenerationOptions
  app_type (FLASHCARDS|QUIZ|PROTOTYPE|MINDMAP|CANVAS)
  free_text_steering_prompt, language_code
  flashcards_generation_options, quiz_generation_options

TailoredReportArtifactGenerationOptions
  description, document_directive
```

### 6.4 Dynamic options

```
GetArtifactCustomizationChoicesRequest { artifact_type }
  → ArtifactCustomizationChoices
      audio_overview_choices   FormatChoices
      video_overview_choices   FormatChoices
      slides_customization_choices
      tailored_report_customization_choices
```

The UI does not hardcode the option lists — it asks the server what choices exist for this artifact type. That is how they ship new formats without a client release.

### 6.5 Media delivery

```
MediaStreamingUrl { url, type }
MediaStreamingType: PROGRESSIVE_STREAMING | ADAPTIVE_STREAMING_HLS
                  | ADAPTIVE_STREAMING_DASH | DOWNLOAD
```

Audio/video are served as media streams, not as blobs in the RPC response.

---

## 7. Notebook-level AI (cheap wins)

```
GenerateNotebookGuide → NotebookGuide
  summary              NotebookSummary { text_summary }
  next_step_suggestions NextStepSuggestions { next_steps[] { suggestion, suggestion_type } }

GeneratePromptSuggestions → PromptSuggestion[] { prompt }
```

`SuggestionConfigId` maps suggestions to artifact types (`AUDIO_OVERVIEW_DEEP_DIVE`, `VIDEO_OVERVIEW_EXPLAINER`, `QUIZ`, `FLASHCARDS`, …). `MagicArtifactType` is the same idea for the "magic" chips.

This is a small amount of work for a large perceived-intelligence gain: the notebook tells you what it can do with *these* sources.

---

## 8. What to copy into our desktop assistant

Our stack: Electron + FastAPI + `pydantic-ai` + sqlite. We already have chat, streaming, workspaces, and a file viewer. What we lack is the notebook layer.

### 8.1 Mapping

| NotebookLM concept | Our equivalent | Effort |
|---|---|---|
| `Project` | `notebooks` table (workspace-scoped) | S |
| `Source` + content-type enum | `sources` table + ingestion pipeline | M |
| `RetrieveRelevantChunks` | `POST /notebooks/{id}/retrieve` → ranked spans | M |
| `TailwindDoc` | Pydantic model, streamed as SSE events | M |
| `Citation` + `Range` | `citations` table written at generation time | M |
| `Artifact` + status enum | `artifacts` table + asyncio worker | M |
| `GetArtifactCustomizationChoices` | static registry in v1, server-driven later | S |
| `GenerateNotebookGuide` | one LLM call on notebook open | S |
| `CheckSourceFreshness` | `sha256` + `synced_at` on source | S |
| `batchexecute` | **do not copy** — use plain JSON/SSE | — |
| `a2ui_block` | **defer** — generative UI is a project of its own | — |

### 8.2 The five decisions worth stealing

1. **Retrieval is a service, not a helper.** Expose it as its own endpoint with `query + source_filter → ranked chunks with spans`. It makes citations possible, makes recall testable, and lets you swap the ranker without touching chat.

2. **Stream a document, not markdown.** Define a `AnswerDoc` with `body[]` of typed elements and an `annotations` map from element ID → character range. Markdown is a rendering target, not the wire format. Retrofitting this later is the single most expensive mistake available.

3. **Citations are written at generation time.** The model must emit citation objects with source ID + range as part of its structured output. Do not try to reconstruct citations by string-matching the answer against the corpus afterwards.

4. **Artifacts are jobs with a status enum.** `INITIALIZED → PROCESSING → READY|FAILED`, plus `PENDING_REVIEW` for the outline-approval step and `SCHEDULED` for deferred work. One table, one asyncio worker, progress over the existing SSE channel.

5. **Two type axes on sources.** "Where it came from" (URL, Drive, upload) and "what the parser should do" (PDF, markdown, audio transcript) are different enums. Collapsing them causes parser-selection bugs.

### 8.3 What not to copy

- **`batchexecute`.** It exists because Google's frontends share one RPC envelope. We have FastAPI; use JSON + SSE.
- **Protobuf over gRPC.** Only worth it if we need the Android client. We don't.
- **`a2ui_block`.** Generative UI inside the answer document is a large surface. Defer until the document model is stable.
- **The `FANTASY_MAP` artifact type.** Yes, really. It is a decorative map. Skip.

### 8.4 Suggested schema delta

```sql
notebooks(id, workspace_id, title, policy, created_at, updated_at)

sources(id, notebook_id, origin_kind, parser_kind, uri, title,
        sha256, bytes, words, status, added_at, synced_at)

source_chunks(id, source_id, ordinal, text, start_offset, end_offset,
              page, heading_path, token_count)

chunk_vectors(chunk_id, embedding)      -- sqlite-vec
chunk_fts(chunk_id, text)               -- FTS5

answer_docs(id, message_id, body_json, annotations_json, created_at)
citations(id, answer_doc_id, object_id, source_id, chunk_id,
          start_offset, end_offset, quote)

artifacts(id, notebook_id, kind, status, options_json, output_path,
          model, cost_json, created_at, started_at, completed_at)
```

`answer_docs.body_json` holds the `StructuralElement[]`; `annotations_json` holds the object-ID → range map. `citations` is the queryable projection for "which sources did this answer use".

---

## 9. If we want to go deeper

The proto contracts answer "what is the shape". They do not answer "how good is the retrieval" or "how is audio actually produced". For those:

### 9.1 HAR capture (best value per minute)

1. Open `notebook.google.com` in Chrome, sign in.
2. DevTools → Network → check **Preserve log**, filter to `Fetch/XHR`.
3. Do one full pass: create notebook → add 3 sources → ask 3 questions → generate one mind map → generate one audio overview.
4. Right-click any request → **Save all as HAR with content**.
5. Hand me the `.har`.

What that adds: real request/response bodies, real chunk counts, real latency per stage, real streaming frame cadence, and the actual `RetrieveRelevantChunks` payloads if the web client calls it.

**Redact first:** the HAR contains your cookies and document text. Strip `Cookie`, `Authorization`, and any `at=`/`f.sid=` values before sharing.

### 9.2 Bundle analysis

The web bundle is at `notebook.google.com/_/LabsTailwindUi/...`. With an authenticated session I can pull it and extract prompt templates, feature flags, and the client-side document renderer. Lower value than the HAR — the protos already told us the model.

### 9.3 What I would *not* spend time on

- Decompiling the Android APK. The protos are already extracted and public.
- Trying to call the API ourselves. It is against Google's ToS, it breaks on their schedule, and we do not need it — we are copying the design, not the service.

---

## 10. Caveats

- **These are unofficial sources.** The protobuf definitions come from a community client (`teng-lin/notebooklm-py`), not from Google. Field names are almost certainly accurate (they are serialized descriptors), but the *semantics* of individual fields are inferred from names and usage.
- **The contract is not the implementation.** Knowing there is a `RetrieveRelevantChunks` RPC tells us retrieval is a service. It does not tell us chunk size, embedding model, whether there is a reranker, or how `mode` behaves.
- **Google changes this without notice.** Method IDs and field numbers are internal. Treat the document as a design reference, not a spec to build against.
- **Do not ship a client that calls these endpoints.** That is a ToS problem and an operational liability. The value here is architectural.

---

## Appendix — source files consulted

- `teng-lin/notebooklm-py` — `_android/proto/google/internal/labs/tailwind/orchestration/v1/{read,chat,artifacts,sources,notebooks}_pb2.py`, `labs/language/tailwind/common/protos/provenance_pb2.py`
- `LocalKinAI/notebooklm-go` — `rpc.go` (method IDs, envelope encoding, response decoding)
- `Experto-AI/pynotebooklm` — `docs/reverse_engineering_findings.md` (RPC map, streaming endpoint)
