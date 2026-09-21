# Setup SeaTicket Server

## Prerequisites

Install Docker and Docker Compose by following the [official Docker installation guide](https://docs.docker.com/engine/install/). SeaTicket stores persistent application, database, SeaDB, SeaSearch, and Caddy data under `/opt` by default.

Prepare an S3-compatible storage service before deployment. Create the following buckets and obtain credentials with read and write access to all of them:

- A bucket for uploaded files (`S3_FILE_BUCKET`).
- A bucket for web crawl data (`S3_WEB_CRAWL_BUCKET`).
- A bucket for SeaSearch data (`S3_SEASEARCH_BUCKET`).

You will configure the S3 endpoint, access key ID, secret access key, and bucket names in `.env`.

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
| `INIT_SEATICKET_TEAM_ADMIN_EMAIL` | Email address for the first SeaTicket team administrator. | Required on first deployment only |
| `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` | Password for the first SeaTicket team administrator. | Required on first deployment only |
| `S3_HOST` | S3-compatible storage endpoint. | Required |
| `S3_KEY_ID` | S3 access key ID. | Required |
| `S3_SECRET_KEY` | S3 secret access key. | Required |
| `S3_FILE_BUCKET` | S3 bucket for uploaded files. | Required |
| `S3_WEB_CRAWL_BUCKET` | S3 bucket for web crawl data. | Required |
| `S3_SEASEARCH_BUCKET` | S3 bucket for SeaSearch data. | Required |
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
| `MYSQL_DB_HOST` | Bundled MariaDB service hostname. | `db` |
| `MYSQL_DB_PORT` | MariaDB port. | `3306` |
| `MYSQL_DB_USER` | SeaTicket database user. | `seaticket` |
| `MYSQL_DB_NAME` | SeaTicket database name. | `seaticket_db` |

On first startup, SeaTicket creates the database and user, then imports the schema. With the default persistent `SEATICKET_MYSQL_VOLUME`, later restarts and image upgrades do not re-import the schema. Keep both database passwords backed up.

### Cache Settings

The standard deployment includes Redis. No changes are required for normal use.

| Variable | Description | Default |
| --- | --- | --- |
| `REDIS_HOST` | Bundled Redis service hostname. | `redis` |
| `REDIS_PORT` | Redis port. | `6379` |
| `REDIS_PASSWORD` | Redis password. | (none) |

### S3 Storage Settings

`seaqa-web`, the indexer, and SeaSearch share the S3 credentials and endpoint configured above. Keep `S3_AWS_REGION`, `S3_USE_V4_SIGNATURE`, and `S3_PATH_STYLE_REQUEST` at their defaults unless required by the storage provider.

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

SeaTicket automatically initializes MariaDB, imports the application schema, creates the first team, creates its team administrator, and creates that administrator's initial workspace. It does not create a system administrator. Sign in with `INIT_SEATICKET_TEAM_ADMIN_EMAIL` and `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` to use SeaTicket.

System administrators are optional maintenance accounts. See [Account Management](../administration/account_management.md) when you need to create one.

## Directory Structure

### `/opt/seadb-data`

`/opt/seadb-data` stores SeaDB data and logs.

### `/opt/seaticket/seaticket-data`

`/opt/seaticket/seaticket-data` stores SeaTicket configuration, logs, and persistent application data.

- `conf`: SeaTicket configuration files.
- `logs`: SeaTicket logs, including `seaqa-web.log`.
- `seaqa-web-data`: Avatar files and custom media.

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
