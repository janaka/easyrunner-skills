---
name: er-app-create
description: Register a new application on an EasyRunner server in one shot. Wraps `er app add`. Use when the user says "add an app", "register X on my server", or "create a new app on easyrunner". Handles both flow_a (build from source) and flow_b (pre-built registry image with --compose-file).
license: MIT
compatibility: Requires `er` CLI with one-shot `er app add ... --deploy-flow flow_b --compose-file ...` support. See `commands-reference.md`.
metadata:
  author: easyrunner
  version: "1.0"
---

# Register a new app on an EasyRunner server

This skill wraps **one** CLI command: `er app add`. It collects the right
inputs, picks the right flow, then runs it.

## Inputs to gather

Ask the user (or infer from context) — do not run `er app add` until **all
required** inputs for the chosen flow are known:

| Input | Required for | Notes |
|---|---|---|
| `name` | both | Friendly app name, e.g. `marketing-site`. Used everywhere; pick something stable. |
| `server_name` | both | Friendly name of the target server (from `er server list`). |
| `repo_url` | flow_a | HTTPS Git URL of the app repo. Required for flow_a; optional for flow_b. |
| `--deploy-flow` | both | `flow_a` (default) builds from source. `flow_b` deploys a pre-built image. |
| `--compose-file` | flow_b | Path to a local docker-compose file to snapshot. **Requires `--deploy-flow flow_b`.** |
| `--custom-domain` | optional | Domain the app should be served at (e.g. `marketing.example.com`). Configures Caddy + DNS. |
| `--description` | optional | One-line description. |
| `--default-deploy-branch` | optional | Git branch to deploy from by default. Defaults to `main`. |

## Pick the flow

- The repo has a `Dockerfile` + `.easyrunner/docker-compose-app.yaml` and the
  user wants `er` to build it → **flow_a**. The `er-app-repo-prep` skill is
  the right precursor.
- The user is pointing at a pre-built image on a registry (Docker Hub, GHCR,
  etc.) and just wants to run it → **flow_b**, and they need to supply a
  local compose file via `--compose-file`. No repo URL is needed.

## Commands

### flow_a (build from source — the common case)

```bash
er app add <name> <server_name> <repo_url> \
  [--description "..."] \
  [--custom-domain marketing.example.com] \
  [--default-deploy-branch main]
```

`--deploy-flow flow_a` is the default; you can omit it.

### flow_b (pre-built registry image)

```bash
er app add <name> <server_name> \
  --deploy-flow flow_b \
  --compose-file ./path/to/compose.yaml \
  [--custom-domain app.example.com] \
  [--description "..."]
```

The compose file is **read and validated on the local machine at command
time**. Re-run with `er app update-details ... --compose-file <path>` to
change it later. The repo URL argument is allowed but optional for flow_b
(pass an empty string or omit if your shell allows).

## Validation the CLI will perform

Don't try to replicate these — let the CLI fail loudly:

- `--compose-file` without `--deploy-flow flow_b` → rejected.
- flow_a with no `repo_url` → rejected.
- App name already exists on that server → rejected.
- `--custom-domain` triggers a DNS lookup/setup step (Cloudflare). The CLI
  prints what it did or warns and skips on failure.

## After it succeeds

The app is registered in the local store but **not deployed**. Tell the user
to either:

1. Hand off to the `er-app-deploy` skill, or
2. Run `er app deploy <name> <server_name>` themselves.

For flow_b apps registered without `--compose-file`, the CLI emits an info
line reminding the operator to set one via `er app update-details` before
deploying. Surface that to the user.

See `commands-reference.md` for the exact flag list confirmed against the
current CLI.
