---
title: Initialization and Configuration
slug: initialization
description: A comprehensive guide to initializing the ThreadsPipe class, including required and optional parameters, configuration examples, and the update_param method.
sidebar_label: Initialization
sidebar_position: 3
---

# Initialization and Configuration

The `ThreadsPipe` class is the core interface for interacting with the Threads API. Initializing an instance requires understanding its parameters, which control authentication, behavior, and advanced features like file uploads and rate limiting. This page covers the `__init__` method, required and optional parameters, practical examples, and the `update_param` method for runtime adjustments.

For installation prerequisites, refer to the [Installation](../installation.md) page.

## Required Parameters

The `ThreadsPipe` constructor requires two essential parameters for authentication and API access:

- **`user_id` (int)**:  
  The unique identifier for the Threads user account. This is obtained from the data returned by the access token generation process (e.g., via the Threads Graph API's OAuth endpoints). It identifies the account for posting, retrieving data, and other operations.

- **`access_token` (str)**:  
  The access token for the Threads account. Use a long-lived access token for production (recommended over short-lived ones for stability). Tokens are generated via Meta's OAuth flow and include scopes like `threads_basic`, `threads_content_publish`, etc. See the [introduction](../introduction.md) for token generation details.

:::caution Token Expiration Warning
Access tokens expire—short-lived ones last about 1 hour, while long-lived ones extend to 60 days. Monitor expiration and implement refresh logic using the Threads refresh endpoint (`https://graph.threads.net/refresh_access_token`). Failing to refresh can lead to authentication errors. Always test tokens before use.

:::

## Optional Parameters

The constructor includes several optional parameters to customize behavior. Defaults are provided for most, but adjust them based on your use case. Key ones include:

- **`disable_logging` (bool, default: False)**:  
  Controls Python's logging module. Set to `True` to suppress debug/info logs, which is useful in production to reduce output noise.

:::warning Logging Disablement
Disabling logging (`disable_logging=True`) hides valuable debug information, making troubleshooting harder. Use sparingly and enable it during development.

:::

- **`handle_hashtags` (bool, default: True)**:  
  Automatically processes hashtags at the end of posts. Threads allows only one hashtag per post, so this splits and distributes them across chained (threaded) posts if the content exceeds the 500-character limit or includes multiple media items (up to 20).

- **`auto_handle_hashtags` (bool, default: False)**:  
  An advanced mode for hashtag handling. When `True`, it intelligently skips posts with existing hashtags in the body and distributes multiple end-hashtags across threads more dynamically.

- **`gh_bearer_token` (str, default: None)**, **`gh_repo_name` (str, default: None)**, **`gh_username` (str, default: None)**, **`gh_api_version` (str, default: "2022-11-28")**, **`gh_upload_timeout` (int, default: 300)**:  
  These enable temporary file storage on GitHub for local uploads. Threads API requires public URLs for media, so local files are uploaded to a GitHub repo, used in the post, and deleted afterward.  
  - `gh_bearer_token`: A fine-grained GitHub Personal Access Token (PAT) with repo write/delete permissions. Generate at [github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta).  
  - `gh_repo_name`: The target repo (must exist and be public or accessible via the token).  
  - `gh_username`: Your GitHub username.  
  - `gh_api_version`: GitHub REST API version.  
  - `gh_upload_timeout`: Upload timeout in seconds (e.g., 5 minutes default).  
  This setup is ideal for local images/videos without a public server.

- **`wait_on_rate_limit` (bool, default: False)**:  
  If `True`, the library waits and retries on rate limit hits (250 posts/day default, 1000 replies/day) instead of failing. Useful for high-volume posting but may increase memory usage with concurrent requests.

- **`check_rate_limit_before_post` (bool, default: True)**:  
  Checks remaining rate limits before each post. Set to `False` to skip checks for speed, but monitor manually to avoid 429 errors.

- **`threads_api_version` (str, default: "v1.0")**:  
  The Meta Threads API version. Stick to "v1.0" unless a newer version is required.

Other options include wait times for post/media processing (e.g., `wait_before_post_publish=True`, `post_publish_wait_time=35` seconds) to ensure media finishes uploading before publishing—recommended to avoid failures.

## Basic Initialization Example

Here's a simple setup:

```python
from threadspipe import ThreadsPipe

# Assuming you have user_id and access_token from OAuth
api = ThreadsPipe(
    user_id=123456789,  # Your Threads user ID
    access_token="your_long_lived_access_token",
    disable_logging=False,  # Enable logs for debugging
    handle_hashtags=True,
    threads_api_version="v1.0"
)
```

This creates a basic instance ready for posting (see [Posting and Replies](../posting-replies.md)).

## GitHub Setup for Temporary File Storage

For local file uploads (e.g., images/videos on your machine), configure GitHub integration. First, create a repo and generate a fine-grained PAT with `contents:write` and `contents:delete` for that repo.

Example:

```python
api = ThreadsPipe(
    user_id=123456789,
    access_token="your_access_token",
    gh_bearer_token="ghp_your_fine_grained_pat_here",  # From GitHub settings
    gh_username="your_github_username",
    gh_repo_name="temp-threads-media",  # Existing repo for temp storage
    gh_api_version="2022-11-28",
    gh_upload_timeout=300,  # 5 minutes
    handle_hashtags=True
)

# Now post with local files
api.post("Hello Threads! #example", files=["/path/to/local/image.jpg"])
# File uploads to GitHub temporarily, posts URL to Threads, then deletes from GitHub
```

:::tip GitHub Repo Best Practices
Use a dedicated repo for temp files to avoid clutter. Ensure it's public or token-accessible. For security, scope the PAT to only this repo and minimal permissions. After posting, files auto-delete on success or error.

:::

## Rate Limiting Configuration

Threads enforces limits (250 posts/day, 1000 replies/day). Configure to handle gracefully:

```python
api = ThreadsPipe(
    user_id=123456789,
    access_token="your_access_token",
    wait_on_rate_limit=True,  # Wait and retry on limits
    check_rate_limit_before_post=True  # Always check before posting
)

# Before posting, you can manually check limits
limits = api.get_rate_limit_status()
print(f"Posts remaining: {limits['post_limit']}")
```

If `wait_on_rate_limit=True`, the library pauses on 429 errors. For bulk operations, combine with `check_rate_limit_before_post=False` to optimize but risk overages.

:::caution Rate Limit Risks
Exceeding limits results in temporary bans. Always check via the rate limit endpoint (`/threads_publishing_limit`). High-volume apps should implement exponential backoff.

:::

## API Version Selection

Select versions for Threads or GitHub APIs:

```python
# Use a hypothetical future Threads API version
api = ThreadsPipe(
    user_id=123456789,
    access_token="your_access_token",
    threads_api_version="v2.0"  # If available; default is v1.0
)

# Custom GitHub API version
api.update_param(gh_api_version="2023-01-01")
```

Monitor Meta's [Threads API docs](https://developers.facebook.com/docs/threads) for updates. Mismatching versions can cause endpoint errors.

## Updating Parameters with `update_param`

The `update_param` method allows runtime reconfiguration without reinitializing the class. It's useful for token refreshes or dynamic tweaks. Pass only the parameters to update (others retain defaults).

### Method Signature
```python
def update_param(
    self,
    user_id: int = None,
    access_token: str = None,
    # ... other params as in __init__
) -> None:
```

### Description
Updates class attributes and endpoints (e.g., refreshes URLs with new tokens). Call before actions to ensure changes apply. Not all updates guarantee immediate effect if mid-operation.

### Example: Token Refresh and Config Update
```python
# Initial setup
api = ThreadsPipe(user_id=123456789, access_token="old_token")

# Later, refresh token and update
new_token = "refreshed_long_lived_token"  # From refresh endpoint
api.update_param(
    access_token=new_token,
    user_id=123456789,  # If changed
    wait_on_rate_limit=True,  # Enable rate wait
    disable_logging=True  # Disable logs now
)

# Now safe to post
api.post("Updated config! #threads")
```

For full parameter details, see the `__init__` docstring in the source code.

## Additional Warnings and Tips

- **Token Security**: Never hardcode tokens in code—use environment variables (e.g., via `python-dotenv`).  
- **Processing Waits**: Always enable `wait_before_post_publish` and `wait_before_media_item_publish` (default: True) with at least 31-35 seconds to avoid publish failures.  
- **Testing**: Start with [CLI Usage](../cli-usage.md) or [Testing and Contribution](../testing-contribution.md) for safe experiments.  
- **Scopes**: Ensure your access token has required scopes (e.g., `threads_content_publish` for posting).  

For advanced usage like media handling, see [Media Handling](../media-handling.md).