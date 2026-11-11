# How to upgrade

Upgrading SeaTicket involves updating the Docker Compose files, adjusting any required configuration, pulling the target Docker images, and restarting the services. This guide uses `/opt/seaticket` as the deployment directory, as in [Setup SeaTicket](../setup/setup_seaticket.md).

!!! warning "Back up before upgrading"

    Make a full backup of your databases, persistent data, `.env`, Docker Compose files, and application configuration before upgrading. Keep a record of the currently deployed image versions and any custom changes.

    An upgrade may change the database or stored data. Do not assume that switching back to an older Docker image will safely roll back an upgrade. Schedule a maintenance window, as restarting the containers interrupts service.

## How to update SeaTicket Server

1. Update the `.yml` files.
2. Follow the extra upgrade notices and update the necessary configuration, especially image versions.
3. Apply the changes and verify the services.

### Updating the `.yml` files

Download the latest standard Compose files into your deployment directory:

```bash
cd /opt/seaticket
wget -O caddy.yml https://manual.seaticket.ai/repo/docker/caddy.yml
wget -O seadb.yml https://manual.seaticket.ai/repo/docker/seadb.yml
wget -O seaticket.yml https://manual.seaticket.ai/repo/docker/seaticket.yml
wget -O seasearch.yml https://manual.seaticket.ai/repo/docker/seasearch.yml
```

If you have enabled the notification server, also update its Compose file:

```bash
wget -O seaticket-notification.yml https://manual.seaticket.ai/repo/docker/seaticket-notification.yml
```

!!! warning "The existing `.yml` files will be overwritten"

    These commands replace the named files. Save any custom changes beforehand, then review and reapply them as needed. The commands do not overwrite `.env` or `seaticket-data/conf/seaticket_config.yaml`, and do not yet affect running containers.

### Extra upgrade notices and configuration

Review the [Extra Upgrade Notices](extra-upgrade-notice.md) for the target version and any intervening versions before proceeding. Follow any required upgrade steps, including database changes, at the stage specified in the notices.

Keep your existing `.env` and application configuration. Do not replace them with the setup templates, which would discard your deployment-specific values. Instead, compare them with the current templates and apply only the necessary changes:

- **Image versions:** Update the image variables in `.env` to the versions required by the target release. These include `SEATICKET_WEB_IMAGE`, `SEATICKET_AI_IMAGE`, `SEATICKET_EVENTS_IMAGE`, and `SEATICKET_INDEXER_IMAGE`, plus `SEATICKET_NOTIFICATION_IMAGE` if the notification server is enabled. Review dependency image requirements as well, including SeaDB and SeaSearch; do not assume their version numbers match SeaTicket's.
- **Environment variables:** Add or adjust settings required by the target release. If new Compose files are required, update `COMPOSE_FILE`, retaining any enabled optional files and custom overrides. See [Environment Variables](../configuration/environment_variables.md).
- **Application configuration:** Apply any required changes to `seaticket-data/conf/seaticket_config.yaml` or other configuration files. Preserve your model settings, API keys, credentials, secrets, and custom paths. See [seaticket_config.yaml](../configuration/seaticket_yaml.md).

!!! important "Downloading new `.yml` files does not update image versions set in `.env`"

    An image variable already set in `.env` overrides the corresponding default in the Compose file. Update these variables explicitly when the target release requires new image tags. If the tag remains unchanged, such as `1.0-latest`, you still need to run `docker compose pull` to download the updated image.

### Applying changes

After completing the required configuration changes, validate the Compose configuration and pull the images before stopping the running services:

```bash
cd /opt/seaticket
docker compose config --quiet
docker compose pull
```

Resolve any configuration or image download errors before proceeding. Then restart the services:

```bash
docker compose down
docker compose up -d
```

Do not use `docker compose down -v`, as it removes named volumes and can cause data loss.

Check the container status and startup logs:

```bash
docker compose ps
docker compose logs --follow
```

Once the services are ready, open your SeaTicket URL and verify that you can sign in and access your existing workspaces. Check that syncing, search, and any enabled integrations work as expected.

## Naming conventions and update path

SeaTicket uses the following version naming conventions:

- **1 → 2** is a major version upgrade.
- **1.0 → 1.1** is a minor version upgrade.
- **1.0.1 → 1.0.2** is a patch upgrade.

Before upgrading, check the [Extra Upgrade Notices](extra-upgrade-notice.md) for the target version and any intervening versions to determine whether intermediate upgrades or database updates are required. Do not assume that every older version can be upgraded directly to the latest release, or that restarting the containers automatically applies all database changes. Follow any version-specific instructions before completing the upgrade.
