# Theme Transplant — borrow a template's look, keep the PptxGenJS engine

> Single-file proposal. Status: **draft / exploring**. Owner: renan.

## TL;DR

Add a **third, middle path** between the two existing slide-generation modes. Today you either:

- **Default** — your 10 code-rendered layouts in *your* colors (PptxGenJS, client-side), or
- **Template-native** — the user's actual `.pptx` masters, slot-filled (pptx-automizer, server-side).

The gap: a user has a branded deck but wants *your* clean layouts, just wearing *their* brand. The middle path — **"theme transplant"** — extracts the brand's **color palette + fonts (+ optional logo/background)** from an uploaded `.pptx` and feeds them into the existing default PptxGenJS pipeline as a `PresentationTheme`. Same renderer, same 10 layouts, same masters — only the theme tokens change.

```
 DEFAULT                      ←  THIS CHANGE  →             TEMPLATE-NATIVE
 (PptxGenJS)                   (theme transplant)           (pptx-automizer)
┌────────────────┐         ┌────────────────────┐        ┌──────────────────┐
│ YOUR 10 layouts│         │ YOUR 10 layouts    │        │ THEIR layouts    │
│ YOUR colors    │         │ THEIR colors+fonts │        │ THEIR colors     │
│ code-rendered  │         │ code-rendered      │        │ THEIR masters    │
│                │         │ (theme transplant) │        │ slot-filled      │
└────────────────┘         └────────────────────┘        └──────────────────┘
  you own design            you own LAYOUT,               they own everything
                            they own THE LOOK
```

## Why

1. **Real demand, no current answer.** Users with a brand deck have exactly two options today, and both are wrong for "I like your layouts, just match my brand." Default ignores their brand; template-native forces them onto their own (often messy) masters and the heavier server-side automizer flow.
2. **It's cheap.** The bridge is almost entirely *existing* code. `PresentationTheme` (`shared/types/presentation.ts:17`) is already the exact token bag PptxGenJS consumes — `primaryColor`, `secondaryColor`, `accentColor`, `fontFamily`, `logoUrl`, `footerText`. Every master in `client/src/engine/masters/index.ts` is already driven entirely off it. `style-extractor.ts` already cracks the `.pptx` zip with JSZip and parses OOXML. The renderer, masters, layouts, and generate-slide flow **do not change at all.**
3. **It dodges the automizer trap.** "Use their real masters in PptxGenJS" is structurally impossible — `defineSlideMaster()` only accepts simple rects/lines/text/images, not native OOXML masters. That's *why* template-native exists. By scoping this to **tokens only**, we get a clean, robust win and avoid reinventing automizer.

## The constraint that defines scope

A template gives you two fundamentally different kinds of design data:

```
PptxGenJS consumes TOKENS.          automizer consumes STRUCTURE.
  • hex colors                        • real slide masters
  • font names                        • native placeholder shapes
  • a background image                • the template's actual OOXML
  • numbers (sizes, positions)        • brand graphics, SmartArt, etc.
```

| What you pull from the template | Goes into PptxGenJS? | In scope? |
|---|---|---|
| Color palette (accent1–6, dk/lt) | ✅ → `theme.primaryColor` etc. | **Yes** |
| Fonts (major/minor) | ✅ → `theme.fontFamily` | **Yes** |
| Logo / footer | ✅ already supported | **Yes (reuse)** |
| Master **background as a flat image** | ✅ → master `background:{path}` | **Stretch / phase 2** |
| The template's **real masters/layouts** | ❌ structurally impossible | **No → that's automizer** |

**This is a theme transplant, not a master transplant.** The decks wear the brand's colors and fonts (and maybe a background image), still rendered by your own 10 layouts. If a user needs the template's *actual* master shapes, they want template-native — not this.

## What changes

### A. New: theme-token extractor (the only genuinely new code)

`server/src/pptx/automizer/theme-extractor.ts` (new) — reads `ppt/theme/theme1.xml` from the uploaded buffer and maps it to a partial `PresentationTheme`.

```ts
export interface ExtractedThemeTokens {
  primaryColor?: string;    // from <a:clrScheme> accent1
  secondaryColor?: string;  // accent2
  accentColor?: string;     // accent3 (or accent2 if only 2 distinct)
  fontFamily?: string;      // <a:fontScheme><a:majorFont><a:latin typeface=...>
  palette: string[];        // accent1..6 + dk1/lt1, deduped — for diagnostics / future use
  warnings: string[];       // theme-color refs we couldn't resolve to srgb, etc.
}

export async function extractThemeTokens(buffer: Buffer): Promise<ExtractedThemeTokens>;
```

- Parse `<a:clrScheme>`: resolve `accent1..6`, `dk1/lt1` to six-digit hex. `<a:sysClr lastClr="...">` and `<a:srgbClr val="...">` both resolve; unresolved scheme refs go to `warnings`.
- Parse `<a:fontScheme>`: `majorFont` → `fontFamily`. Ignore `minorFont` for v1 (PptxGenJS uses one font per run; major reads as the brand font).
- Never throws — returns `{ palette: [], warnings: ['...'] }` on any parse failure (mirror `style-extractor.ts`'s swallow-and-degrade contract).
- Reuses the existing JSZip + regex-OOXML approach from `style-extractor.ts`; no new XML dependency.

### B. New endpoint (thin)

`POST /api/extract-theme` — multipart upload of a `.pptx`, returns `{ success, data: ExtractedThemeTokens }`. Validates `isPptxBuffer` first (reuse `constants.ts`). Rate-limited like the rest of `/api`.

> Could piggyback on the existing `/api/extract-template` instead of a new route — decide in **Open question 3**.

### C. Client: theme source on the default flow (NOT a third pipeline)

In the default-mode input screen, add an optional **"Match a brand deck"** affordance: upload `.pptx` → call `/api/extract-theme` → merge non-empty tokens over the plan's theme **after** `/api/plan` returns, **before** generation starts.

```
        upload .pptx (optional)
             │
             ▼
   ┌──────────────────────┐
   │  /api/extract-theme  │  ← NEW (reads ppt/theme/theme1.xml)
   └──────────┬───────────┘
              │ ExtractedThemeTokens
              ▼
   ┌──────────────────────┐
   │ /api/plan (UNCHANGED)│  ← LLM still plans your 10 layouts
   └──────────┬───────────┘
              │ plan.theme = { ...plan.theme, ...extracted }   ← merge, extracted wins
              ▼
   ┌──────────────────────┐
   │ generatePptx (CLIENT)│  ← existing PptxGenJS engine, UNTOUCHED
   └──────────────────────┘
```

- Merge rule: extracted tokens win over LLM-chosen tokens; any field the extractor left empty keeps the LLM's value (graceful partial extraction).
- Persist the extracted tokens on the history entry so re-opens and regenerates stay on-brand. Reuse the existing `logoUrl`/`footerText` wiring.

### D. Out of scope (explicitly)

- Lifting the template's real masters/layouts (→ template-native).
- Master-background-as-image (tracked as **phase 2** below; ships separately if at all).
- `minorFont`, per-placeholder font sizing, theme gradients/effects.
- Any change to the template-native pipeline.

## Design decisions

- **Option, not mode.** This is the **default flow with a theme source swap**, not a new state-machine axis. `App.tsx`'s `mode` stays `'default' | 'template-native'`. Avoids a third pipeline and keeps the UX fork to one clear question. (See Open question 2 — this is the recommended answer.)
- **Tokens-only fidelity.** Recolor/re-font the *existing* flat masters. Do **not** chase the template's background graphics in v1.
- **Extractor degrades gracefully.** Partial extraction is a feature: any token we can't resolve falls back to the LLM's choice, never blocks generation.
- **Server-side extraction.** Keep OOXML parsing on the server next to `style-extractor.ts`, even though the default render is client-side — consistency and reuse of `isPptxBuffer`/JSZip.

## Risks / unknowns

- **Color role mapping is a heuristic.** `accent1→primary, accent2→secondary, accent3→accent` is a guess; some brand themes put their hero color in `dk2` or `accent4`. Mitigation: extract the full `palette[]`, ship the simple mapping, let the user re-order in a follow-up if it's wrong. Surface the chosen mapping in the UI so it's not a black box.
- **Contrast.** Borrowed colors might render dark-on-dark in your masters (e.g. a brand whose accent1 is near-white). Mitigation: a contrast guard could swap text color — note as a possible follow-up, not v1 scope.
- **Two reasons to upload a `.pptx`.** Users could confuse "borrow the look" with "use the template." Mitigation: distinct entry points with explicit copy — "Borrow colors & fonts (our layouts, fast)" vs. "Use this template exactly (their layouts)."
- **Font availability.** A borrowed font not installed on the viewer's machine falls back at open time — same existing limitation as any `fontFamily`; acceptable, but worth a tooltip.

## Phase 2 (separate, optional) — background fidelity

If tokens-only feels too plain, a later change could rasterize/lift the template master's background as a flat image and set it via PptxGenJS master `background: { path }`. This buys "looks like their deck" without automizer, but starts chasing fidelity automizer already does better. **Deliberately deferred** — ship tokens first, see if it's even needed.

## Open questions

1. **Background fidelity** — tokens-only for v1 (recommended), or include master-background-as-image now? *Leaning: tokens-only; background is phase 2.*
2. **Mode vs. option** — third `mode` value, or an option on the default flow? *Leaning: option on default flow.*
3. **New endpoint vs. reuse** — dedicated `POST /api/extract-theme`, or extend `/api/extract-template` to also return theme tokens? *Leaning: dedicated thin route; extraction is cheap and independently useful.*
4. **Color role mapping** — ship the fixed `accent1/2/3` mapping, or let the user pick which palette swatch maps to primary/secondary/accent? *Leaning: fixed mapping v1, manual override later.*

## Rough task outline

> Not committed scope — a sketch to size the work. ~1 new file + 1 route + 1 client affordance.

### 1. Server — extractor
- [ ] 1.1 `server/src/pptx/automizer/theme-extractor.ts`: `extractThemeTokens(buffer)` parsing `<a:clrScheme>` + `<a:fontScheme>` from `ppt/theme/theme1.xml`, never-throws contract.
- [ ] 1.2 Resolve `<a:srgbClr>` and `<a:sysClr lastClr>`; push unresolved scheme refs to `warnings`.
- [ ] 1.3 Map accent1/2/3 → primary/secondary/accent; majorFont → fontFamily; collect full `palette[]`.

### 2. Server — route
- [ ] 2.1 `POST /api/extract-theme` (multipart), `isPptxBuffer` guard, `{ success, data }` envelope, wired in `app/index.ts`.

### 3. Shared
- [ ] 3.1 Export `ExtractedThemeTokens` from `shared/` (re-export shim) so client + server share the type.

### 4. Client
- [ ] 4.1 "Match a brand deck" optional upload on the default input screen → `extractTheme()` typed fetch wrapper in `api/client.ts`.
- [ ] 4.2 Merge extracted tokens over `plan.theme` (extracted wins on non-empty) before generation kicks off.
- [ ] 4.3 Show the resolved palette + chosen mapping (transparency, not a black box).
- [ ] 4.4 Persist extracted tokens on the history entry; survive re-open / regenerate.

### 5. Manual verification
- [ ] 5.1 Upload 2–3 real brand decks; confirm generated PptxGenJS deck adopts colors + font; confirm graceful fallback on a theme-less / malformed file.

## Why this is smaller than it sounds

The renderer, masters, layouts, plan prompt, slide prompt, and generate-slide flow are all **untouched**. The change is one ~120-line extractor, one thin route, and a theme-merge wire-through on the client. Everything rides on the existing default engine. The discipline is staying token-only and resisting the pull toward real-master fidelity — that pull leads straight back into automizer's job.
