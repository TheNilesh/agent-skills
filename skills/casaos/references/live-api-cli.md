# Live API And CLI Reference

Use this reference for a live CasaOS host. The CLI repository is `IceWhaleTech/CasaOS-CLI`, described as a command-line tool for testing and diagnosing CasaOS.

## Docker CLI — Always Available

CasaOS runs every app as a standard Docker Compose stack. All Docker and `docker compose` commands work natively alongside `casaos-cli`:

```bash
# Running containers (all CasaOS apps appear here)
docker ps
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Logs (container_name = service name in the compose file)
docker logs <container_name>
docker logs -f --tail 100 <container_name>
docker logs --since 1h <container_name>
docker logs --timestamps <container_name>

# Exec into a container for live debugging
docker exec -it <container_name> sh
docker exec -it <container_name> bash

# Inspect mounts, env vars, network, restart policy
docker inspect <container_name>
docker inspect <container_name> | jq '.[0].Mounts'
docker inspect <container_name> | jq '.[0].Config.Env'

# Resource usage across all containers
docker stats
docker stats <container_name> --no-stream

# Compose-level operations (project name = appid)
docker compose -p <appid> ps
docker compose -p <appid> logs -f
docker compose -p <appid> restart
docker compose -p <appid> down          # stops; does NOT uninstall from CasaOS
docker compose -p <appid> pull          # pull latest images (then restart)

# Equivalent using the stored compose file path
docker compose -f /var/lib/casaos/apps/<appid>/docker-compose.yml ps
```

`casaos-cli app-management logs <appid>` wraps `docker logs`. Prefer `docker logs` directly when you need timestamps, `--since`, or `-f` follow mode.

## Root URL

`casaos-cli` has a global root URL flag:

```bash
casaos-cli --root-url localhost:80 <command>
casaos-cli -u localhost:80 <command>
```

If no flag is provided, the CLI reads `/etc/casaos/gateway.ini` and uses `localhost:<gateway port>` when available. Otherwise it defaults to `localhost:80`.

Base API paths used by the CLI:

- CasaOS health/API: `/v2/casaos`
- App management: `/v2/app_management`
- Message bus: `/v2/message_bus`
- Local storage: `/v2/local_storage`
- Users: `/v2/users`

Prefer CLI commands over hand-written curl when the CLI exists because it formats responses and uses the expected service paths.

## Healthcheck

```bash
casaos-cli healthcheck services
casaos-cli healthcheck ports-in-use
casaos-cli healthcheck logs --dir /tmp
```

Direct API fallback:

```bash
curl -sS http://localhost:80/v2/casaos/health/services
curl -sS http://localhost:80/v2/casaos/health/ports
curl -fL http://localhost:80/v2/casaos/health/logs -o casaos-logs.zip
```

Use healthcheck first when diagnosing "CasaOS is down", "app store is not responding", "ports conflict", or service status questions.

## App Management

List and inspect:

```bash
casaos-cli app-management list apps
casaos-cli app-management list app-stores
casaos-cli app-management search
casaos-cli app-management search --category Media
casaos-cli app-management search --type official
casaos-cli app-management search --recommend
casaos-cli app-management show local <appid>
casaos-cli app-management show local <appid> --yaml
casaos-cli app-management show global
casaos-cli app-management logs <appid> --lines 200
```

Install or modify compose apps:

```bash
casaos-cli app-management install --file docker-compose.yml --dry-run
casaos-cli app-management install --file docker-compose.yml
casaos-cli app-management apply <appid> --file docker-compose.yml --dry-run
casaos-cli app-management apply <appid> --file docker-compose.yml
```

Lifecycle:

```bash
casaos-cli app-management start <appid>
casaos-cli app-management stop <appid>
casaos-cli app-management restart <appid>
casaos-cli app-management update app <appid>
casaos-cli app-management update app <appid> --force
casaos-cli app-management uninstall <appid>
casaos-cli app-management uninstall <appid> --no-remove-config
```

App stores and global variables:

```bash
casaos-cli app-management register app-store <url>
casaos-cli app-management unregister app-store <id>
casaos-cli app-management set global OPENAI_API_KEY sk-...
casaos-cli app-management show global
```

Notes:

- `install` accepts aliases `add`, `create`, and `up`.
- `uninstall` accepts aliases `remove`, `delete`, and `down`.
- `install` and `apply` accept YAML compose files and support `--dry-run`.
- `show local <appid> --yaml` is the quickest way to retrieve the active compose definition before editing.
- `set global` stores environment variables that are injected into compose apps unless an app defines the same variable itself.

## Message Bus

Inspect available types:

```bash
casaos-cli message-bus list event-types
casaos-cli message-bus list action-types
```

Subscribe:

```bash
casaos-cli message-bus subscribe socketio
casaos-cli message-bus subscribe websocket events --source-id <source> --event-names name1,name2
casaos-cli message-bus subscribe websocket actions --source-id <source> --action-names name1,name2
```

Trigger an action:

```bash
casaos-cli message-bus trigger --source-id <source> --action-name <name> --properties key=value,other=value
```

Treat trigger commands as state-changing. First list action types or inspect the service that owns the action.

Direct websocket shapes used by the CLI:

```text
ws://<root>/v2/message_bus/event/<source-id>?names=<comma-separated-events>
ws://<root>/v2/message_bus/action/<source-id>?names=<comma-separated-actions>
```

Socket.IO endpoint:

```text
http://<root>/v2/message_bus/socket.io
```

## Local Storage

```bash
casaos-cli local-storage list merges
casaos-cli local-storage set merge \
  --fstype fuse.mergerfs \
  --mount-point /DATA \
  --source-base-path /mnt \
  --source-volume-uuids uuid1,uuid2
```

Storage merge changes affect disks and mount behavior. Confirm the user's target volumes and desired outcome before running `set merge`.

## User Events And QR

```bash
casaos-cli user list events
casaos-cli qrcode
casaos-cli version
```

`qrcode` renders a QR code for the CasaOS Web UI root URL.

## Working With Installed App Files Directly

CasaOS stores each installed app's compose file at:

```
/var/lib/casaos/apps/<appid>/docker-compose.yml
```

Common file operations:

```bash
# List all installed apps by directory
ls /var/lib/casaos/apps/

# Read an app's live compose file
cat /var/lib/casaos/apps/<appid>/docker-compose.yml

# Inspect app data directory
ls /DATA/AppData/<AppName>/

# Preferred: retrieve compose via CLI (includes resolved CasaOS state)
casaos-cli app-management show local <appid> --yaml > docker-compose.yml

# Edit then apply back
casaos-cli app-management apply <appid> --file docker-compose.yml --dry-run
casaos-cli app-management apply <appid> --file docker-compose.yml
```

Do not edit `/var/lib/casaos/apps/<appid>/docker-compose.yml` directly without following up with `apply` — CasaOS keeps internal state that must stay in sync with the file.

## API Discovery

CasaOS exposes its core OpenAPI document through the gateway:

```bash
curl -sS http://localhost:80/doc/v2/casaos/openapi.yaml
```

When direct API calls fail, first check whether the gateway port differs:

```bash
sudo test -f /etc/casaos/gateway.ini && sudo sed -n '1,80p' /etc/casaos/gateway.ini
```

Then retry with `http://localhost:<port>`.
