# Portal Domain

SeaTicket can expose a project's support portal through either of these domain types:

- A project subdomain managed by the SeaTicket administrator, such as `example.portal-seaticket.com`.
- A custom domain managed by the project administrator, such as `support.yourdomain.com`.

Single-server deployments default to `ENABLE_PORTAL_DOMAIN: false` and serve portals at `/portals/<token>/` on the main SeaTicket site. This mode does not require a separate Portal-mode instance or portal-domain DNS and TLS configuration. Follow this guide to enable separate portal domains explicitly.

This guide uses the following example values:

| Item | Example |
| --- | --- |
| SeaTicket domain | `cloud.seaticket.ai` |
| Portal service root domain | `portal-seaticket.com` |
| Project portal domain | `example.portal-seaticket.com` |
| Project custom domain | `support.yourdomain.com` |
| SeaTicket management upstream | `http://seaqa-web:80` |
| SeaTicket portal upstream | `http://seaqa-web-portal:80` |

!!! important

    Portal domains require `ENABLE_PORTAL_DOMAIN: true` in the `seaqa-web` configuration shared by both Web instances. Setting `PORTAL_SERVICE_ROOT_DOMAIN` or starting a Portal-mode instance alone does not enable portal domains. These settings do not create DNS records, start the Portal-mode service, create reverse proxy routes, or issue TLS certificates. Complete all of the application, DNS, service, ingress, and TLS steps below.

## Deployment architecture

Portal domains must be served by a separate `seaqa-web` instance running in Portal mode:

| Public host | Application mode | Upstream example |
| --- | --- | --- |
| `cloud.seaticket.ai` | Main mode (default) | `seaqa-web:80` |
| `*.portal-seaticket.com` | Portal mode | `seaqa-web-portal:80` |
| Verified custom domains | Portal mode | `seaqa-web-portal:80` |

Portal mode is enabled with the following environment variable:

```env
SEAQA_APP_MODE=portal
```

The Portal-mode instance uses the same SeaTicket image version, database, Redis, SeaDB, S3 storage, configuration file, and shared data as the main instance. Give the Portal instance a different Compose service and container name, and do not replace the main instance.

Download the Portal-mode Web and Caddy extension Compose files into `/opt/seaticket`:

```bash
cd /opt/seaticket
wget https://manual.seaticket.ai/0.9/repo/docker/seaticket/seaqa-web-portal.yml
wget https://manual.seaticket.ai/0.9/repo/docker/seaticket/caddy-portal.yml
```

Add the files to `COMPOSE_FILE` in `/opt/seaticket/.env` and configure the Portal domain values:

```env
COMPOSE_FILE='caddy.yml,caddy-portal.yml,seaqa-web.yml,seaqa-web-portal.yml'
COMPOSE_PATH_SEPARATOR=','

PORTAL_SERVICE_ROOT_DOMAIN=portal-seaticket.com
PORTAL_CERTS_VOLUME=/opt/caddy-certs
```

`seaqa-web-portal.yml` copies the backend, storage, secret, volume, and network settings used by the main Web service and adds `SEAQA_APP_MODE=portal`.

`caddy-portal.yml` does not start a second Caddy container. Docker Compose merges files by service name, so its `caddy` section extends the existing `caddy` service from `caddy.yml`. The base file still owns the Caddy image, ports, Docker socket, persistent data, and network; the Portal extension mounts `PORTAL_CERTS_VOLUME` at `/certs` so the wildcard certificate is available inside the existing Caddy container.

Keeping this mount in a separate Compose file makes Portal support optional. Deployments that do not use Portal domains can continue using only `caddy.yml` without creating a wildcard certificate directory.


## Configure the portal service root domain

Enable portal domains and set the root domain in `/opt/seaticket/seaticket-data/conf/seaticket_config.yaml`, shared by the main and Portal-mode Web instances:

```yaml
seaqa-web:
  ENABLE_PORTAL_DOMAIN: true
  PORTAL_SERVICE_ROOT_DOMAIN: portal-seaticket.com
```

## Install the wildcard certificate

Place one certificate and private key valid for `*.portal-seaticket.com` in the directory configured by `PORTAL_CERTS_VOLUME`:

```bash
mkdir -p /opt/caddy-certs
cp <wildcard-fullchain-file> /opt/caddy-certs/portal-fullchain.pem
cp <wildcard-private-key-file> /opt/caddy-certs/portal-privkey.pem
chmod 600 /opt/caddy-certs/portal-privkey.pem
```

This one wildcard certificate is shared by all Portal subdomains.

Validate the merged Compose configuration, then restart the services

After Caddy loads the `*.portal-seaticket.com` route, new project Portal subdomains reuse the installed wildcard certificate automatically. The deployment administrator only needs to replace this one certificate before it expires and restart Caddy to load the renewed files.

## Configure DNS for project portal domains

Create a wildcard DNS record that points all project subdomains to the public ingress serving SeaTicket.

When the ingress has a public IP address:

| Type | Name | Value |
| --- | --- | --- |
| `A` | `*.portal-seaticket.com` | `<ingress-public-ipv4>` |

When the ingress already has a DNS name:

| Type | Name | Value |
| --- | --- | --- |
| `CNAME` | `*.portal-seaticket.com` | `<ingress-hostname>` |

Do not create both a CNAME and an A record for the same wildcard name.

Verify that the wildcard record resolves before continuing:

```bash
dig +short example.portal-seaticket.com
```

## Configure the ingress and TLS

The public ingress must:

1. Accept `*.portal-seaticket.com` in addition to `cloud.seaticket.ai`.
2. Forward the SeaTicket management domain to the main-mode instance.
3. Forward Portal subdomains and custom domains to the Portal-mode instance.
4. Preserve the original `Host` header so SeaTicket can identify the project portal.
5. Set `X-Forwarded-Proto` to the client-facing protocol.
6. Serve the configured certificate valid for `*.portal-seaticket.com`.

A single wildcard certificate covers all first-level project subdomains under `portal-seaticket.com`. Once the wildcard DNS record and certificate are active, adding a project Portal does not require another DNS record or certificate.

The equivalent Caddy behavior is shown below. Adapt the syntax to the ingress used by the deployment:

```caddyfile
https://*.portal-seaticket.com {
    tls /certs/portal-fullchain.pem /certs/portal-privkey.pem
    reverse_proxy seaqa-web-portal:80
}
```

Caddy preserves the original `Host` header and sets the standard forwarded headers by default. Other ingress implementations must preserve the original `Host` header and set `X-Forwarded-Proto` to the client-facing protocol explicitly when they do not do so automatically.

!!! note "Docker deployment with Caddy"

    The base Docker Compose files route only `SEATICKET_SERVER_HOSTNAME` to the main-mode instance. The additional `seaqa-web-portal.yml` labels route `*.portal-seaticket.com` and verified custom domains to `seaqa-web-portal`. The `caddy_0.tls` label loads the shared wildcard certificate. If TLS is terminated by an external load balancer instead, configure the wildcard certificate and the same routing there, and preserve the original `Host` header through every proxy layer.

## Configure TLS for custom domains

Unlike the Portal service domain, customer-owned domains cannot be covered by one administrator-managed wildcard certificate. The Portal-mode service provides an authorization endpoint for Caddy on-demand TLS. Caddy should call it over the private application network:

```text
http://seaqa-web-portal/internal/portal/custom-domain/allow-tls?domain=<requested-domain>
```

It returns success only when the custom domain is verified and its project Portal is enabled. The labels in `seaqa-web-portal.yml` generate the equivalent of this Caddy behavior:

```caddyfile
{
    on_demand_tls {
        ask http://seaqa-web-portal/internal/portal/custom-domain/allow-tls
    }
}

https:// {
    tls {
        on_demand
    }
    reverse_proxy seaqa-web-portal:80
}
```

The `caddy_0` labels create the explicit Portal route using the shared `*.portal-seaticket.com` certificate. The `caddy_1` labels create the catch-all HTTPS route with on-demand TLS. The global `caddy.on_demand_tls.ask` label restricts certificate issuance to custom domains approved by SeaTicket.

## Expected certificate behavior

After DNS, Caddy, and SeaTicket are configured:

| Domain type | Certificate behavior |
| --- | --- |
| Project Portal subdomains such as `example.portal-seaticket.com` | They reuse the configured `*.portal-seaticket.com` certificate. Creating another project subdomain does not trigger or require separate certificate issuance. |
| Verified custom domains such as `support.yourdomain.com` | They are not covered by the Portal wildcard certificate. On the first TLS connection, Caddy asks SeaTicket whether the domain is authorized and automatically obtains an individual certificate only when the custom domain is verified and its Portal is enabled. |

The shared wildcard certificate is managed by the deployment administrator, while Caddy automatically obtains and renews certificates for approved custom domains. A Portal subdomain is usable after the wildcard DNS record resolves and the shared certificate is installed. A custom domain is usable after its DNS points to the ingress and SeaTicket's authorization endpoint approves it.

Inspect the Caddyfile generated from the Docker labels after deployment:

```bash
docker exec caddy cat /config/caddy/Caddyfile.autosave
```

Confirm that it contains the `*.portal-seaticket.com` site with the wildcard certificate paths, the `on_demand_tls` global option, the SeaTicket `ask` URL, and the catch-all HTTPS site.

!!! warning

    Never enable unrestricted on-demand TLS. Always configure the SeaTicket `ask` endpoint; otherwise arbitrary hostnames can trigger certificate issuance and exhaust CA rate limits.

## Enable and test a project portal

1. Sign in to SeaTicket as a project administrator.
2. Open the project settings and enable **Support portal**.
3. Open the portal settings and select **Portal domain**.
4. Set a unique prefix, such as `example`, and save it.
5. Open `https://example.portal-seaticket.com`.

The prefix must contain 3 to 63 letters, numbers, or hyphens, and cannot begin or end with a hyphen. Reserved prefixes such as `admin`, `api`, `auth`, `static`, `support`, and `www` cannot be used.

You can verify routing from a machine that can reach the ingress:

```bash
curl -I https://example.portal-seaticket.com
```

To test the route before public DNS is available, resolve the domain to the ingress IP only for the curl request:

```bash
curl --resolve example.portal-seaticket.com:443:<ingress-public-ipv4> \
  -I https://example.portal-seaticket.com
```

## Configure a project custom domain

Custom domains are configured by a project administrator after the portal service root domain is working.

1. Open the project's portal settings and select **Custom domain**.
2. Enter the fully qualified domain without a scheme or path, for example `support.yourdomain.com`, and save it.
3. Add the CNAME record displayed by SeaTicket. It points the custom domain to the project's current portal domain:

    | Type | Name | Value |
    | --- | --- | --- |
    | `CNAME` | `support.yourdomain.com` | `example.portal-seaticket.com` |

4. Add the TXT record name and value displayed by SeaTicket. The values are unique to the project and must be copied exactly.
5. Ensure the ingress has already been configured for custom-domain routing and certificate automation.
6. Wait for public DNS propagation, then select **Verify** in SeaTicket.
7. Open `https://support.yourdomain.com` and confirm that it loads the expected portal.

Prefer a subdomain for the custom domain. A DNS zone apex cannot normally use a CNAME record; it requires provider-specific ALIAS, ANAME, or CNAME-flattening support that targets the displayed Portal domain. The custom domain cannot be the SeaTicket management domain, the Portal root domain, or a subdomain below the Portal root domain.

## Deployment checklist

- `ENABLE_PORTAL_DOMAIN: true` and `PORTAL_SERVICE_ROOT_DOMAIN` are set in the `seaqa-web` section of the `seaticket_config.yaml` shared by both Web instances.
- A separate `seaqa-web` instance is running with `SEAQA_APP_MODE=portal`.
- Both Web instances use the same SeaTicket configuration and backend services.
- Both Web instances were restarted after the configuration change.
- `*.portal-seaticket.com` resolves to the public ingress.
- The ingress routes the management domain to main mode and Portal domains to Portal mode.
- The ingress preserves the original `Host` header.
- The TLS certificate covers `*.portal-seaticket.com`.
- The main-mode `seaqa-web` container can query `8.8.8.8` and `1.1.1.1` over UDP and TCP port 53.
- A project portal opens successfully through its generated subdomain.
- Each custom domain has its CNAME, TXT verification record, ingress route, and TLS certificate.

## Upgrade checklist for existing portal-domain deployments

Portal domains now default to disabled. Existing deployments that relied on the previous default must explicitly enable them when upgrading; retaining `PORTAL_SERVICE_ROOT_DOMAIN` and a running Portal-mode instance is not sufficient. Without the switch, SeaTicket uses main-site portal paths, disables portal-domain resolution and custom-domain management, and rejects custom-domain TLS authorization requests.

1. Before upgrading, add `ENABLE_PORTAL_DOMAIN: true` alongside the existing `PORTAL_SERVICE_ROOT_DOMAIN` in the shared `seaqa-web` YAML configuration shown above.
2. Confirm that both Web instances load this configuration, and retain the existing Portal-mode service, DNS, ingress routes, and TLS configuration.
3. Upgrade both Web instances to the same SeaTicket image version and restart both so they load the explicit switch.
4. Verify that a project portal opens through its subdomain, custom-domain management remains available, and the TLS authorization endpoint approves a verified custom domain whose project portal is enabled.
