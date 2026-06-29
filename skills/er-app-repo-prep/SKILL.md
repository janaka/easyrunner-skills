---
name: er-app-repo-prep
description: Prepare a code repository to be deployable by EasyRunner. Use when the user wants to "make this repo work with easyrunner", add a Dockerfile + .easyrunner compose file, or get a project ready before `er app add`. Edits files in the user's repo; does not call the `er` CLI.
license: MIT
compatibility: Targets the EasyRunner compose conventions (xyz.easyrunner.service.* labels, easyrunner_proxy_network external network). See `compose-reference.md` for the label reference.
metadata:
  author: easyrunner
  version: "1.0"
---

# Prepare a repo for EasyRunner deployment

This skill edits files in the user's application repo so that `er app add` and
`er app deploy` will work. It does **not** call the `er` CLI itself.

## Required outputs

Three things must exist in the repo before deployment:

1. A working **`Dockerfile`** that builds the app into a single container image.
   The image must listen on the port declared in the compose label.
2. **`.easyrunner/docker-compose-app.yaml`** — a single-service compose file in
   a fixed shape (see below).
3. The image's listen port must match `xyz.easyrunner.service.port` exactly.

## Steps

1. **Detect the framework.** Inspect the repo for clues:
   - `next.config.{js,ts,mjs}` + `package.json` → Next.js
   - `pyproject.toml` with FastAPI/Flask → Python web
   - `Gemfile` → Ruby
   - Otherwise ask the user to confirm what they're shipping.

2. **Pick the framework label value.** Common values:
   - `nextjs`, `fastapi`, `flask`, `django`, `rails`, `standardbackend`
   If unsure, use `standardbackend` and tell the user it's a free-form label.

3. **Add or update the `Dockerfile`.** Prefer multi-stage builds; produce a
   small final image. The final stage must `EXPOSE` the same port as the
   compose label, and the process must bind `0.0.0.0` (not `127.0.0.1`).

   For Next.js apps, use `output: 'standalone'` in `next.config.js` and a
   standalone runner stage — see the [Next.js reference template](https://docs.easyrunner.xyz/user-guides/nextjs-reference-template/).

   EasyRunner injects build args automatically at deploy time:
   - `EASYRUNNER_APP_URL` / `EASYRUNNER_APP_DOMAIN` — this service's own public
     URL/domain (from its `xyz.easyrunner.service.domain` label).
   - `EASYRUNNER_SERVICE_<NAME>_URL` / `_DOMAIN` — the public URL/domain of each
     web service in the app, so one service can bake in another's URL.
   Consume them with `ARG` in the builder stage when the app needs to know
   a public URL at build time (e.g. `NEXT_PUBLIC_*` vars).

4. **Add `.easyrunner/docker-compose-app.yaml`** with this shape:

   ```yaml
   name: <app-name>

   services:
     app:
       image: <app-name>:latest
       networks:
         - easyrunner_proxy_network
       labels:
         xyz.easyrunner.service.type: web
         xyz.easyrunner.service.domain: <app-domain>
         xyz.easyrunner.service.framework: <framework>
         xyz.easyrunner.service.port: "<port>"
         xyz.easyrunner.service.build-context: "."

   networks:
     easyrunner_proxy_network:
       external: true
   ```

   - The network **must** be `easyrunner_proxy_network` and **must** be marked
     `external: true`. Caddy lives on that network and routes each web service
     to its `xyz.easyrunner.service.domain`.
   - `xyz.easyrunner.service.domain` is **required on every `web` service** and
     must be unique within the app. You can declare several `web` services, each
     on its own domain (see [compose-reference.md](compose-reference.md)).
   - `xyz.easyrunner.service.port` is a **string** (quoted), not a number.
   - Add `volumes:` entries only for caches/data the app genuinely needs to
     persist across redeploys (e.g. `next_cache:/app/.next/cache` for Next.js
     ISR). Do not invent volumes.

5. **Confirm the port matches.** The port in the Dockerfile (`EXPOSE`, the
   server `listen` call) must equal `xyz.easyrunner.service.port`. Mismatches
   are the #1 cause of "deploy succeeds but 502 from Caddy".

6. **Tell the user what's next.** After the files are in place:
   - `er app add <name> <server> <repo-url>` (flow_a, build from source) —
     hand off to the `er-app-create` skill.
   - Then `er app deploy <name> <server>` — hand off to the `er-app-deploy`
     skill.

## What this skill does NOT do

- Does not create the app on the server (`er app add`).
- Does not deploy (`er app deploy`).
- Does not configure DNS or secrets.
- Does not commit/push. Leave changes unstaged so the user reviews them.

See `compose-reference.md` for the full label list and value semantics.
