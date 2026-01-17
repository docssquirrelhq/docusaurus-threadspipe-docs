---
title: Changelog and API Reference
slug: changelog-reference
description: Comprehensive changelog compiling release notes and detailed API reference for the ThreadsPipe class, including method signatures, parameters, returns, exceptions, and version history.
sidebar_label: Changelog & API
sidebar_position: 10
---

# Changelog and API Reference

This page provides a compiled changelog based on the project's release history and a detailed method-by-method API reference for the `ThreadsPipe` class. The current version of the library is **0.4.5** (as defined in `pyproject.toml`). Version history is tied to releases documented in `CHANGELOG.md`, with key updates, fixes, and additions noted below.

## Changelog

The changelog is compiled from `CHANGELOG.md` in reverse chronological order (newest first). Each release includes added features, fixes, and other changes relevant to the Threads API integration and library functionality.

### Version 0.4.5 - 2025-03-27
#### Fix
- Removed redundancies and moved mutable class attributes into the `__init__` method.

### Version 0.4.4 - 2024-11-19
#### Fix
- Added dots to the end of each post split when post length is less than the post limit but exceeds it after adding tags. This bug was originally fixed in v0.4.1 but overlooked for split posts.

### Version 0.4.3 - 2024-11-19
#### Fix
- Uncommented lines of code that were accidentally commented out during testing.

### Version 0.4.2 - 2024-11-19
#### Fix
- Perfected the bug fix from v0.4.1; previously, the update stripped the rest of the post text and replaced it with the tag.

### Version 0.4.1 - 2024-11-19
#### Fix
- Fixed a bug in the `__split_post__` method where posts slightly under 500 characters exceeded the limit after adding tags.

### Version 0.4.0 - 2024-11-12
#### Added
- Support for the Meta's Threads API update (October 28, 2024): Added the `shares` metric for post insights.
#### Fix
- Handled single-scope cases in the `get_auth_token` method (scope property now processes single scopes correctly).

### Version 0.3.0 - 2024-10-14
#### Added
- Support for Meta's Threads API update (October 9, 2024): Added post quoting and reposts.
#### Fix
- In `cli.py`, changed the return statement in `__refresh_token__` status code check to use `pprint.pp`.

### Version 0.2.1 - 2024-09-27
#### Fix
- Fixed a bug in the base64 file check method that caused errors for binary files.

### Version 0.2.0 - 2024-09-20
#### Fix
- Added a method to delete temporarily uploaded GitHub files if post debug checks fail before publishing.
#### Added
- Support for Meta's Threads API update (September 19, 2024): Increased media file limit per post to 20 (updated in README).

### Version 0.1.6 - 2024-09-18
#### Fix
- Fixed a bug in the `pipe` method where tags at the end of posts with newlines or spaces were ignored.
#### Added
- Added `link_attachment_url` to return fields for `get_post` and `get_posts`.
- Added `threads_api_version` parameter to the class initializer and `update_param` method for customizing the Threads API version.

### Version 0.1.5 - 2024-09-17
#### Fix
- Fixed a bug in `__split_post__` where subsequent batches after adding hashtags reset to the beginning.

### Version 0.1.4 - 2024-09-17
#### Added
- Test case for supported file URL formats.
- Support for IP address file URLs.
- Support for ports in file URLs.
#### Fix
- Updated RegExp for file URLs to match more formats; previous RegExp failed for some URLs, causing errors.

### Version 0.1.3 - 2024-09-17
#### Fix
- Bug fix in the `pipe` method.

### Version 0.1.2 - 2024-09-16
#### Fix
- Bug fix in the `pipe` method.

### Version 0.1.1 - 2024-09-16
#### Fix
- Bug in the `send_post` method.

### Version 0.1.0 - 2024-09-16
#### Added
- `link_attachment` parameter to `pipe` for explicitly adding links to text-only posts.
- Response object to the `__tp_response_msg__` method in `ThreadsPipe`.

**Deprecation Notes**: No methods or parameters are currently deprecated in v0.4.5. Future versions may deprecate older API version handling if Meta's Threads API evolves significantly. Always check the changelog for breaking changes.

## API Reference

The `ThreadsPipe` class is the core interface for interacting with Meta's Threads API. It handles authentication, posting, retrieving data, insights, and more. Below is a method-by-method reference extracted from `threadspipe.py` docstrings, including signatures, parameters, returns, and exceptions. Methods are listed in the order they appear in the source code.

Internal methods (prefixed with `__`) are documented but noted as private; use public methods where possible.

### `ThreadsPipe.__init__(self, user_id: int, access_token: str, disable_logging: bool = False, wait_before_post_publish: bool = True, post_publish_wait_time: int = 35, wait_before_media_item_publish: bool = True, media_item_publish_wait_time: int = 35, handle_hashtags: bool = True, auto_handle_hashtags: bool = False, gh_bearer_token: str = None, gh_api_version: str = "2022-11-28", gh_repo_name: str = None, gh_username: str = None, gh_upload_timeout: int = 60 * 5, wait_on_rate_limit: bool = False, check_rate_limit_before_post: bool = True, threads_api_version: str = 'v1.0') -> None`

#### Description
Initializes a `ThreadsPipe` instance for interacting with the Threads API. Handles authentication, logging, rate limiting, and GitHub integration for temporary file uploads.

#### Parameters
- `user_id: int` - The Threads user ID (from authentication response).
- `access_token: str` - Long-lived access token (recommended).
- `disable_logging: bool = False` - Disable Python logging if `True`.
- `wait_before_post_publish: bool = True` - Wait for media processing before publishing (recommended).
- `post_publish_wait_time: int = 35` - Wait time in seconds (min 30s).
- `wait_before_media_item_publish: bool = True` - Wait for media item processing.
- `media_item_publish_wait_time: int = 35` - Wait time for media items.
- `handle_hashtags: bool = True` - Automatically handle hashtags in posts.
- `auto_handle_hashtags: bool = False` - Conditionally add hashtags only if none exist.
- `gh_bearer_token: str = None` - GitHub fine-grained token for temp file uploads.
- `gh_api_version: str = "2022-11-28"` - GitHub API version.
- `gh_repo_name: str = None` - GitHub repo for temp uploads.
- `gh_username: str = None` - GitHub username.
- `gh_upload_timeout: int = 300` - Upload timeout in seconds.
- `wait_on_rate_limit: bool = False` - Wait on rate limits if exceeded.
- `check_rate_limit_before_post: bool = True` - Check quota before posting.
- `threads_api_version: str = 'v1.0'` - Threads API version (added in v0.1.6).

#### Returns
`None`

#### Exceptions
- None explicitly raised; invalid tokens may cause runtime errors in API calls.

#### Version History
- Introduced in v0.1.0.
- `threads_api_version` added in v0.1.6.

### `ThreadsPipe.pipe(self, post: Optional[str] = "", files: Optional[List] = [], file_captions: List[Union[str, None]] = [], tags: Optional[List] = [], reply_to_id: Optional[str] = None, who_can_reply: Union[str, None] = None, chained_post: bool = True, persist_tags_multipost: bool = False, allowed_country_codes: Union[str, List[str]] = None, link_attachments: List[str] = [], quote_post_id: Union[str, int, None] = None, persist_quoted_post: bool = False)`

#### Description
Sends posts or replies to Threads. Supports text, media (up to 20 per post), chaining for long content, quoting, and geo-gating. Automatically splits content exceeding limits (500 chars text, 20 media).

#### Parameters
- `post: Optional[str] = ""` - Post text (auto-splits if >500 chars).
- `files: Optional[List] = []` - Media files (paths, URLs, bytes, base64; auto-splits if >20).
- `file_captions: List[Union[str, None]] = []` - Captions per file (use `None` for no caption).
- `tags: Optional[List] = []` - Hashtags (appended if `handle_hashtags` enabled).
- `reply_to_id: Optional[str] = None` - ID to reply to.
- `who_can_reply: Union[str, None] = None` - Reply restrictions (from `who_can_reply_list`: 'everyone', 'accounts_you_follow', 'mentioned_only').
- `chained_post: bool = True` - Enable auto-chaining for long content.
- `persist_tags_multipost: bool = False` - Keep tags across chained posts.
- `allowed_country_codes: Union[str, List[str]] = None` - Geo-gate by country codes (requires permission).
- `link_attachments: List[str] = []` - Links for text-only posts (one per chained post).
- `quote_post_id: Union[str, int, None] = None` - ID of post to quote (added in v0.3.0).
- `persist_quoted_post: bool = False` - Attach quote to all chained posts.

#### Returns
`dict` - Success: `{'info': 'success', 'data': body, 'message': str}`; Error: `{'info': 'error', 'error': body, 'message': str}`. Includes post IDs.

#### Exceptions
- `Exception`: Raised if no text or media provided.
- Error dict if geo-gating ineligible or rate-limited.
- Runtime errors for invalid media (e.g., unsupported formats).

#### Version History
- Introduced in v0.1.0.
- `link_attachment` added in v0.1.0.
- Quoting support in v0.3.0.
- Bug fixes in v0.1.2–0.1.6, v0.2.0–0.4.5 (e.g., splitting, tags, media limits).

### `ThreadsPipe.get_access_tokens(self, app_id: str, app_secret: str, auth_code: str, redirect_uri: str)`

#### Description
Exchanges authorization code for short- and long-lived access tokens.

#### Parameters
- `app_id: str` - App ID from Threads dashboard.
- `app_secret: str` - App secret from dashboard.
- `auth_code: str` - One-time code from auth redirect.
- `redirect_uri: str` - Matching redirect URI from auth.

#### Returns
`dict` - `{'user_id': str, 'tokens': {'short_lived': dict, 'long_lived': dict}}` or error dict.

#### Exceptions
- Error dict if requests fail (status >201).

#### Version History
- Introduced in v0.1.0.
- Fix for single-scope in v0.4.0.

### `ThreadsPipe.refresh_token(self, access_token: str, env_path: str = None, env_variable: str = None)`

#### Description
Refreshes a long-lived token (must be >24h old, expires in 60 days). Optionally updates .env file.

#### Parameters
- `access_token: str` - Long-lived token to refresh.
- `env_path: str = None` - Path to .env file.
- `env_variable: str = None` - Env var name to update.

#### Returns
`dict` - New token JSON or error.

#### Exceptions
- Runtime errors for invalid tokens.

#### Version History
- Introduced in v0.1.0.
- CLI fix in v0.3.0.

### `ThreadsPipe.get_posts(self, since_date: Union[str, None] = None, until_date: Union[str, None] = None, limit: Union[str, int, None] = None)`

#### Description
Retrieves all user posts (including replies).

#### Parameters
- `since_date: Union[str, None] = None` - Start date (ISO format).
- `until_date: Union[str, None] = None` - End date.
- `limit: Union[str, int, None] = None` - Max results.

#### Returns
`dict` - JSON list of posts (fields: id, text, media_url, etc.).

#### Exceptions
- API errors returned as JSON.

#### Version History
- Introduced in v0.1.0.
- `link_attachment_url` added in v0.1.6.

### `ThreadsPipe.get_post(self, post_id: str)`

#### Description
Retrieves a single post's data.

#### Parameters
- `post_id: str` - Post ID.

#### Returns
`dict` - Post JSON (fields: id, text, media, etc.).

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.
- `link_attachment_url` added in v0.1.6.

### `ThreadsPipe.get_profile(self)`

#### Description
Retrieves user profile data.

#### Parameters
None.

#### Returns
`dict` - Profile JSON (id, username, name, bio, etc.).

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.get_post_replies(self, post_id: str, top_levels: bool = True, reverse: bool = False)`

#### Description
Retrieves replies to a post (top-level or deep).

#### Parameters
- `post_id: str` - Post ID.
- `top_levels: bool = True` - `False` for deep replies.
- `reverse: bool = False` - Reverse order if `True`.

#### Returns
`dict` - Replies JSON.

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.get_user_replies(self, since_date: Union[str, None] = None, until_date: Union[str, None] = None, limit: Union[int, str] = None)`

#### Description
Retrieves all user replies.

#### Parameters
- `since_date: Union[str, None] = None` - Start date.
- `until_date: Union[str, None] = None` - End date.
- `limit: Union[int, str] = None` - Max results.

#### Returns
`dict` - Replies JSON.

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.hide_reply(self, reply_id: str, hide: bool)`

#### Description
Hides or unhides a reply under a post.

#### Parameters
- `reply_id: str` - Reply ID.
- `hide: bool` - `True` to hide, `False` to unhide.

#### Returns
`dict` - API response JSON.

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.get_post_insights(self, post_id: str, metrics: Union[str, List[str]] = 'all')`

#### Description
Retrieves insights for a post (views, likes, etc.).

#### Parameters
- `post_id: str` - Post ID.
- `metrics: Union[str, List[str]] = 'all'` - Metrics (from `threads_post_insight_metrics`: 'views', 'likes', etc.; 'all' for all).

#### Returns
`dict` - Insights JSON.

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.
- `shares` added in v0.4.0.

### `ThreadsPipe.get_user_insights(self, user_id: Optional[str] = None, metrics: Union[str, List[str]] = 'all', since_date: Union[str, None] = None, until_date: Union[str, None] = None, follower_demographic_breakdown: str = 'country')`

#### Description
Retrieves user-level insights (views, followers, etc.).

#### Parameters
- `user_id: Optional[str] = None` - User ID ('me' for current).
- `metrics: Union[str, List[str]] = 'all'` - From `threads_user_insight_metrics`.
- `since_date: Union[str, None] = None` - Start date.
- `until_date: Union[str, None] = None` - End date.
- `follower_demographic_breakdown: str = 'country'` - From `threads_follower_demographic_breakdown_list`.

#### Returns
`dict` - Insights JSON.

#### Exceptions
- API errors as JSON.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.get_post_intent(self, text: str = None, link: str = None)`

#### Description
Generates a share intent URL for posting.

#### Parameters
- `text: str = None` - Post text.
- `link: str = None` - Link to attach.

#### Returns
`str` - Intent URL.

#### Exceptions
None.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.get_follow_intent(self, username: Union[str, None] = None)`

#### Description
Generates a follow intent URL.

#### Parameters
- `username: Union[str, None] = None` - Username ('me' auto-uses current).

#### Returns
`str` - Intent URL.

#### Exceptions
None.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.update_param(self, user_id: Optional[int] = None, access_token: Optional[str] = None, disable_logging: Optional[bool] = None, wait_before_post_publish: Optional[bool] = None, post_publish_wait_time: Optional[int] = None, wait_before_media_item_publish: Optional[bool] = None, media_item_publish_wait_time: Optional[int] = None, handle_hashtags: Optional[bool] = None, threads_api_version: Optional[str] = None, auto_handle_hashtags: Optional[bool] = None, gh_bearer_token: Optional[str] = None, gh_api_version: Optional[str] = None, gh_username: Optional[str] = None, gh_repo_name: Optional[str] = None, gh_upload_timeout: Optional[int] = None, wait_on_rate_limit: Optional[bool] = None, check_rate_limit_before_post: Optional[bool] = None)`

#### Description
Updates instance parameters post-initialization.

#### Parameters
- Matches `__init__` parameters (optionals only).

#### Returns
`None`

#### Exceptions
None.

#### Version History
- Introduced in v0.1.0.
- `threads_api_version` added in v0.1.6.

### `ThreadsPipe.get_quota_usage(self, for_reply: bool = False)`

#### Description
Checks rate limit quota (posts or replies).

#### Parameters
- `for_reply: bool = False` - Check reply quota if `True`.

#### Returns
`dict` - Quota JSON or `None` on error.

#### Exceptions
- Error dict on API failure.

#### Version History
- Introduced in v0.1.0.

### `ThreadsPipe.is_eligible_for_geo_gating(self)`

#### Description
Checks if the account can use geo-gating.

#### Parameters
None.

#### Returns
`dict` - `{'is_eligible_for_geo_gating': bool}` or error.

#### Exceptions
- Error dict on API failure.

#### Version History
- Introduced in v0.1.0 (tied to geo-gating support).

### Internal Methods
- `__send_post__(...)`: Private; handles core posting logic. Returns post ID or error.
- `__split_post__(self, post: str, tags: List[str] = [])`: Private; splits long posts. Returns list of split strings. Fixed in v0.4.1–0.4.5.
- `__handle_media__(self, media_files: List[Any])`: Private; processes files (uploads to GitHub if needed). Returns list of media objects or error. Fixed in v0.2.1.
- `__tp_response_msg__(message: str, body: Any, response: Any = None, is_error: bool = False)`: Static private; formats responses. Added in v0.1.0.
- Other utilities (e.g., `__should_handle_hash_tags__`, `__quote_str__`): Private helpers; no public exceptions.

For full source and examples, refer to [threadspipe.py](../src/threadspipepy/threadspipe.py). Links to other docs: [Introduction](../introduction.md), [Posting & Replies](../posting-replies.md).
