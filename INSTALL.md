# Install

```bash
npx skills add JifChat/skills
```

Then paste a JifChat API key (`jc_...`). Setup for that key is in `jifchat/SKILL.md`.

## Fallback

Copy the `jifchat/` folder into the agent skills directory, for example `~/.cursor/skills/jifchat`, `~/.claude/skills/jifchat`, or `~/.grok/skills/jifchat`. Open a new session after copying.

## Hermes

```bash
hermes skills install JifChat/skills
```

For a named profile (not the default one):

```bash
hermes -p <profile> skills install JifChat/skills
```
