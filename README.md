# JifChat skill

Skill for agents that generate images and videos on a JifChat canvas.

```bash
npx skills add JifChat/skills
```

Then paste a `jc_` API key. Details are in `INSTALL.md`.

## Layout

- `jifchat/SKILL.md` — agent instructions
- `jifchat/references/one-shot.md` — the one shot to copy, not all 180 nodes
- `jifchat/references/ugc-reference-board.json` — sanitized UGC reference board. Every media URL field is `{{user-upload}}`.

Seedance 2.5 prompt guide: https://docs.volcengine.com/docs/ark/seedance-2-5-prompt-guide?lang=zh
