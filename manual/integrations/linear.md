# Linear

SeaTicket connects to Linear to sync issues into a project. Authentication uses Linear's OAuth 2.0 authorization code flow.

Linear is connected per project by a project admin from the **New connection** dialog. Before anyone can connect, a system administrator must register an OAuth application with Linear and configure SeaTicket with its credentials.

## Prerequisites

- Workspace administrator access to a Linear workspace. Only workspace admins can create OAuth applications.
- A SeaTicket hostname that is publicly reachable, because Linear redirects the user's browser back to the callback URL.

## Step 1: Create the OAuth app

1. In Linear, open **Settings** → **API** → **OAuth applications** and create a new application. You can also open the [new application page](https://linear.app/settings/api/applications/new) directly.
2. Fill in the application details and create it in the workspace whose issues you want to sync. A new application is private by default, so it can only be authorized from the workspace it was created in; enable **Public** if you need other workspaces to authorize it.
3. Copy the **Client ID** and **Client Secret** from the application page. You will need them in [Step 3](#step-3-configure-seaticket).

!!! note
    Linear applications do not require you to select scopes. SeaTicket sends no scope parameter when connecting, so Linear grants its default read access, which is sufficient for syncing issues.

!!! note "Choosing a workspace for the application"
    From [Linear's OAuth 2.0 documentation](https://linear.app/developers/oauth-2-0-authentication):

    > It is highly recommended you create a workspace for the purpose of managing the OAuth2 Application, as each admin user will have access.

## Step 2: Set the callback URL

Add the following to the application's **Callback URLs**:

```
https://<seaticket-host>/linear/oauth/callback/
```

Replace `<seaticket-host>` with your SeaTicket server hostname (the value of `SEATICKET_SERVER_HOSTNAME`). The callback URL must exactly match the `LINEAR_REDIRECT_URL` configured in SeaTicket. You can register more than one callback URL; enter each on its own line.

## Step 3: Configure SeaTicket

Provide the application credentials to `seaqa-web` using either the `seaticket_config.yaml` file (recommended) or environment variables.

Add the following to your `seaticket_config.yaml`:

```yaml
global:
    LINEAR_CLIENT_ID: "<your-linear-client-id>"
    LINEAR_CLIENT_SECRET: "<your-linear-client-secret>"
    LINEAR_REDIRECT_URL: "https://<seaticket-host>/linear/oauth/callback/"
```

## Step 4: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 5: Connect Linear in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **Linear**.
3. Click **Connect Linear** and, when redirected to Linear, sign in and approve the access request.
4. After you grant access, select the Linear **Team** to sync and save the connection.

## Troubleshooting

- **The callback URL does not match.** The value of `LINEAR_REDIRECT_URL` must exactly match one of the callback URLs registered in the Linear application, including the scheme and trailing slash.
- **"Linear OAuth settings are invalid."** At least one of the `LINEAR_*` settings is empty when `seaqa-web` reads its configuration. Check that the credentials are present under the `seaqa-web` section of `seaticket_config.yaml` (or set as environment variables) and that the container was restarted.
- **The connection fails after you grant access.** Ensure your SeaTicket hostname is publicly reachable, because Linear redirects the browser back to the callback URL.
- **You cannot select a Team.** SeaTicket only lists teams after Linear has been authorized for the project. Complete the authorization to enable the Team field.
