# One shot

Video Generator 1 on the Kate canvas (`examples/kate-canvas.json`). It is a Seedance image-to-video clip: uploaded stills in, one continuous shot out.

| | |
|---|---|
| Node | `videoGenerator-ztbYay5SDl8FN7N8RGnSm` |
| Name | Video Generator 1 |
| Model | `seedance-2.0-i2v` |
| Duration | 9 seconds |
| Aspect | 9:16 |
| Resolution | 720p |
| Audio | off |

## Inputs

Both inputs are image uploads (no generated keyframe on this node).

- Start still, handle `image-in-0`: **Image Upload 17**, file `S4_1_923b.png`, 941×1672.
- End still, handle `lastframe-in`: **Image Upload 14**, 941×1672. The export stored that file under a private storage path; the sanitized canvas uses `{{user-upload}}` there.

The directing prompt lives on the video node (`data.prompt`). Nothing is wired to `text-in`.

## The shot

One continuous take, no cuts, in a scorching Thai rice field, played as an exaggerated Thai TV comedy commercial.

The start still locks the characters, the large purple bag, the field, and the framing. A shirtless man in a straw hat snatches the bag from a woman, then digs through it without the contents being shown. The camera rotates to an over-the-shoulder medium shot from behind the woman as he throws the bag away and points at a helmeted person, shouting the Thai line "เดี๋ยว! ข้าวอยู่ไหน่น่ะ?!" (Wait! Where is the rice?!). The end still is the last frame. The prompt keeps the bag purple, forbids subtitles, music, and scene cuts, and holds on his angry face for the last second.

Timing written on the node: snatch (0–2s), frantic search (2–4s), throw and camera move (4–6s), shouted line (6–8s), hold (8–9s).
