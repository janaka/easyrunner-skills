---
name: er-app-update
description: Reconfigure an existing EasyRunner app — change domain, branch, repo URL, deploy flow, or refresh the snapshotted compose file. Wraps `er app update-details`. Use when the user says "change the domain on X", "update the compose for X", "switch X to a different branch", or similar.
license: MIT
compatibility: Requires `er` CLI. Changes take effect on the next `er app deploy`; this command does not redeploy.
metadata:
  author: easyrunner
  version: "1.0"
---

# Update details on a registered EasyRunner app

This skill wraps **one** command: `er app update-details`. It changes saved
config for an existing app — it does not redeploy. Tell the user to run
`er app deploy` afterwards for the changes to take effect.

## Synopsis

```
er app update-details NAME SERVER_NAME
                      [--app-name NAME]
                      [--description TEXT]
                      [--custom-domain DOMAIN]
                      [--repo-url URL]
                      [--default-deploy-branch BRANCH]
                      [--deploy-flow {flow_a|flow_b}]
                      [--compose-file PATH]
```

## What you can change

| Flag | Effect |
|---|---|
| `--app-name` | Rename the app (friendly name). |
| `--description` | Replace the short description. |
| `--custom-domain` | Set/replace the public domain. Triggers a DNS check + setup via the linked Cloudflare account, same as `er app add`. |
| `--repo-url` | Point flow_a at a different Git repo. |
| `--default-deploy-branch` | Change the default branch deployed when no `--branch` is passed. |
| `--deploy-flow` | Switch between `flow_a` and `flow_b`. |
| `--compose-file` | Refresh the snapshotted compose file for a flow_b app. **Requires the app to already be flow_b**, or pass `--deploy-flow flow_b` alongside. |

Pass only the flags you actually want to change. Omitted fields are left alone.

## Validation rules the CLI enforces

- `--compose-file` requires the effective flow to be `flow_b`. If the app is
  flow_a and you don't pass `--deploy-flow flow_b` in the same command, the
  CLI rejects with: `--compose-file requires the app to be flow_b. Pass
  --deploy-flow flow_b.`
- `--deploy-flow` must be `flow_a` or `flow_b`. Anything else is rejected.
- The compose file is parsed and validated locally with Podman's compose
  loader before being saved.

## Common workflows

### Change the custom domain

```bash
er app update-details marketing-site my-vps \
  --custom-domain new.example.com
```

Then redeploy:

```bash
er app deploy marketing-site my-vps
```

The CLI sets up DNS at command time, but Caddy isn't reconfigured until the
next deploy.

### Bump a flow_b image (e.g. OpenClaw beta)

1. Edit the local compose file to point at the new image tag.
2. Snapshot the updated compose into the app:

   ```bash
   er app update-details openclaw my-vps --compose-file ./openclaw-compose.yaml
   ```

3. Redeploy:

   ```bash
   er app deploy openclaw my-vps
   ```

### Switch an app from flow_a to flow_b

```bash
er app update-details my-app my-vps \
  --deploy-flow flow_b \
  --compose-file ./my-app-compose.yaml
```

You can pass both `--deploy-flow flow_b` and `--compose-file` in the same
invocation; the CLI evaluates them together.

### Change the default deploy branch

```bash
er app update-details my-app my-vps --default-deploy-branch production
```

## What this skill does NOT do

- Does not redeploy. Always remind the user to run `er app deploy` for
  changes to apply.
- Does not register a new app — that's `er-app-create`.
- Does not remove an app — that's `er app remove`.
- Does not manage secrets / env vars — outside this skill set.
