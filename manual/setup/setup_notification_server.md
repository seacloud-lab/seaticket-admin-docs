# Setup Notification Server

`seaqa-notification` is the WebSocket service used to deliver realtime notifications to the browser, including connection syncing events, user notifications.

## Workflow

The notification flow is:

1. `seaqa-indexer` publishes a message to the Redis Pub/Sub channel `connection_sync` after a connection sync finishes.
2. `seaqa-notification` subscribes to `connection_sync`, keeps project WebSocket subscriptions in memory, and forwards the message to clients of the matching `project_uuid`.
3. `seaqa-web` requests `project-notification-jwt-token`, opens a WebSocket connection, subscribes to the current project, and shows a success or failure toast when the message arrives.

The sync notification payload is:

```json
{
  "project_uuid": "d918792d-f3f0-4e62-afe9-9ae0595a145e",
  "connection_id": 25,
  "status": "completed",
  "type": "connection-sync"
}
```

## WebSocket Authentication

When a user enters a project page, `seaqa-web` requests `project-notification-jwt-token` to validate permissions and issue a JWT token.

- The token is valid for 3 days.
- The browser connects to `seaqa-notification` with this token.
- `seaqa-notification` validates the token and stores the WebSocket connection in memory.
- `ExpirationScheduler` tracks token expiration time for each subscribed project.
- When a token expires, `seaqa-notification` sends a `jwt-expired` message to the browser.
- The browser then requests a new token and re-subscribes to the project.

## Docker Deployment

The SeaTicket Docker deployment uses `Caddy`, not `nginx`, as the reverse proxy. For this reason, you do not need to manually edit an `nginx.conf` file when you deploy `seaqa-notification` with the provided Docker Compose files.

### Download the compose file

Download `seaqa-notification.yml` into the same directory as the other SeaTicket compose files:

```bash
cd /opt/seaticket
wget https://manual.seaticket.ai/0.9/repo/docker/seaqa-notification.yml
```

### Modify `.env`

Add the following settings:

```env
ENABLE_NOTIFICATION_SERVER=true
```

`seaqa-web` connects to `/ws` on the current public SeaTicket host. This notification path is fixed and is not configured through `.env` or `seaticket_config.yaml`.

### Update `COMPOSE_FILE`

Add `seaqa-notification.yml` to `COMPOSE_FILE`:

```env
COMPOSE_FILE='caddy.yml,seaqa-web.yml,seaqa-notification.yml'
```

Then start or restart the services:

```bash
docker compose up -d
```

### Reverse proxy behavior

In the Docker example, `seaqa-notification.yml` adds a Caddy path route for `/ws`, and `seaqa-web.yml` continues to serve the rest of the site.

You only need to make sure that:

- `seaqa-notification` can access the same Redis and `JWT_PRIVATE_KEY` as the other SeaTicket services.
