# 01 — SDK swap, install & setup

The single largest change: you are moving off the **deprecated** `@google-cloud/vertexai`
library onto the **Google Gen AI SDK** (`@google/genai`). Gemini 3.x is not reachable from
the old SDK, so this step is mandatory, not optional.

---

## 1. Dependency change

```bash
# Remove the deprecated Vertex AI SDK
npm uninstall @google-cloud/vertexai

# Install the Google Gen AI SDK (Vertex/enterprise backend included)
npm install @google/genai
```

`package.json` diff:

```diff
   "dependencies": {
-    "@google-cloud/vertexai": "^1.x"
+    "@google/genai": "^1.x"
   }
```

> Node 18+ recommended. The SDK supports both ESM (`import`) and CJS (`require`).

---

## 2. Client initialization

### Before (Gemini 2.5, deprecated SDK)

```js
const { VertexAI } = require('@google-cloud/vertexai');

const vertexAI = new VertexAI({
  project: 'YOUR_PROJECT_ID',
  location: 'europe-west1',
});

// A model handle you reused for every call:
const model = vertexAI.getGenerativeModel({ model: 'gemini-2.5-flash' });
```

### After (Gemini 3.5 Flash, Google Gen AI SDK)

```js
import { GoogleGenAI } from '@google/genai';

// Keep region + model in one place so you can flip them without touching logic.
const LOCATION = 'europe-west1';          // fallback: 'eu' (EU multi-region) or 'global'
const MODEL    = 'gemini-3.5-flash';

const ai = new GoogleGenAI({
  vertexai: true,                          // use the Vertex / enterprise backend, not the API-key backend
  project: 'YOUR_PROJECT_ID',
  location: LOCATION,
});

// No persistent model handle. You pass `model` on each request instead.
```

Key structural differences:

- **`vertexai: true` is required.** Without it the SDK targets the API-key Gemini
  Developer backend, which is a different product with different data governance.
  (Recent SDK builds also accept `enterprise: true` as an alias reflecting the
  "Gemini Enterprise Agent Platform" rename — `vertexai: true` remains the documented,
  safe choice.)
- **No `getGenerativeModel`.** There is no long-lived per-model object; the model id is a
  per-request field. This is why config also moves to per-request (see file 02).
- **CommonJS** equivalent if you're not on ESM:
  ```js
  const { GoogleGenAI } = require('@google/genai');
  ```

---

## 3. Region / location — the one knob to keep isolated

The `location` you pass to `GoogleGenAI` decides where the request is served and where data
is processed. For your EU requirement:

| `location` value | Meaning | Use when |
|------------------|---------|----------|
| `'europe-west1'` | Single region, Belgium | Your project has 3.5 Flash pinned there (verify — see README) |
| `'eu'` | EU multi-region, data stays in EU | `europe-west1` 404s but you need EU residency |
| `'global'` | Google-managed global routing | Residency not required / best availability |

Because the client is the only place `location` appears, switching endpoints is a
one-line change. Do **not** sprinkle region strings through the codebase.

> There is **no `europe-west1` for the API-key backend** — regions only exist on the
> Vertex/enterprise backend, which is exactly why `vertexai: true` matters for you.

---

## 4. Authentication (unchanged in spirit)

Auth is still Google Cloud ADC (Application Default Credentials) — same as your 2.5 setup:

```bash
# Local dev
gcloud auth application-default login

# Or a service account key
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa-key.json
```

Required IAM role on the service account / principal: **`roles/aiplatform.user`**
(Vertex AI User). No new roles are introduced by Gemini 3.5.

The `@google/genai` SDK reads ADC automatically when `vertexai: true` — you do **not** pass
an API key for the enterprise backend.

---

## 5. Environment-variable pattern (recommended)

Keep everything overridable so promoting through dev→staging→prod (and flipping region on a
404) needs zero code edits:

```js
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({
  vertexai: true,
  project:  process.env.GOOGLE_CLOUD_PROJECT,
  location: process.env.GOOGLE_CLOUD_LOCATION ?? 'europe-west1',
});

export const MODEL = process.env.GEMINI_MODEL ?? 'gemini-3.5-flash';
export { ai };
```

```bash
# .env
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=europe-west1
GEMINI_MODEL=gemini-3.5-flash
```

Next: [02-code-changes.md](./02-code-changes.md) — the actual call rewrites.
