# JifChat skill

Skill for agents that generate images and videos on a JifChat canvas. The instructions are in `SKILL.md`. Worked material is in `examples/`.

Seedance 2.5 prompt guide: https://docs.volcengine.com/docs/ark/seedance-2-5-prompt-guide?lang=zh

## Install

Copy `SKILL.md` and the `examples/` directory together into a folder named `jifchat`. The skill reads `examples/one-shot.md` and `examples/kate-canvas.json`, so those files have to sit next to `SKILL.md`.

### Cursor

User-level:

```bash
mkdir -p ~/.cursor/skills/jifchat
cp SKILL.md ~/.cursor/skills/jifchat/SKILL.md
cp -R examples ~/.cursor/skills/jifchat/examples
```

For one project, use `.cursor/skills/jifchat/` or `.agents/skills/jifchat/` in that repo.

### Claude Code

```bash
mkdir -p ~/.claude/skills/jifchat
cp SKILL.md ~/.claude/skills/jifchat/SKILL.md
cp -R examples ~/.claude/skills/jifchat/examples
```

For one project, use `.claude/skills/jifchat/`.

### Hermes

```bash
mkdir -p ~/.hermes/skills/jifchat
cp SKILL.md ~/.hermes/skills/jifchat/SKILL.md
cp -R examples ~/.hermes/skills/jifchat/examples
```

From a checkout of this repository you can also run:

```bash
hermes skills install JifChat/skills --name jifchat
```

Hermes keeps `SKILL.md` and the `examples/` files it references.

### Grok

```bash
mkdir -p ~/.grok/skills/jifchat
cp SKILL.md ~/.grok/skills/jifchat/SKILL.md
cp -R examples ~/.grok/skills/jifchat/examples
```

For one project, use `.grok/skills/jifchat/`. Grok also loads skills from the Cursor and Claude Code directories above.

Open a new session after copying so the agent picks up the skill.

## Examples

- `examples/one-shot.md` — the continuous Seedance image-to-video shot Demo 1 copies.
- `examples/kate-canvas.json` — sanitized Kate canvas. Every media URL field is `{{user-upload}}`.
