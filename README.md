# docker-grafana-stack

Centralized logging: Loki (storage/query), Grafana (UI), Alloy (log shipping - both
Docker-container logs and a syslog receiver for appliances that can't run an agent).

LAN-only, plain HTTP by default - no TLS in front of this. Logging infra doesn't need public
exposure; hosts on the LAN talk to it directly. An optional Traefik overlay exists if you
specifically want Grafana's UI reachable via a real HTTPS hostname (see "Optional Traefik
overlay" below) - `loki`/`alloy` stay LAN-only regardless of whether it's applied.

## Two deployment shapes, one repo

Same `alloy` service runs everywhere - full-stack host and agent-only hosts alike. Only
`LOKI_PUSH_URL` in `.env` differs between them. `loki`/`grafana` live in a separate file,
`docker-compose.full.yml` - layering it in (`-f`/`include:`) is what brings them up, not a
Compose profile. (Originally used `profiles: ["full"]` instead of a separate file - switched
after confirming live that TrueNAS's app engine doesn't honor `COMPOSE_PROFILES` from `.env`
the same way the plain CLI does. Ordinary `${VAR}` substitution into YAML content worked fine
through TrueNAS's `include:`; profile activation - a control-plane decision about which
services exist in the resolved model at all, not a text substitution - apparently isn't read
the same way. Multi-file layering doesn't have this problem: "which services run" is purely
"which files you list", nothing depends on an env var being honored correctly by whatever's
doing the resolving.)

- **Full-stack host** (currently: TrueNAS) - runs all three. Receives everything: its own
  Docker container logs, syslog from network gear, and pushes from every agent-only host.
- **Agent-only host** (`docker-green`, `docker-mcp-agents`, etc.) - runs only `alloy`,
  shipping that host's own Docker container logs to the full-stack host's Loki.

Deliberately not split into a separate `docker-alloy` repo for the agent-only case - it's the
exact same image/config either way, and a second repo would just be a second place to forget
to bump the version pin.

## Setup

```sh
cp .env.example .env
```

Fill in:
- `LOKI_DATA_PATH` / `GRAFANA_DATA_PATH` / `ALLOY_DATA_PATH` - real dataset/directory paths.
  Only `ALLOY_DATA_PATH` matters on an agent-only host (the other two services never start
  there, so their paths are unused).
- `LOKI_PUSH_URL` - `http://loki:3100/loki/api/v1/push` on the full-stack host (same compose
  network, resolves by service name); the real LAN hostname/IP on an agent-only host, e.g.
  `http://truenas.lan.example.internal:3100/loki/api/v1/push`.
- `GRAFANA_ADMIN_PASSWORD` - full-stack host only (unused on an agent-only host, since
  `docker-compose.full.yml` is never applied there).

**Before first start on the full-stack host**, match the data directories' ownership to what
the official images expect - this is the single most common first-boot failure with these
images (silent permission-denied, container restart-loops):

```sh
chown -R 10001:10001 "$LOKI_DATA_PATH"
chown -R 472:472 "$GRAFANA_DATA_PATH"
```

(`ALLOY_DATA_PATH` doesn't need this - Alloy's image runs as root by default.)

```sh
docker compose up -d                                        # agent-only host: starts just alloy
docker compose -f docker-compose.yml -f docker-compose.full.yml up -d   # full-stack host
```

## Optional Traefik overlay

Same pattern as every other `docker-truenas` stack: `docker-compose.traefik.yml` is an
overlay, not a replacement - applying it swaps Grafana from a direct `:3000` LAN port to
routing through the shared Traefik instance with a real HTTPS hostname. `loki`/`alloy` are
untouched either way; nothing in this design expects Loki's push/query API to be reachable
over the public internet.

```sh
docker compose -f docker-compose.yml -f docker-compose.full.yml -f docker-compose.traefik.yml up -d
```

Needs `GRAFANA_HOST`, `NETWORK` (defaults to `traefik`, matching the shared external network
every other stack joins), and `MONITORING_STACK` set in `.env` - see `.env.example`. Not
needed at all for the default LAN-only deployment.

## Deploying the full stack on TrueNAS (Custom App)

TrueNAS SCALE's own App system runs this, not a plain `docker compose` CLI invocation - via
Apps > Discover Apps > Custom App > Install via YAML, using Compose's `include:` directive to
point at the real file instead of pasting YAML into the UI (same pattern already used for
`docker-traefik-portainer` on this TrueNAS).

Clone into a **sibling** subfolder of the data directories, not the same path - this repo's
own working tree has top-level folders literally named `alloy/`, `loki/`, `grafana/`, which
would otherwise collide with data directories of the same name (Alloy's runtime state would
end up written directly into this git working tree):

```sh
git clone git@github.com:homelab-bg/docker-grafana-stack.git /mnt/nvme_pool1/Apps/grafana/stack
cd /mnt/nvme_pool1/Apps/grafana/stack
cp .env.example .env
# edit .env: LOKI_DATA_PATH/GRAFANA_DATA_PATH/ALLOY_DATA_PATH point at the sibling data dirs
# (/mnt/nvme_pool1/Apps/grafana/{loki,grafana,alloy}), a real GRAFANA_ADMIN_PASSWORD, and
# LOKI_PUSH_URL=http://loki:3100/loki/api/v1/push
chown -R 10001:10001 /mnt/nvme_pool1/Apps/grafana/loki
chown -R 472:472 /mnt/nvme_pool1/Apps/grafana/grafana
```

Then in TrueNAS's Custom App YAML editor - **always use `path:` with a list, even for just
two files**. A plain list of top-level `include:` entries (`- file1`, `- file2`) is treated by
Compose as independent sub-projects, not a base+override pair - confirmed live that this
raises `services.grafana conflicts with imported resource` the moment two of them define the
same service:

```yaml
include:
  - path:
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.yml
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.full.yml
```

Add `docker-compose.traefik.yml` as a third entry in that same `path:` list if you also want
the Traefik overlay:

```yaml
include:
  - path:
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.yml
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.full.yml
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.traefik.yml
```

`include:` resolves `${VAR}` substitutions against a `.env` sitting next to the included
file(s) (standard Compose Spec behavior, confirmed already working this way for
`docker-traefik-portainer`) - the `.env` created above is picked up automatically, nothing to
paste into a separate TrueNAS environment-variables form.

**Why `docker-compose.full.yml` is a separate file rather than a Compose profile**: originally
`loki`/`grafana` used `profiles: ["full"]` instead, activated via `COMPOSE_PROFILES=full` in
`.env`. Confirmed live that TrueNAS's app engine doesn't honor that the same way the plain CLI
does - only `alloy` came up, `loki`/`grafana` didn't, despite the env var genuinely being
present in `.env`. Ordinary `${VAR}` substitution into YAML content (`LOKI_DATA_PATH` etc.)
worked fine through the same `include:`, so this looks like profile activation specifically -
a control-plane decision about which services exist in the resolved model, not a text
substitution - isn't read from `.env` the same way by whatever TrueNAS uses internally to
resolve `include:`. Switched to a separate file instead: multi-file layering is already
proven to work on this exact TrueNAS (same mechanism the Traefik overlay needs anyway), and
"which services run" becomes purely "which files you list" - nothing depends on an env var
being honored correctly by an engine whose internals aren't fully visible from outside it.

## Pointing syslog-only devices at this

For hardware that can't run Alloy (UniFi UDM, switches, Proxmox via `rsyslog` forwarding,
TrueNAS's own system logs) - point its remote-logging/syslog-server setting at the full-stack
host's IP, UDP port `514`. No agent, no config on this side beyond what's already in
`alloy/config.alloy`.

## Verify

- Alloy's own UI (component graph, live debugging): `http://<host>:12345`
- Grafana (full-stack host only): `http://<host>:3000` - Loki is pre-provisioned as the
  default datasource, nothing to click through manually.
- Quick query from Grafana's Explore view once logs are flowing: `{container=~".+"}` should
  show every container Alloy has discovered on that host.
- Containers carrying `logging_jobname`/`stackname` Docker labels (the `docker-truenas`
  stacks already do, via their own `x-common-labels` anchor) get those promoted to real `job`/
  `stack` Loki labels too - e.g. `{stack="home-assistant-stack"}`. Containers without those
  labels just don't get them; nothing breaks either way.

## Adding a new agent-only host

1. Clone this repo onto the host.
2. `cp .env.example .env`, set `ALLOY_DATA_PATH` and `LOKI_PUSH_URL` (pointing at the real
   Loki host).
3. `docker compose up -d` - only `alloy` starts, since `docker-compose.full.yml` is never
   applied here.
