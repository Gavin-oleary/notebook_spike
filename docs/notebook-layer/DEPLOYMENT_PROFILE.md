# Deployment Profile — trust tiers, policy injection, agent surface

**Status:** draft
**Date:** 2026-11-19
**Supersedes:** the `local_only | cloud_ok | redact_then_cloud` policy model in `CODE_BIBLE.md` §9 and `FEATURES.md` §9

The notebook layer was specced against a generic "local vs cloud" model. The actual deployment differs in three ways that matter:

1. **Three trust tiers, not two.** Ollama on-device, an internal Azure API, and (optionally) external providers. "Cloud" is not one thing.
2. **Agent-first.** Skills + tools + MCP/ACP/A2A via `pydantic-ai`. The notebook cannot be only a REST surface the UI calls.
3. **Governed.** NIST AI RMF, SP 800-218A, and DoD generative-AI guidance are implemented in operational code.

This document corrects the spec against that reality. It deliberately does **not** define a data taxonomy — see §3.

---

## 1. Trust tiers

| Tier | What it is | Data leaves the device | Controlled by |
|---|---|---|---|
| **T0** | Ollama, on-device | No | The user |
| **T1** | Internal Azure API | Yes, to company infrastructure | The company |
| **T2** | External public provider | Yes, to a third party | The vendor |

The previous `local_only | cloud_ok` enum collapsed T1 and T2. That is the bug: a notebook marked `cloud_ok` would be permitted to reach a public provider, which is a different decision from reaching the internal endpoint.

**T1 is not "cloud" in the risk sense.** It is a governed endpoint inside the company boundary. Treating it as equivalent to T2 either over-restricts — users fall back to a worse local model for no reason — or under-restricts.

The tiers are a fact about the deployment. They are not a policy.

---

## 2. What the notebook layer owns, and what it does not

| Owns | Does not own |
|---|---|
| Which tier a call is destined for | Whether that call is permitted |
| Passing source/chunk identity to the policy | What those sources are classified as |
| Recording the decision | The taxonomy the decision is based on |
| Failing closed when the policy is unavailable | The policy's rules |

The notebook layer is a **policy consumer and an evidence producer**. It is not a policy author.

This matters because the deployment already has a control plane — whatever implements the NIST and DoD requirements. The notebook layer must compose with it, not compete with it.

---

## 3. Policy is injected, not defined here

The layer defines an interface. The deployment supplies an implementation.

```python
class TrustTier(StrEnum):
    T0_LOCAL = "t0_local"
    T1_INTERNAL = "t1_internal"
    T2_EXTERNAL = "t2_external"


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
    obligations: list[str] = Field(default_factory=list)  # e.g. "log", "redact", "review"


class EgressPolicy(Protocol):
    async def decide(self, request: EgressRequest) -> EgressDecision: ...
```

The policy receives **identity, not content**: source IDs, chunk IDs, destination tier, model, purpose. It resolves whatever it needs — classification, ACLs, handling caveats, data-owner rules — from its own store.

That is the whole point. The notebook layer does not know what "sensitive" means in this organisation, and should not pretend to.

### Default implementation

```python
class TierCeilingPolicy:
    """Deployment default: allow T0/T1, deny T2. Replace with the real policy."""

    def __init__(self, max_tier: TrustTier = TrustTier.T1_INTERNAL) -> None:
        self._max = max_tier

    async def decide(self, request: EgressRequest) -> EgressDecision:
        order = [TrustTier.T0_LOCAL, TrustTier.T1_INTERNAL, TrustTier.T2_EXTERNAL]
        allowed = order.index(request.tier) <= order.index(self._max)
        return EgressDecision(
            allowed=allowed,
            reason=f"tier {request.tier} vs ceiling {self._max}",
        )
```

This is a **default**, not a taxonomy. It exists so the system fails closed before the real policy is wired in. Replacing it is a one-line change at composition time.

---

## 4. The gate

```python
async def decide_egress(
    *,
    policy: EgressPolicy,
    request: EgressRequest,
) -> EgressDecision:
    """The only place that decides whether data may leave the device."""
    try:
        decision = await policy.decide(request)
    except Exception:
        decision = EgressDecision(allowed=False, reason="policy error")
    if decision is None:
        decision = EgressDecision(allowed=False, reason="policy returned no decision")
    await log_egress(request, decision)
    return decision
```

Invariants:

1. **One gate.** Every outbound model call passes through `decide_egress`. There is no second path.
2. **Fail closed.** A policy that raises, returns `None`, or is missing is a deny.
3. **The gate never inspects content.** It passes identity and records the answer.
4. **Every decision is logged**, including denials.
5. **A denial is a normal outcome.** The chat path falls back to T0; it does not raise.

The test that matters:

```python
async def test_denied_policy_makes_no_outbound_calls(monkeypatch):
    calls = []
    monkeypatch.setattr(httpx.AsyncClient, "send", lambda *a, **k: calls.append(a))
    await run_chat(notebook=nb, policy=DenyAllPolicy(), tier=TrustTier.T1_INTERNAL)
    assert calls == []
```

---

## 5. Agent surface

The notebook is not only a UI feature. In a skills/tools app it must be reachable by the agent, and gated by the existing `ToolProfile` allowlist.

### 5.1 Tools

Prefix every tool so `ToolProfile.tool_prefixes` can allowlist a group.

| Tool | Purpose | Minimum profile |
|---|---|---|
| `document_read` | path → markdown | all |
| `document_list` | directory → document inventory | all |
| `document_extract_tables` | path → tables | all |
| `notebook_create` | new notebook | manual / smart / bypass |
| `notebook_add_source` | add a source | manual / smart / bypass |
| `notebook_list_sources` | inventory | all |
| `notebook_retrieve` | query → ranked spans | all |
| `notebook_ask` | grounded answer | all |
| `artifact_generate` | enqueue an artifact | manual / smart / bypass |
| `artifact_get` | status + output | all |

Read-only profiles get the read tools and none of the write tools. That falls out of the existing allowlist mechanism with no new code.

**Caveat from the existing research:** `load_capability` must stay exempt from any allowlist, or skills cannot load. Allowlisting `notebook_*` must not filter it.

### 5.2 Skills

Two skills, because the *judgment* is what needs teaching, not the mechanics:

- **`document-handling`** — when to read a file directly vs. when to create a notebook. The rule: one file, one question → `document_read`. Many files, or a question that spans them → notebook.
- **`notebook-research`** — retrieval discipline: cite or refuse, never answer from memory, prefer `notebook_retrieve` before `notebook_ask` when the user wants to see the evidence.

Skills are instruction-only (no bundled scripts), so the `in_run_workspace=False` limitation does not bite.

### 5.3 MCP

Expose the same handlers as an MCP server for external clients. The handlers are the same functions; the MCP layer is a thin adapter.

Do not make MCP the primary path. The in-process tool call is the fast path; MCP is for reach.

### 5.4 A2A / ACP

The notebook is a natural A2A agent ("research agent": capability card, task lifecycle, streaming). **Defer it.** A2A adds a task-state model and a capability-negotiation surface that is not worth building before the notebook itself is stable.

Note the shape now so the boundary stays clean: the A2A agent would wrap `notebook_*` tools, not reimplement them.

---

## 6. What the notebook layer adds to the compliance surface

The existing controls — permission profiles, `ApprovalReviewer`, `ToolCallJudge`, shell/file allowlists — govern **what the agent may do to the machine**. They are execution controls.

The notebook layer is the first feature that:

- **ingests arbitrary documents** the user did not author,
- **retains them** in a local corpus,
- **emits disseminable content** (reports, decks, audio) that a human may forward,
- and **crosses trust boundaries** with retrieved text.

None of those are execution controls. They are new *concerns* the deployment's existing control framework will need to cover:

| Concern | Question it raises |
|---|---|
| Data handling | What did we ingest, how long do we keep it, how is it deleted? |
| Content release | Who approved this artifact before it left? |
| Accuracy claims | How do we know retrieval and citation are correct? |
| Boundary crossing | What left the device, and was it allowed? |

The notebook layer's job is to make each of these **answerable from records**, not to define the answers. The answers belong to the policy and the control framework.

---

## 7. Evidence the layer emits

This is the concrete contribution to whatever NIST / DoD implementation already exists. Each item is a record the layer produces reliably; mapping them to control objectives is the deployment's call.

| Record | Contents | Answers |
|---|---|---|
| **Egress record** | notebook, source IDs, chunk IDs, tier, model, purpose, decision, reason, timestamp | What left, where, and was it allowed? |
| **Provenance record** | source hash, origin, parser + version, ingest timestamp | Where did this text come from, and how was it read? |
| **Generation record** | model, model version, tier, prompt hash, token counts, cost | What produced this, with what? |
| **Evaluation record** | retrieval recall@k, citation-quote validation rate, refusal rate, benchmark version + date | Were the grounding claims tested? |
| **Artifact lineage** | sources → model → reviewer → export timestamp | Who released this, and from what? |

Two notes:

- **The retrieval benchmark is an evaluation record**, not just an engineering tool. Commit it, date it, keep the history — it is the retained evidence that grounding was tested.
- **Provenance marking on artifacts should be embedded in the artifact**, not only in a metadata field, if the deployment requires a forwarded file to still say what it is. Whether that is required is a policy question, not a notebook-layer one.

### Supply chain

The parser dependencies (`pymupdf`, `python-docx`, `python-pptx`, `openpyxl`, `trafilatura`, `faster-whisper`) are new third-party code processing untrusted input. Pin them, record versions, include them in the SBOM. `meta.parser` + `meta.parser_version` on every `ParsedDocument` is a provenance record, not just a cache key.

---

## 8. What changes in the spec

| Document | Change |
|---|---|
| `CODE_BIBLE.md` §9 | Replace the policy enum with tiers + injected `EgressPolicy` |
| `CODE_BIBLE.md` §1 | I-6 becomes "egress fails closed"; I-12 becomes "policy is injected, not defined here" |
| `CODE_BIBLE.md` §4 | Drop `policy` / `classification` columns; policy is runtime-injected |
| `FEATURES.md` §9 | Replace NB-801 with the policy-provider interface |
| `DOCUMENT_LAYER.md` | No classification at ingest — the parser records provenance only |
| `README.md` | Add this document to the reading order |

---

## 9. Open questions

1. **What is the policy implementation?** Is there an existing service the gate should call, or does this need a new one?
2. **Is T2 enabled at all?** If not, the default `TierCeilingPolicy` is the whole policy and the design gets simpler.
3. **Does the policy need source metadata, or only IDs?** Determines whether `EgressRequest` carries a descriptor or the policy resolves it.
4. **Does artifact export require review before release?** If yes, the release gate is a feature; if no, it is not.
5. **Retention period for sources and artifacts?** Drives whether hard-delete is a feature or a policy.
