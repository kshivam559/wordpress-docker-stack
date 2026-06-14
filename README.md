# WordPress (single-site) — Caddy + PHP-FPM Docker stack

This repository provides a Docker-based WordPress single-site stack using Caddy as the reverse proxy, PHP-FPM for PHP execution, MariaDB for storage and Redis for caching. The README focuses on Docker installation and project-specific setup.

## Overview

Services in this workspace:

- `caddy` — Caddy v2 as the public reverse proxy and automatic TLS (ACME) provider. Configuration is in `caddy/Caddyfile`.
- `php` — custom PHP-FPM image (see `php/Dockerfile`) for running WordPress.
- `mariadb` — MariaDB for the WordPress database. Configuration and persisted data live in `mariadb/`.
- `redis` — Redis for object caching and transient storage.

## Project layout

```
.
├── docker-compose.yml
├── README.md
├── caddy/
│   └── Caddyfile
├── html/                 # web root (place WordPress files here)
├── mariadb/
│   └── my.cnf
├── php/
│   ├── Dockerfile
│   ├── php.ini
│   └── www.conf
└── redis/
    └── redis.conf
```

## Requirements

- Docker
- Docker Compose v2 (the `docker compose` plugin)
- A domain name DNS record pointing to the server running this stack
- Ports `80` and `443` reachable from the Internet for ACME/TLS (if using public certs)

## Install Docker (Ubuntu example)

For Ubuntu-based servers, install Docker Engine and the Compose plugin with these commands:

```bash
apt update && apt install -y ca-certificates curl gnupg lsb-release && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
chmod a+r /etc/apt/keyrings/docker.gpg && \
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
apt update && \
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify:

```bash
docker --version
docker compose version
```

## Docker daemon log rotation

Prevent logs from consuming excessive disk space by creating `/etc/docker/daemon.json` with:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Then restart Docker:

```bash
systemctl restart docker
```

## Initial setup (project-specific)

1. Edit `docker-compose.yml` and update environment variables and placeholders (DB credentials, site domain names, etc.).
2. Update `caddy/Caddyfile` to set your site domain(s). Caddy will obtain TLS certificates automatically when ports 80/443 are reachable.
3. Ensure Caddy can persist certs and data — the compose file typically mounts a `caddy` data volume or folder; confirm permissions if you see errors.
4. Place WordPress core and theme files in `html/` (this repo does not include WordPress core by default).
5. Ensure secrets and credentials are set via environment variables or a secrets manager; avoid committing credentials to source control.

Example file permissions (run on the host if needed):

```bash
chmod -R 777 html
chown -R www-data:www-data html
```

Adjust ownership as appropriate for the user the `php` container expects.

## Start the stack

Build and run the stack:

```bash
docker compose up -d --build
```

Check container health:

```bash
docker compose ps
```

## Importing data (AdminNeo)

After the stack is running, you can import SQL dumps using the `adminneo` service.

1. Open AdminNeo:

- Local server: `http://127.0.0.1:8080`
- Remote server: create an SSH tunnel first, then open the same local URL:

```bash
ssh -L 8080:127.0.0.1:8080 <user>@<server-ip>
```

2. Log in using your MariaDB credentials from `docker-compose.yml`:

- System: MySQL
- Server: `mariadb` (Docker service name)
- Username: `MYSQL_USER` (default in this repo: `wpuser`)
- Password: `MYSQL_PASSWORD` (default in this repo: `strong_pass`)
- Database: `MYSQL_DATABASE` (default in this repo: `wordpress`)

3. In AdminNeo, open **Import** (or SQL command execution), upload your `.sql` file, and run it.

4. If the dump contains old URLs, run a WordPress-safe search/replace after import (recommended) before going live.

### CLI fallback (without AdminNeo)

To import a `.sql` dump directly into the running MariaDB container:

```bash
docker exec -i mariadb sh -c 'exec mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' < dump.sql
```

For root-based import:

```bash
docker exec -i mariadb sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"' < dump.sql
```

## WordPress configuration suggestions

Add or update the following in `wp-config.php` to match the container network and caching:

```php
// Recognize HTTPS when behind Caddy
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = 'on';
}

define('WP_REDIS_HOST', 'redis');
define('WP_REDIS_PORT', 6379);
define('WP_REDIS_DATABASE', 0);
define('WP_CACHE_KEY_SALT', 'your_site_salt_');

define('WP_MEMORY_LIMIT', '64M');
define('WP_MAX_MEMORY_LIMIT', '128M');

define('FS_METHOD', 'direct');
define('DISALLOW_FILE_EDIT', true);
define('DISABLE_WP_CRON', true);
```

These settings:

- let WordPress detect HTTPS through the reverse proxy
- connect WordPress to the `redis` service
- set sensible memory limits and disable file editing in production
- disable built-in WP-Cron so host cron can run scheduled tasks predictably

### WP-Cron host schedule

With `DISABLE_WP_CRON` enabled in `wp-config.php`, schedule cron from the host every 5 minutes.

1. Open crontab:

```bash
crontab -e
```

2. Add this line:

```bash
*/5 * * * * flock -n /tmp/wpcron.lock docker exec -i php php -d memory_limit=64M -q /var/www/html/wp-cron.php > /dev/null 2>&1
```

This prevents overlapping cron runs and executes cron inside the `php` container.

## Common runtime commands

```bash
docker compose up -d
docker compose down
docker compose logs php --tail=100
docker compose logs mariadb --tail=100
docker compose logs caddy --tail=100
docker compose restart php
docker system prune -af --volumes
```

## Troubleshooting

- If Caddy fails to provision certificates, verify DNS records and that ports `80`/`443` are forwarded to the host.
- If WordPress cannot connect to the database, check `docker compose logs mariadb` and verify env vars in `docker-compose.yml`.
- If PHP returns 502 or downloads PHP files, inspect `docker compose logs php` and `docker compose logs caddy`.
- If logs grow quickly, ensure Docker log rotation is enabled (see Docker daemon log rotation section).

## Production notes

- Do not commit secrets; use environment variables or a secrets manager.
- Keep `caddy/Caddyfile` and `php` config tuned for your deployment (timeouts, client_max_body_size, etc.).
- Consider configuring backups for `mariadb` data and periodic pruning of old Docker images/volumes.

---

If you'd like, I can also update `docker-compose.yml` to add explicit healthchecks, or add a short `Makefile` with common commands. Let me know which you'd prefer next.
