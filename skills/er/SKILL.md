---
name: er
description: Router / overview skill for EasyRunner. Use when the user mentions easyrunner, the `er` CLI, deploying an app to their own server, or asks something like "deploy this to my easyrunner box". Points at the right sub-skill (repo prep, app create, deploy, update).
license: MIT
compatibility: Requires the `er` CLI installed locally. Targets `er` CLI version recorded in the plugin manifest (`min_er_version`). Run `er --version` to check; upgrade with `brew upgrade easyrunner-cli` or `pipx upgrade easyrunner-cli`.
metadata:
  author: easyrunner
  version: "1.0"
---

# EasyRunner router

EasyRunner is a single-server self-hosting PaaS. The `er` CLI configures a remote
Ubuntu host as a web host and deploys web apps to it via Podman + systemd Quadlet
units, fronted by Caddy.

This skill is the entry point. **It does not run commands itself.** It picks the
right sub-skill for the task.

## Pick a sub-skill

| User intent | Use this skill |
|---|---|
| "Make this repo deployable to easyrunner" / "add a Dockerfile + compose for easyrunner" | `er-app-repo-prep` |
| "Register a new app on my server" / "add an app called X" | `er-app-create` |
| "Deploy this app" / "ship the latest commit" / "redeploy X" | `er-app-deploy` |
| "Change the domain / branch / compose / description on an app" | `er-app-update` |

If the user is doing a clean greenfield rollout, the natural order is:

1. `er-app-repo-prep` — get the repo into the shape `er` expects
2. `er-app-create` — register the app on a server
3. `er-app-deploy` — push it live

## Version compatibility

These skills target a specific `er` CLI version recorded in the plugin's
`plugin.json` under `compatibility.min_er_version`. Before running anything,
confirm:

```bash
er --version
```

If the local CLI is older than `min_er_version`, ask the user to upgrade:

```bash
# Homebrew
brew upgrade easyrunner-cli
# or pipx
pipx upgrade easyrunner-cli
```

If the local CLI is newer than `min_er_version`, the skill instructions may
omit newer flags but should still work for the documented happy path. Prefer
`er <command> --help` over guessing.

## Ground rules for every sub-skill

- Always read `er <command> --help` first when unsure. The CLI's `--help` text
  is the source of truth for current flags.
- Never invent flags. If a flag isn't in the skill or `--help`, don't pass it.
- The CLI is interactive in places (DNS confirmations, prereq prompts). When
  driving it from an agent, surface the prompt back to the user — don't auto-
  answer destructive prompts.
- Server names are referenced by their friendly name (e.g. `my-vps`), not IP.
- App names are referenced by their friendly name (e.g. `marketing-site`).

## Out of scope here

Server provisioning, license setup, mesh/wireguard, secret vault, and DNS
linking are not yet covered by sub-skills. For those, point the user at
`er <noun> --help` and the [easyrunner docs](https://docs.easyrunner.xyz).
