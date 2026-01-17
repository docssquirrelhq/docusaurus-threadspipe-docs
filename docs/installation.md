---
title: Installation and Setup
slug: installation
description: Guide to installing ThreadsPipe, verifying the setup, configuring a development environment, and obtaining access tokens for the Threads API.
sidebar_label: Installation
sidebar_position: 2
---

# Installation and Setup

This guide covers installing the ThreadsPipe library via pip, verifying the installation, setting up a development environment, and configuring access to the Threads API. ThreadsPipe (version 0.4.5, as defined in `pyproject.toml`) is a Python library for interacting with Instagram's Threads using Meta's official Threads API. It requires Python >=3.8.

The core dependencies include `filetype`, `python-dotenv`, and `colorama`. Optional CLI dependencies add `colorama` for enhanced terminal output.

## Installing via pip

ThreadsPipe is available on PyPI. Install the base library with:

```bash
pip install threadspipepy
```

To include optional CLI dependencies (for the `threadspipepy` command-line tool):

```bash
pip install threadspipepy[cli]
```

This installs the library and its dependencies as specified in `pyproject.toml`:

- **Core dependencies**: `filetype` (for media type detection), `python-dotenv` (for environment variable management), `colorama` (for colored terminal output).
- **CLI extras**: `colorama` (ensures CLI formatting works).

If you're using a virtual environment (recommended), create one first:

```bash
python -m venv threads_env
source threads_env/bin/activate  # On Windows: threads_env\Scripts\activate
pip install threadspipepy[cli]
```

## Verifying Installation

After installation, verify by importing the library in Python:

```python
from threadspipepy import ThreadsPipe

# Check version indirectly via help or attributes
print(ThreadsPipe.__threads_auth_scope__)  # Should print available scopes without errors
```

For CLI verification (if installed with `[cli]`):

```bash
threadspipepy --help
```

This should display the CLI help message, confirming the `threadspipepy` script (defined in `pyproject.toml` under `[project.scripts]`) is available. Example output:

```
ThreadsPipe CLI is a tool to get short and long lived access tokens and to refresh long lived access tokens
usage: threadspipepy [-h] {access_token,refresh_token} ...
```

If you encounter import errors, ensure your Python version is >=3.8 and reinstall if needed.

## Setting Up a Development Environment

For development or contribution:

1. Clone the repository:
   ```bash
   git clone https://github.com/paulosabayomi/ThreadsPipe-py.git
   cd ThreadsPipe-py
   ```

2. Create and activate a virtual environment:
   ```bash
   python -m venv dev_env
   source dev_env/bin/activate  # On Windows: dev_env\Scripts\activate
   ```

3. Install in editable mode with all dependencies:
   ```bash
   pip install -e .[cli]
   ```

   This uses `hatchling` as the build backend (from `pyproject.toml`) and installs dependencies locally. The `-e` flag allows changes to the source code to take effect immediately.

4. (Optional) Install additional tools for development:
   ```bash
   pip install pytest black flake8  # For testing, formatting, and linting
   ```

5. Verify:
   ```python
   from threadspipepy import ThreadsPipe
   print("Installation successful!")
   ```

For more on testing and contribution, see the [Testing & Contribution](/docs/testing-contribution) page.

## Creating a Facebook Developer App for Threads API Access

ThreadsPipe uses Meta's Threads API, which requires a Facebook Developer app. Follow these steps:

1. **Create a Developer Account**:
   - Go to [developers.facebook.com](https://developers.facebook.com/) and log in with your Facebook account.
   - If new, create a developer account by verifying your email/phone.

2. **Create a New App**:
   - Click **My Apps** > **Create App**.
   - Select **Other** > **Business** (or Consumer if personal).
   - Enter app details: Display Name (e.g., "My Threads App"), App Purpose, and your email.
   - Click **Create App** (may require verification).

3. **Add the Threads Product**:
   - In the app dashboard, go to **Add Products to Your App**.
   - Find **Threads API** and click **Set Up**.
   - This enables the Threads API under the Graph API.

4. **Configure Basic Settings**:
   - Navigate to **Settings** > **Basic**.
   - Note your **App ID** and **App Secret** (click **Show** for secret; treat as password).
   - Set **App Domains** if needed (e.g., `localhost` for local dev).
   - Under **Add Platform**, select **Website** and add a site URL (e.g., `http://localhost:3000`).

5. **Set Up Redirect URI**:
   - Go to **Products** > **Threads API** > **Basic Display** (or directly under Settings).
   - In **Valid OAuth Redirect URIs**, add your redirect URI (e.g., `https://example.com/callback` or `http://localhost:8080/auth` for local testing).
   - This URI must match what you use in the authorization flow. Save changes.

6. **App Review (if needed)**:
   - For production, submit for review under **App Review** > **Permissions and Features**.
   - Required permissions: `threads_basic`, `threads_content_publish`, etc. (see scopes below).

Your app is now ready. Reference: [Meta for Developers - Threads API](https://developers.facebook.com/docs/threads).

## Obtaining App ID and Secret

- **App ID**: Found in **Settings** > **Basic** (public).
- **App Secret**: Also in **Settings** > **Basic** (hidden; generate if lost).

Store securely, e.g., in a `.env` file:

```env
THREADS_APP_ID=your_app_id
THREADS_APP_SECRET=your_app_secret
```

Load with `python-dotenv`: `from dotenv import load_dotenv; load_dotenv()`.

## Generating Short/Long-Lived Access Tokens

ThreadsPipe handles OAuth 2.0 for Threads. The flow involves:

1. **Authorization Code (via `get_auth_token` or manual)**: Opens a browser for user consent.
2. **Exchange for Tokens (via `get_access_tokens` or CLI)**: Swap code for short-lived (1 hour) and long-lived (60 days) tokens.
3. **Refresh (via `refresh_token` or CLI)**: Extend long-lived tokens (after 24 hours, before expiry).

Scopes are defined in `ThreadsPipe.__threads_auth_scope__`:

```python
{
    'basic': 'threads_basic',
    'publish': 'threads_content_publish',
    'read_replies': 'threads_read_replies',
    'manage_replies': 'threads_manage_replies',
    'insights': 'threads_manage_insights'
}
```

Use `'all'` for all scopes.

### Using the Library (from `threadspipe.py`)

Initialize `ThreadsPipe` after obtaining tokens (see [Initialization](/docs/initialization)).

#### Step 1: Get Authorization Code

```python
from threadspipepy import ThreadsPipe

# Replace with your values
app_id = "your_app_id"
redirect_uri = "http://localhost:8080/callback"  # Must match app settings

tp = ThreadsPipe(user_id=None, access_token=None)  # Temp instance for auth
tp.get_auth_token(
    app_id=app_id,
    redirect_uri=redirect_uri,
    scope='all',  # Or list: ['basic', 'publish']
    state='optional_csrf_state'  # Optional, prevents CSRF
)
```

- This opens `https://threads.net/oauth/authorize/...` in your browser.
- Grant permissions; you'll be redirected to `redirect_uri?code=AUTH_CODE#_`.
- Extract `auth_code` from the URL (strip `#_` at end). Note: Code is single-use.

#### Step 2: Exchange for Tokens

```python
app_secret = "your_app_secret"
auth_code = "extracted_auth_code_from_redirect"

tokens_response = tp.get_access_tokens(
    app_id=app_id,
    app_secret=app_secret,
    auth_code=auth_code,
    redirect_uri=redirect_uri
)

if "error" in tokens_response:
    print(f"Error: {tokens_response['message']}")
    print(tokens_response["error"])
else:
    print("Tokens obtained:")
    print(tokens_response)  # {'user_id': ..., 'tokens': {'short_lived': ..., 'long_lived': ...}}

# Update instance
user_id = tokens_response['user_id']
long_lived_token = tokens_response['tokens']['long_lived']['access_token']
tp.update_param(access_token=long_lived_token, user_id=user_id)
```

Error handling: If HTTP status >201 (e.g., invalid code), returns `{"message": "Could not generate ... token", "error": response.json()}`.

#### Step 3: Refresh Long-Lived Token

```python
# After 24 hours, before 60 days
refreshed = tp.refresh_token(
    access_token=long_lived_token,
    env_path=".env",  # Optional: Auto-update .env
    env_variable="THREADS_ACCESS_TOKEN"
)

if "error" in refreshed:
    print(f"Refresh error: {refreshed['message']}")
    print(refreshed["error"])
else:
    print("Token refreshed:", refreshed['access_token'])
    tp.update_param(access_token=refreshed['access_token'])
```

Errors return similar dicts (status >201).

For full flow, handle logging: Set `disable_logging=False` in `__init__`.

### Using the CLI (from `threadspipepy/cli.py`)

The CLI wraps token exchange and refresh. No direct auth window; get `auth_code` manually first.

#### Generate Tokens

```bash
threadspipepy access_token \
    --app_id=your_app_id \
    --app_secret=your_app_secret \
    --auth_code=extracted_auth_code \
    --redirect_uri=http://localhost:8080/callback \
    --env_path=.env \
    --env_variable=THREADS_ACCESS_TOKEN
```

- Outputs tokens (user_id, short/long-lived).
- Optionally updates `.env`.
- Errors: Pretty-prints dict and exits (e.g., invalid code).

#### Refresh Token

```bash
threadspipepy refresh_token \
    --access_token=your_long_lived_token \
    --env_path=.env \
    --env_variable=THREADS_ACCESS_TOKEN \
    --auto_mode=true  # Optional: Pull token from .env if not provided
```

- Outputs new token.
- Updates `.env` if specified.
- Errors: Pretty-prints and exits.

For full CLI details, see [CLI Usage](/docs/cli-usage).

## Troubleshooting

- **Invalid Redirect URI**: Ensure it matches exactly in app settings.
- **Token Errors**: Check scopes, app review status. Use long-lived tokens for production.
- **Rate Limits**: API enforces limits; handle via retries in library.
- **Dependencies**: If `colorama` issues on Windows, reinstall.

Next: Learn [Initialization](/docs/initialization) to start using the API.
