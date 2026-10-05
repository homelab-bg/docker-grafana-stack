# docker-grafana-stack

Centralized logging: Loki (storage/query), Grafana (UI), Alloy (log shipping - both
Docker-container logs and a syslog receiver for appliances that can't run an agent).

LAN-only, plain HTTP throughout - no TLS/Caddy in front of this. Logging infra doesn't need
public exposure; hosts on the LAN talk to it directly.

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

## Adding a new agent-only host

1. Clone this repo onto the host.
2. `cp .env.example .env`, set `ALLOY_DATA_PATH` and `LOKI_PUSH_URL` (pointing at the real
   Loki host), leave `COMPOSE_PROFILES` unset.
3. `docker compose up -d` - only `alloy` starts.
