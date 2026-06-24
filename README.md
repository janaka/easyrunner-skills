# EasyRunner Skills

Agent skills that let you drive [EasyRunner](https://easyrunner.xyz) — a
CLI-first single-server, self-hosting PaaS — straight from your coding agent, without
learning the `er` CLI. Ask the agent to prep, register, deploy, or update an
app and these skills route it through the right `er app` commands.

EasyRunner turns a single, low-cost server into your own app platform — deploy
from a git repo or a prebuilt image, give each app a custom domain with
automatic HTTPS, and pack in as many apps as the box can hold. Scales to multiple servers across providers. Push-button simplicity, on infrastructure you fully own.

| Skill | Wraps | What it does |
|---|---|---|
| [`er`](./skills/er/SKILL.md) | — | Router / overview — picks the right sub-skill |
| [`er-app-repo-prep`](./skills/er-app-repo-prep/SKILL.md) | (repo edits) | Make a repo deployable (`Dockerfile`, `.easyrunner` compose, service labels) |
| [`er-app-create`](./skills/er-app-create/SKILL.md) | `er app add` | Register a new app on a server |
| [`er-app-deploy`](./skills/er-app-deploy/SKILL.md) | `er app deploy` | Deploy, verify, and manage app lifecycle |
| [`er-app-update`](./skills/er-app-update/SKILL.md) | `er app update-details` | Reconfigure an existing app |

All skills follow the cross-runtime [`SKILL.md`](https://agentskills.io) format
and wrap only the public `er` CLI surface — no secrets.

## Prerequisites

- An EasyRunner server you can reach, plus the `er` CLI installed locally — see
  [easyrunner.xyz](https://easyrunner.xyz) for setup.
- One of the supported agent runtimes below.

## Install

### Claude Code

```text
/plugin marketplace add janaka/easyrunner-skills
/plugin install easyrunner-skills@easyrunner-skills
```

### Codex CLI

```bash
codex plugin install github:janaka/easyrunner-skills
```

### GitHub Copilot CLI

```bash
gh copilot extension install janaka/easyrunner-skills
```

### Gemini CLI

```bash
gemini extension install github:janaka/easyrunner-skills
```

> The exact subcommand for each runtime evolves; check the runtime's docs if a
> snippet fails. All four runtimes consume the same `SKILL.md` files.

## Version compatibility

Each skill targets a specific `er` CLI command surface, recorded in
[`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) under
`compatibility.min_er_version`. Check your CLI with `er --version`; if it's
older than `min_er_version`, upgrade with `brew upgrade easyrunner-cli` or
`pipx upgrade easyrunner-cli`.

## About

This repository is a **generated mirror** of the skills maintained in the
EasyRunner project and is published automatically — don't edit it directly.
For issues or requests, see [easyrunner.xyz](https://easyrunner.xyz).

Licensed under the MIT License — see [LICENSE](./LICENSE).
