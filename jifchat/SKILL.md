---
name: jifchat
version: 2.1.0
description: Generate images and videos on the user's JifChat infinite canvas (jif.dev). Use when the user asks to generate, create, or render an image, product shot, poster, or short video with JifChat, or says "/jifchat", "/generate image", "/generate video", or "make a similar UGC video". Results land in a JifChat canvas project and spend the user's JifChat credits.
---

# JifChat

JifChat is an AI design canvas. This skill creates canvas projects and runs image and video generation on the user's own account through the Canvas API that is live in production.

Default base URL is `https://chat.jif.dev`. Override only with `JIFCHAT_BASE_URL` (a key works only on the environment that created it).

## What this skill can and can't do

**Can**
- Create canvas projects and generate images and videos in them
- Upload image, video, and audio files, then place them on the canvas
- Build a graph with `POST /nodes` and `POST /edges` (text prompts, uploads, generators) and lay it out with `position`

**Can't**
- Touch projects shared with the user (owner-only; others return 404)
- Chat with JifChat's assistant, manage billing, or create API keys

Templates are not available.

## Setup

1. Sign in to JifChat → **Settings → Balance → API Keys** → **Create**. Copy the `jc_...` key — it is shown once.
2. Save it where this skill can read it:
   ```bash
   install -m 600 /dev/null ~/.jifchat_key && printf '%s' 'jc_YOUR_KEY' > ~/.jifchat_key
   ```
   `~/.jifchat_key` is the only recommended location. Never patch that file with an edit tool — the diff prints the key into the transcript. To replace a key, overwrite it with the `printf` above; to read it, use `$(cat ~/.jifchat_key)`.
3. Optional: to run against a non-production environment you own, set `JIFCHAT_BASE_URL` to that environment's origin. Default is `https://chat.jif.dev`.

> OAuth sign-in (authorize in the browser, no key copy-paste) is planned. Until then, setup is the manual key above.

## Calling the API

Call the API with curl. A bare Python client gets Cloudflare 1010.

Start every shell session with:

```bash
B=${JIFCHAT_BASE_URL:-https://chat.jif.dev}
K=${JIFCHAT_API_KEY:-$(cat ~/.jifchat_key 2>/dev/null)}
[ -n "$K" ] || echo "No JifChat key — see Setup"
```

Every curl sends `Authorization: Bearer $K`. **Never print, echo, or log the key.** If the key is missing, stop and walk the user through Setup. Check the key without spending credits:

```bash
curl -s "$B/api/v1/canvas/projects" -H "Authorization: Bearer $K"
```

200 = OK, 401 = bad key.

The canvas link for a project is `$B/canvas/$PID`.

### Surface

Production mounts the canvas key routes under `/api/v1/canvas`. Use these; do not invent hosts or paths.

- `POST /projects` and `GET /projects` — create (`{"title":"..."}` → `_id`) and list
- `GET /projects/:id`, `PUT /projects/:id`, `DELETE /projects/:id`, `POST /projects/:id/copy`
- `POST /projects/:id/nodes` — add one node
- `POST /projects/:id/edges` — connect two nodes
- `POST /files/{kind}` — upload (`kind` is `images`, `videos`, or `audio`)
- `POST /projects/:id/run-node`, `POST /projects/:id/run-video-node`, `POST /projects/:id/run-text-node`, `POST /projects/:id/check-run`
- `GET /models`, `GET /balance`
- `GET/POST /groups` — team-scoped project groups

`PUT /projects/:id` updates the title and other project fields. A body that contains `nodes` is a 400, and nothing in that body is written. Build the graph with `POST /nodes` and `POST /edges`.

`position` is optional on `POST /nodes` and on the run routes. If it is omitted the server places the node at the previous node's x + that node's layout width + 200, same y. Widths: `textNode` 320, `imageUpload` 320, `videoUpload` 320, `audioUpload` 345, generators 736. A measured 9:16 video card is about 720×1280, so the next shot row is `y += 1480` (measured, not a layout constant). **Always send `position` anyway.** Explicit positions keep a 200px gap (text at 0, generator at 520).

## Uploads

`POST /api/v1/canvas/files/{kind}` with multipart form-data, field name `file`. `kind` is `images`, `videos`, or `audio`. Images also require `width` and `height` form fields (pixels). Uploads are free.

```bash
curl -s -X POST "$B/api/v1/canvas/files/images" \
  -H "Authorization: Bearer $K" \
  -F "file=@./still.png" \
  -F "width=941" \
  -F "height=1672"
```

201 returns `{ success, url, filepath, filename, type }`. Put that `url` on an upload node (`imageUpload` uses `data.imageUrl` or `data.mediaUrl`; `videoUpload` and `audioUpload` use `data.mediaUrl`). The stored URL must be https Firebase Storage or fal media. Anything else is 400. Upload URLs can be reused on a new project.

The server may downscale a large upload (the image strategy caps the long side at 2000px). The `width` / `height` in the 201 response are the stored dimensions — treat them as authoritative and pass those when you place the file, not the numbers you measured locally.

Checks, each a 400: kind, then `file_id` (a UUID if you send one; omit it and one is generated), then mime, then size, then width and height for images. Images: jpeg, gif, png, webp, up to 30 MB (HEIC/HEIF is rejected). Videos: mp4, quicktime, webm, x-m4v, ogg, up to 200 MB. Audio: mpeg, mp3, wav, x-wav, ogg, webm, aac, flac, mp4, up to 15 MB. An image whose bytes cannot be decoded is 400 and nothing is stored. A third upload while two are in flight is 429. Videos or audio on a file strategy with no upload handler is 501.

When you need a product packshot and the brand site blocks automated fetches (403 / bot wall), pull the archived image from the Wayback Machine using its `id_` raw form (`https://web.archive.org/web/<timestamp>id_/<original-url>`), which returns the original bytes rather than the viewer page. If that fails, ask the user for the file. Never invent a product URL.

If `POST /files/audio` is 400 or 501, upload that file with `POST /files/videos` and place the URL on an `audioUpload` node (`audio-out` → `audio-in-0`). Audio alone is not a Seedance reference. If the upload still fails, ask the user to upload on the canvas or pass an https Firebase or fal URL. Do not invent a host or a URL.

## Nodes, edges, layout

**Node body** — `{ nodeId, nodeType, position, data }`. `nodeId` is a non-empty string, at most 256 characters. `nodeType` is `textNode`, `imageUpload`, `videoUpload`, `audioUpload`, `imageGenerator`, `textGenerator`, or `videoGenerator`. Existing nodes are never overwritten.

- `textNode`: `data.text` (clamped to 8000 characters). This is how an **image** generator gets its prompt.
- `imageGenerator` / `videoGenerator`: `data.model`, plus stored settings in camelCase (`resolution`, `aspectRatio`, `duration`, `quality`, `imageSize`, `generateAudio`). The server seeds `status` and the result fields. Do not send those.
- An image generator's prompt is a `textNode` wired `text-out` → `text-in`. The canvas Run button stays disabled without that edge.
- Video `@Image1` lives in the video node's `data.prompt`. `@` in a connected text node does nothing. The spoken line for that shot goes on that same `data.prompt`.

**Edge body** — `{ source, sourceHandle, target, targetHandle }`. The API builds the edge id and stroke. One edge per target slot. Text fan-in is one (`409` if the slot is taken).

Handles are 0-based.

- Text: `text-out` → `text-in`
- Images: `image-out` → `image-in-0`, then `image-in-1`, and so on. The first image slot is `image-in-0`.
- End still: `image-out` → `lastframe-in` (video generators only)
- Reference clips on reference-to-video: `video-out` → `video-in-0` .. `video-in-2`, `audio-out` → `audio-in-0` .. `audio-in-2`

**Layout.** Gap is 200. Next `x` = previous card width + 200. Widths: `textNode` 320, `imageUpload` 320, `videoUpload` 320, `audioUpload` 345, generators 736. A measured 9:16 video card is about 720×1280, so the next shot row is `y += 1480` (measured, not a layout constant). Explicit positions in the examples keep a 200px gap (text at 0, generator at 520).

```bash
post_node() { curl -s -X POST "$B/api/v1/canvas/projects/$PID/nodes" -H "Authorization: Bearer $K" -H 'Content-Type: application/json' -d "$1"; }
post_edge() { curl -s -X POST "$B/api/v1/canvas/projects/$PID/edges" -H "Authorization: Bearer $K" -H 'Content-Type: application/json' -d "$1"; }

# image: text card, then the generator 200px to its right (320 + 200 = 520)
post_node '{"nodeId":"prompt-1","nodeType":"textNode","position":{"x":0,"y":0},"data":{"text":"a shiba inu on a surfboard, watercolor"}}'
post_node '{"nodeId":"img-1","nodeType":"imageGenerator","position":{"x":520,"y":0},"data":{"model":"nano-banana-pro","resolution":"1K","aspectRatio":"1:1"}}'
post_edge '{"source":"prompt-1","sourceHandle":"text-out","target":"img-1","targetHandle":"text-in"}'
```

End-still video (start on `image-in-0`, end on `lastframe-in`). Do not run it until the user says yes — see Journey.

```bash
post_node '{"nodeId":"start","nodeType":"imageUpload","position":{"x":0,"y":0},"data":{"imageUrl":"<url from POST /files/images>"}}'
post_node '{"nodeId":"end","nodeType":"imageUpload","position":{"x":0,"y":620},"data":{"imageUrl":"<url from POST /files/images>"}}'
post_node '{"nodeId":"vid-1","nodeType":"videoGenerator","position":{"x":520,"y":0},"data":{"model":"seedance-2.5-i2v","prompt":"@Image1 holds the bag, then shouts the line.","duration":"5","aspectRatio":"9:16","resolution":"720p","generateAudio":false}}'
post_edge '{"source":"start","sourceHandle":"image-out","target":"vid-1","targetHandle":"image-in-0"}'
post_edge '{"source":"end","sourceHandle":"image-out","target":"vid-1","targetHandle":"lastframe-in"}'
```

Write image prompts on the text node and video prompts on `data.prompt` before `run-*`. Never send a probe `parameters.prompt` at a node whose text you still need. Run calls send `parameters` to the model. They do not read `data.prompt`. Provider keys are snake_case (`aspect_ratio`, `image_url`, `end_image_url`, `image_urls`, `image_size`). A `parameters.prompt` on `run-node` / `run-video-node` also creates or updates the upstream text node (it will not add a second text edge). For video, still put `@Image1` in `data.prompt`; copying `@` onto the text node does nothing.

## Models

One rule, not a menu. `seedance-2.0-i2v` and `kling-v3` in `references/ugc-reference-board.json` are the old export, not choices.

- End still (`lastframe-in`): model id `seedance-2.5-i2v`. Start still is `image_url` (the image on `image-in-0`). End still is `end_image_url`. The bare id `seedance-2.5` is reference-to-video and does not forward an end frame.
- Several reference images and no end lock: `seedance-2.5`, with those images in `image_urls`. Cite them as `@Image1`, `@Image2`, … in the video node's `data.prompt` (`image-in-0` is `@Image1`).
- Cheap default image: `nano-banana-pro`. `1K` / `2K` / `4K` is that model's `resolution` only.
- `gpt-image` uses `quality` (`low`, `medium`, `high`) and size presets (`image_size`: `auto`, `square_hd`, `square`, `portrait_4_3`, `portrait_16_9`, `landscape_4_3`, `landscape_16_9`). It does not take 1K/2K/4K. `references/character-consistency.md` overrides the cheap default for a repeated character: sheet and keyframes use `gpt-image` at quality `high`.
- Image edit with reference stills: run model `${model}/edit` plus `image_urls`. Do not store `/edit` on the node.

Quote cost from `GET /api/v1/canvas/models` **on the host that will run the job** — re-fetch it there, never reuse a quote from another host. If `models` has an entry for the model, use that table with duration and resolution. If `models` is empty, quote the flat `imageCreditCost` (a 4K image run is `imageCreditCost4K`) or `videoCreditCost` on that same response (a video quote is that flat amount, not a per-second rate). If you cannot reach `/models`, say "price unknown" — never quote from memory or from the reference board.

A run that fails at the provider is **never charged**: its record reads `creditCost: 0`, so there is nothing to refund. Snapshot `GET /balance` before and after every run and confirm spend on the balance, not on the run row. If the balance moved for a run that reads `failed`, that is the save-race bug below — report it, do not blindly re-run.

```bash
curl -s "$B/api/v1/canvas/models" -H "Authorization: Bearer $K"
curl -s "$B/api/v1/canvas/balance" -H "Authorization: Bearer $K"
```

## Journey

Templates are not available.

- **Missing subject or action:** ask once for every missing detail, then stop. Do not create a project yet.
- **One cheap default image:** create the project and run (`nano-banana-pro`, `resolution` `1K`). Always send `position`.

```bash
curl -s -X POST "$B/api/v1/canvas/projects" \
  -H "Authorization: Bearer $K" -H 'Content-Type: application/json' \
  -d '{"title":"<short descriptive title>"}'
# → { "_id": "..." } — this is $PID

curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" \
  -H "Authorization: Bearer $K" -H 'Content-Type: application/json' \
  -d '{"nodeId":"img-1","model":"nano-banana-pro","parameters":{"prompt":"<the prompt>","resolution":"1K"},"position":{"x":520,"y":0}}'
```

- **Any video, including one video:** `POST` the nodes and edges, do not run, reply with the canvas link (`$B/canvas/$PID`), a plain plan, and the cost, then wait.
- **No:** `DELETE` the project.

```bash
curl -s -X DELETE "$B/api/v1/canvas/projects/$PID" -H "Authorization: Bearer $K"
```

- **Yes:** reply that it started, with the link and a rough ETA, then run upstream first (a keyframe image before the video that uses it).
- Before generating keyframe stills, add up what the whole job will cost (every still plus every video) and check it against `GET /balance`. Stills for a video that the balance cannot pay for are wasted spend.
- Pass `image_url` and `end_image_url` for `seedance-2.5-i2v`, or `image_urls` for `seedance-2.5`.

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-video-node" \
  -H "Authorization: Bearer $K" -H 'Content-Type: application/json' \
  -d '{"nodeId":"vid-1","model":"seedance-2.5-i2v","parameters":{"prompt":"<same directing line as data.prompt>","image_url":"<start url>","end_image_url":"<end url>","duration":5,"aspect_ratio":"9:16","resolution":"720p"}}'
```

- Expect roughly 5–12 minutes per video clip when running serially. Poll about every 10–15 seconds — not a tight loop, and do not treat a slow run as a failure.
- **Poll** `GET /projects/:id` about every 10–15 seconds and read the run whose `responseUrl` matches. Do not poll `check-run`: while a finished video is being saved, a `check-run` call starts a second poller that marks the run `failed` (`Invalid file source`) even though the video was in fact made. After about 8 minutes, stop and hand over the canvas link. The project stays; do not re-generate. Watch the server's own concurrency cap (`CANVAS_MAX_CONCURRENT_RUNS`, 5 by default): on `Too many concurrent canvas runs`, wait. Do not change the cap.

```bash
curl -s "$B/api/v1/canvas/projects/$PID" -H "Authorization: Bearer $K" \
  | jq '.runs[] | select(.responseUrl=="<run.responseUrl>") | {status, outputUrl, creditCost, errorMessage}'
```

- Call `check-run` only to recover a run still `running` after about 10 minutes (server restart). Call it once, not in a loop.

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/check-run" \
  -H "Authorization: Bearer $K" -H 'Content-Type: application/json' \
  -d '{"responseUrl":"<run.responseUrl>"}'
```

- While videos are running, ask the user to close this project's canvas tab. The canvas also calls `check-run` and can trigger the same false failure.

- **Finished:** send the file in chat and the canvas link.
- **Error:** the reply includes the canvas link.
- Running more than one job in parallel? Keep one state file per run (`$PID`, the `responseUrl`s, the balance snapshot) so the runs cannot overwrite each other's bookkeeping.
- **Later edits** stay in that project. Do not create a new one for "make it slower" or a line change. `POST /nodes` will not overwrite the existing node; change duration or prompt on the run `parameters`, and keep `@Image1` on the video node's `data.prompt`.
- If the canvas tab is open, tell the user to refresh before editing so autosave does not write the old graph back.

### Multi-shot

Same character, product, or location in 2+ shots: read `references/character-consistency.md` before building the graph. Do not clone all 180 nodes. Read `references/one-shot.md` for the one shot to copy.

- One character still, reused on every shot (same upload, wired to each shot).
- The same character description in every video prompt.
- The spoken line on that video node's `data.prompt`.
- **Check the line** before moving on: transcribe the clip with a neutral pass, then a second pass biased toward the expected words. Digits and Latin words are the usual misses. Never trust a single pass.
- Run shot 1 and wait before the rest.
- Do not run a video on a keyframe that already looks wrong.
- Chain the shots as keyframe n+1 from shot n's last frame: wire shot n's output still to `image-in-0` as the new `image_url`. One character still, reused on every shot, is the fallback for a hard cut.
- Action beats need at least 2 seconds of screen time, and must not sit in the last second of a clip that carries a spoken line — the line needs that second.
- After a multi-shot stitch, `POST /files/videos` and add a `videoUpload` node, then send the canvas link.

Stitch the clips before uploading. Normalize each clip first, then concat without re-encoding:

```bash
for f in shot*.mp4; do
  ffmpeg -y -i "$f" -c:v libx264 -crf 18 -video_track_timescale 30000 -c:a aac "norm_$f"
done
printf "file 'norm_shot1.mp4'\nfile 'norm_shot2.mp4'\n" > list.txt
ffmpeg -y -f concat -safe 0 -i list.txt -c copy stitch.mp4
```

Keeping `-video_track_timescale 30000` on every clip is what makes the `-c copy` concat work.

## Demo 1 — one UGC shot

`references/ugc-reference-board.json` is the 180-node board. **Do not clone all 180 nodes.** Copy the one shot in `references/one-shot.md` (Video Generator 1): product and reference uploads, an image-generator keyframe only when a still has to be generated, and one video node. URL fields in the JSON are the placeholder `{{user-upload}}`. They are not fetchable. Upload the user's files with `POST /files/images`, or use an https Firebase or fal URL they pass.

A new run of that shot uses `seedance-2.5-i2v` so the end still is kept. Wire `image-in-0` and `lastframe-in`, put the directing prompt (including the spoken line) on the video node's `data.prompt`, then follow the video journey: post the graph, send the link, the plan, and the cost, and wait. Do not run before yes.

## Errors

| Status | Meaning | Do |
|---|---|---|
| 400 | bad model, handle, field, or URL; image missing width/height; PUT body contains `nodes` | read the error, fix once. A rejected PUT saved nothing |
| 401 | missing/invalid key | re-check Setup and `JIFCHAT_BASE_URL` |
| 402 | out of credits — the body carries `required` and `available` | report both, tell the user to top up in Settings → Balance. A 402 is refused before the provider is called, so nothing was generated and nothing was charged |
| 404 | project or node missing, or not theirs | re-check `$PID` |
| 409 | node already running, or edge slot taken | wait, or use a new `nodeId` / slot |
| 429 | rate limited, or too many uploads in flight (default 2) | wait, retry once |
| 501 | videos/audio upload not supported by the active file strategy | for audio, `POST /files/videos` and place the URL on an `audioUpload` node; otherwise ask the user to upload on the canvas or pass an https Firebase or fal URL |
| 524 | gateway timeout on a run | do not re-run that `nodeId`. Read `runs[].outputUrl` from `GET /projects/:id` |
| — | `run-*` hangs or returns an empty body | run every `run-*` call with `--max-time 300`. On a timeout or empty body, read `runs[]` first: a terminal run means the job did land — do not re-run it |

`Invalid file source` on a video run is a known false failure: the video was generated and saved, but the run was marked `failed` while saving. Do not retry right away. Compare `GET /balance` with the snapshot from before the run. If it dropped by the run cost, that charge landed on a run that reads `failed` — report it as a bug (a failed run must never be charged) and tell the user the video is in their JifChat files (the file name contains `canvas_<nodeId>_`). Ask before re-running. If the balance did not drop, retry the same payload once.

`fetch failed`: retry the same payload up to 3 times before changing files or the project.

An error reply includes the canvas link. Full spec: `$B/api/openapi.json`.
