# Docker Deployment Guide

This repository includes a production-focused Docker setup built around `docker compose`.

## Stack

- `app`: Laravel application served by Apache on PHP `8.3`
- `db`: MySQL `8.4`
- `db_data`: named volume for MySQL persistence
- `app_storage`: named volume for Laravel `storage/`

The app is exposed on port `8080` by default.

## Prerequisites

- Docker
- Docker Compose v2

Verify both are available:

```bash
docker --version
docker compose version
```

## First Deployment

1. Copy the Docker environment template:

   ```bash
   cp .env.docker.example .env
   ```

2. Generate a stable application key and set it in `.env`:

   ```bash
   docker run --rm php:8.3-cli php -r "echo 'base64:'.base64_encode(random_bytes(32)).PHP_EOL;"
   ```

3. Update the required values in `.env`:

   - `APP_KEY`
   - `APP_URL`
   - `DB_DATABASE`
   - `DB_USERNAME`
   - `DB_PASSWORD`
   - `DB_ROOT_PASSWORD`

4. Set `APP_URL` to the public HTTPS URL used in deployment.

   This matters for GitHub OAuth callbacks and webhook URLs. For local Docker usage, `http://localhost:8080` is fine.

5. Build and start the stack:

   ```bash
   docker compose up -d --build
   ```

6. Optionally seed the database on first boot:

   ```bash
   docker compose exec app php artisan db:seed --force
   ```

7. Verify the deployment:

   ```bash
   docker compose ps
   curl http://127.0.0.1:8080/up
   ```

The health endpoint returns `200` only after Laravel boots and the database connection succeeds.

## Default Runtime Behavior

On startup, the app container:

- waits for MySQL to become reachable
- runs `php artisan migrate --force`
- ensures `public/storage` exists
- warms Laravel config and view caches

This makes updates idempotent and removes the need for manual migration steps during a normal redeploy.

## Updating a Deployment

Use the same rebuild-and-redeploy flow for code changes:

```bash
git pull
docker compose up -d --build
```

## Useful Commands

View service status:

```bash
docker compose ps
```

Tail logs:

```bash
docker compose logs -f app
docker compose logs -f db
```

Run Artisan commands:

```bash
docker compose exec app php artisan migrate --force
docker compose exec app php artisan optimize:clear
docker compose exec app php artisan tinker
```

Open a shell in the app container:

```bash
docker compose exec app bash
```

Stop the stack:

```bash
docker compose down
```

Stop the stack and remove volumes:

```bash
docker compose down -v
```

Use `docker compose down -v` only when you intentionally want to delete MySQL data and persisted Laravel storage.

## Persistence

The deployment uses named volumes:

- `db_data` persists MySQL data
- `app_storage` persists Laravel file storage, sessions, and cache files

This setup is intended for a single-node deployment where file-backed sessions and cache are acceptable.

## Production Notes

- PHP is pinned to `8.3` because the current dependency lockfile is not compatible with PHP `8.4`.
- Queue processing is configured as `sync`.
- Session and cache drivers are file-based.
- TLS termination and reverse proxy setup are not included in this stack.
- Redis, dedicated queue workers, and multi-instance deployment are out of scope for this configuration.

## Troubleshooting

If the app does not become healthy:

1. Check service status:

   ```bash
   docker compose ps
   ```

2. Inspect logs:

   ```bash
   docker compose logs --tail=200 app
   docker compose logs --tail=200 db
   ```

3. Verify `.env` values:

   - `APP_KEY` must be set
   - `DB_*` values must match the database service configuration
   - `APP_URL` should match the deployed URL

4. Rebuild after env or code changes:

   ```bash
   docker compose up -d --build
   ```
