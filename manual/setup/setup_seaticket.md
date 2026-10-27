# Setup SeaTicket Server

## Getting started

- This guide uses `/opt/seaticket` as the deployment directory for Docker Compose files and `.env`.
- Keep the default data paths unless you need to customize them. If you use different paths, adjust the commands in this guide accordingly.

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

Components communicate using their built-in Docker service addresses. You do not need to configure internal endpoints for a single-host deployment. Keep the service names unchanged; see [internal server URLs](../configuration/seaticket_yaml.md#internal-server-urls-optional) for custom deployments.

## Configure `.env`

Edit `/opt/seaticket/.env`. For a standard deployment, change only the required settings below and leave the remaining template values at their defaults.

| Variable | Description | Value |
| --- | --- | --- |
| `SEATICKET_SERVER_HOSTNAME` | Public hostname or domain of the SeaTicket server. | Required |
| `JWT_PRIVATE_KEY` | Random string of at least 32 characters. Generate with `pwgen -s 40 1`. | Required |
| `SECRET_KEY` | Random string of at least 32 characters. Back it up and do not change it after deployment. | Required |
| `MYSQL_DB_PASSWORD` | Password for the SeaTicket MariaDB user. | Required |
| `INIT_SEATICKET_MYSQL_ROOT_PASSWORD` | Password for the bundled MariaDB `root` user. | Required on first deployment only |
| `INIT_SEATICKET_TEAM_ADMIN_EMAIL` | Email address for the first SeaTicket team administrator. | Required on first deployment only |
| `INIT_SEATICKET_TEAM_ADMIN_PASSWORD` | Password for the first SeaTicket team administrator. | Required on first deployment only |

The database and related services are initialized automatically. Keep the remaining template values at their defaults; see [Environment Variables](../configuration/environment_variables.md) for additional options.

## Configure `seaticket_config.yaml`

Edit `/opt/seaticket/seaticket-data/conf/seaticket_config.yaml`. Configure at least one valid LLM in `global.LLM_MODELS` and an embedding model in `global.EMBEDDING_MODEL`.

For example, to use OpenAI, replace the file contents with the following and fill in your API key:

```yaml
global:
  LLM_MODELS:
    - type: openai
      model: gpt-4o-mini
      key: <your-openai-api-key>
  EMBEDDING_MODEL:
    type: openai
    model: text-embedding-3-small
    key: <your-openai-api-key>
```

Use model IDs available to your account. For other providers or self-hosted services, see [LLMs configurations](../configuration/seaticket_yaml.md#llms-configurations) and [Embedding model](../configuration/seaticket_yaml.md#embedding-model) for provider-specific settings such as `type` and `url`. All other YAML settings are optional and can use their defaults.

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
