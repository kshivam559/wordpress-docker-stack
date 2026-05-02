# WordPress Docker Stack

This repository contains a Docker-based WordPress deployment stack with a single documentation source in this README.

## Overview

The stack includes:

- Traefik as the public reverse proxy and Let's Encrypt TLS terminator
- Nginx as the web server in front of WordPress
- PHP-FPM 8.3 for application execution
- MariaDB 10.11 for the database
- Redis for object caching and transient storage support
- AdminNeo for database administration

## Project Layout

```text
.
├── docker-compose.yml
├── html/
├── mariadb/
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── default.conf
├── php/
│   ├── Dockerfile
│   ├── php.ini
│   └── www.conf
└── traefik/
    └── acme.json
```

## Service Roles

- `traefik` exposes ports 80 and 443, routes traffic to the Nginx container, and manages TLS certificates.
- `nginx` serves the WordPress site from `html/` and forwards PHP requests to `php:9000`.
- `php` builds a custom PHP-FPM image with MySQL, GD, Redis, and OPcache support.
- `mariadb` stores the WordPress database in the local `mariadb/` directory.
- `redis` provides a cache backend.
- `adminneo` is available locally on port `8080` for database management.

## Requirements

- Docker
- Docker Compose v2
- A domain name pointed to the server running this stack
- Valid DNS records for the configured hostnames

## Installation

For Ubuntu-based servers, install Docker Engine and the Compose plugin with the following commands:

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

Verify the installation:

```bash
docker --version
docker compose version
```

## Docker Log Rotation

Configure Docker daemon log rotation to prevent logs from consuming excessive disk space.

Create `/etc/docker/daemon.json` with:

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

## Initial Setup

1. Review `docker-compose.yml` and replace the placeholder values:
   - `yourmail@mail.com` in the Traefik ACME configuration
   - `example.com` and `www.example.com` in the Traefik and Nginx configuration if your domain is different
   - `MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, and `MYSQL_PASSWORD` for MariaDB

2. Make sure the ACME storage file exists and is writable by Docker:

   ```bash
   chmod 600 traefik/acme.json
   ```

3. Place your WordPress files inside `html/`.

   The current repository does not include WordPress core files in that directory yet, so the site root is expected to be populated before the stack is used.

4. Start the stack:

   ```bash
   docker compose up -d --build
   ```

5. Fix ownership for the web root if needed:

   ```bash
   chown -R www-data:www-data html
   chmod -R 777 html
   ```

## Importing Data (AdminNeo)

After the stack is built and running you can import database dumps using AdminNeo.

1. Confirm containers are healthy:

   ```bash
   docker compose ps
   ```

2. Open AdminNeo (local):

   - `http://127.0.0.1:8080`

   If the server is remote, create an SSH tunnel first:

   ```bash
   ssh -L 8080:localhost:8080 root@your-server-ip
   ```

3. Connect in AdminNeo UI:

   - System/Driver: MySQL
   - Server: `mariadb` (service name)
   - Username / Password: use the credentials from `docker-compose.yml`
   - Database: `wordpress` (or the database you created)

4. Import via the AdminNeo UI: use the Import/SQL feature and upload your `.sql` dump.

5. Alternative - import from the host (stdin) without AdminNeo UI:

   ```bash
   docker exec -i mariadb sh -c 'exec mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' < dump.sql
   ```

   Replace the environment variables or substitute credentials directly if needed.

## WordPress Configuration

Add the following configuration to `wp-config.php` to align WordPress with the container setup:

```php
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
   $_SERVER['HTTPS'] = 'on';
}

define('WP_REDIS_HOST', 'redis');
define('WP_REDIS_PORT', 6379);

define('WP_MEMORY_LIMIT', '64M');
define('WP_MAX_MEMORY_LIMIT', '128M');

define('FS_METHOD', 'direct');
define('DISALLOW_FILE_EDIT', true);

define('DISABLE_WP_CRON', true);
```

These settings:

- force WordPress to recognize HTTPS when it is behind Traefik
- connect WordPress to the Redis service in the Docker network
- set front-end and admin memory limits to match the application profile
- disable the theme/plugin file editor for safety
- allow direct filesystem operations inside the container
- disable built-in WP-Cron so the host cron job can handle scheduled tasks

## Runtime Commands

Use these commands for normal operations and maintenance:

```bash
docker compose up -d
docker compose down
docker compose logs php --tail=50
docker compose logs nginx --tail=50
docker compose logs mariadb --tail=50
docker compose logs traefik --tail=50
docker compose restart php
docker system prune -af --volumes
```

## WordPress Cron

Use the host machine's crontab to run `wp-cron.php` every 5 minutes without overlapping executions.

1. Open the crontab editor:

   ```bash
   crontab -e
   ```

2. Add the following line:

   ```bash
   */5 * * * * flock -n /tmp/wpcron.lock docker exec -i php php -d memory_limit=64M -q /var/www/html/wp-cron.php > /dev/null 2>&1
   ```

This runs every 5 minutes, uses `flock` to prevent concurrent runs, and executes cron inside the PHP container.

## Access

- Website: `https://example.com` and `https://www.example.com`

## Runtime Notes

- PHP is configured with `cgi.fix_pathinfo=0` for safer file handling.
- PHP memory limit is set to `128M`.
- Uploads are configured up to `20M`.
- `max_execution_time` and `max_input_time` are both set to `60` seconds.
- `session.cookie_secure=1` is enabled for secure sessions over HTTPS.
- OPcache is enabled and timestamp validation is disabled for performance.
- Nginx includes security-oriented rules for blocking invalid methods, suspicious user agents, PHP execution in uploads, `.htaccess`, and `xmlrpc.php`.
- Traefik handles HTTPS automatically through the configured ACME resolver.

## Troubleshooting

- If certificates are not issued, verify DNS records, the ACME email, and that ports `80` and `443` are reachable.
- If WordPress cannot reach the database, verify the MariaDB container is running and the credentials in `docker-compose.yml` match the application configuration.
- If PHP files return as downloads or 502 errors appear, check the Nginx and PHP-FPM containers with `docker compose logs nginx` and `docker compose logs php`.
- If logs grow quickly on the host, keep the Docker daemon log rotation settings in place.

## Notes For Production

- Replace hardcoded credentials with environment variables before sharing or deploying widely.
- Consider keeping `traefik/acme.json` out of source control if this repository is used outside a private server setup.
- Review the domain names in the Nginx and Traefik configuration before going live.
