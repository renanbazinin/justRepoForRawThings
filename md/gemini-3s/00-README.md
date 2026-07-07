# Migration Plan: Vertex AI Gemini 2.5 → Gemini 3.5 Flash (Node.js)

> Target: migrate an enterprise Vertex AI workload from the **Gemini 2.5** series to
> **`gemini-3.5-flash`**, keeping data in **`europe-west1`** where possible.
> Language: **Node.js**.

This is the index. The plan is split into focused files so you can hand each one to a
different owner (platform, app, QA).

| File | What it covers |
|------|----------------|
| **00-README.md** (this file) | Executive summary, breaking-changes matrix, region reality check |
| [01-sdk-and-setup.md](./01-sdk-and-setup.md) | The SDK swap (`@google-cloud/vertexai` → `@google/genai`), install, client init, region/auth |
| [02-code-changes.md](./02-code-changes.md) | Side-by-side before/after for every call pattern |
| [03-request-response-payloads.md](./03-request-response-payloads.md) | The **new** request payload + response JSON (thinking, thought signatures, usage) |
| [04-migration-checklist.md](./04-migration-checklist.md) | Step-by-step rollout, testing, rollback |

---

## TL;DR — the two things that actually break you

1. **The SDK changed.** `@google-cloud/vertexai` (the `VertexAI` / `getGenerativeModel`
   pattern) is **deprecated and end-of-life as of 2026-06-24**. Gemini 3.x is **only**
   exposed through the new **Google Gen AI SDK**, `@google/genai`. This is the biggest
   part of the migration — it's a client-library rewrite, not a model-string swap.

2. **Thinking is now first-class and on by default.** Gemini 3.5 Flash is a reasoning
   model. You control reasoning with **`thinkingLevel`** (`minimal` | `low` | `medium` |
   `high`), *not* the old numeric `thinkingBudget`. During multi-turn function calling you
   must **echo back `thoughtSignature`** blocks or you get a 4xx. Sampling params
   (`temperature`, `topP`, `topK`) are **no longer recommended** — leave them at default.

If you only remember one code change: model access moves from
`vertexAI.getGenerativeModel({model}).generateContent(req)` to
`ai.models.generateContent({ model, contents, config })`.

---

## Breaking-changes matrix (2.5 → 3.5 Flash)

| Area | Gemini 2.5 (old) | Gemini 3.5 Flash (new) | Action |
|------|------------------|------------------------|--------|
| **npm package** | `@google-cloud/vertexai` | `@google/genai` | Replace dependency |
| **Client** | `new VertexAI({project, location})` | `new GoogleGenAI({vertexai:true, project, location})` | Rewrite init |
| **Model handle** | `vertexAI.getGenerativeModel({model})` | none — pass `model` per call | Remove handle |
| **Call** | `model.generateContent(request)` | `ai.models.generateContent({model, contents, config})` | Rewrite calls |
| **Config location** | on the model *and* in `generationConfig` | one `config` object per request | Consolidate |
| **Response text** | `result.response.text()` / `.candidates[0]...` | `response.text` (property) | Update readers |
| **Model ID** | `gemini-2.5-flash` / `gemini-2.5-pro` | `gemini-3.5-flash` | Swap string |
| **Reasoning control** | `thinkingConfig.thinkingBudget` (int) | `thinkingConfig.thinkingLevel` (enum) | Replace param |
| **Thinking default** | off / low unless budgeted | **on, default `medium`** | Re-tune for latency/cost |
| **Multi-turn tools** | no signature needed | **must return `thoughtSignature`** | Preserve history verbatim |
| **`temperature`/`topP`/`topK`** | commonly tuned | **not recommended; keep default 1.0** | Remove overrides |
| **`candidateCount`** | supported | **removed** | Delete param |
| **Media control** | fixed | `mediaResolution` (low→ultra_high) | Optional tuning |
| **Image segmentation** | supported | **unsupported** — keep 2.5 for it | Keep a 2.5 fallback |
| **Chat** | `model.startChat()` | `ai.chats.create({model})` | Rewrite |
| **Region** | `europe-west1` single-region | Prefer `europe-west1`, but verify — see below | Confirm before GA |

---

## ⚠️ Region reality check for `europe-west1`

You said you've confirmed `europe-west1` is available for your project — good, **build to
that**. But this is enterprise, so here is the honest picture from Google's docs and
current availability signals, because it's the one thing that will silently 404 you:

- Gemini 3.x models are commonly served from the **`global`** endpoint and the **`eu`
  multi-region** endpoint (both keep EU workloads GDPR-covered under the Vertex/Gemini
  Data Processing Addendum).
- Several sources note that **single** European regions (e.g. `europe-west3`,
  `europe-west4`) had *limited* single-region availability for early Gemini 3.x. Whether
  `gemini-3.5-flash` is pinned to `europe-west1` for *your project* can differ by
  allow-listing and GA date.

**Do this before writing GA code (2 minutes):**

```bash
# Does europe-west1 actually publish gemini-3.5-flash for your project?
gcloud ai models list \
  --region=europe-west1 \
  --project=YOUR_PROJECT_ID | grep -i "gemini-3.5-flash"

# Or hit the endpoint directly:
curl -s -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://europe-west1-aiplatform.googleapis.com/v1/projects/YOUR_PROJECT_ID/locations/europe-west1/publishers/google/models/gemini-3.5-flash:generateContent" \
  -d '{"contents":[{"role":"user","parts":[{"text":"ping"}]}]}'
```

- **If it returns a completion** → set `location: 'europe-west1'` and you're done.
- **If it 404s / "model not found"** → fall back to `location: 'eu'` (EU multi-region,
  data stays in EU) or `location: 'global'`. All three keep the code identical — only the
  `location` string changes. The migration code below is written so switching is a
  one-line change.

The rest of this plan assumes `europe-west1`; every snippet has the location isolated to a
single constant so you can flip it without touching logic.

---

## Sources

- [Vertex AI SDK migration guide (`@google-cloud/vertexai` → `@google/genai`)](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/deprecations/genai-vertexai-sdk)
- [Gemini 3 Developer Guide](https://ai.google.dev/gemini-api/docs/gemini-3)
- [What's new in Gemini 3.5 Flash](https://ai.google.dev/gemini-api/docs/whats-new-gemini-3.5)
- [Gemini 3.5 Flash model page (Vertex / Gemini Enterprise Agent Platform)](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/gemini/3-5-flash)
- [Thinking](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/thinking) · [Thought signatures](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/thought-signatures)
- [Vertex locations / endpoints](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations)
