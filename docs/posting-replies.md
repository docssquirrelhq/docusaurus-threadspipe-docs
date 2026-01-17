---
title: Posting and Replying with the Pipe Method
slug: posting-replies
description: A comprehensive guide to using the pipe method in ThreadsPipe for posting, replying, handling long content, media attachments, geo-gating, quoting, and more.
sidebar_label: Posting & Replies
sidebar_position: 4
---

# Posting and Replying with the Pipe Method

The `pipe` method in ThreadsPipe is the primary interface for creating new posts or replies on Threads (Instagram's text-based app). It supports rich features like text posts, media attachments (images and videos), replies, thread-like chaining for long content, geo-gating, quoting other posts, and link attachments. This method abstracts away much of the complexity of the Threads API, including content splitting, media uploading, and error handling.

Whether you're posting a simple text update or a multi-media thread, `pipe` handles validation, preprocessing, and API interactions. Below, we'll cover the parameters, advanced behaviors, examples, error handling, and rate limits.

For an overview of ThreadsPipe, see the [Introduction](../introduction.md). To get started, ensure you've installed and initialized the library as described in [Installation](../installation.md) and [Initialization](../initialization.md).

## Method Signature

```python
def pipe(
    self,
    post: Optional[str] = "",
    files: Optional[List] = [],
    file_captions: List[Union[str, None]] = [],
    tags: Optional[List] = [],
    reply_to_id: Optional[str] = None,
    who_can_reply: Union[str, None] = None,
    chained_post: bool = True,
    persist_tags_multipost: bool = False,
    allowed_country_codes: Union[str, List[str]] = None,
    link_attachments: List[str] = [],
    quote_post_id: Union[str, int, None] = None,
    persist_quoted_post: bool = False
) -> Union[dict, requests.Response]
```

### Returns
- On success: A dictionary with `{'message': 'Post piped to Instagram Threads successfully!', 'is_error': False, 'response': publish_request, 'body': {'media_ids': [list_of_media_ids]}}`.
- On error: A dictionary with `{'message': 'Error description', 'is_error': True, 'body': error_details, 'response': request_object}` or a `requests.Response` object for low-level errors.

## Key Parameters

### `post` (str, default: "")
The main text content of your post or reply. This can be any length, but Threads limits single posts to 500 characters. If longer, ThreadsPipe automatically splits it into a thread (chained posts) unless `chained_post=False`.

- Supports hashtags: If `handle_hashtags=True` (class-level setting), trailing hashtags (e.g., "My post #tag1 #tag2") are extracted and distributed across chains. Use `tags` to provide them explicitly.
- Empty string (`""`) is allowed for media-only posts.
- Whitespace is stripped automatically.

:::tip
For long posts, include continuity markers like "1/5" manually to guide readers.
:::

### `files` (List, default: [])
A list of media files to attach. Supports:
- Local file paths (e.g., `"/path/to/image.jpg"`).
- URLs (e.g., `"https://example.com/video.mp4"` – validated for HTTP/HTTPS).
- Base64-encoded strings (e.g., `"data:image/jpeg;base64,/9j/4AAQSkZJRgABAQ..."`).
- Bytes (e.g., `open('file.jpg', 'rb').read()`).

Threads limits single posts to 20 media items. If more, ThreadsPipe splits into batches unless `chained_post=False`. Mixed media (images/videos) is handled as carousels where possible.

Local files and bytes are temporarily uploaded to a GitHub repository (configured via `gh_username`, `gh_repo_name`, `gh_bearer_token`) for public access during posting, then deleted on success or failure.

### `file_captions` (List[Union[str, None]], default: [])
Captions (alt text) for media files, aligned by index with `files`. Use `None` for files without captions. The list length doesn't need to match `files` exactly – it's padded or truncated.

Example: For 3 files, `["Caption for first", None, "Caption for third"]`.

These are used as accessibility descriptions in the API.

### `tags` (List[str], default: [])
Explicit hashtags to add to the post. Overrides any extracted from `post`. Distributed across chained posts unless `persist_tags_multipost=True` (repeats the full list on each).

No effect if both `handle_hashtags` and `auto_handle_hashtags` (class-level) are `False`.

### `reply_to_id` (str, optional)
The media ID of the post to reply to (e.g., `"1234567890"`). If provided, the entire chain becomes a reply thread to that post. The first batch replies directly to `reply_to_id`, and subsequent batches reply to the previous one.

### `who_can_reply` (str, optional)
Controls who can reply to your post. Options from `ThreadsPipe.who_can_reply_list`:
- `"everyone"` (default).
- `"accounts_you_follow"`.
- `"mentioned_only"`.

Appended as `reply_control` in the API request.

### `chained_post` (bool, default: True)
Enables automatic splitting and chaining for content exceeding limits (500 chars or 20 media). Set to `False` to truncate to the first batch only (raises no error but discards excess).

### `persist_tags_multipost` (bool, default: False)
If `True`, repeats the full `tags` list (or extracted hashtags) on every chained post. Useful for single persistent hashtags like `#MySeries`.

### `allowed_country_codes` (Union[str, List[str]], optional)
Geo-gating: Restricts visibility to specific countries (requires user eligibility). Provide as a comma-separated string (e.g., `"US,CA,NG"`) or list (e.g., `["US", "CA", "NG"]`).

Check eligibility first with `ThreadsPipe.is_eligible_for_geo_gating()`. If ineligible, an early error is returned.

### `link_attachments` (List[str], default: [])
URLs to attach as link previews (text-only posts only; no media). Limited to one per post. For multiples in chained posts, they are distributed across batches.

Example: `["https://example.com/article"]` – URL-encoded in the API.

### `quote_post_id` (Union[str, int, None], optional)
The ID of a post to quote (embeds it in your post). For chains, attaches to the first batch by default; set `persist_quoted_post=True` to include on all.

### `persist_quoted_post` (bool, default: False)
If `True`, the quoted post is attached to every chained post.

## Handling Long Posts and Media Splitting

ThreadsPipe automatically manages content exceeding limits by creating thread-like structures (root post + chained replies).

### Long Text (>500 Characters)
- The `post` is split into ~500-character chunks using `__split_post__`.
- The first chunk becomes the root post (or reply if `reply_to_id` provided).
- Subsequent chunks are posted as replies to the previous one, with ellipses (`...`) for continuity.
- Hashtags are handled specially: Extracted and distributed (e.g., one per chain) or persisted if `persist_tags_multipost=True`.
- Chunk sizes adjust to fit tags + ellipses within 500 chars.

If `chained_post=False`, only the first 500 chars are used.

### Long Media (>20 Files)
- Files are batched into groups of 20.
- The first batch attaches to the first text chunk (root).
- Remaining batches attach to subsequent text chunks or as media-only replies if no more text.
- Captions align by index across batches.
- Each file is processed individually: URLs validated, locals/bytes uploaded to GitHub temporarily.

Mixed text/media: Media batches align with text batches where possible; extras chain as replies.

:::caution
Chaining increases API calls – monitor rate limits (see below). For very long content, test incrementally.
:::

## Geo-Gating with `allowed_country_codes`

Geo-gating restricts posts to specific countries, enhancing privacy or targeting. 

1. Verify eligibility: Call `api.is_eligible_for_geo_gating()`. Returns `True` if allowed, or an error dict if not.
2. Provide codes: Use ISO 2-letter codes (e.g., `["US", "GB"]` or `"US,GB"`).
3. In `pipe`: Appended as `allowlisted_country_codes` to media containers and publish requests.

If ineligible, `pipe` returns early: `{'message': 'You are attempting to send a geo-gated content but you are not eligible for geo-gating', 'is_error': True}`.

Available codes can be fetched via `api.get_allowlisted_country_codes()` (requires permission).

Example use case: Limit a promotional post to North America.

## Quoting Posts with `quote_post_id`

Quoting embeds another post's content in yours, like a retweet with commentary.

- Provide the post ID (str or int, e.g., `123456`).
- Attaches to the root post by default.
- For chains: Use `persist_quoted_post=True` to quote on every segment (e.g., for consistent context in threads).

Quoting works with replies and media. The quoted post must be public or accessible.

## Link Attachments

For text-only posts (`files=[]`), add clickable links with previews.

- Provide URLs in `link_attachments` (e.g., `["https://docs.threads.net"]`).
- One link per post; multiples distribute across chains.
- Only works without media – API limitation.
- URL-encoded automatically.

Ideal for sharing articles or external resources.

## Full Examples

### Basic Text Post
```python
from threadspipe import ThreadsPipe

api = ThreadsPipe(username="your_username", password="your_password")

result = api.pipe(
    post="Hello, Threads! This is my first post. #ThreadsPipe"
)
print(result['body']['media_ids'][0])  # Posted media ID
```

### Reply with Local Files and Captions
```python
result = api.pipe(
    post="Replying to your post with images!",
    files=[
        "/path/to/local/image1.jpg",
        open("/path/to/local/video.mp4", "rb").read()  # Bytes
    ],
    file_captions=[
        "A beautiful landscape",
        None  # No caption for video
    ],
    reply_to_id="1234567890",  # Target post ID
    who_can_reply="accounts_you_follow"
)
```

### URL and Base64 Media (Long Post with Splitting)
```python
import base64

# Example base64 (truncated for brevity)
with open("image.jpg", "rb") as f:
    base64_image = base64.b64encode(f.read()).decode()

long_text = "This is a very long post that exceeds 500 characters. " * 10  # >500 chars

result = api.pipe(
    post=long_text,
    files=[
        "https://example.com/image.jpg",  # URL
        base64_image,  # Base64 string
        "https://example.com/video.mp4"
    ],
    file_captions=["URL image", "Base64 image", "Video from URL"],
    chained_post=True,  # Enable splitting
    tags=["#LongThread", "#MediaExample"]  # Explicit tags
)
```

### Geo-Gated Post with Quoting and Links
```python
# Assume eligibility checked
result = api.pipe(
    post="Exclusive content for select countries!",
    allowed_country_codes=["US", "CA"],
    quote_post_id=123456,
    persist_quoted_post=True,
    link_attachments=["https://example.com/exclusive-link"],
    who_can_reply="mentioned_only"
)
```

### Many Files (>20) – Auto-Splitting
```python
files_list = [
    f"/path/to/image_{i}.jpg" for i in range(25)  # 25 local files
] + [
    "https://example.com/extra.jpg"
]

result = api.pipe(
    post="A thread with 26 media items!",
    files=files_list,
    file_captions=[f"Caption for {i}" if i % 2 == 0 else None for i in range(26)],
    chained_post=True
)
# First post: Text + first 20 images
# Reply 1: Next 6 images (media-only)
```

## Error Responses

Errors return a dict with `is_error: True` for easy checking. Common cases:

- **No Content**: `Exception` raised if `post=""` and `files=[]`.
- **Invalid File/URL**: `{'error': 'Invalid file type or URL'}` from media processing (e.g., non-HTTP URL, undecodable base64).
- **Geo-Gating**: `{'message': 'You are attempting to send a geo-gated content but you are not eligible...', 'is_error': True}`.
- **Container Creation Failure** (e.g., API 400): Deletes temp files, returns `{'message': 'An error occurred while creating media container...', 'body': api_response, 'is_error': True}`.
- **Publish Failure** (e.g., status != 'FINISHED'): Waits up to 35s (configurable via `media_item_publish_wait_time`), then errors with deletion.
- **HTTP Errors**: Full `requests.Response` or `{'message': 'Could not publish post', 'body': {...}, 'is_error': True}`.

Always check `result.get('is_error')` before using `media_ids`.

:::danger
Local file uploads use GitHub – ensure your tokens have repo/write/delete scopes. Timeouts: 5 minutes default.
:::

## Rate Limits and Quota Checks

Threads enforces strict limits to prevent spam:
- 250 posts per day.
- 1000 replies per day.

### Quota Checking
- Enable via class init: `ThreadsPipe(..., check_rate_limit_before_post=True)`.
- Before each `pipe`, fetches quota via `get_quota_usage(for_reply=bool(reply_to_id))`.
- If exceeded (status 429), returns `{'message': 'Rate limit exceeded!', 'body': quota_data, 'is_error': True}`.
- For replies, uses reply-specific quota.

### Handling Limits
- Set `wait_on_rate_limit=True` in init to pause and retry (not default; memory-intensive for long chains).
- No built-in exponential backoff – implement manually for production.
- Monitor via logs: Errors logged with `logging.error`.

:::info
Quotas reset daily (UTC). For high-volume use, batch posts across accounts or schedule via cron. See [Advanced Features](../advanced-features.md) for custom waits.
:::

This guide covers the core of posting and replying. For media-specific details, see [Media Handling](../media-handling.md). If you encounter issues, check [Testing & Contribution](../testing-contribution.md).
