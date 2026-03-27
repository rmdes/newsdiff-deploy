# newsdiff-deploy

Docker Compose deployment for [NewsDiff](https://github.com/rmdes/newsdiff) — self-hosted news article diff tracker.

## Architecture

```
              :8080 (LISTEN_PORT)
                │
           ┌────▼────┐
           │  nginx   │ reverse proxy
           └────┬─────┘
       ┌────────┴────────┐
       │ AP paths        │ everything else
       ▼                 ▼
  ┌─────────┐      ┌─────────┐
  │   bot   │      │   app   │  SvelteKit + workers
  │  :8001  │      │  :3000  │
  └────┬────┘      └────┬────┘
       │                │
       ▼                ▼
  ┌─────────┐      ┌──────────┐
  │  redis  │      │ postgres │
  └─────────┘      └──────────┘
```

- **app** — SvelteKit web frontend, feed poller, syndicator workers
- **bot** — ActivityPub server (Botkit/Fedify), handles incoming federation
- **nginx** — routes `.well-known/webfinger`, `/ap/`, `/users/`, `/nodeinfo/` to bot; everything else to app
- **migrate** — one-shot service that runs DB migrations on startup, then exits
- **postgres** — database
- **redis** — shared by app (BullMQ) and bot (Fedify KV + message queue)

Both `app` and `bot` use the same Docker image with different entrypoints.

## Quick start

```bash
git clone https://github.com/rmdes/newsdiff-deploy.git
cd newsdiff-deploy

# Configure
cp .env.example .env
# Edit .env — at minimum set ORIGIN and POSTGRES_PASSWORD

# Start
docker compose up -d

# Check logs
docker compose logs -f app bot
```

The app listens on `LISTEN_PORT` (default: 8080). Put a TLS-terminating reverse proxy in front (Caddy, Traefik, nginx with certbot, etc.).

## TLS setup

NewsDiff requires HTTPS for ActivityPub federation. The simplest option is [Caddy](https://caddyserver.com/) as a front proxy with automatic HTTPS:

```
# Caddyfile
diff.yourdomain.com {
    reverse_proxy localhost:8080
}
```

Or add a Caddy service to `docker-compose.override.yml`:

```yaml
services:
  caddy:
    image: caddy:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
    depends_on:
      - nginx
    restart: unless-stopped

volumes:
  caddy-data:
```

## Configuration

All configuration is via environment variables in `.env`. See `.env.example` for the full list.

| Variable | Required | Description |
|----------|----------|-------------|
| `ORIGIN` | Yes | Public URL (e.g., `https://diff.yourdomain.com`) |
| `POSTGRES_PASSWORD` | Yes | Database password |
| `BOT_USERNAME` | No | ActivityPub bot username (default: `bot`) |
| `BOT_NAME` | No | Bot display name (default: `NewsDiff Bot`) |
| `BLUESKY_HANDLE` | No | Bluesky handle for syndication |
| `BLUESKY_PASSWORD` | No | Bluesky app password |
| `LISTEN_PORT` | No | Port nginx listens on (default: `8080`) |
| `SYNDICATE_RATE_MS` | No | Min gap between posts in ms (default: `300000` = 5 min) |

## Updating

```bash
docker compose pull
docker compose up -d
```

The `migrate` service runs automatically on every startup, applying any new database migrations.

## Building from source

If you prefer to build the image locally instead of pulling from the registry:

```bash
git clone https://github.com/rmdes/newsdiff.git
docker build -t newsdiff:local ./newsdiff

# Then in docker-compose.yml, replace:
#   image: ghcr.io/rmdes/newsdiff:latest
# with:
#   image: newsdiff:local
```

## Data

| Volume | Contents |
|--------|----------|
| `postgres-data` | Database |
| `redis-data` | Federation state, job queue |
| `app-data` | Bot profile config, diff card images, uploaded images |

## Backup

```bash
# Database
docker compose exec postgres pg_dump -U newsdiff newsdiff > backup.sql

# Restore
docker compose exec -T postgres psql -U newsdiff newsdiff < backup.sql
```
