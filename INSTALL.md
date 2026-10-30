# Install

## skills.sh (Cursor, Claude Code, Codex, and other Agent Skills clients)

```bash
npx skills add JifChat/skills
```

## Hermes

```bash
hermes skills install JifChat/skills/skills/jifchat
```

For a named profile (not the default one):

```bash
hermes -p <profile> skills install JifChat/skills/skills/jifchat
```

Hermes prompts for `JIFCHAT_API_KEY` when the skill loads (it reads `required_environment_variables`).

## OpenClaw / ClawHub

```bash
clawhub install @jifchat/jifchat
# or
openclaw skills install @jifchat/jifchat
```

OpenClaw injects `JIFCHAT_API_KEY` (the skill's `primaryEnv`) from its skill config.

## API key

Get a JifChat API key (`jc_...`) from JifChat → **Settings → Balance → API Keys → Create**.

- Recommended: set it as the `JIFCHAT_API_KEY` environment variable.
- Fallback: save it to `~/.jifchat_key` (mode 600). The skill reads this file only when `JIFCHAT_API_KEY` is not set, so existing file-based setups keep working.

Full setup, including the file command, is in `skills/jifchat/SKILL.md`.

The skill needs `curl` and `jq`. `ffmpeg` is optional and only used to stitch multi-shot videos.

## Fallback

Copy the `skills/jifchat/` folder into the agent skills directory, for example `~/.cursor/skills/jifchat`, `~/.claude/skills/jifchat`, or `~/.grok/skills/jifchat`. Open a new session after copying.
