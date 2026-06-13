# CasaOS Compose App Reference

Use this reference when drafting or editing CasaOS-compatible Docker Compose files for live import through CasaOS App Management.

## Core Rules

- Create a valid Docker Compose file first.
- Add CasaOS metadata with `x-casaos` at both service level and top level.
- Use a top-level `name`; it becomes the store/app ID.
- Keep `name` lowercase and use only letters, numbers, `_`, and `-`; start with a letter or number.
- Do not use `latest` image tags. Pin a specific version.
- Prefer `restart: unless-stopped`.
- Use `/DATA/AppData/<AppName>/...` for app config/state (substitute the actual app name, e.g. `/DATA/AppData/jellyfin/config`). `$AppID` is a CasaOS template variable that resolves to the same value at runtime; both forms are equivalent in compose files.
- Use `$TZ`, `$PUID`, `$PGID` when the image supports them.
- Use `${WEBUI_PORT}` or `${WEBUI_PORT:-default}` for CasaOS-assigned web UI ports.
- Match top-level `x-casaos.main` to an actual service key.
- Match top-level `x-casaos.port_map` to the published web UI port.
- Optionally declare supported architectures in top-level `x-casaos.architectures`

## Service-Level Metadata

Describe user-facing envs, ports, and volumes that CasaOS should show/edit:

```yaml
services:
  app:
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
```

Keep this metadata synchronized with the actual `environment`, `ports`, and `volumes` sections.

## Top-Level Metadata

```yaml
x-casaos:
  architectures:
    - amd64
    - arm64
    - arm
  main: app
  author: CasaOS User
  category: Utilities
  description:
    en_US: Full description.
  developer: Upstream Project
  tagline:
    en_US: One-line tagline.
  title:
    en_US: App Name
  icon: ""
  screenshot_link: []
  thumbnail: ""
  tips: {}
  index: /
  port_map: ${WEBUI_PORT:-8080}
  scheme: http
  store_app_id: app
  version: "1.2.3"
```

Common categories include Media, Downloader, Utilities, Network, Backup, Productivity, and Developer. Follow existing CasaOS category names when targeting the official AppStore.

## Port Pattern

For a web UI listening inside the container on `8080`:

```yaml
ports:
  - target: 8080
    published: ${WEBUI_PORT:-8080}
    protocol: tcp
x-casaos:
  ports:
    - container: "8080"
      description:
        en_US: Web UI port
```

Top level:

```yaml
x-casaos:
  port_map: ${WEBUI_PORT:-8080}
  scheme: http
  index: /
```

## Volume Pattern

```yaml
volumes:
  - type: bind
    source: /DATA/AppData/$AppID/config
    target: /config
  - type: bind
    source: /DATA/Media/Music
    target: /music
```

Use user media paths such as `/DATA/Media/...` only when the app needs access to a user's library. Keep secrets and databases under `/DATA/AppData/$AppID/...`.

## Validation And Import

Validate YAML/Compose:

```bash
docker compose -f docker-compose.yml config
```

Dry-run CasaOS import on the target host:

```bash
casaos-cli app-management install --file docker-compose.yml --dry-run
```

Apply to an existing app:

```bash
casaos-cli app-management apply <appid> --file docker-compose.yml --dry-run
casaos-cli app-management apply <appid> --file docker-compose.yml
```

Retrieve an installed app before editing:

```bash
casaos-cli app-management show local <appid> --yaml > docker-compose.yml
```

## Review Checklist

- Image tags are pinned.
- No accidental host paths outside `/DATA/...` unless required and explained.
- `container_name` and service key are stable.
- `x-casaos.main`, `store_app_id`, and top-level `name` align.
- `port_map` opens the intended Web UI.
- Service-level metadata only lists real envs/ports/volumes.
- Sensitive values use placeholders or global env vars, not real secrets.
- `docker compose config` and CasaOS dry-run pass before live install/apply.
