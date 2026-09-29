---
name: jifchat
description: Generate images and videos on the user's JifChat infinite canvas (jif.dev). Use when the user asks to generate, create, or render an image, product shot, poster, or short video with JifChat, or says "/jifchat", "/generate image", "/generate video", or "make a similar UGC video". Results land in a JifChat canvas project and spend the user's JifChat credits.
---

# JifChat

JifChat is an AI design canvas. This skill lets you create canvas projects and run image/video generation nodes in them through the JifChat Canvas API, on the user's own account.

## What this skill can and can't do

**Can**
- Create new canvas projects and generate images / videos in them
- Build complete multi-node canvas workflows (text prompts → image generators → video generators)
- Put results on the canvas as properly connected, laid-out nodes that show up in the JifChat UI

**Can't (yet)**
- Touch projects shared with the user (owner-only; others return 404)
- Upload local files — reference images must already be public URLs (an upload endpoint is planned)
- Use templates (`/api/v1/templates` is planned)
- Chat with JifChat's assistant, manage billing, or create API keys

**Never** delete a project, or run generation in a project you didn't create in this session, without asking first.

## Project rule — always fresh

**Create a NEW project for every new user request.** Do NOT list the account's projects and do NOT reuse an existing one — the account may hold unrelated work, and each request deserves its own canvas. Only reuse an existing project when the user *explicitly* refers to a previous generation ("make THAT one slower", "add another scene to the drama") — and ask which one if ambiguous.

```bash
# Step 1 of every request: fresh project
curl -s -X POST "$B/api/v1/canvas/projects" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d '{"title":"<short descriptive title>"}'
# → { "_id": "..." } — use this _id as $PID for the whole request
```

## Setup

1. Sign in to JifChat → **Settings → Balance → API Keys** → **Create**. Copy the `jc_...` key — it is shown once.
2. Save it where this skill can read it (either works):
   ```bash
   install -m 600 /dev/null ~/.jifchat_key && printf '%s' 'jc_YOUR_KEY' > ~/.jifchat_key
   ```
   or `export JIFCHAT_API_KEY=jc_YOUR_KEY` in your shell profile.
3. Optional: point at another environment, e.g. `export JIFCHAT_BASE_URL=https://staging.jif.dev`. Default is `https://chat.jif.dev`. A key only works on the environment it was created on.

> OAuth sign-in (authorize in the browser, no key copy-paste) is planned. Until then, setup is the manual key above.

## Calling the API

Start every shell session with:

```bash
B=${JIFCHAT_BASE_URL:-https://chat.jif.dev}
K=${JIFCHAT_API_KEY:-$(cat ~/.jifchat_key 2>/dev/null)}
[ -n "$K" ] || echo "No JifChat key — see Setup"
```

Then send `-H "Authorization: Bearer ***" -H 'Content-Type: application/json'` on every request. **Never print, echo, or log the key.** If the key is missing, stop and walk the user through Setup. Check the key without spending credits: `curl -s "$B/api/v1/canvas/projects" ...` — 200 = OK, 401 = bad key.

## The two paths — pick by job size

**One generation?** → Path 1 (single run call — the API builds the node structure for you).
**Two or more generations, or a pipeline?** → Path 2 (PUT the whole graph, then run).

### Path 1 — single generation

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"img-1","model":"nano-banana-pro","parameters":{"prompt":"a shiba inu on a surfboard, watercolor"},"position":{"x":320,"y":100}}'
```

- **Always send `position`** — never omit it. The API auto-creates the full structure: the `textNode` holding your prompt (placed 320px left of your position) + the edge + the generator. If a text node is already wired, its text is updated instead — never duplicated.
- Response: `{"success":true,"run":{"status":"completed","outputUrl":"https://...","creditCost":150}}`

Video (async — 202 + poll, see *Polling*):
```json
{"nodeId":"vid-1","model":"seedance-2.5","parameters":{"prompt":"...","duration":5,"aspect_ratio":"16:9","resolution":"720p"},"position":{"x":320,"y":100}}
```

To animate a generated image, pass its `outputUrl` as a reference: `"image_urls":["<outputUrl>"]`, cite it in the prompt as `@Image1`.

### Path 2 — multi-node workflow: PUT the whole graph

For 2+ generations (multi-scene videos, storyboard→video pipelines, avatar image + avatar video): **declare the entire node graph in ONE `PUT`**, then run the nodes. Do NOT issue per-node create calls; compose the structure once.

```bash
curl -s -X PUT "$B/api/v1/canvas/projects/$PID" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d @- <<'EOF'
{ ...full graph JSON — see worked example below... }
EOF
```

**⚠ The PUT replaces the ENTIRE nodes/edges arrays — anything you omit is DELETED.** On a fresh project (your default) the arrays are empty, so this is safe. If you must PUT into a project that already has content: GET it first, keep every existing node untouched, add only your new ones, send the complete arrays back.

**Node shape** — every node needs `id`, `type`, `position` `{x,y}`, and type-specific `data`:
- `textNode`: `{"text": "<the prompt>"}`
- `imageGenerator`: `{"model": "nano-banana-pro", "status": "idle", "resultImage": null, "errorMessage": null, "prompt": ""}`
- `videoGenerator`: `{"model": "seedance-2.5", "status": "idle", "resultVideo": null, "errorMessage": null, "prompt": ""}`

**Edge shape** — every edge needs `id` (e.g. `"edge-1"`), `source`, `target`, `sourceHandle`, `targetHandle`, `type: "highlightable"`, `animated: false`, `style: {"stroke": "<color>"}`:
- text: `text-out` → `text-in` (at most ONE text edge per generator), stroke `"var(--handle-text-lit)"`
- images: `image-out` → `image-in-1`, `image-in-2`, … (one edge per slot), stroke `"var(--handle-image-lit)"`

**Layout formula — use it, don't improvise:** each generator occupies a column 450px apart; its textNode sits 320px left of it; rows 240px apart.

**Worked example — 2 image scenes feeding a video:**

```json
{
  "nodes": [
    {"id": "scene1-prompt", "type": "textNode", "position": {"x": 0, "y": 100}, "data": {"text": "scene 1: a dog discovers a surfboard on the beach, watercolor"}},
    {"id": "scene1", "type": "imageGenerator", "position": {"x": 320, "y": 100}, "data": {"model": "nano-banana-pro", "status": "idle", "resultImage": null, "errorMessage": null, "prompt": ""}},
    {"id": "scene2-prompt", "type": "textNode", "position": {"x": 0, "y": 340}, "data": {"text": "scene 2: the dog rides a wave at sunset, watercolor"}},
    {"id": "scene2", "type": "imageGenerator", "position": {"x": 320, "y": 340}, "data": {"model": "nano-banana-pro", "status": "idle", "resultImage": null, "errorMessage": null, "prompt": ""}},
    {"id": "video-prompt", "type": "textNode", "position": {"x": 770, "y": 220}, "data": {"text": "@Image1 discovers the wave, @Image2 rides it — 5s cinematic"}},
    {"id": "vid-1", "type": "videoGenerator", "position": {"x": 1090, "y": 220}, "data": {"model": "seedance-2.5", "status": "idle", "resultVideo": null, "errorMessage": null, "prompt": ""}}
  ],
  "edges": [
    {"id": "edge-1", "source": "scene1-prompt", "sourceHandle": "text-out", "target": "scene1", "targetHandle": "text-in", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-text-lit)"}},
    {"id": "edge-2", "source": "scene2-prompt", "sourceHandle": "text-out", "target": "scene2", "targetHandle": "text-in", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-text-lit)"}},
    {"id": "edge-3", "source": "video-prompt", "sourceHandle": "text-out", "target": "vid-1", "targetHandle": "text-in", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-text-lit)"}},
    {"id": "edge-4", "source": "scene1", "sourceHandle": "image-out", "target": "vid-1", "targetHandle": "image-in-1", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-image-lit)"}},
    {"id": "edge-5", "source": "scene2", "sourceHandle": "image-out", "target": "vid-1", "targetHandle": "image-in-2", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-image-lit)"}}
  ]
}
```

The video generator gets its own textNode for the directing prompt; the scene images feed its `image-in-1`/`image-in-2` slots; cite connected images in prompts as `@Image1`, `@Image2` in wiring order.

**Then run the nodes** — same `nodeId` as in the graph, **omit `parameters.prompt`** (the prompt already lives on the canvas; re-sending it would overwrite your text node):

```bash
# images first (sync — result is in the response)
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d '{"nodeId":"scene1","model":"nano-banana-pro","parameters":{}}'

# then the video (async — poll)
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-video-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"vid-1","model":"seedance-2.5","parameters":{"duration":5,"aspect_ratio":"16:9","resolution":"720p"}}'
```

Run nodes in dependency order (upstream first); each run spends credits only on success.

## Polling (video only)

After `run-video-node` returns `{"run":{"status":"running","responseUrl":"..."}}`, poll every ~10s:

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/check-run" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d '{"responseUrl":"<responseUrl>"}'
```

Until `status` is `completed` (use the output URL) or `failed` (report the error). Tell the user it's in progress — videos take minutes. **Give up after ~8 minutes**: report the run is still going, hand over the canvas link, and note the project persists — the user can ask again later and you'll read the state back instead of re-generating. Never poll past the threshold.

## Models

| Use | Model | Notes |
|---|---|---|
| Image (default) | `nano-banana-pro` | with refs: `${model}/edit` + `image_urls`, ≤14 |
| Image | `gpt-image` | with refs: `${model}/edit` + `image_urls`, ≤16 |
| Video (default) | `seedance-2.5` | refs optional: ≤9 `image_urls`, cite as `@Image1` |
| Video | `grok-i2v` | image-to-video, needs `reference_image_urls` |

Use the defaults unless the user asks otherwise. With reference images (product photo, logo): use `<model>/edit` in the run request only — never store `/edit` in node data.

## Demo 1 — one UGC shot

Read `examples/one-shot.md`, and `examples/kate-canvas.json` when you need the graph. That export is the 180-node Kate canvas. **Do not clone all 180 nodes.**

Demo 1 builds a few nodes copied from **one** shot (Video Generator 1 in the example):

- a product image upload
- reference stills (image uploads)
- an optional image-generator keyframe, only when a still has to be generated
- one text node (the directing prompt)
- one Seedance video node

Use Path 2 for that small graph. Follow the layout formula. Wire each upload `image-out` → `image-in-1`, `image-in-2`, … in order, and an end still to `lastframe-in` when the shot has one. Wire the text node `text-out` → `text-in`. Cite stills in the prompt as `@Image1`, `@Image2` in wiring order. Write the prompt as one continuous shot, in the spirit of `examples/one-shot.md`. For Seedance wording, see the prompt guide linked from the repo README.

URL fields in the example JSON are the placeholder `{{user-upload}}`. They are not fetchable. Put the user's real image URLs on the upload nodes.

**Explicit “make a similar UGC video” runs.** When the user says “make a similar UGC video”, that sentence is the go-ahead: create a fresh project, PUT this one-shot graph, and run it (keyframe first if you added one, then the Seedance video). Do not stop after describing the graph, and do not ask for a second confirmation.

**“Make it slower” edits that video node.** Stay in the project that holds the shot. Raise the duration on that Seedance video node (adjust the prompt's timing beats only if they would no longer match), PUT the graph, and re-run **that** node. Do not create a new project, do not add a second video node, and do not clone the canvas.

**Template-name jobs still ask permission.** If the user names a template, or asks to run a job by template name, say what you are about to run and wait for a yes. A template name is not permission to spend credits.

## Rules

- **Fresh project per request** — see the Project rule.
- **Always send `position`** — Path 1 single node, Path 2 layout formula.
- **Brands:** never generate brand content from text alone — models don't know real logos/colors. Ask for 2–4 reference image URLs (logo, palette, product) and use an `/edit` model.
- **Credits:** each run spends the user's credits, charged on success (image ≈150 on nano-banana-pro). Before a batch (>3 runs) or any video, say what you're about to run and get a yes. An explicit “make a similar UGC video” or “make it slower” is that yes (see Demo 1). A template name is not.
- **One run per node at a time** — 409 means that node is still running; wait or use a new `nodeId`.
- **Never send an empty array** for an image param (`[]` → 422) — omit the param instead.

## Sync result to canvas (temporary)

Today a successful run does **not** update the node's status on the canvas (it stays `idle` with no image) — structure and positions are handled for you. Until the status write-back lands, after each completed run:

1. `GET $B/api/v1/canvas/projects/$PID` → take `nodes` and `edges`.
2. On the node you ran, set `data.status = "completed"` and `data.resultImage = <outputUrl>` (videos: `data.resultVideo`).
3. `PUT $B/api/v1/canvas/projects/$PID` with `{"nodes": [...], "edges": [...]}` — **all** nodes and edges, unchanged except that one node.

Skip this step if the user has the project open and is editing it — `PUT` replaces the whole node list.

## Errors

| Status | Meaning | Do |
|---|---|---|
| 400 | invalid request (bad model id, bad handle, missing field) | read the error message — it names the problem; fix and retry once |
| 401 | missing/invalid key | re-check Setup and `JIFCHAT_BASE_URL` |
| 402 | out of credits | tell the user to top up in Settings → Balance |
| 404 | project/node missing or not theirs | you probably used a wrong `$PID` — re-check the create-project response |
| 409 | node already running, or edge slot taken | wait, or use a new `nodeId`/slot |
| 429 | rate limited | wait ~30s, retry once |

Full spec: `$B/api/openapi.json` · agent guide: `$B/llms.txt`
