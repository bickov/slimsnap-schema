# SlimSnap Schema

**The open JSON format for screenshots fed to AI agents.**

Terminal-based AI agents — [Claude Code](https://www.anthropic.com/claude-code), [Aider](https://aider.chat), [Codex CLI](https://github.com/openai/codex), [Cursor CLI](https://docs.cursor.com/cli) — can't accept image input. To talk to them about a UI, you have to describe it in English. That's slow, lossy, and the agent still guesses where things are.

SlimSnap is a Mac app that turns any screenshot into a small JSON blob: OCR'd text, element bounding boxes, your annotations. You paste it into the terminal like code. Your agent reads it like code.

This repository is the open, MIT-licensed **specification** of that JSON format. The desktop app that produces it is closed-source (free during launch, with a paid tier planned); the format itself is free for anyone to read, write, validate, or implement.

→ Get the app: **[slimsnap.ai](https://slimsnap.ai)**

---

## Quick example

```json
{
  "schema_version": "2.0",
  "captured_at": "2026-07-09T18:17:46Z",
  "mode": "single",
  "capture": { "width_px": 1440, "height_px": 900 },
  "screen": { "title": "Login — Acme", "app": "Safari" },
  "frames": [
    {
      "index": 0,
      "kind": "spatial",
      "dims": { "width_px": 1440, "height_px": 900 },
      "position": { "scroll_y_px": 0 },
      "elements": [
        { "id": "e1", "type": "label",  "value": "Sign in to your account", "bbox": [0.36, 0.18, 0.28, 0.06] },
        { "id": "e2", "type": "input",  "value": "Email",                   "bbox": [0.36, 0.32, 0.28, 0.06] },
        { "id": "e3", "type": "button", "value": "Sign in",                 "bbox": [0.36, 0.54, 0.28, 0.07], "color": "#3B82F6" }
      ],
      "annotations": [
        { "id": "a1", "type": "arrow", "color": "#EF4444", "from": [0.85, 0.30], "to": [0.55, 0.57], "intent": "highlight" }
      ]
    }
  ],
  "estimated_tokens": 340
}
```

A typical 1440×900 screenshot is ~7,500 vision tokens. The same screen as SlimSnap JSON is ~600 text tokens. **~12× fewer tokens, same information, works anywhere text does.**

---

## Why a format, not just an OCR dump

OCR alone gives you text without spatial meaning. A vision-model embedding gives you spatial meaning without text the agent can quote. SlimSnap gives the agent **both** in a layout it can reason about:

- **`frames`** — the unit of content. A regular screenshot is one frame; a scrollable capture (a whole page, a long chat, an endless feed) is several frames stacked top to bottom. Each frame is one coherent coordinate space.
- **`elements`** — every visible UI primitive with a normalized `bbox` (so coordinates stay sane regardless of the image's pixel size).
- **`annotations`** — what the human pointed at, as structured intent (`highlight`, `explain`, `action`, `question`), not just pixels of a red arrow.
- **`screen`** — optional context (URL, window title, app name) that the model uses to disambiguate.
- **Deterministic IDs** — every element and annotation has a stable string ID, unique across all frames, so the agent can refer back to "e4" instead of guessing "the green button on the right."

The result: agents stop hallucinating about what's in the image. They cite element IDs. They explain what your annotation arrow meant. They produce diffs against specific buttons.

---

## Schema

The formal JSON Schema (draft 2020-12) lives at [`schema/v2.0.json`](schema/v2.0.json). Use it to validate any SlimSnap export. The previous major version stays available at [`schema/v1.0.json`](schema/v1.0.json) with examples in [`examples/v1/`](examples/v1).

### Top level

| Field              | Type    | Required | Notes                                                              |
| ------------------ | ------- | -------- | ------------------------------------------------------------------ |
| `schema_version`   | string  | yes      | `"2.0"` for this spec.                                             |
| `captured_at`      | string  | yes      | ISO-8601 timestamp.                                                |
| `mode`             | string  | yes      | `"single"` (one frame) or `"scroll"` (scrollable capture).         |
| `capture`          | object  | yes      | `{ width_px, height_px }` of the WHOLE capture, all frames stacked. |
| `screen`           | object  | no       | `{ title?, app?, url? }`. Where the screenshot came from.          |
| `frames`           | array   | yes      | The content, top to bottom. At least one frame.                    |
| `ai_enrichment`    | object  | no       | Reserved for LLM-generated metadata. Absent when not enriched.     |
| `estimated_tokens` | integer | yes      | Approximate token count if fed to an LLM. For routing decisions.   |

### Frame

```ts
{
  index:    number;                      // 0-based, top to bottom
  kind:     "spatial";                   // frames stack vertically (video will add "temporal")
  dims:     { width_px, height_px };     // THIS frame's pixel size
  position: { scroll_y_px: number };     // top edge's offset inside the whole capture
  images?:  [{ file, offset_y_px, height_px }];  // present only when saved to disk
  elements:    Element[];                // coordinates normalized 0..1 within THIS frame
  annotations: Annotation[];             // same coordinate space
}
```

Why frames: a 9,000px-tall scroll capture is useless as one coordinate space (a text line rounds to a 0.001-high bbox) and exceeds most canvas/vision limits as one image. Frames keep every capture — however long — in human-scale pages, each internally consistent. To get absolute pixel position of anything: `y_px = position.scroll_y_px + bbox_y * dims.height_px`.

### Element

```ts
{
  id:    string;            // stable identifier, e.g. "e1"
  type:  "text" | "button" | "input" | "link" | "image" | "label" | "unknown";
  value: string | object;   // OCR text, or richer structured value
  bbox:  [x, y, w, h];      // normalized 0..1 floats
  color?: string;           // average RGB hex of the element region, e.g. "#3B82F6"
}
```

### Annotation

```ts
{
  id:    string;            // "a1"
  type:  "arrow" | "rectangle" | "highlight" | "callout" | "note";
  color: string;            // hex, e.g. "#EF4444"

  // Geometry — only the relevant fields per type are set:
  from?:     [x, y];        // arrow start
  to?:       [x, y];        // arrow end
  position?: [x, y];        // single point (notes)
  bbox?:     [x, y, w, h];  // rectangle / highlight / callout

  text?:       string;      // callout / note copy
  target_ref?: string;      // ID of the element this points at
  intent?:     "highlight" | "explain" | "action" | "question";
}
```

All coordinates are **normalized 0..1** floats relative to the FRAME's dimensions (`frames[].dims`), so the JSON survives image resizing and rescaling. Element and annotation IDs are unique across the whole capture, so a `target_ref` near a frame boundary may point into the adjacent frame.

---

## Validate

Any JSON Schema 2020-12 validator works. With [Ajv](https://ajv.js.org):

```bash
npm install -g ajv-cli ajv-formats
ajv validate -s schema/v2.0.json -d examples/annotated-screenshot.json --spec=draft2020 -c ajv-formats
ajv validate -s schema/v2.0.json -d examples/scroll-capture.json --spec=draft2020 -c ajv-formats
```

Should print `... valid` for each.

---

## Versioning

This spec follows semantic versioning at the field level:

- **Patch** (`1.0.x`) — clarifications, fixed typos, expanded enums. Backward compatible.
- **Minor** (`1.x`)  — new optional fields. Backward compatible.
- **Major** (`2.x`)  — breaking changes (renamed/removed fields, type changes).

The `schema_version` field is a string and may be a major.minor pair (e.g. `"2.1"`). Producers should emit the highest version they fully implement.

**v1 → v2:** v2's breaking change is the frame container. v1's top-level `image` / `elements` / `annotations` became `capture` + `frames[].elements` / `frames[].annotations`. A v1 document reads as a v2 document with one implicit frame: `mode: "single"`, `dims` = `image` dims, `position.scroll_y_px: 0`. Consumers that want to accept both should branch on the presence of `frames`.

---

## Use it

If you're building anything that takes a screenshot and feeds an agent, you're welcome to read or write this format directly — no payment, no attribution required. The desktop app at [slimsnap.ai](https://slimsnap.ai) is the easiest producer; nothing stops you from writing your own.

A few obvious adjacencies if you want to pick up the thread:

- A browser extension that emits SlimSnap JSON from a webpage selection.
- A Linux/Windows producer using their respective OCR APIs.
- A CLI that turns a saved PNG + manual annotations into SlimSnap JSON for batch workflows.
- A Claude Code / Aider slash-command that ingests SlimSnap JSON files directly.

Pull requests welcome on the spec itself — for new element types, new annotation `intent` values, new top-level fields. Keep them additive (minor version) where possible.

---

## License

MIT. Use it however you want.

---

Built by [@bickov](https://github.com/bickov) · spec lives at [github.com/bickov/slimsnap-schema](https://github.com/bickov/slimsnap-schema) · the app at [slimsnap.ai](https://slimsnap.ai)
