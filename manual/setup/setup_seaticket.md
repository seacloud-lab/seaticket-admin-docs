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

wget -O .env https://manual.seaticket.ai/0.9/repo/docker/env
wget https://manual.seaticket.ai/0.9/repo/docker/caddy.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seadb.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seaqa-web.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seaqa-ai.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seaqa-events.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seaqa-indexer.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seasearch.yml
wget -O seaticket-data/conf/seaticket_config.yaml https://manual.seaticket.ai/0.9/repo/docker/seaticket_config.yaml
```

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
| `SEASEARCH_TOKEN` | Base64 authorization token for the SeaSearch account. | Required |

By default, the first SeaSearch account uses the first team administrator credentials. Generate the token after setting `INIT_SEATICKET_TEAM_ADMIN_EMAIL` and `INIT_SEATICKET_TEAM_ADMIN_PASSWORD`:

```bash
echo -n 'INIT_SEATICKET_TEAM_ADMIN_EMAIL:INIT_SEATICKET_TEAM_ADMIN_PASSWORD' | base64
```

Replace the placeholders in the command with their actual values and assign the result to `SEASEARCH_TOKEN`.

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

For an external MySQL/MariaDB server, set `MYSQL_DB_HOST` and `MYSQL_DB_PORT` to its address and provide that server's root password in `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` for the first initialization. The server must allow root connections from the `seaqa-web` container and permit database creation, user creation, and grants. Automatic initialization is supported regardless of the hostname. When the bundled database is not needed, remove the `db` service from `seaqa-web.yml` and remove its `depends_on` entries from all enabled Compose files, including `seaqa-web.yml`, `seaqa-ai.yml`, `seaqa-events.yml`, `seaqa-indexer.yml`, and `seaqa-web-portal.yml` if enabled.

### SeaDB Settings

By default, SeaDB uses `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` as its initial root password and `MYSQL_DB_USER` / `MYSQL_DB_PASSWORD` as its application credentials. You can leave `INIT_SEADB_ROOT_PASSWORD`, `SEADB_USERNAME`, and `SEADB_PASSWORD` empty. Set any of these variables to use separate SeaDB credentials; each override is independent.

On first startup with an empty or missing data directory, the SeaDB container maps the resolved initialization password to `SEADB_ROOT_PASSWORD` and runs `sea-db init` to create the `root` administrator. The `seaqa-web` container then authenticates as `root` to create the resolved application account, assigning `default_role`. Compose supplies the resolved username as `SEADB_USER` inside all SeaTicket containers. SeaTicket services use the application credentials with HTTP Basic authentication to access SeaDB.

For an existing deployment that used the first team administrator credentials for SeaDB, explicitly set `SEADB_USERNAME` and `SEADB_PASSWORD` to the existing SeaDB account values before updating the Compose files. The new defaults do not modify existing accounts. `JWT_PRIVATE_KEY` is required by the SeaDB service but is not an API access token.

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

!!! warning "Existing deployments"

    Existing S3 deployments must explicitly set `ATTACHMENT_STORAGE_BACKEND=s3` before adopting these deployment files. SeaSearch follows that setting unless `SS_STORAGE_TYPE` is explicitly set. Changing a storage backend does not migrate existing attachments or search data.

#### Use S3 as the storage backend

Skip this section for the default `disk` deployment. To store both attachments and SeaSearch data in S3, set:

```dotenv
ATTACHMENT_STORAGE_BACKEND=s3
# Leave SS_STORAGE_TYPE empty to follow ATTACHMENT_STORAGE_BACKEND.
SS_STORAGE_TYPE=
S3_HOST=https://s3-endpoint.example.com
S3_KEY_ID=your-access-key-id
S3_SECRET_KEY=your-secret-access-key
S3_FILE_BUCKET=seaticket-files
S3_WEB_CRAWL_BUCKET=seaticket-connections
S3_SEASEARCH_BUCKET=seaticket-search
```

Create the buckets needed by the selected backends and grant the credentials read and write access to them. The components using S3 share the following endpoint and credentials:

| Variable | Description | Required when |
| --- | --- | --- |
| `S3_HOST` | S3-compatible storage endpoint. | Attachments or SeaSearch use S3 |
| `S3_KEY_ID` | S3 access key ID. | Attachments or SeaSearch use S3 |
| `S3_SECRET_KEY` | S3 secret access key. | Attachments or SeaSearch use S3 |
| `S3_FILE_BUCKET` | Bucket for uploaded files. | `ATTACHMENT_STORAGE_BACKEND=s3` |
| `S3_WEB_CRAWL_BUCKET` | Bucket for connection assets, such as email attachments. | `ATTACHMENT_STORAGE_BACKEND=s3` |
| `S3_SEASEARCH_BUCKET` | Bucket for SeaSearch data. Compose maps this to `S3_SS_BUCKET` unless that variable is explicitly set. | Resolved `SS_STORAGE_TYPE=s3` |

Advanced S3 options are normally left at their defaults:

| Variable | Description | Default |
| --- | --- | --- |
| `S3_AWS_REGION` | AWS region used by SeaSearch. | `us-east-1` |
| `S3_USE_V4_SIGNATURE` | Use Signature Version 4 for SeaSearch S3 requests. | `true` |
| `S3_PATH_STYLE_REQUEST` | Use path-style S3 requests in SeaSearch. | `true` |
| `S3_SSE_C_KEY` | Optional SeaSearch SSE-C encryption key. Requires HTTPS and Signature Version 4. | Empty |

These advanced options configure SeaSearch. The attachment backend currently uses Signature Version 4 and path-style requests directly; it does not read these four options.

To use different storage backends, set `SS_STORAGE_TYPE` explicitly. For example, `ATTACHMENT_STORAGE_BACKEND=disk` with `SS_STORAGE_TYPE=s3` keeps attachments on disk and stores SeaSearch data in S3; only the shared S3 credentials, endpoint, and SeaSearch bucket are needed.

### SeaSearch Settings

| Variable | Description | Default |
| --- | --- | --- |
| `SEASEARCH_URL` | SeaSearch URL reachable from SeaTicket services. | `http://seasearch:4080` |
| `SEASEARCH_TOKEN` | Base64 authorization token for the SeaSearch API. | Required |
| `INIT_SS_ADMIN_USER` | Optional separate initial SeaSearch account. | First team administrator email |
| `INIT_SS_ADMIN_PASSWORD` | Optional separate initial SeaSearch account password. | First team administrator password |

When both `INIT_SS_ADMIN_USER` and `INIT_SS_ADMIN_PASSWORD` are set, SeaSearch uses that separate account. In this case, generate `SEASEARCH_TOKEN` from those values instead.

### Optional Extensions

The template includes the notification service but does not enable it by default. See [Setup Notification Server](setup_notification_server.md) to enable it.

## Configure `seaticket_config.yaml`

Configure LLM models and other application settings in `/opt/seaticket/seaticket-data/conf/seaticket_config.yaml`. See [SeaTicket YAML Configuration](../configuration/seaticket_yaml.md) for details.

## Start SeaTicket

Run Docker Compose from `/opt/seaticket`:

```bash
docker compose up -d
```

On the first deployment, follow initialization progress:

```bash
docker compose logs -f seaqa-web
```

SeaDB initializes its storage and root account before starting its HTTP service. SeaTicket waits for MySQL/MariaDB and SeaDB, initializes the MySQL database, creates the SeaDB application account, initializes SeaDB template tables, then creates the first team, its administrator, and that administrator's initial workspace. It does not create a system administrator. Sign in with `INIT_SEATICKET_TEAM_ADMIN_EMAIL` and `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` to use SeaTicket.

After deployment succeeds, remove `INIT_SEADB_ROOT_PASSWORD` and `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` from `.env` and run `docker compose up -d` to recreate affected containers. Keep the root passwords backed up for maintenance. Normal restarts use persistent initialization state and application credentials. Keep MySQL application credentials (also used by SeaDB by default), any separate SeaDB credentials, and first-team credentials used as the SeaSearch defaults in `.env`.

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
docker compose logs --follow seaqa-web
```

SeaTicket logs are stored under `/shared/logs` in the container and `/opt/seaticket/seaticket-data/logs` on the host by default.

To monitor all SeaTicket log files on the host:

```bash
sudo tail -f $(find /opt/seaticket/seaticket-data/logs/ -type f -name '*.log' 2>/dev/null)
```
