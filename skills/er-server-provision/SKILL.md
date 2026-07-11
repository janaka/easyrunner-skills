---
name: er-server-provision
description: Use when creating, spinning up, provisioning, or tearing down an easyrunner server host itself — the cloud VM — with er server create / init / delete on Hetzner, as opposed to deploying an app onto an already-running server
---

# Provision an easyrunner server (`er server create` → `init`)

## Overview

Two commands take you from nothing to a working host: `er server create` provisions a
cloud VM and registers it, and `er server init` installs the easyrunner stack (ops user,
Podman, Caddy, host firewall) onto it. **Hetzner is the only supported provider today.**

> `er server create` is labelled "legacy" in `--help`, but it is the **only** command that
> provisions a cloud VM — there is no replacement yet, so use it. (`er server add` registers
> an *existing* host you provisioned elsewhere; it does **not** create one.)

## Prerequisites (check first)

```bash
er link list        # Hetzner must be linked — look for `hetzner` with an account name
er server list      # existing servers, so you choose a fresh unique <name>
er license status   # Server Limit vs. count — create aborts if you're at the cap
```

- **Hetzner link is required.** `create` always uses the Hetzner project linked as
  `default`. If it's missing: `er link hetzner default --api-key <KEY>`.
- **License cap.** Reaching the server limit aborts `create` with a message; `delete` an old
  server or raise the license first.
- **No manual Pulumi install.** `create`/`delete` auto-provision the cloud tooling on first
  use (local Pulumi backend) — don't try to install it yourself.

## Provision + bring up

```bash
er server create <name> hetzner           # provisions the VM + firewall + SSH key; registers it
er server init <name> --username root      # installs the easyrunner stack (FIRST init: connect as root)
```

`<name>` is a friendly, unique label (e.g. `nettest`) — the handle for every later command.
`create` prints the server's **IP** and **Server ID** and stores the SSH key under
`~/.ssh/easyrunner_keys/`.

> **First `init` on a freshly-`create`d cloud VM must use `--username root`.** `init` defaults
> to the `easyrunner` ops user, but on a brand-new Hetzner box that user doesn't exist yet —
> only `root` carries the injected SSH key. `init --username root` connects as root, *creates*
> the `easyrunner` user, and installs the stack; every later command then uses `easyrunner`
> (the default). Without `--username root` the first init fails with `Authentication failed`.

Provisioning and init are **separate steps** — `create` alone gives you a bare VM with no
easyrunner stack. Run `init` after.

**What `create` provisions (all hardcoded — there are no size/region/image flags):**

| | |
|---|---|
| Image | `ubuntu-24.04` |
| Size | `cx23` (2 vCPU, 4 GB RAM, shared vCPU) |
| Location | `fsn1` (Falkenstein, DE) — fixed, regardless of the provider's `nbg1` default |
| Cloud firewall | Hetzner-level: **in** TCP 22/80/443; **out** all TCP, UDP 123 (NTP), ICMP |

To use a different size/region/image you must edit `hetzner_resource_factory.py`, or
hand-provision the VM and register it with `er server add <name> <ip>` instead.

> **If `create` fails with `server type <x> not found`, Hetzner has retired that type.**
> The hardcoded size is a moving target (Hetzner periodically retires types, e.g. `cx11`→…→
> `cx22`→`cx23`). Pick the current same-spec replacement from the live catalogue
> (`GET https://api.hetzner.cloud/v1/server_types`) and update `hetzner_resource_factory.py`.

## Verify

```bash
er server doctor <name>   # pass/fail checks (stack, egress policy, fail2ban)
er server status <name>   # runtime state: system stats, per-app run state, mesh identity
```

## End-to-end smoke test — deploy the hello-world app

Provisioning isn't proven until an app actually **builds, routes, and serves** through the
new host. Every fresh server should get one end-to-end deploy before you rely on it. Use the
canonical Next.js hello-world:

```bash
er app add hello <name> git@github.com:janaka/next-helloworld-app.git   # Flow A: SSH URL required (deploy keys)
er app deploy hello <name>    # clones, builds, starts behind Caddy, auto-provisions Cloudflare DNS + TLS
er app status hello <name>    # containers should come up
```

- **Flow A requires an SSH repo URL (`git@github.com:...`), not HTTPS** — deploys authenticate
  with a per-app deploy key over SSH. An HTTPS URL fails deploy with "Only SSH repository URLs
  are supported". (On a hardened host the clone runs over `ssh.github.com:443`, since the #228
  egress policy blocks outbound port 22 — see the firewall note below.)
- `next-helloworld-app` is already easyrunner-ready (Containerfile + `.easyrunner` compose
  with the `xyz.easyrunner.service.*` labels), so it needs **no repo prep** and no domain flag.
- **The public domain is the `xyz.easyrunner.service.domain` label in that repo's compose** —
  not a CLI flag. The repo routes at **`hellonextjs.<first zone on your linked Cloudflare>`**
  (host `hellonextjs` is fixed; the zone is per-account — the canonical repo ships
  `hellonextjs.easyrunner.xyz`). That zone must exist in your **linked Cloudflare** account so
  the A record + TLS provision automatically on first deploy (certs can take ~a minute); to use
  a different zone, edit that one label.
- Load the printed URL — a served hello-world page means the host is good end-to-end. The
  full add/deploy mechanics live in **`er-app-create`** and **`er-app-deploy`**; follow those
  for anything beyond this smoke test.

**Remove the smoke-test app before you tear the server down** (`delete` refuses while a
server still has apps):

```bash
er app remove hello <name>    # <app> <server> — both positional
```

## Testing network-level changes — mind the two firewall layers

A provisioned **and** initialised host has **two independent firewalls**: the Hetzner
**cloud firewall** (added by `create`) and a **host iptables policy** (added by `init`,
including the uid-scoped egress default-deny for hosted workloads). A packet can be dropped
at *either* layer — check both when a rule doesn't behave as expected. After a policy change
lands in easyrunner, backfill the canonical host rules onto an existing server idempotently:

```bash
er server reapply-firewall <name>   # only adds/removes deltas; never drops baseline access
```

> **The Hetzner cloud firewall is written once, at `create` time.** `reapply-firewall` and
> every other `er server` verb touch only the **host iptables** layer. To change the *cloud*
> firewall rules you must edit them in the Hetzner console (or Pulumi), or `delete` and
> re-`create` the server. If the cloud layer is what you're testing, plan for a recreate.

## Teardown — use `delete`, not `remove`

```bash
er server delete <name> hetzner   # DESTROYS the Hetzner VM + firewall, cleans SSH keys, unregisters
```

- **`delete` ≠ `remove`.** `er server delete <name> hetzner` destroys the **cloud VM**.
  `er server remove <name>` only unregisters and uninstalls the stack — it **leaves the paid
  VM running and billing**. On a cloud-provisioned box that leaks a billable server; for
  anything made with `create`, always `delete`.
- `delete` **refuses while the server still has apps** (intentional friction, since it's
  unrecoverable) — remove each app first with `er app remove <app> <name>`.
- **There is no other confirmation prompt.** Once the server is app-free,
  `er server delete <name> hetzner` destroys it immediately and irreversibly — double-check
  `<name>` before running it.
- Stuck mid-operation? `er server delete <name> hetzner --cancel` unlocks the stack;
  `--refresh` reconciles Pulumi state with reality (e.g. after a manual console delete).

## Common Mistakes

| Mistake | Correct |
|---|---|
| `er server create <name>` (no provider) | Provider is a required positional: `er server create <name> hetzner` |
| Expecting `create` to install the stack | It only provisions + registers; run `er server init <name>` after |
| `er server remove` to destroy a cloud VM | `remove` leaves the VM billing — use `er server delete <name> hetzner` |
| `er server delete <name>` (no provider) | Provider positional required: `... delete <name> hetzner` |
| `er server init <name> hetzner` | `init` takes only `<name>` (plus optional `--username`) |
| Passing `--size` / `--region` / `--image` to `create` | No such flags — values are hardcoded (cx23 / fsn1 / ubuntu-24.04) |
| `er server add` to spin up a new VM | `add` registers an *existing* host; `create` provisions a new one |
| `er server delete` while the server still has apps | Remove its apps first, then `delete` |
