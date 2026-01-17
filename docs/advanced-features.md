---
title: Advanced Features
slug: advanced-features
description: Dive into advanced functionalities of ThreadsPipe, such as hashtag management, intent links, quota monitoring, and geo-gating checks. Learn to customize behaviors for rate limits and multi-post chains.
sidebar_label: Advanced Features
sidebar_position: 6
---

# Advanced Features

ThreadsPipe offers a range of advanced features to enhance your interaction with the Threads API. This page covers hashtag handling for seamless tagging in posts, intent links for easy sharing, quota management to avoid rate limits, and geo-gating eligibility for location-based restrictions. We'll also reference key constants from `threadspipe.py` and provide code examples for custom behaviors, such as waiting on rate limits or persisting tags (or quotes) across multi-post chains.

These features build on the core posting and retrieval capabilities described in [Posting and Replies](/docs/posting-replies.md) and [Retrieving Data](/docs/retrieving-data.md). Initialize your `ThreadsPipe` instance with relevant parameters to enable them.

## Constants from threadspipe.py

ThreadsPipe defines several constants that power advanced options. These are class-level attributes and are used for validation and API parameter construction.

### Reply Controls: `who_can_reply_list`

The `who_can_reply_list` constant specifies valid options for controlling who can reply to your posts:

```python
who_can_reply_list = ['everyone', 'accounts_you_follow', 'mentioned_only']
```

- **Usage**: Pass one of these values to the `pipe` method's `who_can_reply` parameter to restrict replies. For example, `'accounts_you_follow'` limits replies to users you follow.
- **Integration**: In the `pipe` method, this appends `&reply_control={who_can_reply}` to the API endpoint.

Example:
```python
from threadspipe import ThreadsPipe

api = ThreadsPipe(user_id="your_user_id", access_token="your_token")
result = api.pipe(
    post="What do you think? #discussion",
    who_can_reply="accounts_you_follow"  # Restricts replies to followed accounts
)
```

### User Insight Metrics: `threads_user_insight_metrics`

This constant lists available metrics for user insights:

```python
threads_user_insight_metrics = ["views", "likes", "replies", "reposts", "quotes", "followers_count", "follower_demographics"]
```

- **Usage**: Use in `get_user_insights` with `metrics='all'` for all metrics or a subset (e.g., `["views", "likes"]`). The `'follower_demographics'` metric requires a `follower_demographic_breakdown` like `'country'`.
- **Integration**: Metrics are comma-joined for API calls (e.g., `&metrics=views,likes`).

Example (see [Retrieving Data](/docs/retrieving-data.md) for full details):
```python
insights = api.get_user_insights(
    metrics='all',  # Fetches all from threads_user_insight_metrics
    follower_demographic_breakdown='country'
)
print(insights['data'][0]['views'])  # Example output: 1500
```

## Hashtag Handling

Hashtag handling automates the extraction, appending, and persistence of tags in posts, especially useful for long texts that exceed the 500-character limit (defined as `__threads_post_length_limit__`). This is controlled via initialization parameters and the `pipe` method.

### Key Parameters

- `handle_hashtags: bool = True`: Manually extracts hashtags from the end of the post text using regex (`r"(?<=\s)(#[\w\d]+(?:\s#[\w\d]+)*)$"`) and appends them to post segments.
- `auto_handle_hashtags: bool = False`: Automatically adds hashtags only if a post segment lacks them (checked via `__should_handle_hash_tags__`, which uses regex to detect `#word` patterns).
- `persist_tags_multipost: bool = False` (in `pipe`): When splitting long posts into chains, repeats the full set of tags across all segments instead of distributing them.

### How It Works

- During `pipe`, if `handle_hashtags` or `auto_handle_hashtags` is enabled and tags are detected (or absent in auto mode), they are stripped from the main text and passed to the private `__split_post__` method.
- `__split_post__` splits the text into 500-character batches, appending tags to each (or the first tag per batch by default). For chains (`chained_post=True`), it adds `...` for continuity.
- If `persist_tags_multipost=True`, all tags are repeated on every batch, ensuring consistency (e.g., for "quotes" or tags in multi-post threads).

This prevents tag loss in long posts and supports custom behaviors like persisting "quotes" (e.g., referenced content) by treating them as tags.

### Code Examples

Basic hashtag extraction:
```python
api = ThreadsPipe(
    user_id="your_user_id",
    access_token="your_token",
    handle_hashtags=True,  # Enable extraction
    auto_handle_hashtags=False
)

result = api.pipe(
    post="Exciting news! Check it out #ThreadsAPI #Python",
    chained_post=False  # Single post; tags appended if space allows
)
# Post becomes: "Exciting news! Check it out!" + appended "#ThreadsAPI #Python"
```

Persisting tags in multi-post chains (custom behavior for long content with quotes):
```python
long_post = "This is a very long post about ThreadsPipe features. " * 10 + "#feature #quote1 #quote2"
result = api.pipe(
    post=long_post,
    persist_tags_multipost=True,  # Repeats "#feature #quote1 #quote2" on each chain segment
    chained_post=True  # Enables splitting into chain
)
# Output: Multiple posts like "This is a very long post... #feature #quote1 #quote2"
```

For auto-handling (add tags if missing):
```python
api = ThreadsPipe(..., auto_handle_hashtags=True)
result = api.pipe(post="Plain text without tags")  # Automatically appends default or extracted tags if configured
```

:::tip
Use `tags=[]` in `pipe` to override extraction with explicit tags, e.g., `tags=["#custom1", "#custom2"]`.
:::

## Intent Links

Intent links generate shareable URLs for posting or following on Threads. These are deep links that pre-fill content and work without authentication.

### `get_post_intent`

Generates a URL for sharing a pre-filled post with optional text and link.

```python
def get_post_intent(self, text: str = None, link: str = None):
    _text = "" if text is None else self.__quote_str__(text)  # URL-encodes text
    _url = "" if link is None else "&url=" + urlp.quote(link, safe="")
    return f"https://www.threads.net/intent/post?text={_text}{_url}"
```

Example:
```python
intent_url = api.get_post_intent(
    text="Share this amazing tool!",
    link="https://github.com/paulosabayomi/ThreadsPipe-py"
)
print(intent_url)
# Output: https://www.threads.net/intent/post?text=Share%20this%20amazing%20tool!&url=https%3A//github.com/paulosabayomi/ThreadsPipe-py
```

### `get_follow_intent`

Creates a follow URL for a specific username (defaults to your profile).

```python
def get_follow_intent(self, username: Union[str, None] = None):
    _username = username if username is not None else self.get_profile()['username']
    return f"https://www.threads.net/intent/follow?username={_username}"
```

Example:
```python
follow_url = api.get_follow_intent(username="example_user")
print(follow_url)
# Output: https://www.threads.net/intent/follow?username=example_user
```

These links can be shared via apps or web for seamless user engagement.

## Quota Management

Monitor and handle API rate limits to prevent 429 errors. The `get_quota_usage` method fetches current usage, integrated into `pipe` for automatic checks.

### `get_quota_usage`

Returns quota details for posts or replies.

```python
def get_quota_usage(self, for_reply=False):
    field = "&fields=reply_quota_usage,reply_config" if for_reply else "&fields=quota_usage,config"
    req = requests.get(self.__threads_rate_limit_endpoint__ + field)
    return req.json() if 'data' in req.json() else None
```

- **Returns**: JSON with `quota_usage`, `quota_total`, and `quota_duration` (in seconds).
- **Integration**: In `pipe` and `__send_post__`, it checks usage against limits. If exceeded and `wait_on_rate_limit=True` (initialization param), it sleeps for `quota_duration`.

### Custom Behavior: Waiting on Rate Limits

Enable automatic waiting during high usage:

```python
api = ThreadsPipe(
    user_id="your_user_id",
    access_token="your_token",
    wait_on_rate_limit=True  # Sleeps if quota exceeded
)

# Manual check
quota = api.get_quota_usage(for_reply=False)  # For posts
usage = quota['data'][0]['quota_usage']
total = quota['data'][0]['config']['quota_total']
if usage >= total:
    print(f"Quota at {usage}/{total}. Waiting {quota['data'][0]['config']['quota_duration']}s...")
    time.sleep(quota['data'][0]['config']['quota_duration'])

result = api.pipe(post="High-volume post")  # Automatically waits if needed
```

For replies, use `for_reply=True`. This ensures reliable posting during bursts.

## Geo-Gating Eligibility

Geo-gating restricts posts to specific countries, available only for eligible accounts.

### `is_eligible_for_geo_gating`

Checks if your account supports geo-restrictions.

```python
def is_eligible_for_geo_gating(self):
    url = self.__threads_profile_endpoint__ + "&fields=id,is_eligible_for_geo_gating"
    req = requests.get(url)
    if req.status_code != 200:
        return self.__tp_response_msg__(message="Could not get geo-gating eligibility...", is_error=True)
    return req.json()  # e.g., {'id': '123', 'is_eligible_for_geo_gating': True}
```

- **Returns**: Profile JSON with `is_eligible_for_geo_gating` boolean.
- **Integration**: In `pipe`, if `allowed_country_codes` is provided (list or comma-separated), it validates eligibility. If ineligible, returns an error. Eligible posts append `&allowlisted_country_codes=US,CA`.

### Usage Example

```python
eligible = api.is_eligible_for_geo_gating()
if eligible['is_eligible_for_geo_gating']:
    result = api.pipe(
        post="Content for select countries only",
        allowed_country_codes=["US", "CA", "NG"]  # Appends &allowlisted_country_codes=US,CA,NG
    )
else:
    print("Account not eligible for geo-gating.")
```

Use `get_allowlisted_country_codes()` to fetch valid country codes (e.g., ISO codes like 'US').

:::caution
Geo-gating requires account verification; ineligible users will see errors in `pipe`.
:::

These advanced features allow for robust, production-ready Threads integrations. For troubleshooting, see [Testing and Contribution](/docs/testing-contribution.md).
