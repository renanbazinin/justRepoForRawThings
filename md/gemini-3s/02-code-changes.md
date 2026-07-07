# 02 — Code changes (before / after)

Every pattern your 2.5 code likely uses, shown old → new. All "after" snippets assume the
`ai` client and `MODEL` from [01-sdk-and-setup.md](./01-sdk-and-setup.md).

Two rules cover 90% of the diff:
1. `vertexAI.getGenerativeModel({model}).<method>(request)` → `ai.models.<method>({ model, contents, config })`
2. All tuning (`generationConfig`, `systemInstruction`, `safetySettings`, `tools`) moves
   into a single per-request **`config`** object.

---

## 2.1 Basic generateContent

**Before (2.5):**
```js
const request = {
  contents: [{ role: 'user', parts: [{ text: 'Why is the sky blue?' }] }],
};
const result = await model.generateContent(request);
console.log(result.response.text());          // method call
```

**After (3.5 Flash):**
```js
const response = await ai.models.generateContent({
  model: MODEL,
  contents: 'Why is the sky blue?',            // string shorthand allowed
});
console.log(response.text);                     // property, not a method
```

- `contents` still accepts the full `[{ role, parts:[{text}] }]` array — the string form is
  just a shorthand for a single user turn. Keep the array form for multi-turn/multimodal.
- **`response.text` is a property now**, not `response.text()`. This is a common breakage.

---

## 2.2 Streaming

**Before (2.5):**
```js
const streamResult = await model.generateContentStream(request);
for await (const item of streamResult.stream) {
  process.stdout.write(item.candidates[0].content.parts[0].text);
}
```

**After (3.5 Flash):**
```js
const stream = await ai.models.generateContentStream({
  model: MODEL,
  contents: 'Write a short story about a baby turtle.',
});
for await (const chunk of stream) {
  if (chunk.text) process.stdout.write(chunk.text);
}
```

- You iterate the returned object directly (no `.stream` property).
- Each `chunk.text` may be empty (e.g. thought/tool chunks) — guard it.

---

## 2.3 System instruction + generation config

**Before (2.5)** — config split between the model handle and the request:
```js
const model = vertexAI.getGenerativeModel({
  model: 'gemini-2.5-flash',
  systemInstruction: { parts: [{ text: 'You are a terse translator.' }] },
  generationConfig: { maxOutputTokens: 512, temperature: 0.2 },
});
const result = await model.generateContent(request);
```

**After (3.5 Flash)** — one `config` object per request:
```js
const response = await ai.models.generateContent({
  model: MODEL,
  contents: 'Translate "hello" to French.',
  config: {
    systemInstruction: 'You are a terse translator.',   // plain string works
    maxOutputTokens: 512,
    // ⚠️ do NOT set temperature/topP/topK — see 2.7
  },
});
```

---

## 2.4 Thinking control (the new reasoning knob) 🆕

This replaces the numeric `thinkingBudget`. Gemini 3.5 Flash thinks by default at
**`medium`**.

**Before (2.5) — numeric budget:**
```js
const model = vertexAI.getGenerativeModel({
  model: 'gemini-2.5-flash',
  generationConfig: { thinkingConfig: { thinkingBudget: 1024 } },
});
```

**After (3.5 Flash) — enum level:**
```js
const response = await ai.models.generateContent({
  model: MODEL,
  contents: 'Plan a 3-step migration.',
  config: {
    thinkingConfig: {
      thinkingLevel: 'low',        // 'minimal' | 'low' | 'medium'(default) | 'high'
      includeThoughts: false,      // set true to receive thought summaries in the response
    },
  },
});
```

Level guidance (from Google's docs):

| `thinkingLevel` | Use for | Trade-off |
|-----------------|---------|-----------|
| `minimal` | Simple lookups, classification, high-QPS | Lowest latency/cost; least reasoning |
| `low` | Chat, straightforward tasks | Fast, still strong quality |
| `medium` *(default)* | Most tasks | Balanced |
| `high` | Complex reasoning, hard problems | Slowest/most tokens |

- **Do not send both** `thinkingLevel` and `thinkingBudget` → **400 error**. Delete every
  `thinkingBudget` from your 2.5 code.
- To cut latency/cost right after cutover, explicitly set `minimal` or `low` — otherwise you
  inherit `medium`, which is more thinking (and more tokens) than 2.5's default.

---

## 2.5 Safety settings

**Before (2.5):**
```js
const model = vertexAI.getGenerativeModel({
  model: 'gemini-2.5-flash',
  safetySettings: [{
    category: HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
    threshold: HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
  }],
});
```

**After (3.5 Flash):**
```js
import { HarmCategory, HarmBlockThreshold, HarmBlockMethod } from '@google/genai';

const response = await ai.models.generateContent({
  model: MODEL,
  contents: '...',
  config: {
    safetySettings: [{
      method: HarmBlockMethod.SEVERITY,          // new: SEVERITY | PROBABILITY
      category: HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
      threshold: HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
    }],
  },
});
```

Enums now import from `@google/genai` (not `@google-cloud/vertexai`).

---

## 2.6 Function calling (⚠️ thought signatures)

**Before (2.5):**
```js
const tools = [{
  function_declarations: [{
    name: 'get_current_weather',
    description: 'Get weather in a location',
    parameters: { type: 'OBJECT', properties: { location: { type: 'STRING' } } },
  }],
}];
const result = await model.generateContent({ contents, tools });
```

**After (3.5 Flash):**
```js
import { Type } from '@google/genai';

const getWeather = {
  name: 'get_current_weather',
  description: 'Get weather in a location',
  parameters: {
    type: Type.OBJECT,
    properties: { location: { type: Type.STRING } },
    required: ['location'],
  },
};

const response = await ai.models.generateContent({
  model: MODEL,
  contents: 'What is the weather in Boston?',
  config: { tools: [{ functionDeclarations: [getWeather] }] },
});

// The model returns a functionCall part; run your tool, then send the result back.
const call = response.functionCalls?.[0];
```

**🔴 The 3.x gotcha — thought signatures.** When you send the tool result back for the
model to continue, the assistant turn Gemini produced contains a `thoughtSignature` on its
`functionCall` part. **You must echo that part back verbatim** in the conversation history,
or you get a 4xx validation error.

The safe pattern: **don't rebuild history by hand — append the model's own content.**

```js
// Build history from the model's actual output, preserving thoughtSignature:
const contents = [
  { role: 'user', parts: [{ text: 'What is the weather in Boston?' }] },
  response.candidates[0].content,                        // ← contains functionCall + thoughtSignature, kept intact
  { role: 'user', parts: [{
      functionResponse: {
        name: 'get_current_weather',
        response: { temperature: '15C', conditions: 'rain' },
      },
  }]},
];

const final = await ai.models.generateContent({
  model: MODEL,
  contents,
  config: { tools: [{ functionDeclarations: [getWeather] }] },
});
console.log(final.text);
```

> If you use `ai.chats.create(...)` (§2.9) the SDK maintains signatures for you — prefer it
> for multi-turn tool use to avoid managing signatures manually.

---

## 2.7 Sampling parameters — remove them

Gemini 3.x is tuned for its defaults. Google **strongly recommends against** setting
`temperature`, `topP`, or `topK`; lowering `temperature` below 1.0 can cause looping or
degraded reasoning.

```diff
  config: {
-   temperature: 0.2,
-   topP: 0.8,
-   topK: 40,
-   candidateCount: 1,      // ⚠️ candidateCount is REMOVED in 3.x → error
    maxOutputTokens: 1024,  // this one is fine to keep
  }
```

Audit your 2.5 code for these four fields and delete them.

---

## 2.8 Controlled generation (JSON output)

**Before (2.5):**
```js
const model = vertexAI.getGenerativeModel({
  model: 'gemini-2.5-flash',
  generationConfig: {
    responseMimeType: 'application/json',
    responseSchema: schema,
  },
});
```

**After (3.5 Flash):**
```js
import { Type } from '@google/genai';

const response = await ai.models.generateContent({
  model: MODEL,
  contents: 'List 3 cookie recipes.',
  config: {
    responseMimeType: 'application/json',
    responseSchema: {
      type: Type.ARRAY,
      items: {
        type: Type.OBJECT,
        properties: { recipeName: { type: Type.STRING } },
        required: ['recipeName'],
      },
    },
  },
});
const recipes = JSON.parse(response.text);
```

---

## 2.9 Chat / multi-turn

**Before (2.5):**
```js
const chat = model.startChat({});
const r1 = await chat.sendMessage('Hello');
console.log((await r1.response).text());
```

**After (3.5 Flash):**
```js
const chat = ai.chats.create({
  model: MODEL,
  config: { thinkingConfig: { thinkingLevel: 'low' } },
});
const r1 = await chat.sendMessage({ message: 'Hello' });
console.log(r1.text);

const history = chat.getHistory();   // signatures preserved internally
```

Prefer chat sessions when you do multi-turn function calling — signature bookkeeping is
handled for you (see §2.6).

---

## 2.10 Multimodal + media resolution 🆕 (optional)

Multimodal parts are the same shape. New in 3.x: `mediaResolution` lets you trade tokens
for fidelity on images/PDF/video.

```js
const response = await ai.models.generateContent({
  model: MODEL,
  contents: [
    { text: 'Summarize this document.' },
    { fileData: { fileUri: 'gs://bucket/doc.pdf', mimeType: 'application/pdf' } },
  ],
  config: {
    // MEDIA_RESOLUTION_LOW | _MEDIUM | _HIGH  (per-part _ULTRA_HIGH also exists)
    mediaResolution: 'MEDIA_RESOLUTION_HIGH',
  },
});
```

- Expect **higher token usage on PDFs** and **lower on video** vs 2.5. Recheck cost.
- **Image segmentation is not supported** on 3.x — keep a `gemini-2.5-flash` path if you
  rely on it.

---

## 2.11 Count tokens

**Before:** `await model.countTokens(request)`
**After:**
```js
const t = await ai.models.countTokens({ model: MODEL, contents: 'Hello world' });
console.log(t.totalTokens);
```

Next: [03-request-response-payloads.md](./03-request-response-payloads.md) — the raw wire format.
