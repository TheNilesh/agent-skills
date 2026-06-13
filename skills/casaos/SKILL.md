---
name: casaos
description: Operate a live CasaOS installation through its HTTP APIs, casaos-cli when present, and CasaOS-compatible Docker Compose app files. Use when diagnosing CasaOS services, inspecting or managing installed CasaOS apps, querying health/logs/ports, interacting with message bus/local storage/user APIs, or drafting/importing/applying CasaOS AppStore-style docker-compose.yml files with x-casaos metadata.
---

# CasaOS Live Operator

Use this skill for managing an already-installed CasaOS system. The repository is `github.com/IceWhaleTech/CasaOS`, described as a open-source Personal Cloud system.

## Operating Rules

- Prefer local execution on the CasaOS host. Some CasaOS APIs and gateway routes are designed for localhost or trusted local access.
- Discover the API root before calling anything:
  - If `casaos-cli` exists, use `casaos-cli --help`; it defaults to `localhost:80` or reads `/etc/casaos/gateway.ini`.
  - Otherwise inspect `/etc/casaos/gateway.ini` for the gateway port, then use `http://localhost:<port>`.
- Use `--root-url <host:port>` with `casaos-cli` for non-default gateways.
- Prefer dry-run flags before applying compose changes.
- Treat app uninstall, app apply, message-bus trigger, storage merge, and direct Docker/systemd mutations as state-changing operations. Confirm intent before destructive or broad changes.

## Docker Is The Runtime

CasaOS manages each app as a standard Docker Compose stack. Every installed app is a real Docker Compose project; the containers are plain Docker containers. This means the full Docker CLI and `docker compose` subcommands work on any CasaOS app without modification.

Use Docker commands freely alongside `casaos-cli` — they operate on the same containers. Prefer casaos-cli:

```bash
# See all running CasaOS app containers
docker ps

# Stream logs for a container (container_name matches the compose service)
docker logs -f <container_name>
docker logs --tail 200 <container_name>

# Exec into a running container
docker exec -it <container_name> sh

# Inspect container config, mounts, env vars, network
docker inspect <container_name>

# Resource usage
docker stats

# Compose-level commands (project name = appid)
docker compose -p <appid> logs -f
docker compose -p <appid> ps
docker compose -p <appid> restart
docker compose -p <appid> down   # stops but does not uninstall from CasaOS

# Compose file is at the standard location
docker compose -f /var/lib/casaos/apps/<appid>/docker-compose.yml ps
```

`casaos-cli app-management logs <appid>` is a thin wrapper around `docker logs`. When it is unavailable or you need more control (timestamps, since, follow), use `docker logs` directly.

## What To Read

- For CLI/API commands, read `references/live-api-cli.md`.
- For drafting CasaOS compose files, read `references/compose-apps.md`.

## Quick Commands

```bash
command -v casaos-cli
casaos-cli --help
casaos-cli healthcheck services
casaos-cli healthcheck ports-in-use
casaos-cli app-management list apps
casaos-cli app-management show local <appid> --yaml
casaos-cli app-management logs <appid> --lines 200
casaos-cli app-management install --file docker-compose.yml --dry-run
```

Direct API fallback:

```bash
curl -sS http://localhost:80/v2/casaos/health/services
curl -sS http://localhost:80/v2/casaos/health/ports
curl -sS http://localhost:80/doc/v2/casaos/openapi.yaml
```

## Host File System Layout

Key paths on a CasaOS host:

| Path | Purpose |
|------|---------|
| `/var/lib/casaos/apps/<appid>/docker-compose.yml` | Active compose file for each installed app |
| `/var/lib/casaos/apps/<appid>/` | Per-app working directory managed by CasaOS |
| `/DATA/AppData/<AppName>/` | App config and state data (bind-mounted into containers) |
| `/DATA/Media/` | User media library (Music, Movies, etc.) |
| `/etc/casaos/` | CasaOS service configuration |
| `/etc/casaos/gateway.ini` | Gateway port and routing config |

When editing an installed app's compose file directly, always work on the copy at `/var/lib/casaos/apps/<appid>/docker-compose.yml` and follow with `casaos-cli app-management apply <appid> --file <path>` to sync CasaOS state. Do not write compose files elsewhere and expect CasaOS to pick them up.

## Compose Drafting

CasaOS apps are Docker Compose apps with extra `x-casaos` metadata. Use pinned image tags, `/DATA/AppData/$AppID/...` bind mounts, and top-level metadata that points CasaOS to the main service and web UI port.

Minimal shape:

```yaml
name: myapp
services:
  myapp:
    image: vendor/myapp:1.2.3
    container_name: myapp
    restart: unless-stopped
    ports:
      - target: 8080
        published: ${WEBUI_PORT:-8080}
        protocol: tcp
    volumes:
      - type: bind
        source: /DATA/AppData/$AppID/config
        target: /config
    environment:
      TZ: $TZ
      PUID: $PUID
      PGID: $PGID
    x-casaos:
      envs:
        - container: TZ
          description:
            en_US: Time zone
      ports:
        - container: "8080"
          description:
            en_US: Web UI port
      volumes:
        - container: /config
          description:
            en_US: Config directory

x-casaos:
  architectures:
    - amd64
    - arm64
  main: myapp
  author: CasaOS User
  category: Utilities
  description:
    en_US: My app description.
  tagline:
    en_US: Short app tagline.
  title:
    en_US: MyApp
  icon: ""
  index: /
  port_map: ${WEBUI_PORT:-8080}
  scheme: http
  store_app_id: myapp
  version: "1.2.3"
```

Validate before import:

```bash
docker compose -f docker-compose.yml config
casaos-cli app-management install --file docker-compose.yml --dry-run
```
