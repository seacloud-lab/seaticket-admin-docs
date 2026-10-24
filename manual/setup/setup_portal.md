# Setup Portal

Portal is an optional service. Complete the [standard deployment](setup_seaticket.md) first. The `seaticket-portal` service uses the same image, configuration, and persistent data as `seaticket`, with `SEAQA_APP_MODE=portal`.

## Download the Compose files

```bash
cd /opt/seaticket
wget https://manual.seaticket.ai/1.0/repo/docker/seaticket-portal.yml
wget https://manual.seaticket.ai/1.0/repo/docker/caddy-portal.yml
```

## Configure routing and certificates

Set the following values in `.env`:

```env
PORTAL_SERVICE_ROOT_DOMAIN=portal.example.com
PORTAL_CERTS_VOLUME=/opt/caddy-certs
COMPOSE_FILE='caddy.yml,seadb.yml,seaticket.yml,seasearch.yml,caddy-portal.yml,seaticket-portal.yml'
```

Append the Portal files to your existing `COMPOSE_FILE`; keep `seaticket-notification.yml` if notifications are already enabled.

Point `*.portal.example.com` to the Caddy host and place the wildcard certificate and private key in `PORTAL_CERTS_VOLUME` as `portal-fullchain.pem` and `portal-privkey.pem`. Caddy mounts this directory read-only at `/certs`.

The portal root domain is also an application setting. Set it under the existing `seaqa-web` section of `seaticket_config.yaml` to match `.env`:

```yaml
seaqa-web:
  PORTAL_SERVICE_ROOT_DOMAIN: portal.example.com
```

Custom domains must point to the Caddy host. Caddy requests permission for on-demand TLS through `http://seaticket-portal/internal/portal/custom-domain/allow-tls`; the supplied Compose file configures this automatically. Ports 80 and 443 must be reachable for public traffic and certificate issuance.

## Start Portal

```bash
docker compose up -d
docker compose logs -f seaticket-portal
```

Portal uses the built-in AI and Indexer addresses on `backend-net`; no internal endpoint settings are required. Its YAML configuration section remains `seaqa-web`, not `seaticket-portal`.
