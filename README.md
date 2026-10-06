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
`profiles: ["full"]`; `alloy` carries no profile, so it's the zero-flag default. Profiles are
additive-only (an untagged service always runs; a profile only ever adds services on top of
that, never removes one) - so the achievable split is "alloy by default, loki+grafana only
when `full` is explicitly activated," not the reverse.

- **Full-stack host** (currently: TrueNAS) - `--profile full` (or `COMPOSE_PROFILES=full` in
  `.env`) brings up all three. Receives everything: its own Docker container logs, syslog
  from network gear, and pushes from every agent-only host.
- **Agent-only host** (`docker-green`, `docker-mcp-agents`, etc.) - no profile flag, so only
  `alloy` starts, shipping that host's own Docker container logs to the full-stack host's Loki.

**Known open issue, not yet resolved**: confirmed live that TrueNAS's app engine doesn't
honor `COMPOSE_PROFILES` from `.env` the same way the plain CLI does when deployed via its
Custom App `include:` mechanism - only `alloy` started there, `loki`/`grafana` didn't, despite
the env var genuinely being set. Ordinary `${VAR}` substitution into YAML content worked fine
through the same `include:`; profile activation specifically - a control-plane decision about
which services exist in the resolved model at all, not a text substitution - apparently isn't
read the same way. This repo went through a separate-file-per-layer structure for a while
specifically to work around that, then back to this single-file+profiles structure (simpler,
correct for every plain-CLI deployment) once higher priority became validating the design on
`docker-green` first. **The TrueNAS-specific profile-activation problem is still real and
still unresolved** - don't assume this works against TrueNAS again without re-testing it.

Deliberately not split into a separate `docker-alloy` repo for the agent-only case - it's the
exact same image/config either way, and a second repo would just be a second place to forget
to bump the version pin.

## Layout

```
docker-compose.yml           # alloy (default) + loki/grafana (profile: full)
docker-compose.traefik.yml   # optional overlay - see below
config/
  alloy/config.alloy
  loki/loki-config.yaml
  grafana/provisioning/datasources/loki.yaml
```

Data (`loki`/`grafana`/`alloy`) lives in Docker-managed named volumes by default - no extra
folder in this repo for it. Override `*_DATA_PATH` in `.env` to use a real bind-mount path
instead (see "Setup" below).

## Setup

```sh
cp .env.example .env
```

Everything in `.env.example` is optional for a quick local test - `LOKI_DATA_PATH`/
`GRAFANA_DATA_PATH`/`ALLOY_DATA_PATH` all fall back to a Docker-managed named volume
(`loki_data`/`grafana_data`/`alloy_data`) if left commented out, so `cp .env.example .env`
with zero edits works on something like `docker-green` - no real dataset needed, and no
chown step either (see below). Fill in for an actual deployment:
- `LOKI_DATA_PATH` / `GRAFANA_DATA_PATH` / `ALLOY_DATA_PATH` - set to a real dataset/directory
  path to use a bind mount instead of the named-volume default (e.g. on TrueNAS, for ZFS
  snapshots/reliability). Only `ALLOY_DATA_PATH` matters on an agent-only host (the other two
  services never start there without the `full` profile active, so their paths are unused).
- `LOKI_PUSH_URL` - `http://loki:3100/loki/api/v1/push` on the full-stack host (same compose
  network, resolves by service name); the real LAN hostname/IP on an agent-only host, e.g.
  `http://truenas.lan.example.internal:3100/loki/api/v1/push`.
- `GRAFANA_ADMIN_PASSWORD` - full-stack host only (unused on an agent-only host, since
  `loki`/`grafana` never start there).

**Ownership only needs attention if you've overridden `*_DATA_PATH` to a real bind-mount
path** - the single most common first-boot failure with these images is a mismatch between
the host directory's ownership and what the image expects (silent permission-denied,
container restart-loops). Not a concern with the named-volume default: Docker auto-
initializes a fresh volume's ownership from the image's own baked-in directory, matching the
container's user automatically. If you have overridden the path:

```sh
chown -R 10001:10001 "$LOKI_DATA_PATH"
chown -R 472:472 "$GRAFANA_DATA_PATH"
```

(`ALLOY_DATA_PATH` doesn't need this - Alloy's image runs as root by default.)

```sh
docker compose up -d                   # agent-only host: starts just alloy
docker compose --profile full up -d    # full-stack host: starts loki + grafana + alloy
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

## Deploying on TrueNAS (Custom App) - currently blocked, see above

TrueNAS SCALE's own App system runs this, not a plain `docker compose` CLI invocation - via
Apps > Discover Apps > Custom App > Install via YAML, using Compose's `include:` directive to
point at the real file instead of pasting YAML into the UI (same pattern already used for
`docker-traefik-portainer` on this TrueNAS).

**This doesn't fully work yet** - see the "Known open issue" note under "Two deployment
shapes" above. `COMPOSE_PROFILES=full` in `.env` isn't honored through TrueNAS's `include:`
the way it is through the plain CLI, so `loki`/`grafana` won't come up this way until that's
actually solved (deferred for now, while validating the rest of the design on `docker-green`).
What follows is accurate for getting `alloy`-only running there today; treat the `full`
profile part as unverified against TrueNAS specifically.

Now that config lives under `config/` rather than top-level `alloy/`/`loki/`/`grafana/`
folders, cloning directly into `/mnt/nvme_pool1/Apps/grafana/` (instead of the sibling
`/stack` subfolder the old structure needed, to avoid colliding with the data directories of
the same name) is probably safe again - worth re-confirming when this is actually revisited,
not assumed:

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

```yaml
include:
  - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.yml
```

If you also want the Traefik overlay, use `path:` with a list rather than a second top-level
`include:` entry - confirmed live that two separate entries defining the same service (here,
`grafana`) raises `services.grafana conflicts with imported resource`, since Compose treats
separate `include:` entries as independent sub-projects, not a base+override pair:

```yaml
include:
  - path:
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.yml
      - /mnt/nvme_pool1/Apps/grafana/stack/docker-compose.traefik.yml
```

`include:` resolves `${VAR}` substitutions against a `.env` sitting next to the included
file(s) (standard Compose Spec behavior, confirmed already working this way for
`docker-traefik-portainer`) - the `.env` created above is picked up automatically, nothing to
paste into a separate TrueNAS environment-variables form.

## Pointing syslog-only devices at this

For hardware that can't run Alloy (UniFi UDM, switches, Proxmox via `rsyslog` forwarding,
TrueNAS's own system logs) - point its remote-logging/syslog-server setting at the full-stack
host's IP, UDP port `514`. No agent, no config on this side beyond what's already in
`config/alloy/config.alloy`.

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
- If using the named-volume default, `docker volume inspect loki_data` (or `grafana_data`/
  `alloy_data`) shows the real host path Docker is actually storing data at - useful since
  there's no local folder in the repo to just go look at.

## Adding a new agent-only host

1. Clone this repo onto the host.
2. `cp .env.example .env`, set `ALLOY_DATA_PATH` (or leave it for the `alloy_data` named-volume
   default) and `LOKI_PUSH_URL` (pointing at the real Loki host).
3. `docker compose up -d` - only `alloy` starts, since the `full` profile is never activated
   here.
