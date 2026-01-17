---
title: CLI Usage
slug: cli-usage
description: A comprehensive guide to using the ThreadsPipe CLI for generating, exchanging, and refreshing access tokens for the Threads API.
sidebar_label: CLI Usage
sidebar_position: 8
---

# CLI Usage

The ThreadsPipe CLI provides a command-line interface to simplify the process of obtaining and managing access tokens for the Threads API. It allows you to generate short-lived and long-lived access tokens from an authorization code and refresh long-lived tokens before they expire. This tool is particularly useful for developers who need to handle authentication without writing custom scripts.

The CLI is built on top of the ThreadsPipe library and uses the same underlying API calls for token management. It integrates seamlessly with `.env` files for storing tokens securely and supports silent operation for automated workflows.

For installation instructions, refer to the [Installation](../installation.md) guide. To enable CLI features, install the optional dependencies:

```bash
pip install threadspipepy[cli]
```

This ensures `colorama` is available for colored output in the terminal.

## Getting Started

Run the CLI without arguments to see the help menu:

```bash
threadspipepy -h
```

This displays the available actions and options with colored descriptions for clarity.

The CLI supports two main actions:
- `access_token`: Generate short-lived and long-lived access tokens from an authorization code.
- `refresh_token`: Refresh an existing long-lived access token to extend its validity.

All commands use `argparse` for argument parsing, with colored help text via `colorama`. Logging is enabled by default at DEBUG level, but can be disabled with the `--silent` flag.

## Generating Tokens

### access_token Command

The `access_token` action exchanges an authorization code (obtained from the Threads authorization window) for a short-lived token (valid for 1 hour) and then for a long-lived token (valid for 60 days). This process mirrors the library's `get_access_tokens` method.

#### Required Arguments
- `--app_id` (`-id`): Your Threads app ID from the Meta developer dashboard.
- `--app_secret` (`-secret`): Your app secret from the dashboard's "Use cases > Customize > Settings" page.
- `--auth_code` (`-code`): The one-time authorization code from the redirect URI after completing the authorization flow. This code expires after a single use.
- `--redirect_uri` (`-r`): The exact redirect URI used in the authorization request (e.g., `https://example.com/redirect`). Mismatches will invalidate the auth code.

#### Optional Arguments
- `--env_path` (`-p`): Path to a `.env` file (e.g., `./.env`). If provided with `--env_variable`, the long-lived token will be automatically saved to this file.
- `--env_variable` (`-v`): The name of the environment variable in the `.env` file to store the long-lived token (e.g., `THREADS_ACCESS_TOKEN`). Requires `--env_path`.
- `--silent` (`-s`): Disables logging output. Pass with or without a value (e.g., `-s` or `-s true`).

#### Process
1. A POST request is made to `https://graph.threads.net/oauth/access_token` with `grant_type=authorization_code` to obtain the short-lived token.
2. A GET request is made to `https://graph.threads.net/access_token` with `grant_type=th_exchange_token` to exchange it for the long-lived token.
3. If `--env_path` and `--env_variable` are specified, the long-lived token is written to the `.env` file using `python-dotenv`.
4. The output includes the user ID, short-lived token details, and long-lived token details (pretty-printed via `pprint`).

If the `.env` file does not exist, it will be created. The CLI changes to the current working directory before updating to ensure path reliability.

#### Example
To generate tokens and save the long-lived one to a `.env` file:

```bash
threadspipepy access_token \
  --app_id=123456789 \
  --app_secret=your_app_secret \
  --auth_code=ABC123def456GHI789 \
  --redirect_uri=https://example.com/redirect \
  --env_path=./.env \
  --env_variable=THREADS_ACCESS_TOKEN
```

Output (example):
```
{
    'user_id': '17841401234567890',
    'tokens': {
        'short_lived': {
            'access_token': 'short-lived-token-here',
            'expires_in': 3600,
            'user_id': '17841401234567890'
        },
        'long_lived': {
            'access_token': 'long-lived-token-here',
            'expires_in': 5184000
        }
    }
}
```

Updated `.env` file:
```
THREADS_ACCESS_TOKEN=long-lived-token-here
```

In silent mode:
```bash
threadspipepy access_token ... -s
```
No logging or pretty-print output; the process runs quietly, only updating the `.env` if specified.

## Refreshing Tokens

### refresh_token Command

Long-lived tokens expire after 60 days but can be refreshed anytime after they are at least 24 hours old. This action sends a GET request to `https://graph.threads.net/refresh_access_token` with `grant_type=th_refresh_token` to obtain a new long-lived token with an extended validity period.

#### Required Arguments (Conditional)
- `--access_token` (`-token`): The existing long-lived token to refresh. Required unless `--auto_mode` is enabled.

#### Optional Arguments
- `--auto_mode` (`-auto`): Set to `true` to automatically load the token from the `.env` file (using `--env_path` and `--env_variable`) and update it after refresh. Omits the need for `--access_token`.
- `--env_path` (`-p`): Path to the `.env` file. Required for `--auto_mode` or auto-updating.
- `--env_variable` (`-v`): Environment variable name for the token. Required for `--auto_mode` or auto-updating.
- `--silent` (`-s`): Disables logging, as described above.

#### Process
1. If `--auto_mode=true`, the token is fetched from the `.env` file using `get_key` from `python-dotenv`.
2. The refresh request is made to the Threads API.
3. If successful and `--env_path`/`--env_variable` are provided, the new token updates the `.env` file.
4. The new long-lived token details are pretty-printed.

:::note
The Threads API enforces the 24-hour minimum age and 60-day expiration. Attempting to refresh too early or an expired token will result in an API error.
:::

#### Examples

Basic refresh with manual token:
```bash
threadspipepy refresh_token \
  --access_token=your-long-lived-token \
  --env_path=./.env \
  --env_variable=THREADS_ACCESS_TOKEN
```

Auto-mode refresh (loads and updates from `.env`):
```bash
threadspipepy refresh_token \
  --auto_mode=true \
  --env_path=./.env \
  --env_variable=THREADS_ACCESS_TOKEN
```

Silent refresh:
```bash
threadspipepy refresh_token ... -s
```

Output (example):
```
{
    'access_token': 'new-long-lived-token-here',
    'expires_in': 5184000,
    'user_id': '17841401234567890'
}
```

## Updating .env Files

The CLI uses `python-dotenv` to read from and write to `.env` files. Provide `--env_path` (relative or absolute path) and `--env_variable` (e.g., `THREADS_ACCESS_TOKEN`) to enable automatic storage:

- For `access_token`: Saves the initial long-lived token.
- For `refresh_token`: Loads the token (in auto-mode) and saves the refreshed one.

If the file doesn't exist, it's created in the specified path. Always pair these arguments; the CLI validates and errors if one is missing.

Example `.env` integration in a workflow:
1. Generate tokens and save to `.env`.
2. Use the token in your ThreadsPipe library scripts (load via `load_dotenv()`).
3. Periodically refresh via cron job: `threadspipepy refresh_token --auto_mode=true --env_path=./.env --env_variable=THREADS_ACCESS_TOKEN`.

## Silent Mode

Use `--silent` (`-s`) to suppress all logging and output, ideal for scripts or CI/CD pipelines. The CLI still performs the token operations and updates `.env` files but produces no console output. Even if passed with a value (e.g., `-s true`), it disables logging via `logging.disable()`.

## Error Handling

The CLI handles errors gracefully but exits with `sys.exit()` on failures:

- **HTTP Errors (Status > 201)**: For token generation or refresh, API errors (e.g., invalid auth code, expired token, or rate limits) are pretty-printed with a "Could not generate/refresh token" message, including the full error response from Threads API.
- **Argument Validation**:
  - Missing required args for `access_token` (e.g., no `--app_id`) trigger argparse errors.
  - Mismatched `--env_path`/`--env_variable`: Logs an error like "You must also provide the env variable" and exits.
  - Invalid action: Logs "Not implemented!" and exits.
- **Token Expiration/Refresh Rules**:
  - Expired tokens (>60 days) or too-new tokens (<24 hours) return API errors like `{"error": {"message": "Invalid token", "type": "OAuthException"}}`.
  - One-time auth codes: Invalid or reused codes cause `{"error": {"message": "Code expired", "type": "OAuthException"}}`.
- **File/Path Issues**: `.env` updates use the current working directory; path errors (e.g., invalid `--env_path`) may raise `dotenv` exceptions, which are logged.

No automatic retries are implemented—re-run with corrected inputs. For production, wrap CLI calls in scripts that check exit codes.

:::caution
Always secure your `.env` files (add to `.gitignore`). Never commit tokens or secrets.
:::

## Integration with the ThreadsPipe Library

The CLI reuses the same API endpoints and logic as the library's authentication methods (e.g., `ThreadsAPI.get_access_tokens()` and `ThreadsAPI.refresh_token()`). Tokens generated or refreshed via CLI can be directly loaded into library instances:

```python
from threadspipepy import ThreadsAPI
from dotenv import load_dotenv

load_dotenv('./.env')  # Loads THREADS_ACCESS_TOKEN
api = ThreadsAPI()
# Use api for posting, retrieving data, etc.
```

This ensures consistency between CLI token management and library usage. For full library integration, see [Initialization](../initialization.md) and [Retrieving Data](../retrieving-data.md).

For advanced scripting or troubleshooting, inspect the CLI source in `threadspipepy/cli.py`.
