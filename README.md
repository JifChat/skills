# JifChat skill

Skill for agents that generate images and videos on a JifChat canvas.

```bash
npx skills add JifChat/skills
```

Then paste a `jc_` API key. Details are in `INSTALL.md`.

## Layout

- `skills/jifchat/SKILL.md` — agent instructions
- `skills/jifchat/references/one-shot.md` — the one shot to copy (the only template). Every media URL field is `{{user-upload}}`.
- `skills/jifchat/references/character-consistency.md` — rules for a character, product, or location reused across shots

Seedance 2.5 prompt guide: https://docs.volcengine.com/docs/ark/seedance-2-5-prompt-guide?lang=zh
