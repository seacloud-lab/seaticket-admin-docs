# Jira

SeaTicket connects to Jira to sync issues into a project. Authentication uses [Atlassian OAuth 2.0 (3LO)](https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/) with the authorization code grant.

!!! note "Atlassian Cloud only"
    The Jira integration supports **Atlassian Cloud** only. Jira Server and Jira Data Center are not supported.

Jira is connected per project by a project admin from the **New connection** dialog. Before anyone can connect, a system administrator must register an OAuth app with Atlassian and configure SeaTicket with its credentials.

## Prerequisites

- An Atlassian account with access to the [Atlassian developer console](https://developer.atlassian.com/console/myapps/).
- A SeaTicket hostname that is publicly reachable, because Atlassian redirects the user's browser back to the callback URL.

## Step 1: Create the OAuth app

1. Go to the [Atlassian developer console](https://developer.atlassian.com/console/myapps/) and click **Create** → **OAuth 2.0 integration**.
2. Enter a name for your app, agree to the developer terms, and create it.
3. Open the app's **Settings** and note the **Client ID** and **Secret**, which you will need in [Step 4](#step-4-configure-seaticket).

## Step 2: Set the callback URL

In the app's **OAuth 2.0 (3LO)** settings, add the following as the **Callback URL**:

```
https://<seaticket-host>/jira/oauth/callback/
```

Replace `<seaticket-host>` with your SeaTicket server hostname (the value of `SEATICKET_SERVER_HOSTNAME`). The callback URL must exactly match the `JIRA_REDIRECT_URL` configured in SeaTicket. Atlassian requires HTTPS, except for `localhost` during development.

## Step 3: Set the scopes

In the app's **Permissions** settings, add the **Jira API** and select the following scopes:

| Scope | Purpose |
| --- | --- |
| `read:jira-work` | Read issues and projects |
| `read:jira-user` | Read user information |

SeaTicket also requests `offline_access` automatically when connecting, so it can obtain a refresh token and keep the connection valid. This scope is sent in the authorization request and does not need to be added to the app.

## Step 4: Configure SeaTicket

Provide the app credentials to `seaqa-web` using either the `seaticket_config.yaml` file (recommended) or environment variables.

Add the following to your `seaticket_config.yaml`:

```yaml
global:
    JIRA_CLIENT_ID: "<your-atlassian-client-id>"
    JIRA_CLIENT_SECRET: "<your-atlassian-client-secret>"
    JIRA_REDIRECT_URL: "https://<seaticket-host>/jira/oauth/callback/"
```

## Step 5: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 6: Connect Jira in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **Jira**.
3. Click **Connect Jira** and, when redirected to Atlassian, sign in and approve the access request.
4. After you grant access, pick the Jira **site** and **project** to sync and save the connection.

## Troubleshooting

- **The callback URL does not match.** The value of `JIRA_REDIRECT_URL` must exactly match the Callback URL registered in the Atlassian app, including the scheme and trailing slash.
- **"Jira OAuth settings are invalid."** The three `JIRA_*` settings are empty in the config that `seaqa-web` actually loaded. Check that the credentials are present under the `seaqa-web` section of `seaticket_config.yaml` (or as environment variables) and that the container was restarted.
- **The connection fails after you grant access.** Ensure your SeaTicket hostname is publicly reachable, because Atlassian redirects the browser back to the callback URL.
