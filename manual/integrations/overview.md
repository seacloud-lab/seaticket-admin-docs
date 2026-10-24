# Integrations

SeaTicket can connect to third-party services and sync their content into projects. Integrations use OAuth or provider-specific application authentication. Register an app with the service provider and configure its credentials as described on the integration's page. Email connections can instead use the unified OAuth client without registering your own app, or an optional client that you manage.

Configure your own integration credentials in the `global` section of `seaticket_config.yaml`. Merge settings into the existing `global` section rather than creating a duplicate YAML key. See each integration's page for the required settings and services to restart.

| Integration | Description |
| --- | --- |
| [Jira](jira.md) | Sync Jira issues using Atlassian OAuth 2.0 (3LO) |
| [Linear](linear.md) | Sync Linear issues using OAuth 2.0 |
| [GitHub](github.md) | Sync GitHub issues using a GitHub App |
| [Confluence](confluence.md) | Sync Confluence Cloud pages using Atlassian OAuth 2.0 (3LO) |
| [Discord](discord.md) | Sync Discord forum posts using an OAuth2 bot |
| [Gmail](gmail.md) | Connect a Gmail mailbox or alias using a unified or self-managed OAuth client |
| [Microsoft 365 and Outlook](microsoft-email.md) | Connect a mailbox, alias, or Microsoft 365 shared mailbox using a unified or self-managed OAuth client |
