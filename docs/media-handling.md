---
title: Media Handling and File Uploads
slug: media-handling
description: Learn how ThreadsPipe handles media attachments, including supported formats, GitHub uploads for local files, captioning, carousels, validation, and troubleshooting.
sidebar_label: Media Handling
sidebar_position: 5
---

# Media Handling and File Uploads in ThreadsPipe

ThreadsPipe provides robust support for attaching media to posts and replies on Threads. This includes handling various input formats, automatic uploads for local files to a temporary GitHub repository, captioning for accessibility, and creating carousel-style multi-media posts. The system enforces Threads API limits (e.g., up to 20 media items per post) and automatically splits excess content into chained threads if needed.

This page details the core logic from `threadspipe.py`, including the `__handle_media__` method, file validation, URL checks, cleanup processes, examples, and troubleshooting tips.

## Supported Formats

ThreadsPipe accepts media in multiple formats within a list passed to methods like `post()` or `reply()`. Supported types include:

- **URLs (strings)**: Direct links to publicly accessible remote files. Examples:
  - Full HTTPS: `https://example.com/photo.jpg`
  - HTTP with IP and port: `http://192.168.1.1:8080/video.mp4`
  - Scheme-less domains: `example.com/path/to/image.png`
  - Supports query parameters: `https://unsplash.com/photo.jpg?w=800&q=80`

  No upload is required; the URL is validated and used directly.

- **Local Paths (strings)**: File system paths to existing files. Examples:
  - Unix: `/path/to/local/img.jpg`
  - Windows: `C:\Users\file.mp4`

  These are automatically uploaded to a temporary GitHub repository for a public URL.

- **Bytes**: Raw binary data, such as from `open('file.jpg', 'rb').read()`. Treated like local files and uploaded to GitHub.

- **Base64 (strings)**: Encoded binary data (e.g., base64-encoded images). Decoded to bytes and uploaded to GitHub.

Only image (e.g., JPG, PNG) and video (e.g., MP4) files are supported. Mixed types in a single list are fully compatible, processed sequentially.

:::tip
Limits: Up to 20 media items per post. Exceeding this triggers automatic chaining into reply threads. No explicit size limits, but Threads API enforces them (e.g., 512MB for videos).
:::

## Automatic GitHub Temporary Upload for Local Files

For local paths, bytes, or base64 inputs, ThreadsPipe uploads files to a user-specified GitHub repository to generate temporary public URLs. This is handled in the `__get_file_url__` method.

### Requirements
- A public GitHub repository (set via `gh_repo_name`, e.g., `'temp-uploads'`).
- GitHub username (`gh_username`).
- A fine-grained personal access token (`gh_bearer_token`) with repository contents read/write/delete permissions.
- Optional: API version (`gh_api_version`, default: `"2022-11-28"`).

These can be set during initialization:
```python
from threadspipe import ThreadsPipe

pipe = ThreadsPipe(
    access_token="your_threads_token",
    gh_username="your_github_username",
    gh_repo_name="your_temp_repo",
    gh_bearer_token="your_github_token"
)
```

If not provided, local uploads fail with an error message guiding token setup.

### Upload Process
1. **Validation**: Uses the `filetype` library to confirm the MIME type (e.g., `image/jpeg` or `video/mp4`).
2. **Filename**: Generates a random 10-character prefix + extension (e.g., `abc123defg.jpg`).
3. **Upload**:
   - Reads file content (from path or bytes).
   - Base64-encodes it.
   - Sends a PUT request to the GitHub Contents API: `https://api.github.com/repos/{gh_username}/{gh_repo_name}/contents/{filename}`.
   - Payload includes a timestamped commit message (e.g., `"ThreadsPipe: At 2023-10-01T12:00:00, Uploaded a file named abc123defg.jpg"`).
   - Timeout: Configurable via `gh_upload_timeout` (default: 300 seconds).
4. **Success**: Stores the download URL, SHA hash, and API link for cleanup.
5. **Failure**: Logs the error (e.g., rate limit or invalid token) and triggers cleanup of prior uploads.

This process ensures local files are accessible via public GitHub URLs, compatible with the Threads API.

## Captioning

Captions (alt text) improve accessibility and can be attached to individual media items.

- **Input**: A list of strings or `None`, matching the media list length (shorter lists are padded with `None`).
  ```python
  file_captions = ['Caption for first image', None, 'Description for video']
  ```
- **Application**: Added as `alt_text` query parameters during media container creation.
- **Batching**: For chained posts, captions are split along with media (e.g., first 20 get the first 20 captions).
- **Carousels**: Applied per child item.

If no caption is provided for an item, it's skipped.

## Carousel Posts

For posts with multiple media items (>1), ThreadsPipe automatically creates a carousel (multi-image/video post).

- **Process** (in `__send_post__`):
  1. Create individual media containers for each item:
     - Parameters: `media_type=IMAGE` or `VIDEO`, `image_url` or `video_url`, `is_carousel_item=true`, `alt_text` (if provided).
  2. Poll each container's status until `"FINISHED"` (default wait: 35 seconds, configurable via `media_item_publish_wait_time`).
  3. Assemble a parent container: `media_type=CAROUSEL`, `children=id1,id2,...`.
  4. Append post text, reply settings, etc., to the parent.
  5. Publish the parent and poll until `"FINISHED"` (wait: `post_publish_wait_time`, default 35s).

- **Limits**: Up to 20 children per carousel. Excess items are split into chained carousels or single posts.
- **Mixed Media**: Supports images and videos in the same carousel.
- **Errors**: If a child fails (e.g., invalid URL), the entire process aborts with cleanup.

Enable chaining for long threads: `pipe.chaining_enabled = True` (default).

## Detailed Logic: __handle_media__

This private method processes the media list, returning a list of validated media dictionaries or an error.

### Key Steps
1. **Initialization**: Clears internal state (`__handled_media__ = []`).
2. **Loop Over Files** (indexed at `i`):
   - **URL Check**: Matches against `__file_url_reg__` regex (see below). If matched:
     - Extract extension (e.g., `.jpg`).
     - Guess MIME with `filetype`.
     - Fallback: HEAD request for `Content-Type` header.
     - Validate status (≤200) and type (IMAGE/VIDEO). Reject otherwise (e.g., "Filetype of the file at index {i} is invalid").
     - Add `{'type': 'IMAGE' or 'VIDEO', 'url': file}`.
   - **Base64 Check**: Regex for base64 chars + decode attempt. If valid, decode to bytes and route to upload.
   - **Local/Bytes**: Check `os.path.exists` for paths or direct bytes. Validate with `filetype.is_image`/`is_video`. Reject non-media (e.g., "Provided file at index {i} must be either an image or video file").
   - **Invalid**: Any other input rejects with "Provided file at index {i} is invalid".
3. **Batching**: If >20 items and chaining enabled, splits into batches (e.g., `files[0:20]`, `files[20:40]`).
4. **GitHub Check**: Ensures credentials before uploads.
5. **Output**: Processed media list or error dict.

Captions are batched similarly.

## File Type Validation

- **Library**: `filetype` for MIME detection (e.g., `filetype.guess_mime(file_bytes)` or `filetype.get_type(extension='.jpg')`).
- **Supported**: Images (e.g., JPEG, PNG, GIF) and videos (e.g., MP4, MOV). Uppercased to `IMAGE`/`VIDEO`.
- **Rejection**: Non-media types (e.g., `text/plain`) trigger early errors and cleanup of prior uploads.
- **Edge Cases**: No extension? Fallback to HEAD request. Unreachable URL? "File at index {i} could not be found so its type could not be determined".

Install via: `pip install filetype`.

## URL Regex Checks

The `__file_url_reg__` (compiled regex) identifies URLs:
- Pattern: Matches optional schemes (http/https), domains/IPs (e.g., `[a-z0-9-]+\.(com|org)`), optional ports (`:\d+`), paths, and queries (`\?[a-z0-9.&=#-_%?]+`).
- Examples:
  - Valid: `https://example.com:8080/path.jpg?param=val`, `192.168.1.1/video.mp4`, `domain.com/file.png`.
  - Invalid: Local paths (`/local/file.jpg`), malformed (`https://example`).
- Non-matches route to local/base64/bytes handling.

## Cleanup: __delete_uploaded_files__

Ensures no temporary files linger on GitHub.

- **Triggers**: After successful publish or on any error (upload fail, invalid type, rate limit).
- **Process**:
  1. Loop over `__handled_media__`.
  2. DELETE request to GitHub API using stored `'_link'` and `sha` (e.g., `https://api.github.com/repos/{user}/{repo}/contents/{file}?sha={sha}`).
  3. Commit message: Timestamped deletion note.
  4. Timeout: `gh_upload_timeout`.
- **Error Handling**: Logs failures (e.g., "File at index {i} was not deleted... status_code: {code}") but continues.
- **Reset**: Clears `__handled_media__` after processing.
- **Scope**: Only GitHub-uploaded files (ignores remote URLs).

## Examples: Mixed Media Types

Here's a complete example with mixed formats:

```python
from threadspipe import ThreadsPipe
import base64

# Initialize with GitHub creds for uploads
pipe = ThreadsPipe(
    access_token="your_threads_token",
    gh_username="your_username",
    gh_repo_name="temp-uploads",
    gh_bearer_token="your_github_token"
)

# Mixed media list
files = [
    "/path/to/local/img-1.jpg",                    # Local path → Upload
    "https://example.com/video.mp4",               # URL → Direct
    open('/path/to/img-2.jpg', 'rb').read(),       # Bytes → Upload
    base64.b64encode(open('img-3.png', 'rb').read()).decode('utf-8'),  # Base64 → Decode + Upload
    "https://unsplash.com/photo.jpg?w=800"         # URL with query → Direct
]

file_captions = [
    'Local image caption',
    None,
    'Bytes image description',
    'Base64 alt text',
    None
]

# Post with carousel (if >1 media)
post_id = pipe.post(
    text="Check out this mixed media post!",
    files=files,
    file_captions=file_captions
)

print(f"Posted with ID: {post_id}")
```

- **Result**: Processed into media dicts with GitHub URLs for locals. Creates a carousel post.
- **Chaining Example**: For 25 files, the first post uses files[0:20]; replies chain the rest.

## Troubleshooting Upload Timeouts/Errors

Common issues and fixes from `threadspipe.py`:

- **Timeouts**:
  - Symptom: "Timeout while uploading" in logs.
  - Fix: Increase `gh_upload_timeout` (e.g., `pipe.update_param(gh_upload_timeout=600)` for 10min). Useful for large videos or slow networks.

- **GitHub Auth/Repo Errors**:
  - Symptom: "To handle local file uploads... provide your GitHub fine-grained access token".
  - Fix: Verify token permissions (repo contents: write/delete). Ensure repo is public and exists. Test via GitHub API directly.

- **Invalid File/Type**:
  - Symptom: "Provided file at index {i} must be either an image or video file, {mime} given".
  - Fix: Use `filetype` to check MIME locally. Ensure only images/videos.

- **Upload Failures (>201 Status)**:
  - Symptom: Logs full GitHub JSON (e.g., rate limit: "API rate limit exceeded").
  - Fix: Enable `wait_on_rate_limit=True` to pause instead of fail. Monitor with `pipe.get_quota_usage()`. Cleanup runs automatically.

- **URL Issues**:
  - Symptom: "File at index {i} could not be found" (HEAD >200).
  - Fix: Ensure public access; test URL in browser.

- **Base64 Invalid**:
  - Symptom: Treated as invalid string.
  - Fix: Validate with regex + decode try. Use proper encoding.

- **Carousel/Publish Errors**:
  - Symptom: Polling logs "ERROR" status with `error_message`.
  - Fix: Check response details. Increase wait times (e.g., `media_item_publish_wait_time=60`). Verify media URLs.

- **General Tips**:
  - Enable debug logging for full traces.
  - Use `check_rate_limit_before_post=True` (default) to pre-check.
  - For chaining fails, inspect first post ID as `reply_to_id`.
  - Test small batches first.

For more, see the [API Reference](../advanced-features) or [Changelog](../changelog-reference). If issues persist, check GitHub issues or contribute via [Testing & Contribution](../testing-contribution).