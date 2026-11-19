# Notebook Layer

Source-grounded research for the desktop assistant: notebooks, retrieval with spans, cited answers, and generated artifacts.

This folder is the spec. It is derived from an extraction of NotebookLM / Gemini Notebook's shipped service contract — see `../notebooklm-architecture-extraction.md`.

## Read in this order

| # | Document | Answers |
|---|---|---|
| 1 | [`../notebooklm-replica-brief-and-plan.md`](../notebooklm-replica-brief-and-plan.md) | What NotebookLM does, and which of our use cases actually fit |
| 2 | [`../notebooklm-architecture-extraction.md`](../notebooklm-architecture-extraction.md) | How it is built — the observed service contract |
| 3 | [`DEPLOYMENT_PROFILE.md`](DEPLOYMENT_PROFILE.md) | Trust tiers, injected policy, agent surface, evidence records |
| 4 | [`DOCUMENT_LAYER.md`](DOCUMENT_LAYER.md) | The shared parser — one library, three consumers |
| 5 | [`FEATURES.md`](FEATURES.md) | What we are building, in what order, with testable acceptance |
| 6 | [`CODE_BIBLE.md`](CODE_BIBLE.md) | How to build it without painting ourselves into a corner |

## The short version

Six decisions carry most of the value:

1. **Three trust tiers, not two.** T0 Ollama on-device, T1 internal Azure, T2 external. "Cloud" is not one thing, and treating T1 as T2 either over-restricts or leaks.
2. **Policy is injected, not defined here.** The layer passes source/chunk identity and the destination tier to an `EgressPolicy` and records the answer. It does not own a data taxonomy.
3. **One parser, three consumers.** `app/documents/` serves the agent tool, notebook ingestion, and the file viewer. It always produces offsets, even though two of three callers ignore them.
4. **Retrieval is a service.** `query + source_filter → ranked chunks with character spans`. Callable without a model, so recall is measurable.
5. **Answers are documents, not markdown.** A typed `AnswerDoc` with an annotation map and citation objects. Markdown is a render target.
6. **The notebook is an agent capability.** Every operation is a prefixed tool (`document_*`, `notebook_*`, `artifact_*`) so `ToolProfile.tool_prefixes` can gate it. The UI and the agent call the same handlers.

Everything else is detail.

## Status

| | |
|---|---|
| Spec | draft |
| Epic in `plans/` | not yet created |
| Code | not started |
| P0 estimate | ~60.5 engineer-days (P1 +39, P2 +25.5) |

## Next step

Promote this into `plans/P2-notebook-layer.md` and split the P0 features into `pending-*` stories under `plans/P2-notebook-layer/`, honoring the dependency column in `FEATURES.md`.

Start with the document layer: **DH-101 → DH-102 → DH-103 → DH-104**. That is ~7 days, ships on its own (the assistant can finally read a PDF), and produces the block model that everything downstream chunks on.

Then **NB-202 → NB-204 → NB-301 → NB-305** — a measurable retrieval service before any chat code exists. That is the only way to tell a retrieval bug from a generation bug later.

## Open questions

`DEPLOYMENT_PROFILE.md` §9 lists five questions that change the design and cannot be answered from the repo — the policy implementation, whether T2 is enabled, what the policy needs to see, whether export requires review, and the retention period.
