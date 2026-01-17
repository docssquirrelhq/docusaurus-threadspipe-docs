---
title: Introduction to ThreadsPipe-py
slug: introduction
description: Overview of ThreadsPipe-py, a Python client for Meta's Threads API, including purpose, features, architecture, and quick start examples.
sidebar_label: Introduction
sidebar_position: 1
---

# Introduction to ThreadsPipe-py

ThreadsPipe-py is a comprehensive Python library designed to simplify interactions with Meta's Threads API. It provides an intuitive interface for developers to post content, manage replies, retrieve insights, and handle media uploads on Threads (Instagram's text-based conversation app). Built on top of the official Meta Graph API, ThreadsPipe-py abstracts away the complexities of API authentication, rate limiting, and media processing, allowing you to focus on creating engaging Threads content.

Whether you're automating social media posts, building analytics tools, or integrating Threads into your application, ThreadsPipe-py offers robust tools to streamline your workflow.

![ThreadsPipe-py Banner](https://via.placeholder.com/800x200?text=ThreadsPipe-py) <!-- Replace with actual banner if available -->

## Badges

[![GitHub](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/paulosabayomi/ThreadsPipe-py)
[![PyPI](https://img.shields.io/pypi/v/threadspipepy)](https://pypi.org/project/threadspipepy/)
[![Tests](https://img.shields.io/badge/Tests-Passing-brightgreen)](https://github.com/paulosabayomi/ThreadsPipe-py/actions) <!-- Assumes tests pass; link to GitHub Actions or similar -->
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python >=3.8](https://img.shields.io/badge/Python->=3.8-blue)](https://www.python.org/)

## Purpose

ThreadsPipe-py serves as a Python client for Meta's Threads API, enabling seamless integration with Threads for tasks like content publishing, data retrieval, and analytics. It addresses common challenges such as handling long-form text (beyond the 500-character limit via chaining), media uploads (including local files via temporary GitHub storage), geo-gating posts, and fetching insights like views, likes, and follower demographics.

The library is particularly useful for:
- Social media automation tools.
- Content management systems.
- Analytics dashboards for Threads performance.
- Bots or scripts for scheduled posting and replying.

By leveraging the Threads API (version v1.0 by default, configurable), it ensures compliance with Meta's guidelines while providing developer-friendly abstractions.

## Key Features

ThreadsPipe-py offers a rich set of features to handle various aspects of Threads interactions:

- **Posting and Replying**: Create text posts, replies, quotes, and chained threads (e.g., split long text >500 characters or media >20 items into batches). Supports hashtags handling, reply controls (e.g., who can reply: everyone, followed accounts, or mentioned only), and geo-gating (restrict posts to specific countries like US, CA, NG—requires user permission).
  
- **Media Handling**: Upload images and videos from local paths, URLs, bytes, or base64 strings. Handles carousels (up to 20 media per post), captions per media, and automatic splitting for large batches. Local files are temporarily uploaded to GitHub for public URLs (requires GitHub fine-grained token).

- **Insights and Analytics**: Retrieve post-level metrics (views, likes, replies, reposts, quotes, shares) and user-level insights (followers count, demographics by country/city/age/gender). Supports date-range filtering (note: insights limited to post-April 13, 2024 data).

- **Data Retrieval**: Fetch user profiles, posts, replies (top-level or deep conversations), and quota usage. Includes methods for hiding/unhiding replies and generating shareable intent links (e.g., post or follow intents).

- **CLI Support**: Command-line tools for generating/refreshing access tokens and managing authentication.

- **Rate Limiting and Error Handling**: Automatic quota checks, optional waiting on limits, and detailed error responses with logging.

- **Advanced Options**: Customizable wait times for media processing (default 35s), API version selection, and environmental variable support for tokens.

## Supported Python Versions and Dependencies

- **Python Versions**: >=3.8 (tested up to latest stable releases).
  
- **Core Dependencies**:
  - `requests`: For API calls to Meta's Graph API.
  - `filetype`: For detecting and validating media types (images/videos).
  - `python-dotenv`: For loading environment variables (e.g., tokens).
  - `colorama`: For CLI color output (optional for core library).

Install via pip: `pip install threadspipepy`. For CLI features: `pip install threadspipepy[cli]`.

See [Installation Guide](/installation) for full setup instructions.

## Licensing

ThreadsPipe-py is licensed under the MIT License, allowing free use, modification, and distribution for personal or commercial projects. Review the full license in the [LICENSE file](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/LICENSE).

## Library Architecture

At its core, ThreadsPipe-py revolves around the `ThreadsPipe` class, which encapsulates all API interactions. Upon initialization, it stores authentication details (user ID, access token) and configurable options (e.g., GitHub params for media uploads, logging toggles).

### Main Components

- **ThreadsPipe Class**:
  - **Initialization**: Requires `user_id` and `access_token` (short- or long-lived from Meta). Optional params include `handle_hashtags` (auto-append extracted tags), `auto_handle_hashtags` (add tags if none present), GitHub credentials for local media, wait times for processing, and API version.
  - **Authentication Flow**: Use `get_auth_token` to open Meta's authorization window, then swap the code for tokens via CLI or manual exchange.
  - **Endpoints**: Internally maps to Meta's Graph API endpoints, such as:
    - Posting/Replies: `https://graph.threads.net/v1.0/me/threads?` (media creation) and `https://graph.threads.net/v1.0/me/threads_publish?` (publishing).
    - Insights: `https://graph.threads.net/v1.0/{user_id}/threads_insights?`.
    - Profile/Posts: `https://graph.threads.net/v1.0/me?fields=...` and `https://graph.threads.net/v1.0/me/threads?`.
    - Media Handling: Temporary GitHub uploads to `https://api.github.com/repos/{username}/{repo}/contents/` for public URLs.
  - **Key Methods**:
    - `pipe()`: Core method for posting/replying (handles text splitting, media batches, chaining).
    - `get_posts()`, `get_post()`, `get_post_replies()`: Data retrieval.
    - `get_post_insights()`, `get_user_insights()`: Analytics.
    - `__send_post__()` (internal): Builds and publishes media containers.
    - Utilities: `get_quota_usage()`, `hide_reply()`, `get_post_intent()`.

The architecture emphasizes modularity: private methods (`__handle_media__`, `__split_post__`, `__get_file_url__`) manage preprocessing (e.g., base64/URL validation, post splitting with tags), while public methods provide clean APIs. Error handling uses a consistent response format (dict with `message`, `body`, `is_error`).

For deeper dives, see [Advanced Features](/advanced-features) and the source code on GitHub.

## Quick Start

### Installation and Setup

1. Install the library: `pip install threadspipepy`.
2. Obtain Meta App credentials (App ID, Secret) from [Meta for Developers](https://developers.facebook.com/).
3. Generate access tokens (see [Initialization Guide](/initialization)).

### Example: Posting a Thread with Media

Here's a basic example from the library's README, demonstrating ease of use:

```python
from threadspipepy import ThreadsPipe

# Initialize (replace with your credentials)
api = ThreadsPipe(
    user_id="your_user_id",
    access_token="your_long_lived_access_token",
    handle_hashtags=True,  # Auto-extract and append hashtags
    gh_bearer_token="your_github_token",  # For local media uploads
    gh_repo_name="temp-uploads",
    gh_username="your_github_username"
)

# Post a long text thread with mixed media (URLs, local files, bytes)
result = api.pipe(
    post="This is a long post that exceeds 500 characters... [full text here]",
    files=[
        "/path/to/local-image.jpg",  # Local file (auto-uploads to GitHub)
        "https://example.com/video.mp4",  # Remote URL
        open("image.png", "rb").read()  # Bytes
    ],
    file_captions=["Caption for image 1", None, "Video description"],
    tags=["#Threads", "#Python"],  # Optional explicit tags
    who_can_reply="accounts_you_follow",  # Restrict replies
    allowed_country_codes=["US", "CA"],  # Geo-gate (if permitted)
    chained_post=True  # Auto-split into thread if needed
)

if not result.get("is_error"):
    print("Post successful:", result["id"])
else:
    print("Error:", result["message"])
```

This single `pipe()` call handles text splitting, media validation/upload, chaining (e.g., first post as root, rest as replies), and publishing after processing wait (default 35s). For replies, add `reply_to_id="target_post_id"`.

### Example: Fetching Insights

```python
# Get user insights (all metrics, last 30 days)
insights = api.get_user_insights(
    since_date="2024-09-01",
    until_date="2024-10-01",
    metrics="all",  # Or ["views", "likes"]
    follower_demographic_breakdown="country"
)
print(insights)  # JSON with views, followers, etc.

# Post-specific insights
post_insights = api.get_post_insights(
    post_id="your_post_id",
    metrics=["views", "likes", "shares"]
)
print(post_insights)
```

These examples highlight the library's simplicity—minimal boilerplate for complex operations. For more, explore [Posting and Replies](/posting-replies), [Media Handling](/media-handling), and [Retrieving Data](/retrieving-data).

## Next Steps

- [Installation](/installation): Full setup guide.
- [Initialization](/initialization): Authentication and config.
- [CLI Usage](/cli-usage): Token management from command line.
- [Testing and Contribution](/testing-contribution): Run tests and contribute.
- [Changelog](/changelog-reference): Release notes.
- [Reference](/advanced-features): API details.

Join the community on GitHub for issues, contributions, or support!
