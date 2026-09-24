# GitHub

SeaTicket connects to GitHub to sync issues into a project. Authentication uses a [GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app). SeaTicket signs a JWT with the app's private key and exchanges it for an installation access token.

!!! note "github.com only"
    SeaTicket uses the `github.com` and `api.github.com` hosts, and these cannot be changed. GitHub Enterprise Server (self-hosted) and GitHub Enterprise Cloud with data residency on a `*.ghe.com` domain are therefore not supported.

GitHub is connected per project by a project admin from the **New connection** dialog. Before anyone can connect, a system administrator must create a GitHub App and configure SeaTicket with the app's credentials.

## Prerequisites

- Admin or owner access to the GitHub account or organization whose repositories you want to sync.
- A SeaTicket hostname that is publicly reachable, because GitHub redirects the user's browser back to the setup URL.

## Step 1: Create the GitHub App

1. In GitHub, open **Settings** → **Developer settings** → **GitHub Apps** and click **New GitHub App**. To create an app owned by a personal account, use the [new GitHub App page](https://github.com/settings/apps/new); for an organization, open `https://github.com/organizations/<organization>/settings/apps/new`.
2. Enter a name and your SeaTicket URL as the homepage URL.
3. Enter `https://<seaticket-host>/github/installation-setup/` as the **Redirect URI**.
4. Leave **Request user authorization (OAuth) during installation** unchecked. Enabling it hides the **Setup URL** field that SeaTicket depends on.

!!! note "Expire user authorization tokens"
    SeaTicket authenticates as the app installation rather than on behalf of a user, so this setting has no effect on it. It governs user access tokens only, while installation tokens expire after an hour and are renewed on demand.

## Step 2: Set the setup URL

Under **Post installation**, enter the following as the **Setup URL**:

```
https://<seaticket-host>/github/installation-setup/
```

GitHub redirects users here after they install the app, which is how SeaTicket records the installation. Select **Redirect on update** so the setup URL also runs when repositories are added to or removed from an installation.

## Step 3: Set the permissions

Under **Repository permissions**, grant:

| Permission | Access | Purpose |
| --- | --- | --- |
| **Issues** | Read and write | Read issues and comments, and update them when the AI agent acts on a ticket |
| **Metadata** | Read-only | Required by GitHub for every app |

Under **Organization permissions**, grant:

| Permission | Access | Purpose |
| --- | --- | --- |
| **Issue types** | Read-only | Read organization issue types for mapping them to agent task types |

!!! note "Issue types"
    Reading issue types requires the **Issue types** organization permission above. They are an organization-level feature, so they only exist on repositories owned by an organization.

## Step 4: Create the app

Under **Where can this GitHub App be installed?**, select **Only on this account** if the app will only be installed on the account that owns it, or **Any account** to let other accounts install it. Then click **Create GitHub App**.

## Step 5: Note the App ID and generate a private key

1. On the app's settings page, note the **App ID** (not the Client ID, which SeaTicket does not use).
2. Note the app's URL slug. GitHub derives it from the app name and shows it at `https://github.com/apps/<slug>`; the slug is not always identical to the name, since spaces become hyphens.
3. Under **Private keys**, click **Generate a private key**. GitHub downloads a `.pem` file, and this is the only opportunity to download it.

## Step 6: Configure SeaTicket

### Place the private key

Copy the `.pem` file to the host directory that is already mounted into the `seaqa-web` container:

```bash
cp <downloaded>.private-key.pem /opt/seaticket/seaticket-data/conf/github-app.pem
```

That host directory is mounted at `/opt/seaticket/conf`, so the path inside the container is `/opt/seaticket/conf/github-app.pem`, and no change to the compose volumes is needed.

The file has to be readable by the account the container runs as. Check its owner and permissions after copying it, for example with `ls -l`.

### Set the app credentials

Add the following to your `seaticket_config.yaml`:

```yaml
global:
    GITHUB_APP_NAME: "<your-app-slug>"
    GITHUB_APP_ID: "<your-app-id>"
    GITHUB_PRIVATE_KEY_PATH: "/opt/seaticket/conf/github-app.pem"
```

!!! warning "A wrong private key path fails silently"
    If `GITHUB_PRIVATE_KEY_PATH` does not point to a readable file, SeaTicket starts with an empty private key rather than reporting an error. The mistake surfaces later, when GitHub API calls fail.

## Step 7: Restart SeaTicket

Restart `seaqa-web` for the new settings to take effect:

```bash
docker compose restart seaqa-web
```

!!! note
    If you used environment variables instead, recreate the container with `docker compose up -d seaqa-web` so it picks up the new variables.

## Step 8: Install the app and connect in the UI

1. Sign in as a project admin and open the project's **Connections**.
2. Click **New connection** and choose **GitHub**.
3. Click **Install GitHub app** and, on GitHub, grant access to the repositories you want to sync.
4. Back in SeaTicket, select the **Repository** to sync and save the connection.

## Troubleshooting

- **The install link returns 404.** `GITHUB_APP_NAME` must be the app's URL slug, not its display name. Compare it with the last part of `https://github.com/apps/<slug>`.
- **"The app has not been installed on the sea-ticket."** SeaTicket has no installation record, which means the setup URL callback did not complete. Check that the **Setup URL** points at `https://<seaticket-host>/github/installation-setup/` and is reachable from the internet, then install the app again.
- **Installing the app does not redirect back to SeaTicket.** If the app is already installed on the organization, GitHub skips the setup flow and does not call the Setup URL. Open the installation's settings, change **Repository access** to another option and then back, and save; because **Redirect on update** is enabled, GitHub then sends the setup redirect and SeaTicket records the installation.
- **Syncing fails with a token error.** SeaTicket could not read the private key. Check that `GITHUB_PRIVATE_KEY_PATH` points to a readable `.pem` file inside the container and that `GITHUB_APP_ID` holds the numeric App ID.
- **Issue types are not listed.** Issue types only exist on repositories owned by an organization. Check that the app is granted the **Issue types** organization permission described in [Step 3](#step-3-set-the-permissions).
