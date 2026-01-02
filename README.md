Opinionated, production-ready Docker Compose stacks for a home / small lab environment: reverse proxy (Traefik), monitoring (Prometheus, Grafana, Loki, Promtail), observability exporters (node_exporter, cAdvisor), security services (WireGuard, AdGuard, Unbound), app stacks (Keycloak, n8n, Nexus, Portainer, Teleport), plus maintenance helpers (Watchtower, Dozzle).

This repository contains independent Compose stacks under folders at the repository root — start, stop and manage each stack independently.

Quick links

- Reverse proxy: [traefik/docker-compose.yml](traefik/docker-compose.yml) — config at [traefik/traefik.yml](traefik/traefik.yml)
- Monitoring: [monitoring/docker-compose.yml](monitoring/docker-compose.yml)
  - Prometheus config: [monitoring/prometheus/prometheus.yml](monitoring/prometheus/prometheus.yml)
  - Grafana datasources: [monitoring/grafana/provisioning/datasources/datasorce.yml](monitoring/grafana/provisioning/datasources/datasorce.yml)
  - Grafana dashboards: [4271_rev4.json](monitoring/grafana/provisioning/dashboards/4271_rev4.json), [9628_rev8.json](monitoring/grafana/provisioning/dashboards/9628_rev8.json), [14055_rev5.json](monitoring/grafana/provisioning/dashboards/14055_rev5.json)
  - Loki config: [monitoring/loki/loki-local-config.yml](monitoring/loki/loki-local-config.yml)
- Exporters / metrics: [node_expoter/docker-compose.yml](node_expoter/docker-compose.yml), [cadvisor/docker-compose.yml](cadvisor/docker-compose.yml)
- Network & security: [net-sec/docker-compose.yml](net-sec/docker-compose.yml)
- App stacks: [keycloak/docker-compose.yml](keycloak/docker-compose.yml), [n8n/docker-compose.yml](n8n/docker-compose.yml), [redis/docker-compose.yml](redis/docker-compose.yml), [portainer/docker-compose.yml](portainer/docker-compose.yml), [teleport/docker-compose.yml](teleport/docker-compose.yml) and [teleport/config/teleport.yml](teleport/config/teleport.yml)
- Maintenance: [watchtower/docker-compose.yml](watchtower/docker-compose.yml), [dozzle/docker-compose.yml](dozzle/docker-compose.yml)
- Other: [cloudflared/.env](cloudflared/.env) (example), [traefik/.gitignore](traefik/.gitignore)

Prerequisites

- Docker (recent stable) and Docker Compose v2 (docker compose CLI)
- External Docker networks referenced by stacks:
  - proxy-net (used by Traefik & apps) — see [traefik/docker-compose.yml](traefik/docker-compose.yml)
  - tunnel-net (Traefik/Cloudflared) and microservices-net used in monitoring — see [monitoring/docker-compose.yml](monitoring/docker-compose.yml)  
    Create external networks if not present:

```sh
docker network create proxy-net
docker network create tunnel-net
docker network create microservices-net
```

Environment

- Put secrets and local overrides into each stack's .env where applicable (examples: [cloudflared/.env](cloudflared/.env), [n8n/.env](n8n/.env)).
- Grafana admin configured via env keys in [monitoring/docker-compose.yml](monitoring/docker-compose.yml): `GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD`.

Quick start (recommended order)

1. Start Traefik (reverse proxy & ACME):

```sh
docker compose -f traefik/docker-compose.yml up -d
```

2. Start core monitoring (Prometheus, Loki, Grafana, Alloy):

```sh
docker compose -f monitoring/docker-compose.yml up -d
```

3. Start exporters & collectors:

```sh
docker compose -f node_expoter/docker-compose.yml up -d
docker compose -f cadvisor/docker-compose.yml up -d
```

4. Start security/network stacks:

```sh
docker compose -f net-sec/docker-compose.yml up -d
```

5. Start application stacks (example):

```sh
docker compose -f redis/docker-compose.yml up -d
docker compose -f keycloak/docker-compose.yml up -d
docker compose -f n8n/docker-compose.yml up -d
docker compose -f portainer/docker-compose.yml up -d
docker compose -f teleport/docker-compose.yml up -d
docker compose -f nexus/docker-compose.yml up -d
```

6. Start maintenance helpers:

```sh
docker compose -f watchtower/docker-compose.yml up -d
docker compose -f dozzle/docker-compose.yml up -d
```

Notes on healthchecks & readiness

- Services include healthchecks (Keycloak, Loki, Alloy, Prometheus, Redis etc.). See health blocks inside individual compose files (e.g., [keycloak/docker-compose.yml](keycloak/docker-compose.yml), [monitoring/docker-compose.yml](monitoring/docker-compose.yml), [redis/docker-compose.yml](redis/docker-compose.yml)).
- Use `docker compose -f <stack> ps` and `docker compose -f <stack> logs -f <service>` for troubleshooting.

Persistent data & volumes

- Compose files define named volumes for persistence (e.g., grafana-storage, prometheus-data, redis-data). See each stack's compose file (e.g., [monitoring/docker-compose.yml](monitoring/docker-compose.yml), [redis/docker-compose.yml](redis/docker-compose.yml)).

Security & TLS

- Traefik is configured to use ACME / DNS via Cloudflare. See [traefik/traefik.yml](traefik/traefik.yml) and provide `CF_DNS_API_TOKEN` via environment.
- Watchtower notifications are configured to use Discord/Slack hooks in [watchtower/docker-compose.yml](watchtower/docker-compose.yml) — replace webhook URL before production.
- Avoid committing secrets; use each stack's `.env` and ignore via .gitignore (e.g., [traefik/.gitignore](traefik/.gitignore), [n8n/.gitignore](n8n/.gitignore)).

Monitoring & dashboards

- Prometheus scrapes configured targets in [monitoring/prometheus/prometheus.yml](monitoring/prometheus/prometheus.yml).
- Grafana provisioning loads datasources from [monitoring/grafana/provisioning/datasources/datasorce.yml](monitoring/grafana/provisioning/datasources/datasorce.yml) and dashboards under [monitoring/grafana/provisioning/dashboards/](monitoring/grafana/provisioning/dashboards/). Example dashboards: [`4271_rev4.json`](monitoring/grafana/provisioning/dashboards/4271_rev4.json), [`9628_rev8.json`](monitoring/grafana/provisioning/dashboards/9628_rev8.json), [`14055_rev5.json`](monitoring/grafana/provisioning/dashboards/14055_rev5.json).

Best practices

- Run stacks in separated Compose files and bring up only what you need.
- Keep backups of persistent volumes (grafana, prometheus, loki, postgres, redis).
- Use non-root UIDs in containers where provided (see [redis/docker-compose.yml](redis/docker-compose.yml)).
- Limit container privileges (most stacks already set `no-new-privileges` or drop caps).

Troubleshooting checklist

- Service not healthy? Check logs:

```sh
docker compose -f <stack> logs -f <service>
docker inspect --format '{{.State.Health}}' <container>
```

- Traefik routing issues: confirm the service labels and that containers are on `proxy-net` (see [traefik/docker-compose.yml](traefik/docker-compose.yml) and app compose labels).
- Grafana missing datasources/dashboards: ensure Grafana container mounts the provisioning folder and the provisioning YAML (`monitoring/grafana/provisioning/datasources/datasorce.yml`) is valid.

Contributing

- Add or update standalone compose stacks under root directories.
- Keep secrets out of the repo; provide example .env or documentation instead.

License

- Repository content is unlicensed in-tree — add a LICENSE file to set usage terms.
