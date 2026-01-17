---
title: Retrieving Data and Insights
slug: retrieving-data
description: Learn how to fetch posts, replies, profiles, and insights using ThreadsPipe methods like get_posts, get_post_insights, and more. Includes parameters, examples, and limitations.
sidebar_label: Retrieving Data
sidebar_position: 5
---

# Retrieving Data and Insights

ThreadsPipe provides a suite of methods to retrieve data from Threads, including posts (threads), replies, user profiles, and analytics insights. All methods return data in JSON format, making it easy to parse and integrate into your applications. These methods are part of the `ThreadsPipe` class and require proper authentication via the `initialize` method (see [Initialization](../initialization.md) for setup).

Key features include:
- Date filtering with `since_date` and `until_date` (ISO format, e.g., "2024-01-01T00:00:00Z").
- Pagination via the `limit` parameter (e.g., 10 or "10") in supported methods.
- Metrics for insights, such as views, likes, replies, reposts, quotes, shares, followers_count, and follower_demographics.
- Response formats: Structured JSON objects or lists, with fields like `id`, `text`, `timestamp`, `media_url`, `views`, etc.

## Limitations

- **Insights Availability**: Post and user insights are only available for content posted after April 13, 2024. User insights are not guaranteed before June 1, 2024. Date filters (`since_date`, `until_date`) are ineffective before April 13, 2024.
- **Permissions**: Insights methods require the 'insights' permission during token generation.
- **Pagination**: Not all methods support `limit`; check individual method docs.
- **Rate Limits**: Threads API enforces rate limits; handle errors gracefully (e.g., via retries).
- **Demographics**: Follower demographics are aggregated and anonymized; breakdowns are limited to country, city, age, or gender.

For full API details, refer to the [Advanced Features](../advanced-features.md) page.

## Fetching Posts and Threads

Use these methods to retrieve individual posts or lists of posts (threads).

### get_posts(since_date=None, until_date=None, limit=None)

Retrieves all posts from the connected account, including threads and replies. Supports date filtering and pagination.

**Parameters**:
- `since_date` (str | None, optional): Start date filter (ISO format). Default: None (no filter).
- `until_date` (str | None, optional): End date filter (ISO format). Default: None (no filter).
- `limit` (str | int | None, optional): Maximum number of posts to return (e.g., 10). Default: None (API default, typically 50).

**Returns**: JSON list of posts, each with fields like `id`, `text`, `media_type`, `media_url`, `permalink`, `timestamp`, `owner` (username), and `children` (replies).

**Example**:
```python
from threadspipe import ThreadsPipe

pipe = ThreadsPipe()
pipe.initialize(access_token="your_token")

# Fetch recent posts with pagination
posts = pipe.get_posts(limit=5, since_date="2024-05-01T00:00:00Z")
print(posts)  # List of post JSON objects
```

This example fetches the 5 most recent posts since May 1, 2024.

### get_post(post_id)

Retrieves a single post by its ID.

**Parameters**:
- `post_id` (str, required): The unique ID of the post.

**Returns**: JSON object with post details, including `id`, `media_type`, `media_url`, `thumbnail_url`, `text`, `timestamp`, `children` (replies), and `link_attachment_url` if applicable.

**Example**:
```python
post = pipe.get_post("post_id_here")
print(post['text'])  # Prints the post text
```

## Fetching Replies

### get_post_replies(post_id, top_levels=True, reverse=False)

Retrieves replies to a specific post. By default, fetches only top-level replies.

**Parameters**:
- `post_id` (str, required): The ID of the post.
- `top_levels` (bool, optional): If True, only top-level replies; if False, includes nested replies. Default: True.
- `reverse` (bool, optional): If True, reverses order (newest first). Default: False.

**Returns**: JSON list of replies, each with `id`, `text`, `timestamp`, `media_type`, `media_url`, `children`, `hide_status`, and `replied_to`.

**Example** (Fetching Replies):
```python
replies = pipe.get_post_replies("post_id_here", top_levels=False, reverse=True)
for reply in replies:
    print(f"Reply: {reply['text']}")
```

This fetches all nested replies in reverse order.

## Fetching Profiles

### get_profile()

Retrieves the profile of the connected account.

**Parameters**: None.

**Returns**: JSON object with `id`, `username`, `name`, `threads_profile_picture_url`, and `threads_biography`.

**Example**:
```python
profile = pipe.get_profile()
print(f"Username: {profile['username']}")
```

## Post Insights

### get_post_insights(post_id, metrics='all')

Retrieves engagement metrics for a specific post.

**Parameters**:
- `post_id` (str, required): The post ID.
- `metrics` (str | list[str], optional): 'all' for all metrics, or a list/comma-separated string from ['views', 'likes', 'replies', 'reposts', 'quotes', 'shares']. See `ThreadsPipe.threads_post_insight_metrics` for options. Default: 'all'.

**Returns**: JSON object with metric counts (e.g., `{'views': 100, 'likes': 50}`).

**Example** (Fetching Analytics):
```python
insights = pipe.get_post_insights("post_id_here", metrics=['views', 'likes'])
print(insights['views'])  # e.g., 100
```

## User Insights

### get_user_insights(user_id=None, since_date=None, until_date=None, follower_demographic_breakdown='country', metrics='all')

Retrieves account-level insights, including aggregated metrics and demographics.

**Parameters**:
- `user_id` (str | None, optional): Target user ID; None for connected account. Default: None.
- `since_date` (str | None, optional): Start date (ISO). Limitations apply (see above).
- `until_date` (str | None, optional): End date (ISO). Limitations apply.
- `follower_demographic_breakdown` (str, optional): Breakdown for demographics ('country', 'city', 'age', 'gender'). Default: 'country'. See `ThreadsPipe.threads_follower_demographic_breakdown_list`.
- `metrics` (str | list[str], optional): 'all' or list/comma-separated from ['views', 'likes', 'replies', 'reposts', 'quotes', 'followers_count', 'follower_demographics']. See `ThreadsPipe.threads_user_insight_metrics`. Default: 'all'.

**Returns**: JSON object with aggregated data (e.g., total views, follower count, demographics breakdown).

**Example** (Fetching Analytics with Demographics):
```python
user_insights = pipe.get_user_insights(
    since_date="2024-06-01T00:00:00Z",
    metrics=['followers_count', 'follower_demographics'],
    follower_demographic_breakdown='country'
)
print(user_insights['followers_count'])  # e.g., 1000
print(user_insights['follower_demographics'])  # Country breakdown
```

:::caution Insights Limitations
User insights require 'insights' permission and are only reliable post-June 1, 2024. Always check API responses for errors.
:::

## Pagination and Best Practices

- Use `limit` in `get_posts` for efficient fetching: Start with small limits (e.g., 10) and paginate using `until_date` based on the last post's timestamp.
- Handle JSON parsing: Responses are dicts/lists; use libraries like `json` or `pydantic` for validation.
- Error Handling: Wrap calls in try-except for API errors (e.g., rate limits).
- For media: See [Media Handling](../media-handling.md) for downloading attachments from retrieved posts.

For CLI usage of these methods, refer to [CLI Usage](../cli-usage.md).