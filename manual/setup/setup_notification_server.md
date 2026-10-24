# Setup Notification Server

`seaticket-notification` is the WebSocket service used to deliver realtime notifications to the browser, including connection syncing events, user notifications.

## Workflow

The notification flow is:

1. `seaticket-indexer` publishes a message to the Redis Pub/Sub channel `connection_sync` after a connection sync finishes.
2. `seaticket-notification` subscribes to `connection_sync`, keeps project WebSocket subscriptions in memory, and forwards the message to clients of the matching `project_uuid`.
3. `seaticket` requests `project-notification-jwt-token`, opens a WebSocket connection, subscribes to the current project, and shows a success or failure toast when the message arrives.

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

When a user enters a project page, `seaticket` requests `project-notification-jwt-token` to validate permissions and issue a JWT token.

- The token is valid for 3 days.
- The browser connects to `seaticket-notification` with this token.
- `seaticket-notification` validates the token and stores the WebSocket connection in memory.
- `ExpirationScheduler` tracks token expiration time for each subscribed project.
- When a token expires, `seaticket-notification` sends a `jwt-expired` message to the browser.
- The browser then requests a new token and re-subscribes to the project.

## Docker Deployment

The SeaTicket Docker deployment uses Caddy as the public reverse proxy (the web image also includes its own internal Nginx). You do not need to manually edit an `nginx.conf` file when you deploy `seaticket-notification` with the provided Docker Compose files.

### Download the compose file

Download `seaticket-notification.yml` into the same directory as the other SeaTicket compose files:

```bash
cd /opt/seaticket
wget https://manual.seaticket.ai/1.0/repo/docker/seaticket-notification.yml
```

### Modify `.env`

Add the following settings:

```env
ENABLE_NOTIFICATION_SERVER=true
```

The browser connects to `/ws` on the current public SeaTicket host. This notification path is fixed and is not configured through `.env` or `seaticket_config.yaml`. Notification uses `http://seaticket/` by default for internal session checks; no internal endpoint configuration is required.

### Update `COMPOSE_FILE`

Append `seaticket-notification.yml` to the existing `COMPOSE_FILE`, keeping all base files and any other enabled optional files:

```env
COMPOSE_FILE='caddy.yml,seadb.yml,seaticket.yml,seasearch.yml,seaticket-notification.yml'
```

Then start or restart the services:

```bash
docker compose up -d
```

### Reverse proxy behavior

In the Docker example, `seaticket-notification.yml` adds a Caddy path route for `/ws`, and `seaticket.yml` continues to serve the rest of the site.

You only need to make sure that:

- `seaticket-notification` can access the same Redis and `JWT_PRIVATE_KEY` as the other SeaTicket services.
