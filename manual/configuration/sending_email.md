# Email Sending 

SeaTicket requires an external SMTP account to send emails. Typical emails include:

- User password resets
- User invitations and registration notices
- System admin adds new members
- System admin resets user password

## Configuration

There are two ways to configure SMTP for system-wide emails.

=== "Environment variables / seaticket_config.yaml"

    SeaTicket reads SMTP settings from environment variables or the `seaqa-web` configuration section in `seaticket_config.yaml` for the `seaticket` service.

    | Key | Description | Example values |
    | --- | --- | --- |
    | `EMAIL_HOST` | SMTP server hostname | `smtp.example.com` |
    | `EMAIL_HOST_USER` | Username for authentication | `username@example.com` |
    | `EMAIL_HOST_PASSWORD` | Password for authentication | `password` |
    | `EMAIL_PORT` | SMTP server port | `25`, `465`, or `587` |
    | `DEFAULT_FROM_EMAIL` | Default From header | `noreply@example.com` |
    | `SERVER_EMAIL` | Sender for error reporting | `noreply@example.com` |

    !!! note
        `EMAIL_USE_TLS` (and any SSL flag) cannot be set via environment variables or `seaticket_config.yaml`. Use `seaqa_web_settings.py` instead.

=== "Configuration file"

    Add the following lines to `seaqa_web_settings.py`:

    ```python
    EMAIL_USE_TLS = False
    EMAIL_HOST = 'smtp.example.com'
    EMAIL_HOST_USER = 'username@example.com'
    EMAIL_HOST_PASSWORD = 'password'
    EMAIL_PORT = 25
    DEFAULT_FROM_EMAIL = EMAIL_HOST_USER
    SERVER_EMAIL = EMAIL_HOST_USER
    ```

!!! warning "SMTP without authentication"
    If you want to use the email service without authentication, leave `EMAIL_HOST_USER` and `EMAIL_HOST_PASSWORD` **blank** (`''`).

!!! warning "TLS only"
    SeaTicket only exposes the `EMAIL_USE_TLS` flag. SSL-specific toggles are not supported; use port `465` with TLS disabled if your provider requires implicit SSL.

Restart SeaTicket after updating SMTP configuration.

## Debugging

If email sending does not work, check `seaticket` logs for SMTP errors. You can also use a tool like [Swaks](https://github.com/jetmore/swaks) to verify SMTP settings.

```bash
swaks --auth -tls \
  --server <EMAIL_HOST> \
  --protocol SMTP \
  --port <EMAIL_PORT> \
  --au <EMAIL_HOST_USER> \
  --ap <EMAIL_HOST_PASSWORD> \
  --from <DEFAULT_FROM_EMAIL> \
  --to <TO_ADDRESS> \
  --h-Subject: "Subject of the mail" \
  --body 'Test email!'
```

## Examples

### Gmail

If you are using Gmail as email server, you can use the following settings.

=== "Environment variables / seaticket_config.yaml"

    ```python
    EMAIL_USE_TLS = True
    EMAIL_HOST = 'smtp.gmail.com'
    EMAIL_HOST_USER = 'username@gmail.com'
    EMAIL_HOST_PASSWORD = 'password'
    EMAIL_PORT = 587
    DEFAULT_FROM_EMAIL = 'username@gmail.com'
    SERVER_EMAIL = 'username@gmail.com'
    ```

=== "Configuration file"

    ```python
    EMAIL_USE_TLS = True
    EMAIL_HOST = 'smtp.gmail.com'
    EMAIL_HOST_USER = 'username@gmail.com'
    EMAIL_HOST_PASSWORD = 'password'
    EMAIL_PORT = 587
    DEFAULT_FROM_EMAIL = 'username@gmail.com'
    SERVER_EMAIL = 'username@gmail.com'
    ```

!!! warning "Allow access to less secure apps"
    Google blocks external apps by default. If you enable 2-step verification, you need an [App Password](https://support.google.com/accounts/answer/185833). Otherwise, enable [Less Secure Apps](https://support.google.com/accounts/answer/6010255).

### Brevo

SeaTicket Cloud can use the SMTP relay of Brevo.

=== "Environment variables / seaticket_config.yaml"

    ```python
    EMAIL_USE_TLS = True
    EMAIL_HOST = 'smtp-relay.sendinblue.com'
    EMAIL_HOST_USER = 'username@domain.com'
    EMAIL_HOST_PASSWORD = 'xsmtpsib-xxx'
    EMAIL_PORT = 587
    DEFAULT_FROM_EMAIL = 'SeaTicket <noreply@domain.com>'
    SERVER_EMAIL = 'noreply@domain.com'
    ```

=== "Configuration file"

    ```python
    EMAIL_USE_TLS = True
    EMAIL_HOST = 'smtp-relay.sendinblue.com'
    EMAIL_HOST_USER = 'username@domain.com'
    EMAIL_HOST_PASSWORD = 'xsmtpsib-xxx'
    EMAIL_PORT = 587
    DEFAULT_FROM_EMAIL = 'SeaTicket <noreply@domain.com>'
    SERVER_EMAIL = 'noreply@domain.com'
    ```

### Infomaniak

If you are using Infomaniak as email server, you can use the following settings.

=== "Environment variables / seaticket_config.yaml"

    !!! warning "SSL support missing"

        Infomaniak requires SSL instead of TLS, therefore a configuration via TLS only is not possible. Please use the configuration file approach instead.

=== "Configuration file"

    ```python
    EMAIL_USE_TLS = False
    EMAIL_USE_SSL = True
    EMAIL_HOST = 'mail.infomaniak.com'
    EMAIL_HOST_USER = 'username@domain.com'
    EMAIL_HOST_PASSWORD = 'password'
    EMAIL_PORT = 465
    DEFAULT_FROM_EMAIL = 'SeaTicket <noreply@domain.com>'
    SERVER_EMAIL = 'noreply@domain.com'
    ```
