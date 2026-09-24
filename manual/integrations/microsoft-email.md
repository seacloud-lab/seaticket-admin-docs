# Microsoft 365 and Outlook

SeaTicket can connect a personal Microsoft 365 or Outlook mailbox to a project using OAuth 2.0. The connection can send email replies and perform mailbox operations through Microsoft Graph.

!!! note "Personal accounts only"
    The SeaTicket user interface currently supports OAuth connections for **personal** Microsoft 365 and Outlook accounts only. The project admin who creates the connection authorizes their own mailbox.

Before a project admin can connect a mailbox, a system administrator must register an application in Microsoft Entra ID and configure SeaTicket with its credentials.

## Prerequisites

- A SeaTicket hostname that is publicly reachable over HTTPS, because Microsoft redirects the user's browser back to SeaTicket.
- Access to the [Microsoft Entra admin center](https://entra.microsoft.com/).
- A project admin who can sign in to the personal Microsoft 365 or Outlook mailbox that will be connected.

## Step 1: Register a Microsoft Entra application

1. Open the [Microsoft Entra admin center](https://entra.microsoft.com/) and go to **App registrations**.
2. Click **New registration**, enter an application name, and select the account types your deployment needs.
3. Under **Redirect URI**, select **Web** and enter:

```
https://<seaticket-host>/api/v1/connections/email/oauth/callback/
```

Replace `<seaticket-host>` with the public SeaTicket URL. SeaTicket derives this URL from `SEAQA_WEB_SERVICE_URL`. It must exactly match the redirect URI registered in Microsoft Entra, including the scheme and trailing slash.

4. Create the registration and copy its **Application (client) ID**.
5. Open **Certificates & secrets**, create a client secret, and copy its **Value**. The value is shown only once.
6. Open **API permissions** -> **Add a permission** -> **Microsoft Graph** -> **Delegated permissions**, then add the following permissions:

| Permission | Purpose |
| --- | --- |
| `User.Read` | Discover the authorized mailbox identity |
| `Mail.ReadWrite` | Read and manage mailbox messages |
| `Mail.Send` | Send email replies |

7. Grant tenant admin consent if your Microsoft Entra tenant requires it.

SeaTicket also requests the standard `openid`, `profile`, `email`, and `offline_access` scopes. `offline_access` allows SeaTicket to refresh the authorization token without requiring the project admin to sign in again for each operation.

## Step 2: Configure SeaTicket

Add the following to `seaticket_config.yaml`:

```yaml
global:
    MICROSOFT_EMAIL_CLIENT_ID: "<your-microsoft-application-client-id>"
    MICROSOFT_EMAIL_CLIENT_SECRET: "<your-microsoft-client-secret>"
```

## Step 3: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 4: Connect Microsoft 365 or Outlook in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **Email**.
3. Choose **Microsoft** as the OAuth provider.
4. Start authorization. SeaTicket opens the Microsoft sign-in page in a popup window.
5. Sign in to the personal Microsoft 365 or Outlook mailbox that this project should use and approve the requested access.
6. When SeaTicket returns to the connection dialog, verify the discovered sender address and save the connection.

!!! note
    The OAuth authorization request and connection creation require project-admin permission. The authorized mailbox is the personal mailbox of the project admin completing this flow.

## Troubleshooting

- **"Email OAuth provider is not configured."** `MICROSOFT_EMAIL_CLIENT_ID` or `MICROSOFT_EMAIL_CLIENT_SECRET` is missing from the configuration loaded by `seaqa-web`. Check the values and restart the container.
- **The redirect URI does not match.** The redirect URI in Microsoft Entra must exactly match `https://<seaticket-host>/api/v1/connections/email/oauth/callback/`.
- **The authorization popup does not open.** Allow popups for the SeaTicket site in the browser, then start authorization again.
- **The callback reports that the request is missing or expired.** OAuth authorization requests expire after 10 minutes. Start the authorization again and complete it in the same browser session.
- **Microsoft does not return a refresh token.** Confirm that the redirect URI is correct, `offline_access` was approved, and the user approved the consent request. Revoke the application's access if needed, then authorize again.
- **The sender address cannot be discovered.** Verify that the authorized account has a usable Microsoft 365 or Outlook mailbox and granted the required Microsoft Graph permissions.
