# `er app add` reference

This file mirrors the flags supported by `er app add` in the targeted CLI
version (see plugin `compatibility.min_er_version`). Always cross-check with
`er app add --help` on the user's machine.

## Synopsis

```
er app add NAME SERVER_NAME [REPO_URL]
           [--description TEXT]
           [--default-deploy-branch BRANCH]
           [--deploy-flow {flow_a|flow_b}]
           [--compose-file PATH]
```

> Public domains are declared per web service in the compose file via
> `xyz.easyrunner.service.domain` — there is no `--custom-domain` flag (removed
> in #222). DNS is provisioned at `er app deploy`.

## Arguments

- `NAME` — friendly app name (required, positional).
- `SERVER_NAME` — friendly name of the target server (required, positional).
- `REPO_URL` — HTTPS Git URL of the app repo. Required for flow_a; optional
  for flow_b.

## Options

| Flag | Type | Default | Purpose |
|---|---|---|---|
| `--description` | string | `""` | Short description of the application. |
| `--default-deploy-branch` | string | `main` | Default Git branch to deploy from; can be overridden per-deploy with `--branch`. |
| `--deploy-flow` | enum | `flow_a` | `flow_a` builds from source; `flow_b` runs a pre-built registry image. |
| `--compose-file` | path | none | Local Docker Compose file to snapshot for the app. **Requires `--deploy-flow flow_b`.** Read and validated at command time. |

## Behaviour rules

- `--compose-file` is only valid with `--deploy-flow flow_b`. The CLI rejects
  the combination otherwise with: `--compose-file requires --deploy-flow flow_b.`
- `flow_a` without a non-empty `REPO_URL` is rejected with: `flow_a requires a
  repository URL. Pass a repo URL, or use --deploy-flow flow_b.`
- Compose-file contents are validated locally with Podman's compose loader
  before the app is saved; a parse error prints the underlying error. For flow_b
  the `service.domain` labels are validated too.
- DNS is **not** set up at `add` time. At `er app deploy`, the CLI reads the
  compose (from the repo when one is defined, else the stored content), then
  creates/confirms an A record per `service.domain` via the linked Cloudflare
  account. Failures there are warnings; the deploy continues.
- A flow_b app registered **without** `--compose-file` is allowed, but the CLI
  prints: `flow_b app registered without a compose file. Set one via 'er app
  update-details --compose-file <path>' before deploying.`

## Examples

```bash
# flow_a — build from a GitHub repo (domains come from the repo's compose)
er app add marketing-site my-vps https://github.com/me/marketing-site.git

# flow_a — with a non-main branch
er app add marketing-site my-vps https://github.com/me/marketing-site.git \
  --default-deploy-branch production

# flow_b — registry image, snapshot a local compose file
er app add openclaw my-vps \
  --deploy-flow flow_b \
  --compose-file ./openclaw-compose.yaml

# flow_b — registered now, compose attached later
er app add openclaw my-vps --deploy-flow flow_b
# ...then
er app update-details openclaw my-vps --compose-file ./openclaw-compose.yaml
```
