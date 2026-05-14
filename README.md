# Video Content Factory

**Compose media transformations and generative AI as one visual graph,
then publish it as a callable applet.**

Video Content Factory is a node-based studio for building media pipelines
that mix classic transformations (resize, trim, splice, overlay, format,
effects) with generative steps (image → video, text → image, AI text,
auto-transcription, AI chapters). You wire blocks together on a canvas,
preview the result live, save it, and turn it into an **applet** — a
named graph with a typed input/output contract that anything else can
call.

---

## The idea

Two things tend to be built separately today: deterministic media
transforms (a Cloudinary URL chain, a video editor timeline) and
generative AI steps (run an i2v model, run a text-to-image model, get a
transcript). Authors end up gluing them by hand — render here, upload
there, paste a URL, hit run.

Video Content Factory treats both as the same primitive: a **block** that
takes typed inputs and produces a typed output. Once everything is just a
block, the interesting part is the **graph between them**:

- Generate a hero clip with i2v from a still and a prompt, splice it onto
  a captured intro, overlay an AI-written headline, deliver as adaptive
  HLS.
- Run AI auto-chapters on a long-form video, then render the chaptered
  player.
- Take user-supplied text, sanitize it through an AI text step, burn it
  into a watermark layer with a generated background image.

These are all the same shape: a few inputs at the top, a few transforms
in the middle, one terminal output at the bottom. The studio's job is to
make that shape easy to build, easy to read back, and easy to publish.

---

## Core concepts

### Graph
A graph is the canvas. It's a directed graph of blocks connected by
typed wires.

- **Inputs** seed the graph: a video, an image, a piece of text. They're
  the loose ends with no incoming wires.
- **Transformation blocks** are deterministic — resize, trim, splice,
  effect, overlay, format / quality, rotate, audio, border, round
  corners.
- **Generative blocks** are AI-backed — image → video (with frames or
  reference images), image → image, text → image, AI text, AI video
  preview / summarization, auto transcription, auto chapters, AI video
  details.
- **Outputs** are how the graph materialises: as a delivered video, an
  image, a player embed, or a piece of text.

Wires carry kinds (image, video, text). Compatible blocks accept them;
the canvas keeps you out of obvious dead ends.

The word "graph" is used in the standard sense of nodes and edges, not
the chart sense — `node = block`, `edge = wire`. The studio's vocabulary
elsewhere lines up with that: "the graph is the spec," "node id," and so
on.

### Applet
An applet is a graph promoted to a contract.

When you save a graph as an applet, the studio:

1. Detects the graph's loose ends (the blocks with no incoming wires)
   and surfaces them as **named parameters** with kinds (`image`,
   `video`, `text`).
2. Detects the terminal output and surfaces its kind (`video`, `image`,
   `text`).
3. Generates a stable HTTP-style contract from those bindings:

   ```
   POST /applets/<applet-name>/run
   Content-Type: application/json

   {
     "params": {
       "input_image":  "folder/asset_id",
       "input_text":   "headline copy"
     }
   }
   ```

#### Two kinds of applet: **URL** and **API**

Every applet exposes the same `POST /applets/<slug>/run` wrapper, but the
**contract shape** depends on what the graph does to the cloud account:

- **URL** — pure transformation. The graph collapses to a single
  Cloudinary URL chain (resize, overlay, splice, format, …). Output is a
  delivery URL of the input asset; **no new resource is materialised**.
  The call is idempotent.

  ```json
  {
    "ok": true,
    "kind": "derive",
    "applet": { "slug": "<name>", "version": "1" },
    "deliveryUrl": "https://res.cloudinary.com/.../t_<slug>/folder/asset_id.jpg",
    "sourcePublicId": "folder/asset_id"
  }
  ```

  URL applets additionally publish as a **named transformation**
  (`t_<slug>`) so callers can skip the server entirely and invoke them as
  a stateless URL:

  ```
  https://res.cloudinary.com/<cloud>/image/upload/$headline_!Hello%20world!/t_<slug>/folder/asset_id.jpg
  ```

- **API** — the graph mints a new resource (image → video, text →
  image, AI text, auto-transcription, auto-chapters, AI details, AI
  tagging, …) or a server-side sidecar. The contract carries a
  **`producedPublicId`** so callers can reference the new asset:

  ```json
  {
    "ok": true,
    "kind": "create",
    "applet": { "slug": "<name>", "version": "1" },
    "status": "ready",
    "producedPublicId": "vcf/<name>/<id>",
    "resourceType": "video",
    "deliveryUrl": "https://res.cloudinary.com/.../vcf/<name>/<id>.mp4",
    "jobId": null
  }
  ```

  API applets do **not** publish as named transformations — there's no
  URL grammar for "call a model and upload the result."

#### URL ⊆ API at runtime

A URL applet can always be **promoted** to an API call by passing a
`store` block. The wrapper executes the chain as the `eager` value of an
Upload call, so the same applet can either deliver a URL or mint a new
asset depending on what the caller sends:

```
POST /applets/<slug>/run
{
  "params": { "input_image": "folder/asset_id" },
  "store": {
    "public_id":   "campaigns/spring/hero",
    "folder":      "campaigns/spring",
    "overwrite":   false,
    "tags":        ["spring", "hero"],
    "access_mode": "public",
    "eager":       "derive"
  }
}
```

The badge on the applet expresses the **floor**: URL means "can be called
as URL (stateless delivery), and as API when storage params are passed";
API means "always API — the materialising step has no URL form."

#### Converting URL ⇄ API

The bindings editor exposes a **Convert** action next to the kind badge
that pins the applet's *default* contract. The choice is persisted on
the applet record (`defaultKind`) and honoured by the playground, the
sidebar list badge, and the `POST /run` snippet:

- **Convert to API** is available on any URL-eligible graph. After
  conversion, every call defaults to materialising a new asset (the
  snippet body includes a `store` block with `public_id` required); the
  URL form is still structurally available and the named-transformation
  panel still shows the URL template.
- **Convert to URL** is available *only* when the underlying graph
  permits a URL chain. Graphs with materialising steps (image → video,
  transcription, AI sidecars, …) are intrinsically API — the button
  is disabled with a tooltip explaining the structural block.

The applet record carries two related fields:

- `urlFormAvailable` (boolean, derived from the saved `graphSnapshot`)
  — the structural ceiling. Computed at save time so the sidebar can
  badge the applet without re-classifying the graph. Old records without
  this field show as **API** (safer unknown state) and get stamped on
  next save.
- `defaultKind` (`"derive" | "create"`) — the author's chosen default,
  coerced to `"create"` whenever `urlFormAvailable === false`.

You author the graph visually, but you ship a graph-as-an-API. Each
parameter is **required**, **optional**, or **ignored**, and gets a
human-chosen API name.

The applet snapshots the graph at save time, so changing the canvas
later doesn't silently change what callers receive — saving again
publishes a new revision.

---

## Vocabulary

| Term | What it is |
|---|---|
| **Factory** | The product — Video Content Factory. The service that hosts canvases, runs graphs, and executes applets at scale. |
| **Canvas** | The drawing surface. The open space where you place blocks and draw lines between them. |
| **Graph** | The structure you build on a canvas — blocks connected by lines. The graph is both the spec and the thing that runs. |
| **Block** | A single processing unit on the canvas. Takes typed inputs, produces a typed output. Could be a transformation, a generative AI step, an input source, or a terminal output. |
| **Dot** | The small circle on the edge of a block — the physical connection point you drag from or drop onto. Each dot belongs to one port (input or output). |
| **Line** | The drawn connection between two dots. Lines carry a media kind (video, image, text) and define the flow of data through the graph. |
| **Bindings** | The configuration layer of an applet — the named parameters, their kinds (image / video / text), and their required / optional / ignored status. Bindings are what turn a raw graph into a typed API contract. |
| **Applet** | A graph promoted to a contract. Once saved, an applet has a stable slug, a versioned snapshot of its graph, and a callable `POST /applets/<slug>/run` endpoint. |

**Internal code naming** (for contributors): `block = node`, `dot = port`, `line = edge or wire`. The customer-facing names above are used in the UI and docs; the code-level names follow graph-theory and DOM conventions.

---

## What's in the box

The studio ships with the building blocks below. They're combinable in
any order; the result is whatever the wiring says.

**Inputs**
- Video, image, text.

**Cloudinary transformations**
- Resize & crop, trim / duration, rotate, audio (volume, mute), border,
  round corners, adjust (brightness / contrast / saturation / vignette).
- Video effect presets: blur, accelerate / decelerate, reverse,
  boomerang, loop, vignette, fade in / out, noise, grayscale.
- Text overlay, image / watermark overlay (asset can be wired in).
- Splice with 40+ transition presets (cut, fade variants, dissolve,
  wipe, slide, smooth, circle / rect crops, diagonals, slices,
  squeezes…). Each segment can carry its own transform chain.
- Format & quality (delivery), manual transformation escape hatch.
- Video → image (still frame extraction).
- AI video preview / summarization.

**Generative**
- Image → video (frames mode, references mode).
- Image → image, text → image.
- AI text (sanitize / rewrite / synthesize from wired text and images).
- Auto transcription, auto chapters, AI video details.

**Outputs**
- Video (delivered URL), video player (Cloudinary Video Player, with
  saved profile support), image, text.

A graph can have many preview outputs, but exactly **one terminal
output** when published as an applet — that's the contract's response
kind.

---

## How you use it

1. Open the studio. Drag blocks from the palette onto the canvas.
2. Wire inputs into transformations into a single output. The output
   updates live as you type.
3. **Save Graph** to name and persist it. The URL captures the active
   graph (`?graph=<name>`) so links are shareable.
4. Click **Applet** to open the publish surface. Review the discovered
   inputs, give them friendly API names, mark each as required /
   optional / ignored, and save. The contract on the right is what
   callers will see.
5. Reload, iterate, save again — each save is a new revision of the
   same applet slug.

Saved graphs can be exported and imported as `.graph.json` files, so a
graph is portable: hand someone the file, they re-open the same canvas.

---

## Design principles

- **One primitive.** Transformations and AI steps are the same kind of
  thing — blocks with typed ports. Adding a new generator never grows a
  separate parallel system.
- **The graph is the spec.** The visible wiring is what runs. There's no
  hidden glue between "the editor" and "the runtime"; the URL chain (or
  applet response) is derived from the canvas, not authored alongside
  it.
- **Snapshots, not live links.** Saving an applet captures the graph as
  it stands. Later edits don't retroactively change what callers see —
  you re-publish to ship.
- **Boring outputs.** A graph's result is always a thing you can link,
  embed, or read. Video / image → a Cloudinary delivery URL or player
  embed. Text → a string. No proprietary format, no opaque blob.
- **No setup tax.** The studio is a single page. Open it, build a
  graph, share the URL.

---

## Status

Today the studio runs locally in the browser; saved graphs and applets
live in `localStorage`, and a few generative blocks are mocked against
fixed sample assets while real provider integrations land. The applet
contract is documented and stable; an execution runtime that honours
`POST /applets/<name>/run` is the next milestone.
