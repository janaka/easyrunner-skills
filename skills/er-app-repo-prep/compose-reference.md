# Compose file reference for EasyRunner apps

The deploy pipeline reads `.easyrunner/docker-compose-app.yaml` from the app
repo and transforms it into Podman Quadlet units managed by systemd.

## Required labels (`xyz.easyrunner.service.*`)

| Label | Required | Example | Notes |
|---|---|---|---|
| `xyz.easyrunner.service.type` | yes | `web` | `web` = publicly routed via Caddy. `internal` / `worker` = not routed (no domain). |
| `xyz.easyrunner.service.domain` | web only | `shop.example.com` | **The public domain for this web service.** Required on every `web` service; must be unique within the app. The compose file is the source of truth for routing — there is no app-level `--domain` (removed in #222). |
| `xyz.easyrunner.service.framework` | web only | `nextjs`, `fastapi`, `flask`, `rails`, `standardbackend` | Free-form; informational. Use `standardbackend` if unsure. |
| `xyz.easyrunner.service.port` | web only | `"3000"` | **Quoted string.** Must match the port the app listens on inside the container. |
| `xyz.easyrunner.service.build-context` | flow_a | `"."` | Path relative to the repo root passed to `podman build`. |

An app can declare **multiple `web` services**, each with its own
`service.domain` — they are all routed (e.g. a storefront, an admin, and an API
on three different domains). DNS A-records for every declared domain are
provisioned at deploy time.

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

at the top level. Caddy attaches to this network and routes each `web` service
to its `service.domain`; without it the container won't be reachable.

## Talking to internal services (e.g. a database or redis)

`internal` / `worker` services are **not** put on `easyrunner_proxy_network` —
put them on a private network you declare yourself, and the consuming service
joins it too. On a private network EasyRunner registers each service under its
**bare compose name**, so siblings reach it the normal docker-compose way:

```yaml
services:
  backend:
    networks: [easyrunner_proxy_network, medusa_internal]
    environment:
      - REDIS_URL=redis://redis:6379   # bare service name resolves
  redis:
    networks: [medusa_internal]        # private only — not exposed publicly
networks:
  easyrunner_proxy_network:
    external: true
  medusa_internal:
    driver: bridge
```

Bare-name resolution is only registered on private networks (never on the shared
`easyrunner_proxy_network`), so names can't collide across apps.

## Env vars injected by EasyRunner

Injected automatically at build time (build args) **and** runtime (container
env). Don't put them in `build.args`/`environment` unless overriding.

Per web service (its own domain):

- `EASYRUNNER_APP_URL` — `https://<this service's domain>`
- `EASYRUNNER_APP_DOMAIN` — this service's bare domain

For every public web service in the app, injected into all containers so they
can address each other publicly (e.g. a storefront calling the backend API):

- `EASYRUNNER_SERVICE_<NAME>_URL` — `https://<that service's domain>`
- `EASYRUNNER_SERVICE_<NAME>_DOMAIN` — that service's bare domain

`<NAME>` is the compose service name upper-cased with non-alphanumerics → `_`
(e.g. `store-front` → `EASYRUNNER_SERVICE_STORE_FRONT_URL`).

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
      xyz.easyrunner.service.domain: chat.example.com

networks:
  easyrunner_proxy_network:
    external: true
```

## Multiple public services (e.g. a Medusa stack)

Three `web` services on three domains, plus an `internal` redis the backend
reaches by bare name on a private network:

```yaml
name: medusa

services:
  storefront:
    image: ghcr.io/acme/storefront:latest
    networks: [easyrunner_proxy_network]
    labels:
      xyz.easyrunner.service.type: web
      xyz.easyrunner.service.framework: nextjs
      xyz.easyrunner.service.port: "8000"
      xyz.easyrunner.service.domain: shop.example.com
  admin:
    image: ghcr.io/acme/admin:latest
    networks: [easyrunner_proxy_network]
    labels:
      xyz.easyrunner.service.type: web
      xyz.easyrunner.service.framework: standardbackend
      xyz.easyrunner.service.port: "7001"
      xyz.easyrunner.service.domain: admin.example.com
  backend:
    image: ghcr.io/acme/backend:latest
    networks: [easyrunner_proxy_network, medusa_internal]
    environment:
      - REDIS_URL=redis://redis:6379
    labels:
      xyz.easyrunner.service.type: web
      xyz.easyrunner.service.framework: standardbackend
      xyz.easyrunner.service.port: "9000"
      xyz.easyrunner.service.domain: api.example.com
  redis:
    image: docker.io/library/redis:7
    networks: [medusa_internal]
    labels:
      xyz.easyrunner.service.type: internal

networks:
  easyrunner_proxy_network:
    external: true
  medusa_internal:
    driver: bridge
```

The storefront can reach the backend's public API via
`EASYRUNNER_SERVICE_BACKEND_URL`, and the backend reaches redis internally by
the bare name `redis`.
