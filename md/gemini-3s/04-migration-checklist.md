# 04 — Migration checklist & rollout

A sequenced, low-risk path. Order matters: prove the SDK+region first, then behavior, then
cost, then cut over.

---

## Phase 0 — Verify access (before writing code)

- [ ] Confirm `gemini-3.5-flash` is served for your project in `europe-west1`
      (see the `gcloud` / `curl` probe in [00-README.md](./00-README.md#️-region-reality-check-for-europe-west1)).
- [ ] If it 404s, decide fallback now: `eu` (EU residency) vs `global`.
- [ ] Confirm the runtime principal has `roles/aiplatform.user`.
- [ ] Confirm quota for `gemini-3.5-flash` in the chosen location (quotas are per-model, per-region).

## Phase 1 — SDK swap (mechanical)

- [ ] `npm uninstall @google-cloud/vertexai && npm install @google/genai`
- [ ] Replace `new VertexAI(...)` + `getGenerativeModel(...)` with `new GoogleGenAI({ vertexai:true, project, location })`.
- [ ] Centralize `location` and `MODEL` into config/env (one place to flip).
- [ ] Update enum imports (`HarmCategory`, `HarmBlockThreshold`, `Type`) to come from `@google/genai`.

## Phase 2 — Rewrite calls (behavioral)

- [ ] `model.generateContent(req)` → `ai.models.generateContent({ model, contents, config })`.
- [ ] `result.response.text()` → `response.text` (property).
- [ ] Streaming: iterate the returned object; guard empty `chunk.text`.
- [ ] Move `systemInstruction`, `maxOutputTokens`, `safetySettings`, `tools`, `responseSchema` into one `config`.
- [ ] Chat: `model.startChat()` → `ai.chats.create({ model, config })`; `sendMessage({ message })`.

## Phase 3 — Gemini 3.x-specific fixes (the ones that error)

- [ ] **Delete** every `thinkingBudget`; replace with `thinkingConfig.thinkingLevel`.
- [ ] **Delete** `temperature`, `topP`, `topK`, `candidateCount`.
- [ ] Set an explicit `thinkingLevel` (`minimal`/`low`) if latency/cost matters — default is `medium`.
- [ ] Function calling: append the model's own `content` (with `thoughtSignature`) to history; don't rebuild it. Prefer `ai.chats.create` for multi-turn tools.
- [ ] If you use **image segmentation**, keep a `gemini-2.5-flash` code path — 3.x doesn't support it.

## Phase 4 — Test

- [ ] Golden-set: run 20–50 representative prompts through 2.5 and 3.5, diff outputs for quality/format regressions.
- [ ] JSON/structured-output cases still parse (`responseSchema`).
- [ ] Tool-calling round-trips complete without 4xx (signature handling correct).
- [ ] Streaming consumers handle empty/thought chunks.
- [ ] Latency check per `thinkingLevel` — pick the lowest level that meets quality.

## Phase 5 — Cost & observability

- [ ] Log `usageMetadata.thoughtsTokenCount` — **new billable dimension**; add it to dashboards.
- [ ] Re-baseline cost: `totalTokenCount` now includes thinking; PDFs cost more, video less.
- [ ] Alert on `400`/`404` spikes during rollout (region + removed-param regressions surface here).

## Phase 6 — Rollout

- [ ] Feature-flag the model id so you can flip `gemini-3.5-flash` ⇄ `gemini-2.5-flash` instantly.
- [ ] Canary a small traffic %, watch error rate + thinking-token cost, then ramp.
- [ ] Keep the 2.5 path available until 3.5 is proven in prod (and permanently if you need segmentation).

---

## Minimal end-to-end reference (copy-paste starting point)

```js
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({
  vertexai: true,
  project:  process.env.GOOGLE_CLOUD_PROJECT,
  location: process.env.GOOGLE_CLOUD_LOCATION ?? 'europe-west1', // fallback: 'eu' / 'global'
});
const MODEL = process.env.GEMINI_MODEL ?? 'gemini-3.5-flash';

export async function ask(prompt) {
  const response = await ai.models.generateContent({
    model: MODEL,
    contents: prompt,
    config: {
      systemInstruction: 'You are a helpful enterprise assistant.',
      maxOutputTokens: 1024,
      thinkingConfig: { thinkingLevel: 'low' },   // tune per workload
      // no temperature / topP / topK / candidateCount on 3.x
    },
  });
  return {
    text: response.text,
    thinkingTokens: response.usageMetadata?.thoughtsTokenCount ?? 0,
    totalTokens: response.usageMetadata?.totalTokenCount ?? 0,
  };
}
```

---

## What is NOT changing (reassurance)

- Auth (ADC / service account), IAM role (`aiplatform.user`).
- `contents` / `parts` message shape, `systemInstruction`, `tools` declarations, `responseSchema`.
- Streaming concept, safety-setting concept.
- Your GCP project, billing, and VPC-SC posture.

The migration is really: **new SDK + reasoning-model semantics (thinking level + thought
signatures) + a region confirmation.**
