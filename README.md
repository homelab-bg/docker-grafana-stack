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
  from network gear, and pushes from every agent-only host. The same `COMPOSE_PROFILES` value
  also selects Alloy's syslog component (see "Pointing syslog-only devices at this" below) -
  one var, no extra step.
- **Agent-only host** (`docker-green`, `docker-mcp-agents`, etc.) - no profile flag, so only
  `alloy` starts, shipping that host's own Docker container logs to the full-stack host's Loki,
  with no syslog receiver active.

**TrueNAS-specific wrinkle, resolved via a third overlay**: confirmed live that TrueNAS's app
engine doesn't honor `COMPOSE_PROFILES` from `.env` the same way the plain CLI does when
deployed via its Custom App `include:` mechanism - only `alloy` started there, `loki`/`grafana`
didn't, despite the env var genuinely being set. Ordinary `${VAR}` substitution into YAML
content works fine through the same `include:` (confirmed - Alloy's syslog file selection,
below, relies on exactly this); profile *activation* specifically - a control-plane decision
about which services exist in the resolved model at all, not a text substitution - isn't read
the same way. `docker-compose.truenas.yml` works around it the same way
`docker-traefik-portainer`'s own TrueNAS overlay already does for `dnsweaver`: unconditionally
un-gates `loki`/`grafana` (`profiles: !reset []`) rather than relying on profile activation at
all - see "Deploying on TrueNAS" below.

Deliberately not split into a separate `docker-alloy` repo for the agent-only case - it's the
exact same image/config either way, and a second repo would just be a second place to forget
to bump the version pin.

## Layout

```
docker-compose.yml           # alloy (default) + loki/grafana (profile: full)
docker-compose.traefik.yml   # optional overlay - see below
docker-compose.truenas.yml   # TrueNAS-only overlay - see "Deploying on TrueNAS" below
config/
  alloy/base.alloy    # always loaded - Docker log shipping, every host
  alloy/full.alloy    # syslog receiver - only loaded when COMPOSE_PROFILES=full
  alloy/empty.alloy   # no-op placeholder - loaded instead on agent-only hosts
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

Needs `GRAFANA_HOST`, `NETWORK` (defaults to `traefik`, matching the shared external
network every other stack joins), and `GRAFANA_ROOT_URL` set in `.env` - see `.env.example`.
`GF_SERVER_ROOT_URL` lives in this overlay, not the base file - it's only ever meaningful once
Grafana is reachable at something other than `http://<host>:3000`, exactly the condition for
applying this overlay at all. Not needed at all for the default LAN-only deployment.
(`STACK_NAME` is unrelated to this overlay specifically - see "Self-labeling" below; it
applies regardless of whether Traefik is used.)

## Self-labeling

`loki`/`grafana`/`alloy` carry the same `x-common-labels` convention the `docker-truenas`
stacks use on their own containers - `logging_jobname`/`stackname` Docker labels, applied
unconditionally in the base `docker-compose.yml` (not gated behind the Traefik overlay).
Alloy's own `discovery.docker` picks these up from every container on the host regardless of
which compose project it belongs to, and promotes them to real `job`/`stack` Loki labels (see
`config/alloy/base.alloy`) - so this stack's own logs show up in Grafana queryable the same
way every other stack's do, e.g. `{stack="grafana-stack"}`.

`STACK_NAME` in `.env` controls the value (defaults to `grafana-stack` if unset - see
`.env.example`). Named `STACK_NAME` here rather than `MONITORING_STACK` (what the
`docker-truenas` stacks call the equivalent variable) - the old name reads as "the stack
that's doing the monitoring," when what it actually controls is "what to label *this* stack
as, inside the monitoring system." Same rename already done in `docker-traefik-portainer`
(confirmed live - `traefik`/`portainer`/`dnsweaver` all now carry `stackname: "traefik-portainer"`)
- **outstanding, tracked separately**: rename `MONITORING_STACK` to `STACK_NAME` in
`docker-home-assistant-stack`, `docker-vscode`, and `docker-hello-world` too, once this is
validated.

When the Traefik overlay is applied, its `traefik.*` labels merge on top of these - Compose
merges `labels:`/`logging:` as maps across `-f` files the same way it merges `networks:`, so
the overlay doesn't need to redeclare `logging_jobname`/`stackname` itself.

## Deploying on TrueNAS (Custom App)

TrueNAS SCALE's own App system runs this, not a plain `docker compose` CLI invocation - via
Apps > Discover Apps > Custom App > Install via YAML, using Compose's `include:` directive to
point at the real files instead of pasting YAML into the UI (same pattern already used for
`docker-traefik-portainer` on this TrueNAS).

**`docker-compose.truenas.yml` is required for a full-stack deployment here, not optional**
- see the "TrueNAS-specific wrinkle" note under "Two deployment shapes" above.
`COMPOSE_PROFILES=full` in `.env` doesn't activate the `full` profile through TrueNAS's
`include:` the way it does through the plain CLI, so without this overlay `loki`/`grafana`
never start. The overlay unconditionally un-gates both services instead
(`profiles: !reset []`), so profile activation is never relied on against TrueNAS at all.

Clone directly into the project folder - no separate `/stack` subfolder needed anymore.
That nesting existed only because the old structure's top-level `alloy/`/`loki/`/`grafana/`
config folders would otherwise collide with sibling data directories of the same name; now
that config lives entirely under `config/`, there's nothing at the repo's top level to
collide with (confirmed by inspection - `config/`, `docker-compose*.yml`, `.env` are the only
top-level entries). `.gitignore` covers `/loki/`, `/grafana/`, `/alloy/` for exactly this case.

If the dataset directories (`loki`/`grafana`/`alloy`) already exist before cloning - likely,
if you've pre-created them as separate ZFS datasets - `git clone` refuses outright
(`destination path '.' already exists and is not an empty directory`, regardless of whether
anything actually collides path-wise). Confirmed live: `git init` in place instead works
fine, since `git checkout` only objects to real path-level conflicts with tracked files, and
there isn't one here:

```sh
cd /mnt/nvme_pool1/Apps/grafana
git init
git remote add origin git@github.com:homelab-bg/docker-grafana-stack.git
git fetch origin main
git checkout main
cp .env.example .env
# edit .env: LOKI_DATA_PATH/GRAFANA_DATA_PATH/ALLOY_DATA_PATH point at sibling data dirs
# inside this same directory (./loki, ./grafana, ./alloy), a real GRAFANA_ADMIN_PASSWORD,
# LOKI_PUSH_URL=http://loki:3100/loki/api/v1/push, and COMPOSE_PROFILES=full
chown -R 10001:10001 /mnt/nvme_pool1/Apps/grafana/loki
chown -R 472:472 /mnt/nvme_pool1/Apps/grafana/grafana
```

**Also confirmed live: Loki failing with `open /etc/loki/config.yaml: permission denied`** even
though the file itself looks fine. Root cause - TrueNAS creates dataset directories owned
`root:root` (or whoever ran `git checkout`) with mode `0770`, i.e. "other" gets zero access,
not even traversal. `loki` runs as UID/GID `10001` (neither the owner nor in the `root`
group), so it can't even descend into the project directory to reach `config/loki/
loki-config.yaml`, regardless of that file's own permissions - same underlying class of issue
as the `technitium_token` permission note in `docker-traefik-portainer`'s README, just at the
directory-traversal level instead of a single file. Fix (needs `sudo` - these directories are
root-owned):
```sh
sudo chmod o+x /mnt/nvme_pool1/Apps/grafana
sudo chmod -R o+rX /mnt/nvme_pool1/Apps/grafana/config
```
Scoped deliberately - the first is non-recursive (only grants traversal through the project
root itself, touches nothing inside it), the second only reaches `config/`, a sibling of the
`loki`/`grafana`/`alloy` dataset directories, never a descendant of them. Neither command can
affect those datasets' own ownership/permissions, which stay exactly as set above. `alloy`
doesn't need this - its image runs as root, unaffected by directory permissions either way.

Also confirmed live: TrueNAS datasets are typically created root-owned, while `git
init`/`checkout` run as your own user - `git pull` then refuses with "detected dubious
ownership in repository" (a post-CVE-2022-24765 safety check, not specific to this repo).
Benign here (the mismatch is just how the dataset's mountpoint got created, not a real
multi-tenant threat) - fix it the way git itself suggests:

```sh
git config --global --add safe.directory /mnt/nvme_pool1/Apps/grafana
```

```yaml
include:
  - path:
      - /mnt/nvme_pool1/Apps/grafana/docker-compose.yml
      - /mnt/nvme_pool1/Apps/grafana/docker-compose.truenas.yml
```

If you also want the Traefik overlay, add it to the same `path:` list - use one `include:`
entry with a `path:` list, not a second top-level `include:` entry, confirmed live that two
separate entries defining the same service (here, `grafana`) raises `services.grafana
conflicts with imported resource`, since Compose treats separate `include:` entries as
independent sub-projects, not a base+override pair:

```yaml
include:
  - path:
      - /mnt/nvme_pool1/Apps/grafana/docker-compose.yml
      - /mnt/nvme_pool1/Apps/grafana/docker-compose.truenas.yml
      - /mnt/nvme_pool1/Apps/grafana/docker-compose.traefik.yml
```

`include:` resolves `${VAR}` substitutions against a `.env` sitting next to the included
file(s) (standard Compose Spec behavior, confirmed already working this way for
`docker-traefik-portainer`) - the `.env` created above is picked up automatically, nothing to
paste into a separate TrueNAS environment-variables form.

## Pointing syslog-only devices at this

For hardware that can't run Alloy (UniFi UDM, switches, Proxmox via `rsyslog` forwarding,
TrueNAS's own system logs) - point its remote-logging/syslog-server setting at the full-stack
host's IP, one of two UDP ports depending on which syslog format the device actually sends.
No agent, no config on this side beyond what's already enabled by `COMPOSE_PROFILES=full` on
that host.

"Syslog" isn't one wire format - `config/alloy/full.alloy` runs two listeners for this reason,
confirmed against a real device (BusyBox `syslogd` on an SMLIGHT SMHub) that only speaks the
legacy one and gets rejected outright by a listener defaulted to the other. RFC5424 (the
current IETF standard) sits on the default syslog port; the legacy format sits on the
non-default one, deliberately - not the other way round:

| Port        | Format  | Confirmed against                                              |
|-------------|---------|------------------------------------------------------------------|
| `514/udp`   | RFC5424 | Not yet confirmed against a real device                          |
| `1514/udp`  | RFC3164 | SMHub (BusyBox `syslogd` - no format option exists on that device) |

Don't assume which port a new device belongs on from its vendor or OS family - rsyslog-based
systems (Proxmox, TrueNAS SCALE) can typically be pointed at either depending on config, and
UniFi's own remote-syslog setting doesn't document a format choice at all. Point the device at
a port, send one test message, and check Alloy's own logs for a parse warning like `expecting
a version value in the range 1-999` (wrong port) before assuming it's working.

Alloy runs in directory-loading mode (`alloy run` against a directory, not a single file) so
the syslog receiver can be a genuinely separate component from the Docker-log-shipping logic
every host runs. `config/alloy/base.alloy` is always mounted; a second mount resolves to
`./config/alloy/${COMPOSE_PROFILES:-empty}.alloy` - `empty.alloy` (no-op) when unset,
`full.alloy` (the real `loki.source.syslog` component) when `COMPOSE_PROFILES=full`. This
deliberately reuses the same `COMPOSE_PROFILES` value already used to gate `loki`/`grafana`,
rather than adding a second, parallel toggle var - so the syslog receiver is only active on
the full-stack host, not every agent-only host by default.

**Known, accepted fragility**: this works by matching the file's name to `COMPOSE_PROFILES`'s
literal value. Compose allows multiple comma-separated profiles in one `COMPOSE_PROFILES`
(e.g. `full,debug`) - if that's ever done here, the substitution would try to mount a
nonexistent `full,debug.alloy` and the container would fail to start. Not a concern today,
since this repo only ever defines the one `full` profile - just worth knowing if a second
profile gets added later.

**TrueNAS note**: this toggle is plain `${VAR}` substitution, which - unlike `COMPOSE_PROFILES`
actually *activating* the `full` profile for `loki`/`grafana` (confirmed broken through
TrueNAS's `include:` mechanism, worked around via `docker-compose.truenas.yml` - see
"Deploying on TrueNAS" above) - works fine through `include:` either way. So on TrueNAS,
setting `COMPOSE_PROFILES=full` in `.env` correctly picks `full.alloy` for this mount on its
own, independent of whatever un-gates `loki`/`grafana` themselves.

## Verify

- Alloy's own UI (component graph, live debugging): `http://<host>:12345`
- Grafana (full-stack host only): `http://<host>:3000` - Loki is pre-provisioned as the
  default datasource, nothing to click through manually.
- Quick query from Grafana's Explore view once logs are flowing: `{container=~".+"}` should
  show every container Alloy has discovered on that host.
- Any container carrying `logging_jobname`/`stackname` Docker labels - this stack's own three
  (see "Self-labeling" above), and the `docker-truenas` stacks via their own `x-common-labels`
  anchor - gets those promoted to real `job`/`stack` Loki labels, e.g.
  `{stack="grafana-stack"}` or `{stack="home-assistant-stack"}`. Containers without those
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
