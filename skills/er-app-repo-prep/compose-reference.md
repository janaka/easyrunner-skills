# Compose file reference for EasyRunner apps

The deploy pipeline reads `.easyrunner/docker-compose-app.yaml` from the app
repo and transforms it into Podman Quadlet units managed by systemd.

## Required labels (`xyz.easyrunner.service.*`)

| Label | Required | Example | Notes |
|---|---|---|---|
| `xyz.easyrunner.service.type` | yes | `web` | Today only `web` (Caddy-routed HTTP) is supported. |
| `xyz.easyrunner.service.framework` | yes | `nextjs`, `fastapi`, `flask`, `rails`, `standardbackend` | Free-form; informational. Use `standardbackend` if unsure. |
| `xyz.easyrunner.service.port` | yes | `"3000"` | **Quoted string.** Must match the port the app listens on inside the container. |
| `xyz.easyrunner.service.build-context` | flow_a | `"."` | Path relative to the repo root passed to `podman build`. |

## Network

Always:

```yaml
networks:
  - easyrunner_proxy_network
```

at the service level, and:

```yaml
networks:
  easyrunner_proxy_network:
    external: true
```

at the top level. Caddy attaches to this network and routes traffic by the
service name; without it the container won't be reachable.

## Build args injected by EasyRunner

These are passed automatically at deploy time — do not put them in `build.args`
unless you want to override them:

- `EASYRUNNER_APP_URL` — `https://<domain>` of the app
- `EASYRUNNER_APP_DOMAIN` — bare domain

Consume them in the `Dockerfile` builder stage with `ARG EASYRUNNER_APP_URL`.

## Volumes

Declare only volumes you actually need to persist across redeploys. A redeploy
recreates the container; anonymous volumes and bind mounts inside the build
tree are wiped.

```yaml
services:
  app:
    volumes:
      - app_data:/data

volumes:
  app_data:
```

## flow_a vs flow_b

- **flow_a** — build from source. EasyRunner clones the repo, runs
  `podman build` using `xyz.easyrunner.service.build-context`, then starts
  the container.
- **flow_b** — registry image. The compose file's `image:` is pulled from a
  registry (Docker Hub, GHCR, etc.). No `build-context` needed. Use this for
  third-party / pre-built apps.

The flow is chosen on `er app add` / `er app update-details` via
`--deploy-flow flow_a|flow_b`.

## Minimal flow_a example

```yaml
name: marketing-site

services:
  app:
    image: marketing-site:latest
    networks:
      - easyrunner_proxy_network
    labels:
      xyz.easyrunner.service.type: web
      xyz.easyrunner.service.framework: nextjs
      xyz.easyrunner.service.port: "3000"
      xyz.easyrunner.service.build-context: "."

networks:
  easyrunner_proxy_network:
    external: true
```

## Minimal flow_b example

```yaml
name: openclaw

services:
  app:
    image: ghcr.io/openclaw/openclaw:beta-2025-06-01
    networks:
      - easyrunner_proxy_network
    labels:
      xyz.easyrunner.service.type: web
      xyz.easyrunner.service.framework: standardbackend
      xyz.easyrunner.service.port: "18789"

networks:
  easyrunner_proxy_network:
    external: true
```
