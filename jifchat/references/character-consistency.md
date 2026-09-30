# Character consistency

Use these rules whenever the same character, product, or location appears in 2+ shots.
They come from the Kate reference board (`ugc-reference-board.json`).

**Lock the look in images, not in video.** Every shot is decided as a still first.
The video model only animates stills; it never invents a character.

1. **Build a character sheet first.** One image generator (`gpt-image`, quality `high`)
   combines the face, outfit, and signature props (e.g. helmet, bag) into a front + side
   full-body sheet. Every later shot of that character references this sheet.

2. **Name every reference node.** Give each upload and key generator a short, unique name
   (`MUM`, `SON`, `PRODUCT`, `FARM1`). Reuse the same reference nodes across shots; never
   upload a second copy of the same person or product.

3. **Cite references by name in the prompt.** Say which reference controls what:
   "The woman is MUM, wearing the outfit from the character sheet. Background: FARM1."
   Don't describe a character's looks from memory when a reference exists.

4. **Grow each shot from the previous still.** For a new angle or beat in the same scene,
   wire the previous keyframe in and ask for the change:
   "Based on SHOT3, over-the-shoulder from MUM toward SON, same place and light."

5. **Fix drift with a revision pass, not a reroll.** If a keyframe drifts (wrong outfit,
   face, or prop), run a revision on that still with the correct reference wired in:
   "Revise SHOT4: change the woman's outfit to MUM's outfit. Remove AI artifacts."

6. **Lock the style with references too.** Wire 1–2 style stills into every keyframe and
   repeat the same short style line (lighting, palette, grain) in each prompt.

7. **Video = image-to-video from finished keyframes only.** Wire the start still to
   `image-in-0` and, when the shot has a clear end pose, the end still to `lastframe-in`.
   For multi-beat shots, wire keyframes in order (`image-in-0`, `image-in-1`, …) and
   describe the sequence. Never text-to-video a recurring character.

8. **Open every video prompt with the continuity line.** "Use the uploaded image as the
   exact visual reference for the characters, clothing, props, and environment. Maintain
   continuity with the previous shot. One continuous shot, no cuts." Then write the
   action as timed beats (0–2s, 2–4s, …).
