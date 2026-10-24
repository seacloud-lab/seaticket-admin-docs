# Gmail

SeaTicket connects Gmail mailboxes and aliases to projects using OAuth 2.0. Connections can send email replies and perform mailbox operations through the Gmail API. Both Gmail accounts and Google Workspace users with Gmail enabled can authorize a mailbox.

By default, use the unified OAuth client: a system administrator does not need to register a Google application or configure a client ID and secret. Deployments that need their own application identity can [use their own OAuth client](#use-your-own-oauth-client).

## Supported connection types

| Type | Authorization and behavior |
| --- | --- |
| Personal | Authorize the Google account whose Gmail mailbox the project should use. This can be a Gmail account or a Google Workspace user account. |
| Alias | Authorize the account that owns the mailbox, then use an alias configured and permitted as a sending identity for that mailbox. An alias is not a separate sign-in account or mailbox. |

!!! note "Gmail service accounts are not supported"
    Gmail connections use user-delegated OAuth authorization. Service-account authentication, including domain-wide delegation, is not supported. There is no separate Gmail shared-mailbox connection type; shared-mailbox connections are available for Microsoft 365 only.

## Prerequisites

- A SeaTicket hostname that is publicly reachable over HTTPS, because Google redirects the user's browser back to SeaTicket.
- A project admin who can complete authorization with a Google account that has access to the intended Gmail mailbox.
- For an alias, configure the address as an allowed sending identity in Gmail before connecting it. See [Send emails from a different address or alias](https://support.google.com/mail/answer/22370).
- Access to [Google Cloud Console](https://console.cloud.google.com/) only if you choose to manage your own OAuth client.

## Connect Gmail using the unified client

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection**, choose **Email**, and select **Gmail** as the provider.
3. Choose a personal-mailbox or alias connection, as appropriate.
4. Start OAuth authorization and sign in to the Google account that owns the mailbox. For an alias, authorize its underlying mailbox account, not the alias address as a separate account.
5. Review the requested access and approve it. Your organization's Google Workspace policies may require administrator approval.
6. Back in the connection dialog, verify the mailbox and sender address. For an alias connection, specify the configured alias as the sender, then save the connection.

Project-admin permission is required to create the connection; it does not mean the authorized Google account must be the same address as the SeaTicket administrator's login.

## Use your own OAuth client

This is optional. A self-managed client may be appropriate when you need:

- **Your own product branding:** use an application name, logo, and policy links that identify your business on Google's consent screen, especially for a public commercial service.
- **An organization-internal application:** eligible Google Workspace organizations can restrict the app to internal users. Internal-only apps may qualify for verification exceptions; this does not remove OAuth consent or organization access-policy requirements.
- **Independent security and compliance administration:** control the app registration, credentials, permissions, and lifecycle under your organization's policies. A self-managed client alone does not guarantee a particular token-routing or data-residency arrangement.
- **Independent API quota management:** manage the Google Cloud project and its project-level quotas separately rather than sharing a client project's quota. Per-user and other Gmail API limits still apply.

Review Google's [OAuth verification requirements and exceptions](https://support.google.com/cloud/answer/9110914) and [Gmail API usage limits](https://developers.google.com/workspace/gmail/api/reference/quota). External apps requesting restricted scopes may require verification and, where applicable, a security assessment. Internal status is not a blanket exemption from all security requirements.

### Step 1: Create a Google OAuth client

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

Gmail mailbox access uses the following scopes:

| Scope | Purpose |
| --- | --- |
| `https://www.googleapis.com/auth/gmail.modify` | Read and modify mailbox messages, including moving messages to trash or spam |
| `https://www.googleapis.com/auth/gmail.send` | Send email replies |
| `https://www.googleapis.com/auth/gmail.settings.basic` | Discover mailbox sending identities, including configured aliases |

Google may require OAuth consent-screen verification before non-test users can authorize scopes that it classifies as sensitive or restricted. Follow the requirements shown in Google Cloud Console for your application.

### Step 2: Configure SeaTicket

Merge the following into the existing `global` section of `seaticket_config.yaml`:

```yaml
global:
    GOOGLE_EMAIL_CLIENT_ID: "<your-google-client-id>"
    GOOGLE_EMAIL_CLIENT_SECRET: "<your-google-client-secret>"
```

### Step 3: Restart SeaTicket

Restart `seaticket` for the new settings to take effect:

```bash
docker compose restart seaticket
```

Then create the connection using the mailbox or alias authorization flow described above, with your self-managed client.

## Troubleshooting

- **"Email OAuth provider is not configured."** If you use a self-managed client, confirm that both `GOOGLE_EMAIL_CLIENT_ID` and `GOOGLE_EMAIL_CLIENT_SECRET` are present under `global` in `seaticket_config.yaml`, then restart `seaticket`.
- **The redirect URI does not match with your own client.** The redirect URI in Google Cloud Console must exactly match `https://<seaticket-host>/api/v1/connections/email/oauth/callback/`.
- **The authorization popup does not open.** Allow popups for the SeaTicket site in the browser, then start authorization again.
- **The callback reports that the request is missing or expired.** OAuth authorization requests expire after 10 minutes. Start the authorization again and complete it in the same browser session.
- **Google does not return a refresh token.** Confirm that the redirect URI is correct and that the user approved the consent request. Revoke the application's access in the Google account if needed, then authorize again.
- **The sender address cannot be discovered.** Verify that the authorized Google account has a usable Gmail sending identity and granted the required Gmail API permissions.
- **An alias cannot send replies.** Confirm that the address is configured and verified as an allowed sending identity for the authorized mailbox. Authorizing an account does not grant permission to send from an arbitrary address.
- **Authorization is blocked by your organization.** Ask your Google Workspace administrator to review the OAuth app and requested scopes under the organization's access policies. Using an internal client does not bypass those policies.
