# Discord

SeaTicket connects to Discord to sync posts from a forum channel into a project. Authorization uses Discord's OAuth2 flow, which adds the SeaTicket bot to your server.

Discord is connected per project by a project admin from the **New connection** dialog. Before anyone can connect, a system administrator must create a Discord application, add a bot to it, and configure SeaTicket with the credentials.

## Prerequisites

- Manage Server permission on the Discord server you want to sync.
- A SeaTicket hostname that is publicly reachable, because Discord redirects the user's browser back to the redirect URL.
- At least one forum channel on the server. SeaTicket syncs forum channels only; text and voice channels cannot be selected.

## Step 1: Create the Discord application

1. Open the [Discord Developer Portal](https://discord.com/developers/applications) and click **New Application**.
2. Enter a name and create the application.
3. Open the **OAuth2** page and note the **Client ID**. Under **Client Secret**, click **Reset Secret** and copy the value.

You will need both in [Step 4](#step-4-configure-seaticket).

## Step 2: Add a bot and copy its token

1. Open the **Bot** page.
2. If no bot exists yet, click **Add Bot** and confirm.
3. Under **Privileged Gateway Intents**, enable **Message Content Intent**. SeaTicket reads message content through the Discord API, and Discord returns messages with empty content without this intent.
4. Click **Reset Token** and copy the token that Discord shows.

!!! warning "The token is shown only once"
    Copy the bot token as soon as it is displayed. If you lose it, reset the token again; that invalidates the previous one, so update `DISCORD_BOT_TOKEN` and restart SeaTicket afterwards.

## Step 3: Set the redirect URL

Under **OAuth2** → **Redirects**, add the following and save your changes:

```
https://<seaticket-host>/discord/oauth/callback/
```

Replace `<seaticket-host>` with your SeaTicket server hostname (the value of `SEATICKET_SERVER_HOSTNAME`). The URL must exactly match the `DISCORD_REDIRECT_URL` configured in SeaTicket.

## Step 4: Configure SeaTicket

Provide the application credentials to `seaqa-web` using either the `seaticket_config.yaml` file (recommended) or environment variables.

=== "seaticket_config.yaml"

    Add a `seaqa-web` section to your `seaticket_config.yaml`:

    ```yaml
    seaqa-web:
      DISCORD_CLIENT_ID: "<your-client-id>"
      DISCORD_CLIENT_SECRET: "<your-client-secret>"
      DISCORD_BOT_TOKEN: "<your-bot-token>"
      DISCORD_REDIRECT_URL: "https://<seaticket-host>/discord/oauth/callback/"
    ```

=== "Environment variables"

    Add the following to your `.env` file:

    ```env
    DISCORD_CLIENT_ID=<your-client-id>
    DISCORD_CLIENT_SECRET=<your-client-secret>
    DISCORD_BOT_TOKEN=<your-bot-token>
    DISCORD_REDIRECT_URL=https://<seaticket-host>/discord/oauth/callback/
    ```

    !!! note
        Environment variables take precedence over `seaticket_config.yaml`. If the same key is set in both places, the environment variable wins.

## Step 5: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 6: Connect Discord in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **Discord**.
3. Click **Install Discord Bot** and, on Discord, choose the server and authorize the bot.
4. Back in SeaTicket, select the **Channel** to sync and save the connection.

The permissions SeaTicket needs (View Channel and Read Message History) are requested as part of this authorization, so you do not need to set permissions for the app in the Developer Portal.

!!! note "Only forum channels are listed"
    The **Channel** list contains forum channels only. If it is empty, the server has no forum channel, or the bot was not granted access to it.

## Troubleshooting

- **"Discord OAuth settings are invalid."** At least one of `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, and `DISCORD_REDIRECT_URL` is empty when `seaqa-web` reads its configuration. Check the values under the `seaqa-web` section of `seaticket_config.yaml` (or as environment variables) and restart the container.
- **"Invalid Discord OAuth state."** or **"Discord server information was not returned."** The authorization flow was interrupted, or no server was selected. Start the connection again from **Install Discord Bot**.
- **"Failed to authorize Discord."** The Client Secret or the redirect URL does not match what is registered in the Discord application. Check both, then restart `seaqa-web`.
- **The Channel list is empty.** The server has no forum channel, or the bot was not granted access to it. Add a forum channel or install the bot again.
- **Synced posts have no content.** The bot is missing the **Message Content Intent**. Enable it under **Privileged Gateway Intents** on the **Bot** page.
- **Bot token rejected.** SeaTicket returns `Invalid Discord bot token` when `DISCORD_BOT_TOKEN` is wrong or was reset in the Developer Portal. Copy the current token, update the setting, and restart `seaqa-web`.
- **Rate limiting.** Discord returns `Discord API rate limited. Please try again later` when it throttles the request. Wait a moment and try again.
