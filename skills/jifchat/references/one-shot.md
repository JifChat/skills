# One shot

Video Generator 1 on the ugc-reference-board (`ugc-reference-board.json`, beside this note). It is one Seedance image-to-video clip: uploaded stills in, one continuous shot out. Copy this one shot, not all 180 nodes.

The reference shot used `seedance-2.0-i2v` and an end still on `lastframe-in`. A new run uses `seedance-2.5-i2v` so the end still is kept (`end_image_url`). `seedance-2.0-i2v` in the export is the old model id, not the model to send.

A new multi-shot run uses `seedance-2.5-i2v` for start and end stills.

|            |                                        |
| ---------- | -------------------------------------- |
| Node       | `videoGenerator-ztbYay5SDl8FN7N8RGnSm` |
| Name       | Video Generator 1                      |
| Export model | `seedance-2.0-i2v` (old). New run: `seedance-2.5-i2v` |
| Duration   | 9 seconds                              |
| Aspect     | 9:16                                   |
| Resolution | 720p                                   |
| Audio      | off                                    |

## Inputs

Both inputs are image uploads (no generated keyframe on this node).

- Start still, handle `image-in-0`: **Image Upload 17**, file `S4_1_923b.png`, 941×1672. On a new run this URL is `image_url`.
- End still, handle `lastframe-in`: **Image Upload 14**, 941×1672. The export stored that file under a private storage path; the sanitized canvas uses `{{user-upload}}` there. On a new run this URL is `end_image_url`.

The directing prompt lives on the video node (`data.prompt`). Nothing is wired to `text-in`. `@` in a connected text node does nothing.

## Read this node without opening the file

`ugc-reference-board.json` is about 240KB. Do not read it whole, and do not pretty-print or rewrite it. From the `jifchat` directory, these pull Video Generator 1 and its edges only. The id below is the one in the file.

```bash
# the video node (id, model, prompt, settings) — not the other 179 nodes
jq -c --arg id 'videoGenerator-ztbYay5SDl8FN7N8RGnSm' '
  .nodes[] | select(.id == $id)
  | {id, type, position, width, height,
     name: .data.name, model: .data.model, duration: .data.duration,
     aspectRatio: .data.aspectRatio, resolution: .data.resolution,
     generateAudio: .data.generateAudio, prompt: .data.prompt}
' references/ugc-reference-board.json

# edges whose target is that node
jq -c --arg id 'videoGenerator-ztbYay5SDl8FN7N8RGnSm' '
  .edges[] | select(.target == $id)
  | {id, source, sourceHandle, target, targetHandle}
' references/ugc-reference-board.json

# the upload nodes on those edges (name and filename only)
jq -c --arg id 'videoGenerator-ztbYay5SDl8FN7N8RGnSm' '
  . as $doc
  | ($doc.edges | map(select(.target == $id))) as $edges
  | $edges[] as $e
  | $doc.nodes[]
  | select(.id == $e.source)
  | {handle: $e.targetHandle, id, type, name: .data.name, filename: .data.filename}
' references/ugc-reference-board.json
```

## The shot

One continuous take, no cuts, in a scorching Thai rice field, played as an exaggerated Thai TV comedy commercial.

The start still locks the characters, the large purple bag, the field, and the framing. A shirtless man in a straw hat snatches the bag from a woman, then digs through it without the contents being shown. The camera rotates to an over-the-shoulder medium shot from behind the woman as he throws the bag away and points at a helmeted person, shouting the Thai line "เดี๋ยว! ข้าวอยู่ไหนน่ะ?!" (Wait! Where is the rice?!). The end still is the last frame. The prompt keeps the bag purple, forbids subtitles, music, and scene cuts, and holds on his angry face for the last second.

Timing written on the node: snatch (0–2s), frantic search (2–4s), throw and camera move (4–6s), shouted line (6–8s), hold (8–9s).
