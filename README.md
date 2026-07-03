# telegram-local-bot-api-server

Standalone infrastructure repo for running the Telegram Local Bot API Server on a NAS with Docker Compose.

This repository is only for the Telegram Local Bot API Server itself. It does not contain bot business logic. Your bot repos should connect to this service through `TELEGRAM_BOT_API_BASE_URL`.

## What Telegram Local Bot API Server does

Telegram provides an official Bot API server that you can self-host. Running it locally is useful when you want:

- large Telegram file upload and download support
- local NAS file handling
- multiple bots sharing one local Bot API endpoint
- a long-running Bot API service that is independent from any single bot repo

This setup is especially helpful if your bots move large media files around your NAS or need one shared local Bot API endpoint.

## When it is useful

- Bots upload or download large files.
- Bots need to work with local NAS-mounted files.
- Multiple bot projects should reuse one local Bot API server.
- You want a persistent Bot API service managed separately from bot code.

## When it is not necessary

- Simple text-only bots.
- Bots that do not handle large files.
- Small projects that are fine using the default `https://api.telegram.org` endpoint directly.

## Requirements

- Docker
- Docker Compose
- Telegram `api_id` and `api_hash`

## Get `api_id` and `api_hash`

1. Sign in to `https://my.telegram.org`.
2. Open **API Development Tools**.
3. Create an application if you do not already have one.
4. Copy the generated `api_id` and `api_hash`.

Do not put bot tokens in this repo. Only the Telegram application credentials belong in `.env`.

## Configure `.env`

1. Copy `.env.example` to `.env`.
2. Set:

```env
TELEGRAM_API_ID=your_api_id
TELEGRAM_API_HASH=your_api_hash
```

3. Leave the default local-mode values unless you have a specific reason to change them.

Notes:

- `TELEGRAM_LOCAL=1` enables local mode.
- `TELEGRAM_STAT=1` enables the internal stats endpoint used by the healthcheck.
- To disable either flag later, leave its value blank in `.env`.
- `TELEGRAM_DATA_DIR` stores the Bot API server working data on the host.
- `TELEGRAM_FILES_DIR` mounts a host directory at `/srv/telegram-local-files` inside the container for NAS-local file workflows.

## Start the service

```sh
docker compose up -d
```

The compose file uses security-first defaults:

- host access is bound to `127.0.0.1:8081`
- by default, other containers on the shared Docker network can use `http://telegram-bot-api:8081`
- port `8081` is not exposed publicly by default
- both this stack and the bot stack must join the external Docker network `telegram-shared`
- the Bot API listens on `8081`; `8082` is only the internal stats endpoint used by the healthcheck

## Check logs

```sh
docker compose logs -f
```

## Stop the service

```sh
docker compose down
```

## Endpoints

Default endpoint for other containers on the same Docker network:

```text
http://telegram-bot-api:8081
```

Optional local NAS testing endpoint:

```text
http://127.0.0.1:8081
```

If a bot runs outside the Docker network and you intentionally expose this service to your LAN, it can use:

```text
http://NAS_LAN_IP:8081
```

That requires changing the host port binding in `docker-compose.yml`. Do not do this unless your LAN is trusted and protected.

## How another bot should connect

Set:

```env
TELEGRAM_BOT_API_BASE_URL=http://telegram-bot-api:8081
```

If the bot is outside the Docker network and you have intentionally exposed the server on your LAN, use:

```env
TELEGRAM_BOT_API_BASE_URL=http://NAS_LAN_IP:8081
```

## Shared Docker network for multiple bot repos

This compose file joins the external Docker network `telegram-shared`. Other compose projects must also join `telegram-shared` so they can reach the service as `telegram-bot-api`.

Example bot compose snippet:

```yaml
services:
  my-bot:
    image: your-bot-image
    environment:
      TELEGRAM_BOT_API_BASE_URL: http://telegram-bot-api:8081
    networks:
      - telegram-shared

networks:
  telegram-shared:
    external: true
    name: telegram-shared
```

Create that shared network once before starting the stacks:

```sh
docker network create telegram-shared
```

## Local file behavior

In local mode:

- `getFile` may return local absolute file paths instead of only remote download paths.
- Bots should be prepared to handle both local file paths and normal remote file downloads.
- The mounted host files directory is available inside the container at `/srv/telegram-local-files`.

If another bot container needs to work with the same NAS-local files directly, mount the same host directory into that bot container too.

## Migration note

If a bot previously used the official Telegram Bot API at `https://api.telegram.org`, call `logOut` before switching that bot to the local server. After switching, make sure the bot can handle local absolute file paths returned by `getFile`.

## Security warning

- Do not expose port `8081` to the public internet.
- Avoid binding to `0.0.0.0` unless you have a trusted LAN and firewall rules in place.
- If remote access is ever required, put the service behind a reverse proxy, terminate TLS there, and add access control.
- The local Bot API server speaks HTTP only. Remote HTTPS access should be handled by a reverse proxy.

## Helper scripts

Small wrappers are included in `scripts/`:

- `scripts/up.sh`
- `scripts/down.sh`
- `scripts/logs.sh`

They call the same `docker compose` commands shown above.

## Troubleshooting

### Missing `TELEGRAM_API_ID` or `TELEGRAM_API_HASH`

- Make sure `.env` exists.
- Confirm both values are set and not left blank.
- Restart with `docker compose up -d` after updating `.env`.

### Container cannot start

- Check logs with `docker compose logs -f`.
- Verify your `api_id` and `api_hash`.
- Make sure the `data/` and `files/` paths are writable on the NAS.

### Bot cannot reach `http://telegram-bot-api:8081`

- Make sure both the bot container and this service are on `telegram-shared`.
- Confirm the bot uses `TELEGRAM_BOT_API_BASE_URL=http://telegram-bot-api:8081`.
- Check that the service is healthy with `docker compose ps`.

### Port `8081` already in use

- Change `TELEGRAM_HTTP_PORT` in `.env` to an unused port.
- Update any host-based test URLs accordingly.
- If you change `TELEGRAM_HTTP_PORT`, update bot-side base URLs to match the new port too.

### Permission issues with mounted `data/` or `files/` directories

- The image stores working data under `/var/lib/telegram-bot-api`.
- On first run, Docker may create host directories automatically.
- If your NAS uses stricter permissions, pre-create the directories and grant write access for the container.

### Switching back to the official Telegram Bot API

- Remove or override `TELEGRAM_BOT_API_BASE_URL` in the bot project so it points back to `https://api.telegram.org`.
- Restart the bot after changing the base URL.
- If webhook or update delivery looks stuck, clear the webhook and restart the bot before retrying.
