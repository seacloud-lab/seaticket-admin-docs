# Microsoft 365 and Outlook

SeaTicket connects Microsoft 365 and Outlook mailboxes, aliases, and Microsoft 365 shared mailboxes to projects using OAuth 2.0. Connections can send email replies and perform mailbox operations through Microsoft Graph.

By default, use the unified OAuth client: a system administrator does not need to register a Microsoft Entra application or configure a client ID and secret. Deployments that need their own application identity can [use their own OAuth client](#use-your-own-oauth-client).

## Supported connection types

| Type | Authorization and behavior |
| --- | --- |
| Personal | Authorize a user with a Microsoft 365 or Outlook mailbox. This connection type is not limited to consumer Microsoft accounts; organization user mailboxes are also supported. |
| Alias | Authorize the user who owns the mailbox and use an alias that the mailbox is permitted to send from. An alias is not a separate mailbox or sign-in account. |
| Shared mailbox | Authorize a Microsoft 365 user who has access to the shared mailbox, then specify the shared mailbox address. Do not sign in directly as the shared mailbox. This type is available for Microsoft 365, not consumer Outlook accounts. |

## Prerequisites

- A SeaTicket hostname that is publicly reachable over HTTPS, because Microsoft redirects the user's browser back to SeaTicket.
- A project admin who can complete authorization with a Microsoft account that has access to the intended mailbox.
- For an alias, the address must belong to the authorized mailbox, and the organization's Exchange Online settings must permit sending from aliases where applicable.
- For a shared mailbox, an organization administrator must grant the authorizing user the required mailbox access and sending rights, such as **Full Access** and **Send As**, or **Send on Behalf** where appropriate. OAuth consent alone does not grant these rights.
- Access to the [Microsoft Entra admin center](https://entra.microsoft.com/) only if you choose to manage your own OAuth client or need tenant-admin approval.

## Connect using the unified client

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection**, choose **Email**, and select **Microsoft** as the provider.
3. Choose the connection type: personal mailbox, alias, or shared mailbox.
4. Start OAuth authorization. For a personal mailbox or alias, sign in as the mailbox owner. For a shared mailbox, sign in as a user with delegated access to that mailbox.
5. Review the requested access and approve it. Your organization's tenant policies may require administrator consent.
6. Back in the connection dialog, verify the mailbox and sender address. For an alias, specify the permitted alias; for a shared mailbox, specify its address. Save the connection.

Project-admin permission is required to create the connection. The authorizing Microsoft account does not have to use the same address as the SeaTicket administrator's login.

!!! note "Shared mailbox permissions"
    Shared-mailbox access uses delegated user authorization. The authorizing user must have mailbox permissions in Exchange Online as well as the required Microsoft Graph consent. An alias does not have its own mailbox; a shared mailbox does. See Microsoft's guidance on [shared and delegated mail access](https://learn.microsoft.com/en-us/graph/outlook-share-messages-folders) and [sending mail from another user](https://learn.microsoft.com/en-us/graph/outlook-send-mail-from-other-user).

## Use your own OAuth client

This is optional. A self-managed client may be appropriate for:

- **Product branding and trust:** manage your own application identity, publisher information, and consent experience for a commercial service.
- **Internal deployments:** register an application restricted to the intended organization and manage it through the tenant's consent and access policies. Administrator approval may still be required.
- **Security and compliance:** control the app registration, credentials, permissions, and lifecycle under your organization's policies. A self-managed client alone does not guarantee a particular token-routing or data-residency arrangement.
- **Independent application administration:** manage usage and service limits separately where Microsoft applies app-specific limits. Microsoft Graph also applies user-, tenant-, and service-specific throttling; a separate client does not remove those limits.

For details, see Microsoft's [application registration guidance](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app), [consent framework](https://learn.microsoft.com/en-us/entra/identity-platform/consent-framework), and [Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling).

### Step 1: Register a Microsoft Entra application

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
| `Mail.ReadWrite.Shared` | Read and manage shared-mailbox messages when using shared-mailbox connections |
| `Mail.Send.Shared` | Send mail from a shared mailbox when using shared-mailbox connections |

7. Grant tenant admin consent if your Microsoft Entra tenant requires it.

For user-mailbox authorization, SeaTicket also requests the standard `openid`, `profile`, `email`, and `offline_access` scopes. Shared-mailbox authorization uses the delegated shared-mail permissions and `offline_access`. The latter allows SeaTicket to refresh authorization without requiring the user to sign in again for each operation. Add permissions for the connection types you intend to use; mailbox access and sending rights must also be configured in Exchange Online.

### Step 2: Configure SeaTicket

Merge the following into the existing `global` section of `seaticket_config.yaml`:

```yaml
global:
    MICROSOFT_EMAIL_CLIENT_ID: "<your-microsoft-application-client-id>"
    MICROSOFT_EMAIL_CLIENT_SECRET: "<your-microsoft-client-secret>"
```

### Step 3: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

Then create the connection using the mailbox, alias, or shared-mailbox authorization flow described above, with your self-managed client.

## Troubleshooting

- **"Email OAuth provider is not configured."** If you use a self-managed client, confirm that both `MICROSOFT_EMAIL_CLIENT_ID` and `MICROSOFT_EMAIL_CLIENT_SECRET` are present under `global` in `seaticket_config.yaml`, then restart `seaqa-web`. Adding them to `.env` alone does not configure the container.
- **The redirect URI does not match with your own client.** The redirect URI in Microsoft Entra must exactly match `https://<seaticket-host>/api/v1/connections/email/oauth/callback/`.
- **The authorization popup does not open.** Allow popups for the SeaTicket site in the browser, then start authorization again.
- **The callback reports that the request is missing or expired.** OAuth authorization requests expire after 10 minutes. Start the authorization again and complete it in the same browser session.
- **Microsoft does not return a refresh token.** Confirm that the redirect URI is correct, `offline_access` was approved, and the user approved the consent request. Revoke the application's access if needed, then authorize again.
- **The sender address cannot be discovered.** Verify that the authorized account has a usable Microsoft 365 or Outlook mailbox and granted the required Microsoft Graph permissions.
- **A shared mailbox cannot be accessed or cannot send replies.** Confirm that authorization was completed by a user with access to the shared mailbox, the shared mailbox address is correct, and both delegated Graph permissions and Exchange Online mailbox/sending rights have been granted.
- **An alias cannot send replies.** Check that the alias belongs to the authorized mailbox and that sending from aliases is enabled where required by the organization's Exchange Online configuration.
- **Administrator approval is required.** Ask the tenant administrator to review the application and requested permissions. Using a unified or self-managed client does not bypass tenant consent policies.
