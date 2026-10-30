# Publishing

Maintainer notes for releasing `skills/jifchat` to ClawHub and the Hermes Skills Hub. The skill is already indexed on skills.sh from this repo.

## ClawHub (OpenClaw)

Publisher: the org publisher **`jifchat`** (already created). The package is `@jifchat/jifchat`.

1. Install the CLI:
   ```bash
   npm i -g clawhub
   ```
2. Log in with an account that has publisher access to `jifchat`:
   ```bash
   clawhub login              # device flow: open the printed URL and authorize
   # or, unattended / CI:
   clawhub login --token <clawhub-api-token>
   clawhub whoami
   ```
   Never commit the token.
3. Dry run first. It resolves the publish without uploading:
   ```bash
   clawhub skill publish ./skills/jifchat --slug jifchat --owner jifchat --version 1.0.0 --dry-run
   ```
4. Publish for real:
   ```bash
   clawhub skill publish ./skills/jifchat --slug jifchat --owner jifchat --version 1.0.0
   ```
5. Wait for the security scan. Every release is checked (VirusTotal, static analysis, NVIDIA SkillSpector) and gets a Clean, Suspicious, or Malicious verdict; the version is not public until the checks finish. Check the verdict on the skill page, or with:
   ```bash
   clawhub inspect @jifchat/jifchat --version 1.0.0
   ```
   A mismatch between declared and actual behavior is flagged, so keep `metadata.openclaw` (`primaryEnv`, `envVars`, `requires.bins`) in line with what `SKILL.md` actually uses.
6. Verify the install:
   ```bash
   openclaw skills install @jifchat/jifchat
   ```

### Version bumps

- `metadata.version` in `skills/jifchat/SKILL.md` is the release version (semver). CI checks it.
- For each release, bump `metadata.version`, merge, then publish with the same `--version`. A published version number can't be reused with different contents.
- Without `--version`, ClawHub publishes the next patch version of changed content.

### License on ClawHub

Everything published on ClawHub is released under **MIT-0** (no attribution required). ClawHub does not support per-skill license overrides, so `SKILL.md` carries no `license` field. This repository itself stays under the MIT `LICENSE`.

## Hermes Skills Hub

Nothing to upload. Hermes discovers the skill through skills.sh and ClawHub, and its centralized hub index re-indexes those sources twice daily. After a ClawHub release, allow up to about 12 hours before it shows up in the index (a direct `hermes skills search` also queries the registries live).

Users can also install straight from this repo:

```bash
hermes skills install JifChat/skills/skills/jifchat
```

Verify:

```bash
hermes skills search jifchat
hermes skills inspect JifChat/skills/skills/jifchat
```

Hermes reads `required_environment_variables` (`JIFCHAT_API_KEY`) from the top level of the frontmatter only, and prompts for the key when the skill loads.
