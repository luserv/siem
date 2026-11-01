# siem-lite

A lightweight, self-hosted security monitoring stack built from small single-purpose tools instead of a Java-based indexer.

| Role | Tool | Purpose |
|---|---|---|
| Detection | [Falco](https://falco.org) | Runtime syscall detection (modern eBPF) |
| Collection | [Grafana Alloy](https://grafana.com/docs/alloy/) | Ships Falco alerts and the systemd journal to Loki |
| Storage / search | [Loki](https://grafana.com/oss/loki/) | Log storage, 7 day retention |
| Dashboards | [Grafana](https://grafana.com/oss/grafana/) | Provisioned `Security Overview` dashboard |

```
Falco ──> /var/log/falco/events.json ─┐
                                       ├─> Alloy ──> Loki ──> Grafana
systemd journal ───────────────────────┘
```

## Requirements

- Docker with Compose v2
- Linux kernel with BTF (`/sys/kernel/btf/vmlinux` must exist) for Falco's modern eBPF driver
- A persistent systemd journal (`/var/log/journal`)

## Quick start

```sh
echo "GRAFANA_ADMIN_PASSWORD=$(openssl rand -hex 12)" > .env
docker compose up -d
docker compose ps
```

Open Grafana at <http://127.0.0.1:3001> and sign in as `admin` with the password from `.env`.
The `Security Overview` dashboard is under **Dashboards**.

Grafana is bound to `127.0.0.1` only. Loki and Alloy are not published outside the Docker network.

## Verify it works

Trigger a Falco rule by reading a sensitive file from a container:

```sh
docker run --rm alpine cat /etc/shadow
```

After about 10 seconds the alert shows up in **Explore** with the query `{job="falco"}`, and in the dashboard.
System journal entries are available with `{job="journal"}`.

## Layout

| Path | Description |
|---|---|
| `docker-compose.yml` | The four services and their volumes |
| `loki-config.yml` | Single-node Loki, filesystem storage, 7 day retention |
| `alloy-config.alloy` | Log sources (Falco file, journal) and the Loki push endpoint |
| `grafana/provisioning/` | Loki datasource and dashboard provider |
| `grafana/dashboards/` | Dashboard JSON, reloaded every 30 seconds |

Dashboards are provisioned read-only (`allowUiUpdates: false`). Edit the JSON file instead of the UI.

## Notes

- **Falco version.** Falco 0.40.0 crashed with `scap_init` errors on kernel 7.2, so the image is pinned to `0.45.2`. The `openat` / `openat2` TOCTOU warnings on startup are non-fatal.
- **Privileges.** The Falco container runs `privileged` and mounts `/proc`, `/etc` and the Docker socket. This is required for syscall capture.
- **No authentication on Loki.** `auth_enabled: false` means anyone who can reach port 3100 can read or inject logs. Never expose it to the internet.

## Monitoring more machines

Install Alloy (and optionally Falco) on each node and point `loki.write` at this host's Loki endpoint, adding a distinct `host` label per machine.
Loki must first be published on an interface reachable by the nodes, and only over a VPN (Tailscale or WireGuard) or behind a TLS reverse proxy with authentication.

## Operations

```sh
docker compose logs -f falco      # follow Falco
docker compose down               # stop, keep data
docker compose down -v            # stop and delete all data
```
