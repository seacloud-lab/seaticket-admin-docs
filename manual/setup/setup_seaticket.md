# Setup SeaTicket Server

## Getting started

The following assumptions and conventions are used in the rest of this document:

- `/opt/seaticket` is the directory for storing SeaTicket Docker Compose files and `.env`. If you choose a different directory, adjust all paths accordingly.
- SeaTicket uses Docker bind mounts to persist data on the host. The default host paths are `/opt/seaticket/seaticket-data` for application data, `/opt/seaticket-mysql/db` for MySQL/MariaDB, `/opt/seadb-data` for SeaDB, `/opt/seasearch-data` for SeaSearch, and `/opt/caddy-data` for Caddy. It is recommended to keep these defaults. If you change them, account for the new paths when following these instructions.
- SeaTicket configuration files, application logs, and locally stored attachments are kept under `/opt/seaticket/seaticket-data`, in the `conf`, `logs`, and `data/attachments` directories, respectively. This directory is mounted at `/shared` inside the SeaTicket containers.

## Download Deployment Files

SeaTicket uses `.env` and Docker Compose files from the same directory. Download the standard deployment files:

```bash
mkdir -p /opt/seaticket/seaticket-data/conf
cd /opt/seaticket

wget -O .env https://manual.seaticket.ai/1.0/repo/docker/env
wget https://manual.seaticket.ai/1.0/repo/docker/caddy.yml
wget https://manual.seaticket.ai/1.0/repo/docker/seadb.yml
wget https://manual.seaticket.ai/1.0/repo/docker/seaticket.yml
wget https://manual.seaticket.ai/1.0/repo/docker/seasearch.yml
wget -O seaticket-data/conf/seaticket_config.yaml https://manual.seaticket.ai/1.0/repo/docker/seaticket_config.yaml
```

`seaticket.yml` includes the main web service, AI, Events, Indexer, MariaDB, and Redis. Notification and Portal are optional and are not required for a standard deployment. SeaTicket and SeaDB images default to `1.0-latest` on Docker Hub.

The standard `.env` selects all required Compose files:

```env
COMPOSE_FILE='caddy.yml,seadb.yml,seaticket.yml,seasearch.yml'
COMPOSE_PATH_SEPARATOR=','
```

Components communicate using their built-in Docker service addresses. You do not need to configure internal endpoints for a single-host deployment. Keep the service names unchanged; see [internal server URLs](../configuration/seaticket_yaml.md#internal-server-urls) for custom deployments.

## Configure `.env`

Edit `/opt/seaticket/.env`. For a standard deployment, change only the required settings below and leave the remaining template values at their defaults.

### Required Settings

| Variable | Description | Value |
| --- | --- | --- |
| `SEATICKET_SERVER_HOSTNAME` | Public hostname or domain of the SeaTicket server. | Required |
| `JWT_PRIVATE_KEY` | Random string of at least 32 characters. Generate with `pwgen -s 40 1`. | Required |
| `SECRET_KEY` | Random string of at least 32 characters. Back it up and do not change it after deployment. | Required |
| `MYSQL_DB_PASSWORD` | Password for the SeaTicket MariaDB user. | Required |
| `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` | Password for the bundled MariaDB `root` user. | Required on first deployment only |
| `INIT_SEADB_ROOT_PASSWORD` | Optional separate SeaDB `root` password. When empty, uses `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` for initialization. | Optional |
| `INIT_SEATICKET_TEAM_ADMIN_EMAIL` | Email address for the first SeaTicket team administrator. | Required on first deployment only |
| `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` | Password for the first SeaTicket team administrator. | Required on first deployment only |
| `SEADB_USERNAME` | Optional separate SeaDB API username. When empty, uses `MYSQL_DB_USER` (default `seaticket`). | Optional |
| `SEADB_PASSWORD` | Optional separate SeaDB API password. When empty, uses `MYSQL_DB_PASSWORD`. | Optional |

The template includes a dedicated internal SeaSearch account and its matching token. No SeaSearch credential changes are needed for a standard deployment. See [SeaSearch](setup_seasearch.md) for custom configuration.

### Common Optional Settings

| Variable | Description | Default |
| --- | --- | --- |
| `SEATICKET_SERVER_PROTOCOL` | Public server protocol: `http` or `https`. | `http` |
| `TIME_ZONE` | Time zone used by containers. | `UTC` |
| `INIT_SEATICKET_TEAM_NAME` | Name of the first SeaTicket team. | `SeaTicket` |
| `ENABLE_NOTIFICATION_SERVER` | Enable the WebSocket notification service. | `false` |
| `LOG_TO_STDOUT` | Write component logs to container standard output. | `false` |

### Database Settings

The standard Compose deployment includes MariaDB and initializes it automatically. Do not create the database, database user, or tables manually.

| Variable | Description | Default |
| --- | --- | --- |
| `MYSQL_DB_HOST` | MySQL/MariaDB hostname; use `db` for bundled MariaDB. | `db` |
| `MYSQL_DB_PORT` | MariaDB port. | `3306` |
| `MYSQL_DB_USER` | SeaTicket database user. | `seaticket` |
| `MYSQL_DB_NAME` | SeaTicket database name. | `seaticket_db` |

On first startup, the entrypoint runs `/scripts/init_mysql.py`. This script uses PyMySQL and `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` to create the database and user, then imports `sql/mysql.sql` with the `mysql` client if the database has no tables. Credentials for the import are stored in a private temporary option file that is removed afterward. It records successful initialization in `seaqa-web-data/.mysql-initialized` under `SEATICKET_VOLUME`. Subsequent startups use the application account and skip root initialization. Keep the MySQL data and SeaTicket application data persistent and back up both database passwords.

For an external MySQL/MariaDB server, set `MYSQL_DB_HOST` and `MYSQL_DB_PORT` to its address and provide that server's root password in `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` for the first initialization. The server must allow root connections from the `seaticket` container and permit database creation, user creation, and grants. Automatic initialization is supported regardless of the hostname. When the bundled database is not needed, remove the `db` service and all `depends_on.db` entries from `seaticket.yml`, and remove `depends_on.db` from `seaticket-portal.yml` if enabled.

### SeaDB Settings

By default, SeaDB uses `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` as its initial root password and `MYSQL_DB_USER` / `MYSQL_DB_PASSWORD` as its application credentials. You can leave `INIT_SEADB_ROOT_PASSWORD`, `SEADB_USERNAME`, and `SEADB_PASSWORD` empty. Set any of these variables to use separate SeaDB credentials; each override is independent.

On first startup with an empty or missing data directory, the SeaDB container maps the resolved initialization password to `SEADB_ROOT_PASSWORD` and runs `sea-db init` to create the `root` administrator. The `seaticket` container then authenticates as `root` to create the resolved application account, assigning `default_role`. Compose supplies the resolved username as `SEADB_USER` inside all SeaTicket containers. SeaTicket services use the application credentials with HTTP Basic authentication to access SeaDB.

`JWT_PRIVATE_KEY` is required by the SeaDB service but is not an API access token.

### Cache Settings

The standard deployment includes Redis. No changes are required for normal use.

| Variable | Description | Default |
| --- | --- | --- |
| `REDIS_HOST` | Bundled Redis service hostname. | `redis` |
| `REDIS_PORT` | Redis port. | `6379` |
| `REDIS_PASSWORD` | Redis password. | (none) |

### Storage Settings

Leave the storage settings at their defaults for a local-disk deployment without S3:

| Variable | Description | Default |
| --- | --- | --- |
| `ATTACHMENT_STORAGE_BACKEND` | Attachment storage backend: `disk` or `s3`. Used by the web service, IO service, indexer, and optional portal. | `disk` |
| `SS_STORAGE_TYPE` | Optional SeaSearch storage override: `disk` or `s3`. If unset or empty, uses `ATTACHMENT_STORAGE_BACKEND`; if both are unset or empty, uses `disk`. | Follows `ATTACHMENT_STORAGE_BACKEND` |

!!! note "Attachment storage with disk backend"

    With `ATTACHMENT_STORAGE_BACKEND=disk`, attachments are saved to `${SEATICKET_VOLUME}/data/attachments` on the host (`/opt/seaticket/seaticket-data/data/attachments` by default). This directory contains `project_attachments` and `connection_attachments` and must be included in backups.

    Inside the web and indexer containers, `/opt/seaticket/data` links to `/shared/data`. The image fixes the attachment root at `/opt/seaticket/data/attachments`; no attachment-root setting is required in `.env` or `seaticket_config.yaml`. All attachment services must share the same `SEATICKET_VOLUME`.

When SeaSearch uses `disk`, its data is persisted under `${SS_DATA_PATH}` on the host (`/opt/seasearch-data` by default).

!!! warning "Changing storage backends"

    Changing a storage backend does not automatically move attachments or search data between backends.

For S3 storage, including using different backends for attachments and SeaSearch, see [S3 Storage](setup_s3.md).

### Optional Extensions

Notification is provided as a separate optional Compose file and is not enabled by default:

- [Setup Notification Server](setup_notification_server.md)

## Configure `seaticket_config.yaml`

Configure LLM models and other application settings in `/opt/seaticket/seaticket-data/conf/seaticket_config.yaml`. See [SeaTicket YAML Configuration](../configuration/seaticket_yaml.md) for details.

## Start SeaTicket

Run Docker Compose from `/opt/seaticket`:

```bash
docker compose up -d
```

On the first deployment, follow initialization progress:

```bash
docker compose logs -f seaticket
```

SeaDB initializes its storage and root account before starting its HTTP service. SeaTicket waits for MySQL/MariaDB and SeaDB, initializes the MySQL database, creates the SeaDB application account, initializes SeaDB template tables, then creates the first team, its administrator, and that administrator's initial workspace. It does not create a system administrator. Sign in with `INIT_SEATICKET_TEAM_ADMIN_EMAIL` and `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` to use SeaTicket.

After deployment succeeds, remove `INIT_SEADB_ROOT_PASSWORD` and `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` from `.env` and run `docker compose up -d` to recreate affected containers. Keep the root passwords backed up for maintenance. Normal restarts use persistent initialization state and application credentials. Keep MySQL application credentials (also used by SeaDB by default), any separate SeaDB credentials, and the preconfigured SeaSearch credentials and token in `.env`. SeaSearch credentials are independent of the first team administrator credentials.

System administrators are optional maintenance accounts. See [Account Management](../administration/account_management.md) when you need to create one.

## Directory Structure

### `/opt/seadb-data`

`/opt/seadb-data` stores SeaDB data and logs.

### `/opt/seaticket/seaticket-data`

`/opt/seaticket/seaticket-data` stores SeaTicket configuration, logs, and persistent application data.

- `conf`: SeaTicket configuration files.
- `logs`: SeaTicket logs, including `seaqa-web.log`.
- `seaqa-web-data`: Avatar files and custom media.
- `data/attachments`: Project and connection attachments when using the `disk` backend.

### `/opt/seasearch-data`

The default `SS_DATA_PATH` stores persistent SeaSearch data. With the `disk` backend, keep this directory backed up along with the SeaTicket application data.

## Find Logs

To monitor service logs from the deployment directory:

```bash
docker compose logs --follow
docker compose logs --follow seaticket
```

SeaTicket logs are stored under `/shared/logs` in the container and `/opt/seaticket/seaticket-data/logs` on the host by default.

To monitor all SeaTicket log files on the host:

```bash
sudo tail -f $(find /opt/seaticket/seaticket-data/logs/ -type f -name '*.log' 2>/dev/null)
```
