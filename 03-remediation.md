# 03 — Remediation

The remediation plan and the exact commands I ran.

## Setup reference

To avoid confusion between service names, container names and host paths:

- **Project root:** `/opt/docker/<project>-wp` — contains `docker-compose.yml`
  and `.env`. All `docker compose` commands below are run from this directory.
- **Docker Compose version:** v5.6.1 (`docker compose`, v2 syntax).
- **Services in `docker-compose.yml`:** `wordpress` and `db`.
- **Containers:** `<project>-wp` (WordPress, image `wordpress:latest`) and
  `<project>-db` (MariaDB, image `mariadb:latest`). The `container_name` for
  each service is set explicitly in `docker-compose.yml`.
- **Host directories and their mount points:**
  - `/opt/docker/<project>-wp/wp_data` → `/var/www/html` (inside `<project>-wp`)
  - `/opt/docker/<project>-db/db_data` → `/var/lib/mysql` (inside `<project>-db`)

In the commands below:

- Host paths start with `/opt/`.
- Container names do not contain a slash and are used with `docker exec`.

## Plan

The goal was to move the container from PHP 8.3.27 to a patched version
(8.3.36 or later) without losing data. The plan was:

1. Verify that the data lives outside the container (bind mounts, not named
   volumes), so container recreation would not touch it.
2. Back up the database and the WordPress files as a precaution.
3. Pull the updated `wordpress` image.
4. Recreate the container from the new image.
5. Verify the new PHP version and run the scanner.

## Step 1: Confirm data lives on the host

Before touching anything, I checked how the data is mounted:

```bash
docker compose config
```

Relevant output (abridged):

```yaml
volumes:
  - type: bind
    source: /opt/docker/<project>-wp/wp_data
    target: /var/www/html
  - type: bind
    source: /opt/docker/<project>-db/db_data
    target: /var/lib/mysql
```

Both services use **bind mounts**. This means `/var/www/html` inside the container is the same directory as `/opt/docker/<project>-wp/wp_data` on the host — not a copy, but the same files. Recreating the container does not touch these directories, because they exist independently of the container.

## Step 2: Back up the database and files

Even though bind mounts meant the data was safe from `--force-recreate`, I took a backup as a precaution. If the new image failed to start, I wanted a known-good state to roll back to.

```bash
# Create a dated backup directory and load .env variables
mkdir -p ~/backups/$(date +%F)
cd /opt/docker/<project>-wp
set -a
source .env
set +a

# Database dump (the database container runs MariaDB, so mariadb-dump is used)
docker exec <project>-db mariadb-dump -u root -p"${DB_ROOT_PASSWORD}" "${WP_DB_NAME}" \
  > ~/backups/$(date +%F)/db_backup.sql

# WordPress files (wp_data is world-readable, so no sudo is needed)
tar -czf ~/backups/$(date +%F)/wp_backup.tar.gz \
  -C /opt/docker/<project>-wp/wp_data .
```

Notes on the commands:

- `mkdir -p ~/backups/$(date +%F)` creates a dated backup directory. The `-p`
  flag creates any missing parent directories (such as `~/backups` if it does
  not exist) and does not fail if the directory already exists.
- `set -a; source .env; set +a` loads the `.env` variables into the shell
  environment, so `${DB_ROOT_PASSWORD}` and `${WP_DB_NAME}` can be used without
  hardcoding them.
- The database is dumped via `mariadb-dump` inside the container, so no
  host-level `sudo` is needed for the backup at all.
- `-p"${DB_ROOT_PASSWORD}"` is the CLI syntax for `mariadb-dump` (and
  `mysql`/`mysqldump`) where the password follows `-p` without a space.
- `wp_data` is owned by `www-data` but world-readable (`drwxr-xr-x`), so the
  `tar` command does not require `sudo`.

## Step 3: Pull the updated image and recreate the container

```bash
docker compose pull wordpress
docker compose up -d --force-recreate wordpress
```

Why `--force-recreate`: `docker compose up -d` on its own recreates a container only if its configuration or image changed. Since the image tag (`wordpress:latest`) did not change, Compose would have kept the old container running on the old PHP version. `--force-recreate` forces Compose to stop and recreate the container from the freshly pulled image, even if the configuration looks unchanged ([Docker docs](https://docs.docker.com/reference/cli/docker/compose/up/)).

The bind-mounted directories (`wp_data` and `db_data`) were not affected — Docker reattached them to the new container, and the files remained on the host throughout.
