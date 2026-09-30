---
name: jifchat
description: Generate images and videos on the user's JifChat infinite canvas (jif.dev). Use when the user asks to generate, create, or render an image, product shot, poster, or short video with JifChat, or says "/jifchat", "/generate image", "/generate video", or "make a similar UGC video". Results land in a JifChat canvas project and spend the user's JifChat credits.
---

# JifChat

JifChat is an AI design canvas. This skill lets you create canvas projects and run image/video generation nodes in them through the JifChat Canvas API, on the user's own account.

## What this skill can and can't do

**Can**
- Create new canvas projects and generate images / videos in them
- Build multi-node canvas workflows (upload stills, image generators, video generators) and connect them with edges
- Put results on the canvas as nodes the JifChat UI can show

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

**One generation?** → Path 1 (single run call — the API creates the generator node for you).
**Two or more generations, a pipeline, or upload nodes on the canvas?** → Path 2 (`POST` each node, then run).

**Never build a canvas by `PUT /api/v1/canvas/projects/$PID` with `nodes`, and never send a whole-graph replace.** That PUT rejects any body that contains `nodes` — including `[]` and `null` — with 400, and it writes nothing in that body (no title, no edges). The only key-surface node write is `POST /api/v1/canvas/projects/$PID/nodes` (validated). There is no documented `POST /edges`. Edges, when you need them, go on that same PUT as `{ "edges": [...] }` with **no** `nodes` key. PUT is title and edges only.

### Path 1 — single generation

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"img-1","model":"nano-banana-pro","parameters":{"prompt":"a shiba inu on a surfboard, watercolor"}}'
```

- `run-node` creates the generator node when that `nodeId` is new. Do not PUT the project to add it.
- Send the prompt in `parameters`. Do not send `status`, `resultImage`, `resultVideo`, `outputText`, `errorMessage`, or `uploadStatus` — the server seeds those.
- Response: `{"success":true,"run":{"status":"completed","outputUrl":"https://...","creditCost":150}}`

Video (async — 202 + poll, see *Polling*):
```json
{"nodeId":"vid-1","model":"seedance-2.5","parameters":{"prompt":"...","duration":5,"aspect_ratio":"16:9","resolution":"720p"}}
```

To animate a generated image, pass its `outputUrl` as a reference: `"image_urls":["<outputUrl>"]`, cite it in the prompt as `@Image1`.

### Path 2 — multi-node workflow: POST each node

For 2+ generations (multi-scene videos, storyboard→video pipelines, avatar image + avatar video) and for upload nodes: **create each node with its own `POST /nodes`**, then run. Do not declare the graph in one PUT.

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/nodes" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"scene1","nodeType":"imageGenerator","data":{"model":"nano-banana-pro","prompt":"scene 1: a dog discovers a surfboard on the beach, watercolor"}}'
```

**Node body** — `nodeId` (non-empty string, ≤256 characters), `nodeType`, and `data`:
- `nodeType` is one of `imageUpload`, `videoUpload`, `audioUpload`, `imageGenerator`, `textGenerator`, `videoGenerator`.
- Existing nodes are never overwritten. A second POST with the same `nodeId` leaves the first node as it was. Change a prompt or duration on the **run** body, not by PUT.
- Generator `data` may include `model`, `prompt` (a string; longer values are capped), `name`, and that type's settings (`resolution`, `aspectRatio`, `quality`, `imageSize`, `duration`, `generateAudio`). Unknown keys are dropped. The server seeds `status: "idle"` and the result fields. Do not send those yourself.
- `data.prompt` on the node is what the canvas shows. The run does **not** read it — always send `parameters.prompt` (and duration, aspect, resolution) on `run-node` / `run-video-node`.
- Upload nodes require an **https** Firebase Storage or fal media URL. `imageUpload` takes `data.imageUrl` (or `data.mediaUrl`). `videoUpload` and `audioUpload` take `data.mediaUrl`. Anything else is 400. `filename` and `mimeType` must be strings when sent; `duration` must be a non-negative number when sent.

**Edges** — only if the canvas should show connections. `PUT` replaces the **edges** array. On a fresh project that array is empty, so send the full list. If the project already has edges, GET first and send every edge you intend to keep. Never include a `nodes` key in that body. Do not send `"edges": []` unless you mean to clear every edge.

Edge shape — `id` (e.g. `"edge-1"`), `source`, `target`, `sourceHandle`, `targetHandle`, `type: "highlightable"`, `animated: false`, `style: {"stroke": "<color>"}`:
- images: `image-out` → `image-in-1`, `image-in-2`, … (one edge per slot), stroke `"var(--handle-image-lit)"`. A shot's end still uses `lastframe-in` when it has one.

`POST /nodes` does not take a position. Do not PUT nodes to lay the canvas out.

**Worked example — 2 image scenes feeding a video:**

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/nodes" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"scene1","nodeType":"imageGenerator","data":{"model":"nano-banana-pro","prompt":"scene 1: a dog discovers a surfboard on the beach, watercolor"}}'

curl -s -X POST "$B/api/v1/canvas/projects/$PID/nodes" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"scene2","nodeType":"imageGenerator","data":{"model":"nano-banana-pro","prompt":"scene 2: the dog rides a wave at sunset, watercolor"}}'

curl -s -X POST "$B/api/v1/canvas/projects/$PID/nodes" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"vid-1","nodeType":"videoGenerator","data":{"model":"seedance-2.5","prompt":"@Image1 discovers the wave, @Image2 rides it — 5s cinematic"}}'

curl -s -X PUT "$B/api/v1/canvas/projects/$PID" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d @- <<'EOF'
{
  "edges": [
    {"id": "edge-1", "source": "scene1", "sourceHandle": "image-out", "target": "vid-1", "targetHandle": "image-in-1", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-image-lit)"}},
    {"id": "edge-2", "source": "scene2", "sourceHandle": "image-out", "target": "vid-1", "targetHandle": "image-in-2", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-image-lit)"}}
  ]
}
EOF
```

Cite connected images in prompts as `@Image1`, `@Image2` in wiring order.

**Then run the nodes** — same `nodeId`, and **send `parameters.prompt`** on the run (plus video duration / aspect / resolution). Do not PUT the node to store the prompt for the run:

```bash
# images first (sync — result is in the response)
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"scene1","model":"nano-banana-pro","parameters":{"prompt":"scene 1: a dog discovers a surfboard on the beach, watercolor"}}'

# then the video (async — poll). Pass the scene output URLs as image_urls.
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-video-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"vid-1","model":"seedance-2.5","parameters":{"prompt":"@Image1 discovers the wave, @Image2 rides it — 5s cinematic","image_urls":["<scene1 outputUrl>","<scene2 outputUrl>"],"duration":5,"aspect_ratio":"16:9","resolution":"720p"}}'
```

Run nodes in dependency order (upstream first); each run spends credits only on success. Read `outputUrl` from the run response. Do not write it back onto the node.

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

Read `references/one-shot.md`, and `references/ugc-reference-board.json` when you need the graph. That export is the 180-node UGC reference board. **Do not clone all 180 nodes.**

Demo 1 copies **one** shot (Video Generator 1 in the example) and creates only those nodes, each with `POST /projects/:id/nodes`:

- a product image upload
- reference stills (image uploads)
- an optional image-generator keyframe, only when a still has to be generated
- one Seedance video node. The directing prompt lives on that node as `data.prompt` (see `references/one-shot.md`) — do not add a separate text node, and do not PUT a graph to place it

Use Path 2 for that small set of nodes. Wire uploads with a PUT of **edges only** (no `nodes` key): each upload `image-out` → `image-in-1`, `image-in-2`, … in order, and an end still to `lastframe-in` when the shot has one. Video Generator 1 uses `image-in-0` for the start still and `lastframe-in` for the end still. Cite stills in the prompt as `@Image1`, `@Image2` in wiring order. Write the prompt as one continuous shot, in the spirit of `references/one-shot.md`. For Seedance wording, see the prompt guide linked from the repo README.

URL fields in the example JSON are the placeholder `{{user-upload}}`. They are not fetchable. Put the user's real image URLs on the upload nodes. Those URLs must be https Firebase Storage or fal media URLs — `POST /nodes` rejects anything else. Do not work around that by PUTting `nodes`.

**Explicit “make a similar UGC video” runs.** When the user says “make a similar UGC video”, that sentence is the go-ahead: create a fresh project, POST this one-shot's nodes (not the whole board), PUT edges with no `nodes` key, and run it (keyframe first if you added one, then the Seedance video). Do not stop after describing the graph, and do not ask for a second confirmation.

**“Make it slower” edits that video node.** Stay in the project that holds the shot. Re-run **that** Seedance node with a higher `parameters.duration` (adjust `parameters.prompt` timing beats only if they would no longer match). `POST /nodes` will not overwrite the existing node, and PUT must not carry `nodes`. Do not create a new project, do not add a second video node, and do not clone the canvas.

**Template-name jobs still ask permission.** If the user names a template, or asks to run a job by template name, say what you are about to run and wait for a yes. A template name is not permission to spend credits.

## Rules

- **Fresh project per request** — see the Project rule.
- **Node writes are `POST /projects/:id/nodes` only.** Never PUT a body that contains `nodes`. PUT is title and edges. The server seeds `status`, `resultImage`, `resultVideo`, `outputText`, `errorMessage`, and `uploadStatus` — do not write them.
- **Brands:** never generate brand content from text alone — models don't know real logos/colors. Ask for 2–4 reference image URLs (logo, palette, product) and use an `/edit` model.
- **Credits:** each run spends the user's credits, charged on success (image ≈150 on nano-banana-pro). Before a batch (>3 runs) or any video, say what you're about to run and get a yes. An explicit “make a similar UGC video” or “make it slower” is that yes (see Demo 1). A template name is not.
- **One run per node at a time** — 409 means that node is still running; wait or use a new `nodeId`.
- **Never send an empty array** for an image param (`[]` → 422) — omit the param instead.

## Errors

| Status | Meaning | Do |
|---|---|---|
| 400 | invalid request (bad model id, bad handle, missing field), or a PUT body that contains `nodes` | read the error message — it names the problem; fix and retry once. For `nodes` on PUT, use `POST /projects/:id/nodes` instead. That rejected PUT saved nothing |
| 401 | missing/invalid key | re-check Setup and `JIFCHAT_BASE_URL` |
| 402 | out of credits | tell the user to top up in Settings → Balance |
| 404 | project/node missing or not theirs | you probably used a wrong `$PID` — re-check the create-project response |
| 409 | node already running, or edge slot taken | wait, or use a new `nodeId`/slot |
| 429 | rate limited | wait ~30s, retry once |

Full spec: `$B/api/openapi.json` · agent guide: `$B/llms.txt`
