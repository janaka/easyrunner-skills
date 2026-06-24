---
name: er-app-deploy
description: Deploy an existing EasyRunner app to its server. Wraps `er app deploy`. Use when the user says "deploy X", "ship the latest commit", or "redeploy X to my server". Handles branch overrides and surfaces deployment failures/recovery guidance.
license: MIT
compatibility: Requires `er` CLI. flow_a apps additionally require a linked GitHub token (`er auth login github`).
metadata:
  author: easyrunner
  version: "1.0"
---

# Deploy an app to its EasyRunner server

This skill wraps **one** command: `er app deploy`. It assumes the app has
already been registered with `er app add` (see the `er-app-create` skill).

## Synopsis

```bash
er app deploy <name> <server_name> [--branch BRANCH]
```

## Inputs

| Input | Required | Notes |
|---|---|---|
| `<name>` | yes | Friendly app name as registered. |
| `<server_name>` | yes | Friendly server name. |
| `--branch` | optional | Git branch to deploy from for this run. Falls back to the app's `default_deploy_branch` (default `main`). flow_a only. |

## Preflight checks (do these first)

1. **App exists on the server.** If unsure, suggest `er app list <server_name>`.
2. **flow_a apps need a GitHub token.** The deploy will fail with
   `❌ No GitHub token found. Please run 'er auth login github' first.` if
   none is linked. If the user hasn't run `er auth login github`, surface
   that as the first thing to do.
3. **flow_b apps need a compose file.** If `er app add` was run without
   `--compose-file`, the deploy needs one set first via
   `er app update-details <name> <server_name> --compose-file <path>`.
4. **CLI version.** `er --version` should be ≥ the plugin's `min_er_version`.

## Running it

For the common case:

```bash
er app deploy marketing-site my-vps
```

To deploy from a non-default branch for one run:

```bash
er app deploy marketing-site my-vps --branch feature/new-landing
```

The CLI shows a 7-step Rich progress bar covering: clone/pull, build, image
load, Quadlet generation, systemd reload, container start, Caddy route
update. On success it prints the public URL if a custom domain is set.

## Common failures and what to tell the user

- **"No GitHub token found"** — `er auth login github` first.
- **SSH / connection errors** — the deployment is resumable. Re-running the
  same command is safe; already-completed steps are skipped or updated.
  The CLI itself surfaces this hint.
- **502 from Caddy after a successful deploy** — almost always a port
  mismatch between the Dockerfile and `xyz.easyrunner.service.port`. Hand
  off to `er-app-repo-prep` to fix the compose label.
- **flow_b: "Failed to pull image"** — the tag in the compose file is wrong
  or no longer published. Update the compose `image:`, then either
  `er app update-details ... --compose-file <path>` and redeploy, or fix the
  tag and redeploy.

## What this skill does NOT do

- Does not edit app config — that's `er-app-update`.
- Does not register a new app — that's `er-app-create`.
- Does not provision the server — that's outside the current skill set.
- Does not link GitHub / Cloudflare — point at `er auth login github`,
  `er link cloudflare`.

After a successful deploy, suggest `er app status <name> <server_name>` for
a live view.
