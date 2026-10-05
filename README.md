# docker-grafana-stack

Centralized logging: Loki (storage/query), Grafana (UI), Alloy (log shipping - both
Docker-container logs and a syslog receiver for appliances that can't run an agent).

LAN-only, plain HTTP by default - no TLS in front of this. Logging infra doesn't need public
exposure; hosts on the LAN talk to it directly. An optional Traefik overlay exists if you
specifically want Grafana's UI reachable via a real HTTPS hostname (see "Optional Traefik
overlay" below) - `loki`/`alloy` stay LAN-only regardless of whether it's applied.

## Two deployment shapes, one repo

Same `alloy` service runs everywhere - full-stack host and agent-only hosts alike. Only
`LOKI_PUSH_URL` in `.env` differs between them. `loki`/`grafana` are gated behind Compose's
`profiles: ["full"]`, so they simply don't start unless that profile is active.

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
- `COMPOSE_PROFILES=full` - **only** on the full-stack host. Leave unset everywhere else.
- `GRAFANA_ADMIN_PASSWORD` - full-stack host only.

**Before first start on the full-stack host**, match the data directories' ownership to what
the official images expect - this is the single most common first-boot failure with these
images (silent permission-denied, container restart-loops):

```sh
chown -R 10001:10001 "$LOKI_DATA_PATH"
chown -R 472:472 "$GRAFANA_DATA_PATH"
```

(`ALLOY_DATA_PATH` doesn't need this - Alloy's image runs as root by default.)

```sh
docker compose up -d        # agent-only host: starts just alloy
docker compose --profile full up -d   # full-stack host: starts loki + grafana + alloy
```

## Optional Traefik overlay

Same pattern as every other `docker-truenas` stack: `docker-compose.traefik.yml` is an
overlay, not a replacement - applying it swaps Grafana from a direct `:3000` LAN port to
routing through the shared Traefik instance with a real HTTPS hostname. `loki`/`alloy` are
untouched either way; nothing in this design expects Loki's push/query API to be reachable
over the public internet.

```sh
docker compose -f docker-compose.yml -f docker-compose.traefik.yml --profile full up -d
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
# (/mnt/nvme_pool1/Apps/grafana/{loki,grafana,alloy}), COMPOSE_PROFILES=full, a real
# GRAFANA_ADMIN_PASSWORD, and LOKI_PUSH_URL=http://loki:3100/loki/api/v1/push
chown -R 10001:10001 /mnt/nvme_pool1/Apps/grafana/loki
chown -R 472:472 /mnt/nvme_pool1/Apps/grafana/grafana
```

Then in TrueNAS's Custom App YAML editor:

```yaml
include:
  - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.yml
  # add this second line too if you also want the Traefik overlay applied:
  # - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.traefik.yml
```

`include:` resolves `${VAR}` substitutions against a `.env` sitting next to the included file
(standard Compose Spec behavior, confirmed already working this way for
`docker-traefik-portainer`) - the `.env` created above is picked up automatically, nothing to
paste into a separate TrueNAS environment-variables form.

**Unverified - check on first deploy**: whether TrueNAS's app engine honors
`COMPOSE_PROFILES` from that `.env` the same way the plain CLI does. If `loki`/`grafana`
don't start alongside `alloy`, that's the first thing to check - haven't been able to confirm
this detail of TrueNAS's internal app engine from outside it.

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
   Loki host), leave `COMPOSE_PROFILES` unset.
3. `docker compose up -d` - only `alloy` starts.
