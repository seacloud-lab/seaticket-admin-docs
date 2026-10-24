# SeaSearch

The standard [SeaTicket deployment](setup_seaticket.md) includes SeaSearch on the internal Docker network at `http://seasearch:4080`. The `.env` template provides a dedicated `seaticket` account, a pre-generated random password, and the matching API token. No SeaSearch configuration is required for a standard deployment, and this account is independent of the first SeaTicket team administrator.

!!! warning "Internal service credentials"

    The credentials in the downloadable template are public and shared by deployments that keep the defaults. SeaSearch does not publish a host port in the standard Compose configuration. Do not expose it using these credentials. If you make SeaSearch accessible outside the deployment, use unique credentials and appropriate network access controls. The Base64 token encodes the credentials; it does not encrypt them.

## Configuration variables

Set these variables in `/opt/seaticket/.env`:

| Variable | Description | Template default |
| --- | --- | --- |
| `SEASEARCH_URL` | SeaSearch URL reachable from SeaTicket services. | `http://seasearch:4080` |
| `INIT_SS_ADMIN_USER` | Initial SeaSearch administrator username. | `seaticket` |
| `INIT_SS_ADMIN_PASSWORD` | Initial SeaSearch administrator password. | Pre-generated random password in `.env` |
| `SEASEARCH_TOKEN` | Base64 authorization token for the SeaSearch API, generated from `username:password`. | Preconfigured to match the template account |

Compose maps `INIT_SS_ADMIN_USER` and `INIT_SS_ADMIN_PASSWORD` to `SS_FIRST_ADMIN_USER` and `SS_FIRST_ADMIN_PASSWORD` in the SeaSearch container. It does not fall back to `INIT_SEATICKET_TEAM_ADMIN_EMAIL` or `INIT_SEATICKET_TEAM_ADMIN_PASSWORD`.

## Customize credentials

Before the first startup, set `INIT_SS_ADMIN_USER` and `INIT_SS_ADMIN_PASSWORD` to your chosen account credentials. Generate the matching token:

```bash
printf '%s' '<username>:<password>' | base64 | tr -d '\n'
```

Replace the placeholders with the actual values and assign the output to `SEASEARCH_TOKEN`. Do not add the `Basic` prefix. The token must always match the account that SeaTicket uses to access SeaSearch.

!!! note "Existing deployments"

    The initialization variables create the first SeaSearch account; changing them does not change an account already stored in persistent SeaSearch data. When upgrading an existing deployment, keep its working SeaSearch credentials and token instead of replacing them with the template defaults. To rotate credentials, change the account in SeaSearch first and then update the matching `.env` values and token.

Keep the SeaSearch credentials and token in `.env`, because the Compose files require them when recreating containers. After changing configuration for an existing deployment, run `docker compose up -d` from `/opt/seaticket` to recreate affected containers.

## Storage

SeaSearch uses local disk by default and persists data under `${SS_DATA_PATH}` on the host (`/opt/seasearch-data` by default). Include this directory in backups.

`SS_STORAGE_TYPE` can explicitly select `disk` or `s3`. If unset or empty, it follows `ATTACHMENT_STORAGE_BACKEND`, then falls back to `disk`. For S3 buckets, credentials, and mixed storage configurations, see [S3 Storage](setup_s3.md).
