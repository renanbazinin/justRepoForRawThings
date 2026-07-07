# 03 — New request & response payloads (wire format)

This is what actually goes over HTTP to Vertex, so you can update logging, schema
validation, proxies, and anything that inspects the JSON. The `@google/genai` SDK builds
these for you, but enterprise teams usually need to see the raw shape.

**Endpoint (europe-west1):**
```
POST https://europe-west1-aiplatform.googleapis.com/v1/projects/YOUR_PROJECT_ID/locations/europe-west1/publishers/google/models/gemini-3.5-flash:generateContent
```
(`:streamGenerateContent` for streaming. For the `eu`/`global` fallback, swap the host and
both `location` path segments accordingly — e.g. `https://aiplatform.googleapis.com/v1/.../locations/global/...`.)

> Note: on the wire Vertex uses **camelCase** JSON keys (`generationConfig`,
> `thinkingConfig`, `thinkingLevel`). Some Google AI Studio docs show snake_case
> (`thinking_level`) — that's the API-key backend. For Vertex/enterprise, use camelCase as
> below.

---

## 3.1 Request — Gemini 2.5 (old, for reference)

```json
{
  "contents": [
    { "role": "user", "parts": [{ "text": "Summarize our Q3 results." }] }
  ],
  "systemInstruction": { "parts": [{ "text": "You are a finance analyst." }] },
  "generationConfig": {
    "temperature": 0.2,
    "topP": 0.8,
    "maxOutputTokens": 1024,
    "candidateCount": 1,
    "thinkingConfig": { "thinkingBudget": 1024 }
  }
}
```

## 3.2 Request — Gemini 3.5 Flash (NEW) ✅

```json
{
  "contents": [
    { "role": "user", "parts": [{ "text": "Summarize our Q3 results." }] }
  ],
  "systemInstruction": { "parts": [{ "text": "You are a finance analyst." }] },
  "generationConfig": {
    "maxOutputTokens": 1024,
    "thinkingConfig": {
      "thinkingLevel": "LOW",
      "includeThoughts": false
    },
    "mediaResolution": "MEDIA_RESOLUTION_MEDIUM"
  }
}
```

What changed in the payload:

| Field | Change |
|-------|--------|
| `generationConfig.thinkingConfig.thinkingBudget` | ❌ removed → replaced by `thinkingLevel` (`"MINIMAL"`/`"LOW"`/`"MEDIUM"`/`"HIGH"`) |
| `generationConfig.temperature` / `topP` / `topK` | ⚠️ omit — not recommended on 3.x |
| `generationConfig.candidateCount` | ❌ removed — sending it errors |
| `generationConfig.mediaResolution` | 🆕 optional media/token control |
| `generationConfig.thinkingConfig.includeThoughts` | 🆕 opt-in thought summaries in the response |

Everything else (`contents`, `systemInstruction`, `tools`, `safetySettings`,
`responseMimeType`, `responseSchema`) keeps the same shape.

---

## 3.3 Response — Gemini 3.5 Flash (NEW) ✅

```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          { "text": "Q3 revenue rose 12% QoQ, driven by ..." }
        ]
      },
      "finishReason": "STOP",
      "avgLogprobs": -0.21
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 38,
    "candidatesTokenCount": 210,
    "thoughtsTokenCount": 512,
    "totalTokenCount": 760
  },
  "modelVersion": "gemini-3.5-flash",
  "responseId": "a1b2c3d4-...."
}
```

New / changed response fields:

| Field | Meaning |
|-------|---------|
| `usageMetadata.thoughtsTokenCount` | 🆕 tokens the model spent **thinking**. **Bill this — it's real cost** and did not exist in 2.5. Track it separately for cost dashboards. |
| `totalTokenCount` | Now **includes** `thoughtsTokenCount`. Reconcile any cost math that assumed `prompt + candidates`. |
| `modelVersion` | Confirms you were served `gemini-3.5-flash`. |
| `parts[].thought` | Present only when `includeThoughts: true` — a summary "thought" part, separate from the answer text. Filter it out of user-facing output. |

### Reading it in Node

```js
const text  = response.text;                                  // final answer
const usage = response.usageMetadata;
console.log('thinking tokens:', usage.thoughtsTokenCount);    // 🆕 monitor this
console.log('total tokens:',    usage.totalTokenCount);
```

---

## 3.4 Function-call response — the `thoughtSignature` 🔴

When Gemini 3.5 Flash decides to call a tool, the response part carries a
**`thoughtSignature`** (an encrypted reasoning token). This is the field you must return
verbatim on the next turn (see [02 §2.6](./02-code-changes.md#26-function-calling--thought-signatures)).

```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "functionCall": {
              "name": "get_current_weather",
              "args": { "location": "Boston" }
            },
            "thoughtSignature": "Cg8BAB3f...ENCRYPTED...=="
          }
        ]
      },
      "finishReason": "STOP"
    }
  ],
  "usageMetadata": { "promptTokenCount": 52, "thoughtsTokenCount": 128, "totalTokenCount": 180 }
}
```

Your follow-up request must include that **entire** model `content` object (with
`thoughtSignature` untouched) followed by your `functionResponse`:

```json
{
  "contents": [
    { "role": "user",  "parts": [{ "text": "Weather in Boston?" }] },
    { "role": "model", "parts": [
        { "functionCall": { "name": "get_current_weather", "args": { "location": "Boston" } },
          "thoughtSignature": "Cg8BAB3f...ENCRYPTED...==" }
    ]},
    { "role": "user",  "parts": [
        { "functionResponse": { "name": "get_current_weather",
                                "response": { "temperature": "15C", "conditions": "rain" } } }
    ]}
  ],
  "tools": [ { "functionDeclarations": [ /* ...same tool decls... */ ] } ]
}
```

- ❌ Drop or edit `thoughtSignature` → **4xx validation error**.
- ✅ The SDK's `response.candidates[0].content` already contains it — append that object,
  don't hand-reconstruct the model turn.
- Using `ai.chats.create(...)` handles this automatically.

---

## 3.5 Error you'll most likely hit during migration

| Symptom | Cause | Fix |
|---------|-------|-----|
| `400 INVALID_ARGUMENT: thinking_budget ... thinking_level` | Sent both thinking params | Remove `thinkingBudget` |
| `400` on `candidateCount` | Field removed in 3.x | Delete it |
| `4xx` after a tool call | Dropped `thoughtSignature` | Echo the model `content` verbatim |
| `404 model not found` in `europe-west1` | 3.5 Flash not pinned to that single region for your project | Set `location:'eu'` or `'global'` (see README) |
| `response.text is not a function` | 3.x makes `text` a property | Use `response.text` (no `()`) |

Next: [04-migration-checklist.md](./04-migration-checklist.md).
