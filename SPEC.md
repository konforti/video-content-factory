# Video Content Factory — Spec

A single-file, browser-based **node/flow editor** for composing Cloudinary
media pipelines (and a few mock AI media-generation steps), then packaging a
flow as a named **applet** with a documented HTTP API contract.

The whole product lives in [`index.html`](./index.html). No build step, no
backend, no framework. Everything below describes what's actually in that file
today — not where it might go next.

---

## 1. At a glance

- **Form factor.** One HTML file. Vanilla JS in a single `(function () { ... })()`
  IIFE. CSS in `<style>`. The page is a 2-column grid (palette + canvas) with a
  topbar.
- **What you do with it.** Drag blocks from the left palette onto the canvas,
  wire them, and the right-hand panes preview the resulting Cloudinary
  delivery URL or rendered video / image / text live as you type.
- **What it produces.** A Cloudinary URL of the form
  `https://res.cloudinary.com/<cloud>/<image|video|raw>/upload/<chain>/<public_id>.<ext>`,
  plus an embeddable Cloudinary Video Player URL when the terminal is an
  output-video block.
- **Where state goes.** `localStorage` (saved flows, saved applets) and
  `?flow=<name>` in the URL (the currently-loaded flow). Flows can also be
  exported / imported as JSON files (`*.flow.json`).
- **Cloud is hardcoded.** `VIDEOFACTORY_CLOUD_NAME = "videofactory"`. Every
  pipeline targets that one Cloudinary cloud.

### Goals
- Let an author build a Cloudinary transformation chain visually instead of
  hand-writing the URL.
- Let the same canvas express AI-style steps (image → video, text → image,
  AI text) with placeholder results, so the wiring story is unified — even
  though the AI parts are mocks today.
- Treat a saved flow as a **callable workflow**: bind its loose ends to
  named API parameters, derive its output kind, and document the resulting
  REST surface.

### Non-goals (today)
- No actual applet runtime / no `POST /applets/.../run` server. The "API"
  panel is a documentation surface for the contract a future runtime would
  honour.
- No real AI calls. The mock generator blocks substitute fixed Cloudinary
  sample assets (`samples/dance-2`, `samples/woman`) and tidy upstream text.
- No multi-cloud, no sign-in, no upload UI beyond the Cloudinary Media
  Library widget pop-up for picking an existing asset.
- No collaboration / multi-user state. `localStorage` is the source of truth.

---

## 2. Glossary

- **Block / node.** A draggable card on the canvas; one entry in the global
  `TYPES` table. Has a `type`, `title`, optional input ports (`ins`), output
  ports (`outs`), `defaultSettings`, `settingsFields`, an icon, and (for
  Cloudinary blocks) a `cldKind` that controls how it compiles to URL pieces.
- **Port.** A typed connection point on a block. `id` is local to the block
  (e.g. `in`, `out`, `text`, `layer`, `image`, `prompt`). A port can declare
  `multiple: true` to accept fan-in from many wires (only `splice.in`,
  `mock_ai_text.text`, and `mock_ai_text.image` do today).
- **Edge / wire.** `{ id, fromNode, fromPort, toNode, toPort }`. Stored in a
  flat `edges` array. Rendered as cubic-bezier SVG paths between port anchors.
- **Flow.** The whole canvas state — nodes + edges + viewport + id sequences.
  Serialized as a flow document (see §6).
- **Terminal output.** An output block that materialises a result the flow
  is "for". One of `output` (video chain), `output_video_player`,
  `output_image`, `output_text`. Anything else is an intermediate step or a
  preview-only output. A flow can have at most **one terminal** to be a valid
  applet; extras are tolerated on the canvas.
- **Applet.** A saved flow promoted to a callable contract — slug = saved
  flow name, plus per-canvas-input role/param-name bindings and a single
  derived output kind.

---

## 3. Runtime architecture

### 3.1 File layout
- `index.html` — markup, CSS, JS (≈ 7.3k lines).
- `README.md` — title only.
- No package manifest, no transpile, no bundler, no tests.

### 3.2 External dependencies (loaded from CDNs at runtime)
| Purpose | Source | Loaded |
|---|---|---|
| Inter + JetBrains Mono fonts | `fonts.googleapis.com` | eagerly via `<link>` |
| Cloudinary Video Player JS + CSS | `cdn.jsdelivr.net/npm/cloudinary-video-player` | lazily, the first time an `output_*video*` block needs to mount a player (`ensureCloudinaryVideoPlayerLib`) |
| Cloudinary Media Library widget | `media-library.cloudinary.com/global/all.js` | lazily, the first time an Input block's "Browse" button is clicked (`loadMediaLibraryScript`) |
| Cloudinary embed iframe | `player.cloudinary.com/embed/?…` | rendered as `<iframe>` for embed-mode previews |

### 3.3 Top-level JS state
All scoped inside the IIFE:
- `nodes: Node[]`, `edges: Edge[]`, `nodeIdSeq`, `edgeIdSeq`.
- Viewport: `panX, panY, zoom` (zoom clamped `MIN_Z = 0.25` … `MAX_Z = 2`).
- Selection / interaction: `selectedNodeId`, `selectedEdgeId`,
  `draggingNode`, `wiring`, `inputWirePull`, `canvasPanPending`,
  `tool: "select" | "hand"`, `spaceDown`.
- Output player registry: `outputCldPlayers: Map<nodeId, CldPlayer>`,
  `outputPlayerRenderGen` (incremented on every render so async player init
  callbacks can detect they're stale).
- Applet editor: `appletEditingSlug`, `appletSavedInputsSnapshot`,
  `appletEditorCleanSnapshot`, `appletEditorPersistedBindingIds`,
  `appletEditorPersistedInputsMap`.
- Dirty tracking: `cleanFlowFingerprint` (string hash of canvas content),
  used to flip the topbar `· unsaved` badge.

### 3.4 Hardcoded constants worth knowing
| Constant | Value | Used for |
|---|---|---|
| `VIDEOFACTORY_CLOUD_NAME` | `"videofactory"` | every Cloudinary URL we build |
| `CLOUDINARY_PLAYER_PROFILES` | `["cld-default", "cld-looping", "cld-adaptive-stream"]` | dropdown on `output_video_player` block |
| `MOCK_AI_VIDEO_PUBLIC_ID` | `"samples/dance-2"` | mock i2v / similar |
| `MOCK_AI_IMAGE_PUBLIC_ID` | `"samples/woman"` | mock i2i / t2i |
| `MOCK_AI_TEXT_FALLBACK` | `"[mock] (no upstream text/asset detected — placeholder result)"` | empty-result text |
| `FLOW_DOC_VERSION` | `1` | flow JSON contract version |
| `FLOW_STORE_KEY` | `"videofactory.workflow.store.v1"` | localStorage key for saved flows |
| `FLOW_QUERY_PARAM` | `"flow"` | URL param for the active flow |
| `BUILTIN_CANVAS_VERSION` | `3` | bumping it forces a re-seed on next boot |
| `BUILTIN_CANVAS_VERSION_KEY` | `"videofactory.workflow.builtinCanvasVersion"` | sticky boot version |
| `APPLET_STORE_KEY` | `"videofactory.applets.store.v1"` | localStorage key for applets |
| `APPLET_RECORD_VERSION` | `1` | applet record contract version |

### 3.5 Cloudinary credentials
- The cloud name is hardcoded.
- The Media Library widget needs an API key; the app reads it from
  `window.VIDEOFACTORY_CLD_API_KEY` (no default, no UI to set it). If
  unset, the widget is created without `api_key` and Cloudinary's own login
  flow takes over inside the popup.
- No API secret is used, ever. Anything that would need it (e.g. listing the
  account's actual Video Player profiles) is left as a static curated list.

---

## 4. UI surface

### 4.1 Layout
```
┌──────────────────────────────────────────────┐
│  Studio / <flow-name>      [Run] [Save] [Applet]
├──────────┬───────────────────────────────────┤
│          │                                   │
│  Add     │   Canvas                          │
│  blocks  │   (pan / zoom / nodes / wires)    │
│  search  │                                   │
│          │                  [⌘ dock: select / hand
│  list    │                   zoom-out / % / zoom-in / fit]
└──────────┴───────────────────────────────────┘
```

- **Topbar.**
  - `Studio / <flow-name>` — clicking the flow name opens the **Load flow**
    popover. Untitled flows show "Untitled" in italics. A diverged-from-saved
    canvas adds an amber "· unsaved" suffix.
  - **Run flow** (⌘↵) — re-validates outputs, refreshes preview URLs,
    flashes a status toast. Doesn't actually call any backend.
  - **Save flow** — opens a name popover anchored to the button.
  - **Applet** — opens the per-flow applet editor (requires the flow to be
    saved first).
- **Palette.** Searchable list grouped into categories (Inputs, Cloudinary,
  Mock AI, Outputs). Drag a row onto the canvas to instantiate.
- **Canvas.**
  - Pan: middle-click drag, space-drag, or "hand" tool drag.
  - Zoom: ⌘-wheel or dock buttons; `Fit view` button frames all nodes.
  - Select tool: click selects, shift-drag wires, drag-on-empty pans on
    threshold (otherwise clears selection).
  - Wires: cubic bezier; clicking an edge selects it; deleting a selected
    edge or node removes it (and dependent edges).
  - Output blocks render their Cloudinary preview directly inside the node
    sheet (`<video>`, `<img>`, Cloudinary Player div, or readonly text).
- **Dock** (bottom-right of canvas): tool toggles, zoom controls, %, fit.
- **Toast.** `#flow-toast` is a popover-API status line for save/load /
  validation messages.

### 4.2 Dialogs (`<dialog>`-based, shown as anchored popovers)
- **Save flow** (`#flow-save-dialog`) — name input + Save / Cancel. Enter
  saves. Click outside or Escape closes.
- **Load flow** (`#flow-load-dialog`) — list of saved flows with Load /
  Export / Delete per row + "Import from file…" leading action.
- **Applet editor** (`#applet-edit-dialog`) — see §8.

---

## 5. Block catalog

Every block is one entry in the `TYPES` array. The shape is roughly:
```ts
type BlockDef = {
  type: string;                       // unique slug, persisted
  title: string;                      // shown in palette + node header
  iconBg, glyphColor, wireColor, dotColor: string;
  ins?: { id: string; label: string; multiple?: boolean }[];
  outs?: { id: string; label: string }[];
  defaultSettings: object;
  settingsFields?: SettingsFieldDef[]; // rendered inside the node sheet
  cldKind?: string;                    // present → contributes URL pieces
  mockGenerator?: { kind: "video" | "image" | "text" };
  placeholder?: string;
};
```
Settings field types: `text`, `textarea`, `select` (with `{ v, label }[]` options).

### 5.1 Inputs
| Block | `cldKind` | Default settings | Notes |
|---|---|---|---|
| `input_video` | — | `{ cloudName, publicId: "/samples/dance-2", resourceType: "video" }` | "Browse" opens the Media Library widget. |
| `input_image` | — | `{ cloudName, publicId: "cld-sample", resourceType: "image" }` | |
| `input_text` | — | `{ prompt: "" }` | The text the rest of the canvas can wire into. |

### 5.2 Cloudinary transformation blocks
All have `cldKind` and one `in`/`out` port unless noted. The `cldKind`
controls how `nodeTransformationPieces(node)` produces URL fragments.

| Block | `cldKind` | What it emits |
|---|---|---|
| `cld_raw` *(Manual transform)* | `raw` | `settings.chain` split on `/` or newline → each piece becomes a raw URL segment. |
| `cld_resize` | `resize` | `c_<crop>,w_<w>,h_<h>,ar_<ar>,g_<gravity>` (only set keys). |
| `cld_trim` | `trim` | `so_<start_offset>,eo_<end_offset>,du_<duration>`. |
| `cld_video_preview` *(Video summarization)* | `video_preview` | `e_preview[:duration_X][:max_seg_Y][:min_seg_dur_Z]`. |
| `cld_video_frame` *(Video → image)* | `video_frame` | `so_<offset>,f_<format>,w_,h_`; flips the URL extension to the image format. |
| `cld_effect` *(Video effect)* | `effect` | Picks one of 11+ presets (blur, accelerate, decelerate, reverse, boomerang, loop, vignette, fade_in, fade_out, noise, grayscale, custom) — see `CLD_VIDEO_EFFECT_PRESETS`. Emits `e_<name>[:<args>]`. |
| `cld_round` *(Round corners)* | `round` | `r_<max\|px>`. |
| `cld_adjust` *(Adjust)* | `adjust` | One `e_<name>:<value>` per non-empty {brightness, contrast, saturation, vignette}. |
| `cld_border` | `border` | `bo_<spec>` e.g. `5px_solid_black`. |
| `cld_text_overlay` *(Text overlay)* | `text_overlay` | Two-piece overlay: `l_text:<font_family>_<size>[_<weight>]:<text>,co_<color>` then `fl_layer_apply,g_,x_,y_`. The `text` input port can be wired (then it wins over any literal). |
| `cld_image_overlay` *(Image / watermark)* | `image_overlay` | `l_<asset>,o_<opacity>` + `fl_layer_apply,g_,x_,y_`. The `layer` input port can be wired to an `input_image` (then it wins over the literal `settings.asset`). |
| `cld_delivery` *(Format & quality)* | `delivery` | `f_<format>,q_<quality>,vc_<codec>`. Drives the URL extension when no `video_frame` is present. |
| `cld_rotate` | `rotate` | `a_<angle\|vflip\|hflip\|…>`. |
| `cld_audio` | `audio` | `e_volume:<vol>` (or `e_volume:mute` if `mute` / `0`). |
| `cld_transcribe` *(Auto transcription)* | `transcribe` | **Trigger only.** Emits no URL pieces today. Cloudinary auto-saves `<id>.transcript` raw files alongside the video. Future block: a Video Player text-track consumer. |
| `cld_auto_chapters` *(Auto chapters)* | `auto_chapters` | **Trigger only.** Cloudinary saves `<id>-chapters.vtt`. Future Video Player block consumes it. |
| `cld_ai_details` *(AI video details)* | `ai_details` | **Trigger only.** Cloudinary saves AI title/description/tags as context metadata. |
| `splice` | `splice` | Multi-fan-in. First wire = base video; each subsequent wire becomes `fl_splice[:transition_(name_<x>)],l_video:<id>/…/fl_layer_apply`. Per-segment per-gap transition (`SPLICE_TRANSITIONS`, 40+ presets — `cut`, fade, dissolve, wipe, slide, smooth, circle/rect, diag, slice, squeeze, …). Per-segment transforms wired between an `input_video` and the splice are inlined inside the segment's layer. |

### 5.3 Mock AI generators
| Block | `mockGenerator.kind` | Inputs | Substitutes |
|---|---|---|---|
| `mock_i2v_frames` *(Image → video, frames)* | `video` | `image`, `endImage`, `prompt` | `MOCK_AI_VIDEO_PUBLIC_ID` |
| `mock_i2v_refs` *(Image → video, refs)* | `video` | `image`, `ref1`, `ref2`, `prompt` | `MOCK_AI_VIDEO_PUBLIC_ID` |
| `mock_i2i` *(Image → image)* | `image` | `image`, `prompt` | `MOCK_AI_IMAGE_PUBLIC_ID` |
| `mock_t2i` *(Text → image)* | `image` | `prompt` | `MOCK_AI_IMAGE_PUBLIC_ID` |
| `mock_ai_text` *(AI text)* | `text` | `text` (multi), `image` (multi) | Prefers wired text (joined and `mockAiTextTidy`'d); else `settings.prompt`; else "[mock AI text] Synthesized response from N images."; else "[mock AI text] No inputs wired …". |

`mockAiTextTidy` collapses whitespace, sentence-cases, ensures terminal
punctuation. Cycles through `mock_ai_text` are guarded by a `visited` set.

### 5.4 Outputs
| Block | Behaviour |
|---|---|
| `output` *(Output · video)* | Compiles full chain into a Cloudinary delivery URL; mounts the Cloudinary Video Player against that URL. |
| `output_video_player` *(Output · video player)* | Two modes via `settings.mode`: `"video"` (chain delivery URL, like above) and `"profile"` (uses Cloudinary Player profile `settings.profile` — the cloud applies the profile's transformations + UI server-side; the chain is ignored on this branch). |
| `output_image` | Image preview. URL extension forced to `jpg/png/webp/auto` based on Format & quality / Video → image upstream. |
| `output_text` | Read-only readout of `buildOutputTextResultForOutput`. |

### 5.5 Block icons
Inline SVGs in `BLOCK_ICON_SVG`, keyed by `type`. Stroke uses `currentColor`
via the block's `glyphColor` token.

---

## 6. Wiring & graph rules

### 6.1 Port semantics
- Default: a port accepts a single wire. Wiring a second source onto a
  single-fan-in input replaces the existing wire.
- `multiple: true` ports accept N wires. Today: `splice.in`,
  `mock_ai_text.text`, `mock_ai_text.image`.
- Wires carry no kind metadata. Compatibility is implicit per block — a
  text overlay's `text` port resolves only via the text walker
  (`resolveUpstreamTextForEdge`), so wiring an image source does nothing
  meaningful even though the wire is allowed.

### 6.2 Cycles
DAG-only by construction: the wiring code rejects an edge that would create
a back-edge from `toNode` to `fromNode`. Splice and AI-text walkers also
guard against cycles via `visited` sets.

### 6.3 Topological walks
- `orderedNodeIds()` — Kahn-ish DFS from sources.
- `orderedPipelineNodes(outputId)` — nodes that participate in the chain
  ending at `outputId`.
- `nodesOnPrimaryPath(outputId)` — nodes on the *first* splice segment
  path; only these contribute to the base URL chain. Other splice branches
  are inlined inside the splice piece.
- `findPrimaryMediaInputNode(startId)` — splice-aware walk back to the
  first `input_video` / `input_image` / mock generator that materialises
  the base media.

---

## 7. URL compilation

### 7.1 Pieces → segments → URL
1. For each node on the primary path with a `cldKind`, call
   `nodeTransformationPieces(node)` → `Piece[]`.
2. Each piece is either a key-value object (e.g. `{ crop: "fill", width: 640 }`)
   or `{ _raw: "<literal segment>" }` for splice / manual-transform output.
3. `transformationPieceToUrlSegment(piece)` joins KV pairs as
   `<short>_<value>` separated by commas (with abbreviations: `crop` → `c_`,
   `width` → `w_`, `effect` → `e_`, `overlay` → `l_`, `flags` → `fl_`, etc.
   — see `escUrlVal` and the abbreviation table inside the function).
4. Image terminals (`output_image`) and pipelines containing `cld_video_frame`
   get a tail piece (`f_*,q_auto`) so browsers receive a concrete still
   format. Video terminals do **not** get an implicit tail — add an explicit
   `cld_delivery` block to override format / quality.
5. Final URL:
   `https://res.cloudinary.com/<cloud>/<image|video|raw>/upload/<segments joined by />/<public_id>.<ext>`.

### 7.2 Embed Player URL
`buildCloudinaryEmbedPlayerUrl(outputId)` returns a query-string URL of the
form `https://player.cloudinary.com/embed/?cloud_name=…&public_id=…`. The
`output_video_player` block's `profile` mode adds `&profile=<profile>` and
the cloud-hosted player applies the profile server-side. The chain is **not**
included in the embed URL — the embed always sources the raw public_id.

### 7.3 Splice compilation
Detail of `case "splice"` inside `nodeTransformationPieces`:
- The first incoming wire's source is treated as the base video and
  contributes via the normal primary-path walk — it is *not* re-emitted as a
  layer.
- For every subsequent incoming wire (in `settings.order`):
  1. Resolve the segment's source (`spliceSegmentLayerChain` walks back
     through any per-segment transforms) and its public id.
  2. Emit `fl_splice[:transition_(name_<X>)],l_video:<layerRef>` (where
     `<X>` is the transition stored on the *previous* segment, or no
     qualifier when `cut`).
  3. Inline the per-segment transforms (each transformed via
     `nodeTransformationPieces` → `transformationPieceToUrlSegment`).
  4. Close with `fl_layer_apply`.
- `settings.order` is an array of edge ids; `settings.transitions` maps
  edge id → transition preset key. Both are kept in sync with live wires
  (stale ids dropped on every render via `syncSpliceOrderFromEdges`).

---

## 8. Applets

### 8.1 Model
One applet per saved flow. The applet's slug **is** the saved flow name —
saving one re-saves the other (`saveAppletFromForm` calls
`persistCurrentFlowUnderNameSilent` on the same name). You can't open the
applet editor for an unsaved flow; the editor toasts "Save the flow first."

### 8.2 Applet record
Stored under `localStorage["videofactory.applets.store.v1"]` as
`{ v: APPLET_RECORD_VERSION, applets: { [slug]: AppletRecord } }`.
```ts
type AppletRecord = {
  v: 1;
  name: string;            // == slug, == saved flow name
  slug: string;
  version: string;         // "1" today; bumped only manually
  description?: string;
  flowSnapshot: FlowDocument;   // see §6
  inputs: AppletInput[];
  output: { kind: "video" | "image" | "text"; nodeId: string };
  updatedAt: string;       // ISO
};

type AppletInput = {
  nodeId: string;          // canvas node id this binding maps to
  kind: "image" | "text" | "video" | "node";
  required: boolean;
  ignored?: boolean;       // present + true → don't expose at all
  paramName?: string;      // POST body key; required unless ignored
};
```

### 8.3 Input discovery
Inputs are discovered automatically: every node with **zero incoming wires**
(`nodeIndegree === 0`) is a candidate, sorted top-to-bottom then
left-to-right by canvas position. Block type → kind:
- `input_image`, `mock_i2i`, `mock_t2i` → `image`
- `input_text`, `mock_ai_text` → `text`
- `input_video`, `mock_i2v_frames`, `mock_i2v_refs` → `video`
- anything else with no incoming wires → `node` (catch-all, opaque)

Each row in the editor exposes:
- A **Role** select: `Required`, `Optional`, `Ignored`. `Ignored` drops the
  input from the API surface entirely; `Optional` leaves it in but
  non-required.
- An **API name** input (the JSON key in the request `params`). Default:
  `input_<kind>` with a numeric suffix to keep names unique. Pattern:
  `^[a-zA-Z_][a-zA-Z0-9_]*$`, max 64 chars.

The editor also flags drift between the saved record and the canvas:
- "New on canvas" badge for inputs that aren't in the persisted record yet.
- "Removed from canvas" cards for persisted inputs whose nodes no longer
  exist; saving drops them.

### 8.4 Output derivation
`deriveAppletOutputSpecFromFlow()` finds terminal-output nodes on the
canvas (`output`, `output_video_player`, `output_image`, `output_text`).
The applet is only valid when there's exactly one terminal — the editor
surfaces "Add one Output block" / "Only one terminal Output allowed.
Mark extras as Preview only." otherwise. The output's `kind` becomes the
applet's response delivery kind.

### 8.5 Documented HTTP API
The editor displays this contract in two collapsible details:

**Request**
```
POST /applets/<encodeURIComponent(slug)>/run
Content-Type: application/json

{
  "params": {
    "<paramName_1>": "<value>",
    "<paramName_2>": "<value>"
  }
}
```
Each `paramName` is the binding's resolved API name. Placeholder values
shown in the editor: `"folder/asset_id"` for `image`/`video`,
`"Example prompt"` for `text`, `"param_value"` for `node`.

**Response (success)**
```jsonc
{
  "ok": true,
  "applet": { "slug": "<slug>", "version": "<version>" },
  "output": { "kind": "video" | "image" | "text" },
  "deliveryUrl": "https://res.cloudinary.com/.../upload/.../public_id.ext" | null,
  "embedPlayerUrl": "https://player.cloudinary.com/embed/?..." // video only, when computable
}
```

**Response (invalid flow)**
```json
{ "ok": false, "error": "invalid_flow", "message": "<editor message>" }
```

> **No server today.** Nothing in `index.html` issues this request or
> answers it. The editor is a documentation surface that locks the contract
> down enough that a future runtime — or a hand-rolled curl — can honour it.

### 8.6 Dirty tracking inside the applet editor
`appletEditorCleanSnapshot` captures the description, normalized bindings,
and a `canvasGraphFingerprint()` at open / save time. The "Unsaved changes"
banner and per-row dirty styling derive from diffing the live state against
that snapshot. Canvas structure changes (input nodes added / removed) are
tracked separately via `appletEditorPersistedBindingIds`.

---

## 9. Persistence

### 9.1 Flow document (v1)
Returned by `serializeFlowDocument()`:
```ts
type FlowDocument = {
  v: 1;
  savedAt: string;            // ISO
  viewport: { panX: number; panY: number; zoom: number };
  nodeIdSeq: number;
  edgeIdSeq: number;
  nodes: { id: string; type: string; x: number; y: number; settings: object }[];
  edges: { id: string; fromNode: string; fromPort: string; toNode: string; toPort: string }[];
};
```
Notes:
- Settings are deep-cloned on serialization (`JSON.parse(JSON.stringify)`).
- On load (`applyFlowDocument`):
  - Unknown `node.type`s are dropped and reported via `unknownTypes`.
  - Edges referencing missing nodes or missing ports are dropped silently.
  - Legacy `node.type === "input"` is migrated to `input_video`.
  - Settings are merged through `mergeNodeSettings(def, raw)` so new fields
    introduced after the snapshot was saved get default values.
  - If `viewport` is missing, the canvas auto-fits.
  - The dirty fingerprint is reset to "clean".

### 9.2 LocalStorage keys
| Key | Shape | Purpose |
|---|---|---|
| `videofactory.workflow.store.v1` | `{ v: 1, lastActive: string \| null, flows: { [name]: FlowDocument } }` | Saved flows. `lastActive` is read but **not** the source of truth — the URL `?flow=` wins. |
| `videofactory.applets.store.v1` | `{ v: 1, applets: { [slug]: AppletRecord } }` | Saved applets. |
| `videofactory.workflow.builtinCanvasVersion` | string number | Tracks the built-in seed-demo version. Bumping `BUILTIN_CANVAS_VERSION` forces the next boot to clear `?flow=` and re-seed the demo. |

Storage failures (full / private mode) surface as toasts: "Could not save
(storage full or private mode)." etc.

### 9.3 URL state
`?flow=<encoded name>`:
- Read on boot (`getFlowIdFromLocation`); if it matches a saved flow,
  that flow is applied.
- Saving / loading / importing updates the URL via
  `history.replaceState` (no page reload).
- `popstate` re-applies the flow named in the URL, falling back to the
  seed demo.

### 9.4 File import / export
- Export: `JSON.stringify(doc, null, 2)` blobbed and downloaded as
  `<safe-name>.flow.json` (filesystem-safe filename via
  `flowDownloadFilename`).
- Import: read text, `JSON.parse`, must have `v === FLOW_DOC_VERSION`,
  apply, then clear `?flow=` so the user can't accidentally overwrite a
  prior save.

---

## 10. Boot sequence

1. CSS variables initialize.
2. `buildPalette("")` populates the palette from `TYPES`.
3. All applet dialogs are force-closed (defensive, against stuck `<dialog>` state).
4. `restoreStartupFlow()`:
   - Read `BUILTIN_CANVAS_VERSION_KEY` from localStorage. If missing or
     less than `BUILTIN_CANVAS_VERSION`, bump it and seed the demo flow
     (`seedDemo()` — Input · video → Resize 720 → Output · video).
   - Otherwise, if `?flow=<name>` matches a saved flow, apply it.
   - Otherwise, seed the demo.
5. `popstate` listener installed.
6. Brand flow-name / dock state synced.

---

## 11. Validation surface

| Action | Validation | User-visible result |
|---|---|---|
| Run flow with no nodes | — | Toast: "Add blocks to the canvas first." |
| Run flow with no Output | — | Toast: "Add an Output block." |
| Run flow with no Input public id on media outputs | — | Toast: "Set a public ID on an Input block." |
| Run flow with media outputs | URL builds for some/all | Toast: "Flow ran — N preview(s) updated." or partial-coverage summary. |
| Save flow with empty name | trim/length | Toast: "Enter a name for this flow." |
| Open Applet editor on unsaved flow | flow has no name | Toast: "Save the flow first." |
| Save applet with > 1 terminal | output spec | Toast: "Only one terminal Output allowed. Mark extras as Preview only." |
| Save applet with no terminal | output spec | Toast: "Add one Output block (video, image, or text)." |
| Save applet with invalid API name | regex / length | Toast: 'Invalid API parameter name "…". Use letters, digits, underscores; start with a letter or _ (max 64).' |
| Save applet with duplicate API name | uniqueness | Toast: 'Duplicate API parameter name "…".' |

---

## 12. Versioning & migrations

- **Flow doc.** `v: 1`. No migration code path exists for `v != 1`; older
  files would be rejected with "Invalid flow file (expected version 1)."
- **Applet record.** `v: 1`. Same story; mismatching records cause the
  store to be re-initialised empty.
- **Built-in seed.** `BUILTIN_CANVAS_VERSION = 3`. Bumping it on a future
  release is the lever for "discard whatever the user had on first boot
  and replant a new seed demo." It does **not** wipe saved flows or
  applets.
- **Block-type rename.** `input` → `input_video` is hard-coded inside
  `applyFlowDocument`. The pattern for future renames is the same line.

---

## 13. Known limits & open questions

- **No applet runtime.** The single biggest open item. The `POST
  /applets/.../run` contract is documented; nothing executes it. A future
  runtime would: load the applet's `flowSnapshot`, substitute each
  required `paramName` into its bound canvas node's settings (image/video
  → `publicId`, text → `prompt`), compile the URL, optionally invoke real
  AI for mock generators, and return `{ ok, applet, output, deliveryUrl,
  embedPlayerUrl? }`.
- **Mock AI is fixed.** Every `mock_*` generator emits a fixed sample
  public id. There's no provider abstraction yet.
- **`cld_transcribe` / `cld_auto_chapters` / `cld_ai_details` are
  trigger-only.** They emit no URL pieces and have no consumer block —
  the artifacts Cloudinary saves (`.transcript`, `.vtt`, AI context
  metadata) are not surfaced in the player or downstream chain. The
  block headers explicitly call out the future "consumer" block that
  would burn-in / display them.
- **One Cloudinary cloud, one API key channel.** No UI for switching
  cloud or providing an API key — `window.VIDEOFACTORY_CLD_API_KEY` is
  the only knob.
- **`localStorage`-only.** No sync, no export of the *applet* store
  (only individual flows export).
- **Single-file scaling.** ~7.3k lines of vanilla JS in one file. There's
  no module boundary; the only structural cut is "things above the
  `// boot` block are declarations / helpers; things below are event
  bindings + boot." Anything growing past here probably wants a real
  bundler.
- **Wires are untyped.** The editor allows you to wire anything to
  anything; meaning is enforced per block. Fine today, will get
  surprising as kinds proliferate.
