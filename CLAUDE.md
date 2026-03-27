# CLAUDE.md — newsdiff-deploy

Docker Compose deployment for [NewsDiff](https://github.com/rmdes/newsdiff).
No application code lives here — only deployment configuration.

## Repository Layout

```
newsdiff-deploy/
├── docker-compose.yml    # 6 services: app, bot, nginx, migrate, postgres, redis
├── nginx.conf            # Reverse proxy: AP paths → bot, everything else → app
├── .env.example          # All configuration variables
├── README.md             # Setup and usage guide
└── CLAUDE.md             # This file
```

## Services

| Service | Image | Entrypoint | Purpose |
|---------|-------|-----------|---------|
| `app` | `ghcr.io/rmdes/newsdiff` | `node build/index.js` | SvelteKit + feed poller + syndicator |
| `bot` | `ghcr.io/rmdes/newsdiff` | `node --import tsx/esm src/bot/index.ts` | ActivityPub federation server |
| `migrate` | `ghcr.io/rmdes/newsdiff` | `node --import tsx/esm src/lib/server/db/migrate.ts` | One-shot DB migrations |
| `nginx` | `nginx:alpine` | default | Reverse proxy |
| `postgres` | `postgres:17-alpine` | default | Database |
| `redis` | `redis:7-alpine` | default | Job queue + federation state |

## Commands

```bash
# Start everything
docker compose up -d

# View logs
docker compose logs -f app bot

# Restart after .env changes
docker compose up -d

# Update to latest image
docker compose pull && docker compose up -d

# Database backup
docker compose exec postgres pg_dump -U newsdiff newsdiff > backup.sql
```

## How nginx routes traffic

- `/.well-known/webfinger`, `/.well-known/nodeinfo` → bot
- `/ap/*`, `/users/*`, `/nodeinfo/*` → bot
- Browser requests to `/ap/*` (no `application/activity+json` Accept header) → redirect to `/about`
- Everything else → app

## Shared state

`app` and `bot` share Redis for Botkit/Fedify state. The app publishes outgoing AP messages via Botkit's session API, while the bot handles incoming federation requests. Both connect to the same Redis instance.

The `app-data` volume is shared between `app` and `bot` for bot profile config (`/data/config/bot-profile.json`) and images (`/data/images/`).
