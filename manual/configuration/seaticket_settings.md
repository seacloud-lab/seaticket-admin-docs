# seaticket_settings.py

`seaticket_settings.py` is used to customize the SeaTicket web (`seaticket`) settings.


## Team role permissions

SeaTicket provides built-in **team roles**, and each role has a set of permissions. You can customize these permissions with the `ENABLED_ROLE_PERMISSIONS` setting.

Built-in team roles:

- `free`
- `starter`
- `pro`
- `business`
- `enterprise`

### Available permissions

The following options can be configured per team role:

| Option | Description | Default |
| --- | --- | --- |
| `can_add_project` | Allow users in the role to create projects. | `True` |
| `can_add_group` | Allow users in the role to create groups. | `True` |
| `can_use_saml` | Allow SAML SSO for this role. | `False` for free/starter/pro, `True` for business/enterprise |
| `ai_credit` | AI credits for the team. Set `-1` for unlimited. | `-1` |

### Configuration example

```python
# seaticket_settings.py
ENABLED_ROLE_PERMISSIONS = {
    'free': {
        'can_add_project': True,
        'can_add_group': True,
        'can_use_saml': False,
        'ai_credit': -1,
    },
    'starter': {
        'can_use_saml': False,
        'ai_credit': 2000,
    },
    'pro': {
        'ai_credit': 5000,
    },
    'business': {
        'can_use_saml': True,
    },
    'enterprise': {
        'can_use_saml': True,
    },
}
```

### Merge behavior

SeaTicket merges your custom settings with the defaults:

- If you only override part of a role’s permissions, the missing fields inherit from the default settings.
- If you omit a role entirely, it inherits from `free` defaults.

For example, the above configuration only changes the keys you specify and keeps the rest consistent with the built-in defaults.

## Basic settings

General site metadata and links for the login and registration pages. Email settings are documented in [Sending Email Notifications](sending_email.md).

```python
# Base name used in email sending
SITE_NAME = 'SeaTicket'

# Browser tab's title
SITE_TITLE = 'SeaTicket'

# privacy policy link and service link
PRIVACY_POLICY_LINK = 'https://your-domain/policy'
TERMS_OF_SERVICE_LINK = 'https://your-domain/terms'

# Redirect URL after logout; falls back to the default logout page when empty
LOGOUT_REDIRECT_URL = 'https://example.com/<your-own-logout-page>'

# Custom language code choices (overrides defaults in `settings.py` when needed)
LANGUAGES = (
    ('en', 'English'),
    ('zh-cn', '简体中文'),
    ('zh-tw', '繁體中文'),
)

# Force default language for anonymous users (e.g., 'en', 'zh-cn')
FORCE_DEFAULT_LANGUAGE = ''

# Path to the background image file of login page(relative to the media path)
LOGIN_BG_IMAGE_PATH = 'img/login-bg.jpg'
```

## UI customization

Visual branding and UI assets for the web interface. Default assets use `LOGO_PATH`/`FAVICON_PATH` values, while `CUSTOM_*` paths are optional overrides that only take effect when the corresponding files exist.

### Logo

`LOGO_PATH` points to the **default** logo. If you want to replace it, keep the same key but change the path to your own file.

```python
# Default logo path (relative to media root)
LOGO_PATH = 'img/SeaTicket.png'

# Optional logo size override
LOGO_WIDTH = ''
LOGO_HEIGHT = 32
```

### Favicon

`FAVICON_PATH`, `FAVICON_NOTIFICATION_PATH`, and `APPLE_TOUCH_ICON_PATH` point to the **default** favicon files. To replace them, keep the keys but change the paths to your own assets.

```python
# Default favicon icons (replace with your own files if needed)
FAVICON_PATH = 'favicons/favicon.ico'
FAVICON_NOTIFICATION_PATH = 'favicons/notification-favicon.ico'
APPLE_TOUCH_ICON_PATH = 'favicons/favicon.ico'
```

### Custom asset overrides

`CUSTOM_*` paths are optional overrides that take effect only when the files exist under `seaqa-web-data`. Use them if you prefer to keep defaults untouched and swap assets by placing files in the `custom/` directory.

```python
# Custom branding assets (optional overrides when files exist)
CUSTOM_LOGO_PATH = 'custom/mylogo.png'
CUSTOM_FAVICON_PATH = 'custom/favicon.ico'
CUSTOM_FAVICON_NOTIFICATION_PATH = 'custom/notification-favicon.ico'
CUSTOM_LOGIN_BG_PATH = 'custom/login-bg.jpg'

# Legacy CSS override path
BRANDING_CSS = ''
# Allow uploading branding CSS via admin UI
ENABLE_BRANDING_CSS = False
# Control whether top bar shows logout icon
SHOW_LOGOUT_ICON = False
```

## User management options

Options for registration, activation, and sign-up restrictions. These settings control the sign-up flow and whether admins are notified.

```python
# Allow users to self-register on web UI
ENABLE_SIGNUP = False
# Auto-activate after sign-up
ACTIVATE_AFTER_REGISTRATION = True

# Send activation email when registration completes
REGISTRATION_SEND_MAIL = False
# Notify system admin after registration
NOTIFY_ADMIN_AFTER_REGISTRATION = False
# Require additional profile fields on signup
REQUIRE_DETAIL_ON_REGISTRATION = False

# List of forbidden organization prefixes
REJECT_REGISTRATION_ORG_PREFIX = []
# Regex-based org name blacklist
REJECT_REGISTRATION_ORG_RE_STR = []

# Force users to agree terms on CN deployments
CN_FORCE_USER_AGREE_TERMS = False

# Remember days for login. Default is 7
LOGIN_REMEMBER_DAYS = 7

# Maximum failed login attempts before lock
LOGIN_ATTEMPT_LIMIT = 5
# Lockout window in seconds
LOGIN_ATTEMPT_TIMEOUT = 15 * 60
# Freeze account when exceeding failures
FREEZE_USER_ON_LOGIN_FAILED = False

# minimum length for user's password
USER_PASSWORD_MIN_LENGTH = 6

# LEVEL based on four types of input:
# num, upper letter, lower letter, other symbols
# '3' means password must have at least 3 types of the above.
USER_PASSWORD_STRENGTH_LEVEL = 3

# default False, only check USER_PASSWORD_MIN_LENGTH
# when True, check password strength level, STRONG(or above) is allowed
USER_STRONG_PASSWORD_REQUIRED = False

# Force user to change password when admin adds/resets a user.
FORCE_PASSWORD_CHANGE = True

# Whether to allow SSO users to set a local password; default True, admin or user can set a local password by 'Reset password'
ENABLE_SSO_USER_CHANGE_PASSWORD = True

# Whether to allow LDAP users to set a local password; default False, when True, admin or user can set a local password by 'Reset password'
ENABLE_LDAP_USER_CHANGE_PASSWORD = False

# Whether or not activate inactive user on first login. Mainly used in LDAP user sync.
ACTIVATE_AFTER_FIRST_LOGIN = False

# Whether to allow users to delete their own accounts
ENABLE_DELETE_ACCOUNT = True

# Whether to allow users to update their own information
ENABLE_UPDATE_USER_INFO = True

# Age of cookie, in seconds (example: 2 weeks). Default in core is 1 day.
SESSION_COOKIE_AGE = 60 * 60 * 24
```

## Security settings

Security hardening, throttling, and login protection. Configure captchas, rate limits, and account lockout policies here.

```python
# For security consideration, please set to match the host/domain of your site, e.g., ALLOWED_HOSTS = ['.example.com'].
# Please refer https://docs.djangoproject.com/en/dev/ref/settings/#allowed-hosts for details.
ALLOWED_HOSTS = ['.myseaticket.com']

# Enable slide captcha challenge
ENABLE_SLIDE_CAPTCHA = False
# Custom slide captcha image resource
SLIDE_CAPTCHA_IMAGE_URL = ''

# Enable Cloudflare Turnstile challenge
ENABLE_TURNSTILE = False
# Cloudflare Turnstile site key
TURNSTILE_SITE_KEY = ''
# Cloudflare Turnstile secret key
TURNSTILE_SECRET_KEY = ''

# Request throttle count within period
REQUEST_RATE_LIMIT_NUMBER = 3
# Request throttle period in seconds
REQUEST_RATE_LIMIT_PERIOD = 60
```

## Two-factor authentication

SeaTicket supports Two-Factor Authentication to enhance account security. There are two ways to enable this feature:

1. System admin can enable it in the system settings page.
2. Or add the following settings to `seaticket_settings.py` and restart SeaTicket.

After that, users will see a “Two-Factor Authentication” section in the user profile page and can bind an authenticator app (e.g., Google Authenticator) via QR code.

```python
# Enable TOTP-based two factor
ENABLE_TWO_FACTOR_AUTH = False
# URL to redirect users for OTP setup when 2FA is required but not yet configured (defaults to `/profile/two_factor_authentication/setup/`)
OTP_LOGIN_URL = '/profile/two_factor_authentication/setup/'
# Force every user to enable 2FA
ENABLE_FORCE_2FA_TO_ALL_USERS = False
# Days to remember trusted devices
TWO_FACTOR_DEVICE_REMEMBER_DAYS = 90

# Enable SMS-based two factor auth
ENABLE_SMS_TWO_FACTOR_AUTH = False
# Allow SMS login
ENABLE_SMS_LOGIN = False

# SMS verification code attempt limit
SEND_SMS_ATTEMPT_LIMIT = 5
# SMS verification code attempt timeout in seconds
SEND_SMS_ATTEMPT_TIMEOUT = 60 * 60  # 1h

# Aliyun SMS provider configuration used by registration/SMS login flows
ALIYUN_SMS_CONFIG = {
    'accessKeyId': '',
    'accessKeySecret': '',
    'signName': '',
    'templateCode': '',
    'regionId': 'cn-hangzhou',
}
```

## User profile options


```python
# Allow users to change contact email
ENABLE_USER_SET_CONTACT_EMAIL = False
# Allow users to edit display name
ENABLE_USER_SET_NAME = True
```

## Group and project limits
### Group member limit.

```python
# Max members in an org/group
GROUP_MEMBER_LIMIT = 500
# Max personal groups per user
PERSONAL_GROUP_LIMIT = 500
```

### Project limit

```python
# Max projects for free organizations
FREE_ORG_PROJECT_LIMIT = 500
# Max projects per group
GROUP_PROJECT_LIMIT = 500

# Max personal projects per user
PERSONAL_PROJECT_LIMIT = 500
# Disable creating personal projects
DISABLE_ADDING_PERSONAL_PROJECTS = False
```

## Custom navigation

```python
# Navigation items injected into top menu
CUSTOM_NAV_ITEMS = []
```

## RESTful API
API throttling related settings. Enlarger the rates if you got 429 response code during API calls. API_THROTTLE_RATES is used to replace the old REST_FRAMEWORK option. API_THROTTLE_RATES is empty by default. You can add your custom THROTTLE_RATE to the option

```python
API_THROTTLE_RATES = {
   'ping': '3000/minute',
   'anon': '60/minute',
   'user': '3000/minute',
   'sync_common_dataset': '60/minute',
   'password_reset': '10/minute',
   'org-admin': '1000/day',
   'app': '1000/minute',
   'import': '20/minute',   # Limit the rate of API calls for importing via excel/csv
   'export': '20/minute',   # Limit the rate of export base, table and view
}
```

## Trash cleaning interval

```python
# trash cleanning interval by days
TRASH_CLEAN_AFTER_DAYS = 30
```

## Project settings

Settings for project file upload limits.

```python
# Maximum size for project image uploads (in MB)
PROJECT_IMAGE_MAX_SIZE = 10  # 10MB
# Maximum size for project file uploads (in MB)
PROJECT_FILE_MAX_SIZE = 25  # 25MB
```

## AI chat settings

Settings for AI chat functionality, controlling how many comments are fetched when AI processes tickets and external issues.

```python
# Maximum number of comments to fetch for each ticket in AI chat
AI_CHAT_TICKET_MAX_COMMENTS_NUM = 20
# Maximum number of comments to fetch for each GitHub issue in AI chat
AI_CHAT_GITHUB_ISSUE_MAX_COMMENTS_NUM = 20
```
