# Video Content Factory

**Compose media transformations and generative AI as one visual workflow,
then publish it as a callable applet.**

Video Content Factory is a node-based studio for building media pipelines
that mix classic transformations (resize, trim, splice, overlay, format,
effects) with generative steps (image → video, text → image, AI text,
auto-transcription, AI chapters). You wire blocks together on a canvas,
preview the result live, save it, and turn it into an **applet** — a
named workflow with a typed input/output contract that anything else
can call.

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

### Workflow
A workflow is the canvas. It's a directed graph of blocks connected by
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
- **Outputs** are how the workflow materialises: as a delivered video, an
  image, a player embed, or a piece of text.

Wires carry kinds (image, video, text). Compatible blocks accept them;
the canvas keeps you out of obvious dead ends.

### Applet
An applet is a workflow promoted to a contract.

When you save a flow as an applet, the studio:

1. Detects the workflow's loose ends (the blocks with no incoming wires)
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

   ```json
   {
     "ok": true,
     "applet": { "slug": "<name>", "version": "1" },
     "output": { "kind": "video" },
     "deliveryUrl": "https://res.cloudinary.com/.../upload/.../public_id.mp4",
     "embedPlayerUrl": "https://player.cloudinary.com/embed/?..."
   }
   ```

When the workflow collapses to a single Cloudinary URL chain — every block
on the path is a transformation, no generative steps and no AI sidecar
producers — the applet additionally publishes as a **named transformation**
(`t_<slug>`) on the cloud account. Callers can then invoke it as a
stateless URL with the applet's parameters spliced in, no server
round-trip:

```
https://res.cloudinary.com/<cloud>/image/upload/$headline_!Hello%20world!/t_<slug>/folder/asset_id.jpg
```

Workflows that materialise new assets (image → video, text → image, AI
text, auto-transcription, auto-chapters, AI details) stay API-only —
there's no URL grammar for "call a model and upload the result." The
publish surface shows which forms an applet supports and, when URL form
isn't available, the specific block that forces server-side execution.

You author the workflow visually, but you ship a workflow-as-an-API.
Each parameter is **required**, **optional**, or **ignored**, and gets a
human-chosen API name.

The applet snapshots the workflow at save time, so changing the canvas
later doesn't silently change what callers receive — saving again
publishes a new revision.

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

A workflow can have many preview outputs, but exactly **one terminal
output** when published as an applet — that's the contract's response
kind.

---

## How you use it

1. Open the studio. Drag blocks from the palette onto the canvas.
2. Wire inputs into transformations into a single output. The output
   updates live as you type.
3. **Save flow** to name and persist it. The URL captures the active
   flow so links are shareable.
4. Click **Applet** to open the publish surface. Review the discovered
   inputs, give them friendly API names, mark each as required /
   optional / ignored, and save. The contract on the right is what
   callers will see.
5. Reload, iterate, save again — each save is a new revision of the
   same applet slug.

Saved flows can be exported and imported as `.flow.json` files, so a
workflow is portable: hand someone the file, they re-open the same
canvas.

---

## Design principles

- **One primitive.** Transformations and AI steps are the same kind of
  thing — blocks with typed ports. Adding a new generator never grows a
  separate parallel system.
- **The graph is the spec.** The visible wiring is what runs. There's no
  hidden glue between "the editor" and "the runtime"; the URL chain (or
  applet response) is derived from the canvas, not authored alongside
  it.
- **Snapshots, not live links.** Saving an applet captures the workflow
  as it stands. Later edits don't retroactively change what callers
  see — you re-publish to ship.
- **Boring outputs.** A workflow's result is always a thing you can
  link, embed, or read. Video / image → a Cloudinary delivery URL or
  player embed. Text → a string. No proprietary format, no opaque blob.
- **No setup tax.** The studio is a single page. Open it, build a
  workflow, share the URL.

---

## Status

Today the studio runs locally in the browser; saved flows and applets
live in `localStorage`, and a few generative blocks are mocked against
fixed sample assets while real provider integrations land. The applet
contract is documented and stable; an execution runtime that honours
`POST /applets/<name>/run` is the next milestone.
