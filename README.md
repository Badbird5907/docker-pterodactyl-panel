# docker-pterodactyl-panel

Easily set up pterodactyl on Docker, adhering to the *one service per container* mantra.

(Well, mostly. Cron and php-fpm run on the same container, but cron doesn't count, right?)

I think it should be compatible with Docker Swarm, but I'm not able to test this myself yet.

## Getting started

1. Copy `docker-compose.yml` to somewhere on your computer.
2. Edit `docker-compose.yml` to choose a secure username and password for your database. You'll need to use the same credentials for both mariadb's `MYSQL_USER`/`MYSQL_PASSWORD` and PHP-FPM's `DB_USERNAME`/`DB_PASSWORD` environment variables.
3. Generate an app key once and export it in your shell before starting:
   - Linux/macOS:
     - `export APP_KEY=base64:YOUR_GENERATED_KEY`
   - Windows PowerShell:
     - `$env:APP_KEY="base64:YOUR_GENERATED_KEY"`
4. Start:
   - `docker compose up -d`
5. Find the php container name using `docker ps` (it should look like `docker-pterodactyl-panel-php-fpm-1`).
6. Run `docker exec -it <php-container-name> php artisan p:user:make` and follow the prompts to set up your first user.

## Portainer deployment (Docker Hub image)

You can use `docker-compose.yml` directly in Portainer.  
`docker-compose.portainer.yml` is kept as an optional explicit Portainer variant.

1. Build and push a fixed image tag to Docker Hub (example tag used here: `v1.12.2-fixed1`).
2. In Portainer, create or edit the stack and paste/import `docker-compose.yml`.
3. Add stack environment variable `APP_KEY` with a stable value (do not rotate it on restarts).
4. Deploy the stack.
5. Create first user:
   - Open `php-fpm` container console (`/bin/sh`) and run `php artisan p:user:make`.

## Why APP_KEY must stay fixed

This stack no longer runs `php artisan key:generate` at container startup.

That is intentional: rotating `APP_KEY` on every restart causes encrypted daemon/API payloads to fail decryption (`The MAC is invalid`) and leads to intermittent or persistent 500 errors.

Set `APP_KEY` once and keep it stable across restarts and upgrades.

## Upgrading safely

Do not run migrations automatically on every boot. Upgrades should be explicit.

1. Backup database and volumes.
2. Update the panel image tag in `docker-compose.yml`.
3. Pull updated images:
   - `docker compose pull php-fpm panel-upgrade`
4. Run migrations once using the one-shot upgrade service:
   - `docker compose --profile upgrade run --rm panel-upgrade`
5. Restart runtime services:
   - `docker compose up -d php-fpm nginx`
6. Smoke test panel login, API endpoints, and wings/daemon connectivity.

### Upgrade flow in Portainer

1. Update `php-fpm` image tag in `docker-compose.yml` (or `docker-compose.portainer.yml` if you use that variant).
2. Redeploy stack.
3. Run migration once in `php-fpm` container console (`/bin/sh`):
   - `php artisan migrate --force`
4. Verify panel and wings/daemon communication.

## Using your own MySQL/MariaDB server

By default this stack includes its own MariaDB server, but with a few modifications you can have it use an existing database server instead. You will need to make several modifications to docker-compose.yml.

1. Edit docker-compose.yml to remove the database server block.
2. Edit docker-compose.yml to remove database from the depends_on section of the php server block.
3. Edit docker-compose.yml to modify php-fpm's DB_HOST and DB_PORT environment variables.
