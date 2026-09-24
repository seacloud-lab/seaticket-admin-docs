# Confluence

SeaTicket connects to Confluence to sync pages from selected spaces into a project. Authentication uses [Atlassian OAuth 2.0 (3LO)](https://developer.atlassian.com/cloud/confluence/oauth-2-3lo-apps/) with the authorization code grant.

!!! note "Confluence Cloud only"
    SeaTicket supports **Confluence Cloud** only. Confluence Server and Confluence Data Center are not supported.

Confluence is connected per project by a project admin from the **New connection** dialog. Before anyone can connect, a system administrator must register an OAuth app with Atlassian and configure SeaTicket with its credentials.

## Prerequisites

- An Atlassian account with access to the [Atlassian developer console](https://developer.atlassian.com/console/myapps/).
- A SeaTicket hostname that is publicly reachable, because Atlassian redirects the user's browser back to the callback URL.
- A project admin who has access to the Confluence workspace and spaces to sync.

## Step 1: Create the OAuth app

1. Go to the [Atlassian developer console](https://developer.atlassian.com/console/myapps/) and click **Create** -> **OAuth 2.0 integration**.
2. Enter an app name, choose the **OAuth 2.0 (3LO)** grant type, and create the app.
3. Open the app's **Settings** and note the **Client ID** and **Secret**. You will need them in [Step 4](#step-4-configure-seaticket).

## Step 2: Set the callback URL

1. Open the app's **Authorization** settings.
2. Configure OAuth 2.0 (3LO) and add the following **Callback URL**:

```
https://<seaticket-host>/confluence/oauth/callback/
```

Replace `<seaticket-host>` with your SeaTicket server hostname (the value of `SEATICKET_SERVER_HOSTNAME`). The callback URL must exactly match the `CONFLUENCE_REDIRECT_URL` configured in SeaTicket. Atlassian requires HTTPS, except for `localhost` during development.

## Step 3: Set the scopes

In the app's **Permissions** settings, add the **Confluence API** and select these scopes:

| Scope | Purpose |
| --- | --- |
| `search:confluence` | Search and read Confluence content during synchronization |
| `read:space:confluence` | List accessible spaces |
| `read:confluence-user` | Read user information returned with Confluence content |
| `report:personal-data` | Meet Atlassian's user-data reporting requirement |

SeaTicket also requests `offline_access` when connecting. This allows SeaTicket to refresh its access token and keep the project connection active.

## Step 4: Configure SeaTicket

Provide the app credentials to `seaqa-web` using either the `seaticket_config.yaml` file (recommended) or environment variables.

Add the following to your `seaticket_config.yaml`:

```yaml
global:
    CONFLUENCE_CLIENT_ID: "<your-atlassian-client-id>"
    CONFLUENCE_CLIENT_SECRET: "<your-atlassian-client-secret>"
    CONFLUENCE_REDIRECT_URL: "https://<seaticket-host>/confluence/oauth/callback/"
```

## Step 5: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 6: Connect Confluence in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **Confluence**.
3. Click **Connect Confluence** and, when redirected to Atlassian, sign in and approve the access request.
4. After authorization, select the Confluence **workspace** to sync.
5. Optionally select one or more **spaces**. Leave this empty to sync all spaces that the authorized account can access in the selected workspace.
6. Save the connection.

## Troubleshooting

- **The callback URL does not match.** The value of `CONFLUENCE_REDIRECT_URL` must exactly match the Callback URL in the Atlassian app, including the scheme and trailing slash.
- **"Confluence OAuth settings are invalid."** At least one of the `CONFLUENCE_*` settings is empty in the configuration loaded by `seaqa-web`. Check the credentials and restart the container.
- **No workspace is listed.** Confirm that the authorized Atlassian account can access at least one Confluence Cloud workspace and that the app was granted the required Confluence scopes.
- **A space is missing.** The authorized account must have permission to view that space in Confluence.
- **The connection later stops syncing with an authorization error.** The user may have revoked access, or the Atlassian refresh token may have expired. Reconnect Confluence from the project's **Connections** page.
