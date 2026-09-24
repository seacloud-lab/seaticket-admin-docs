# Gmail

SeaTicket can connect a personal Gmail mailbox to a project using OAuth 2.0. The connection can send email replies and perform mailbox operations through the Gmail API.

!!! note "Personal accounts only"
    The SeaTicket user interface currently supports OAuth connections for **personal** Gmail accounts only. The project admin who creates the connection authorizes their own mailbox.

Before a project admin can connect a mailbox, a system administrator must register an OAuth application with Google and configure SeaTicket with its credentials.

## Prerequisites

- A SeaTicket hostname that is publicly reachable over HTTPS, because Google redirects the user's browser back to SeaTicket.
- Access to [Google Cloud Console](https://console.cloud.google.com/).
- A project admin who can sign in to the personal Gmail mailbox that will be connected.

## Step 1: Create a Google OAuth client

1. In [Google Cloud Console](https://console.cloud.google.com/), create or select a project.
2. Enable the **Gmail API** for the project.
3. Configure the OAuth consent screen. If the application is external, add test users while the application remains in testing.
4. Open **APIs & Services** -> **Credentials** and create an **OAuth client ID** for a **Web application**.
5. Add the following authorized redirect URI:

```
https://<seaticket-host>/api/v1/connections/email/oauth/callback/
```

Replace `<seaticket-host>` with the public SeaTicket URL. SeaTicket derives this URL from `SEAQA_WEB_SERVICE_URL`. It must exactly match the redirect URI registered in Google, including the scheme and trailing slash.

6. Copy the client ID and client secret.

SeaTicket requests these Gmail API scopes for a personal mailbox:

| Scope | Purpose |
| --- | --- |
| `https://www.googleapis.com/auth/gmail.modify` | Read and modify mailbox messages, including moving messages to trash or spam |
| `https://www.googleapis.com/auth/gmail.send` | Send email replies |
| `https://www.googleapis.com/auth/gmail.settings.basic` | Discover the authorized mailbox's sending identity |

Google may require OAuth consent-screen verification before non-test users can authorize scopes that it classifies as sensitive or restricted. Follow the requirements shown in Google Cloud Console for your application.

## Step 2: Configure SeaTicket

Add the following to `seaticket_config.yaml`:

```yaml
global:
    GOOGLE_EMAIL_CLIENT_ID: "<your-google-client-id>"
    GOOGLE_EMAIL_CLIENT_SECRET: "<your-google-client-secret>"
```

## Step 3: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 4: Connect Gmail in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **Email**.
3. Choose **Gmail** as the OAuth provider.
4. Start authorization. SeaTicket opens the Google sign-in page in a popup window.
5. Sign in to the personal Gmail mailbox that this project should use and approve the requested access.
6. When SeaTicket returns to the connection dialog, verify the discovered sender address and save the connection.

!!! note
    The OAuth authorization request and connection creation require project-admin permission. The authorized mailbox is the personal mailbox of the project admin completing this flow.

## Troubleshooting

- **"Email OAuth provider is not configured."** `GOOGLE_EMAIL_CLIENT_ID` or `GOOGLE_EMAIL_CLIENT_SECRET` is missing from the configuration loaded by `seaqa-web`. Check the values and restart the container.
- **The redirect URI does not match.** The redirect URI in Google Cloud Console must exactly match `https://<seaticket-host>/api/v1/connections/email/oauth/callback/`.
- **The authorization popup does not open.** Allow popups for the SeaTicket site in the browser, then start authorization again.
- **The callback reports that the request is missing or expired.** OAuth authorization requests expire after 10 minutes. Start the authorization again and complete it in the same browser session.
- **Google does not return a refresh token.** Confirm that the redirect URI is correct and that the user approved the consent request. Revoke the application's access in the Google account if needed, then authorize again.
- **The sender address cannot be discovered.** Verify that the authorized Google account has a usable Gmail sending identity and granted the required Gmail API permissions.
