---
title: Testing and Contribution Guide
slug: testing-contribution
description: A guide to running tests, ensuring code quality, and contributing to the ThreadsPipe project, including pytest usage, coverage, linting, and best practices for pull requests.
sidebar_label: Testing & Contribution
sidebar_position: 9
---

# Testing and Contribution Guide

This guide covers how to run and write tests for the ThreadsPipe project, ensure code quality through linting and coverage, and contribute effectively. The project uses pytest for unit testing, with automated workflows in GitHub Actions to validate changes. Tests focus on core functionalities like post splitting, media handling (e.g., base64 checks and URL validation), and error handling in the `ThreadsPipe` class.

## Running Tests Locally

To maintain code quality, run tests locally before submitting changes. The project is set up for Python 3.8+ and uses standard tools like pytest.

### Installing Dependencies for Testing

First, install the development dependencies:

```bash
pip install -e .[dev]
```

This includes `pytest`, `pytest-cov` for coverage, and linting tools like `pylint` or `black` (as implied by the workflow).

### Running Pytest

Execute the unit tests with:

```bash
pytest
```

This runs all tests in the `tests/` directory, primarily `tests/test.py`. Key test examples include:

- **Post Splitting**: The `test_post_splitting_test` verifies the internal `__split_post__` method, which splits long text posts into 500-character batches (Threads API limit) while preserving tags. For instance, it asserts that a lengthy lorem ipsum text results in a list where initial batches are exactly 500 characters long:

  ```python
  # Example from test.py (abridged)
  post_text = "Lorem ipsum dolor sit amet... [long text exceeding 500 chars multiple times]"
  splitted_post = th_init.__split_post__(post=post_text, tags=["#tag1", "#tag2"])
  assert isinstance(splitted_post, list)
  assert len(splitted_post) > 0
  assert len(splitted_post[0]) == 500  # First batch
  for i in range(len(splitted_post) - 1):
      assert len(splitted_post[i]) == 500  # All but last
  ```

- **Base64 Checks**: `test___is_base64___method` tests the `__is_base64__` static method to detect valid Base64-encoded media (e.g., images). It uses a known Base64 GIF string and contrasts it with invalid text:

  ```python
  # Example from test.py
  b64_img = "R0lGODlhPQBEAPeoAJosM//AwO/AwHVYZ/z595kzAP/s7P+goOXMv8+fhw... [full Base64 string]"
  assert th_init.__is_base64__(b64_img) is True
  random_text = "hdbchdschsabchbcldschdbcdhcbsydgcdycbdlcbascba"
  assert th_init.__is_base64__(random_text) is False
  ```

- **URL Validation**: `test__is_url__method` uses a regex (`__file_url_reg__`) to validate remote file URLs for uploads, supporting HTTPS/HTTP, ports, and IPs while rejecting local paths:

  ```python
  # Example from test.py (abridged)
  urls = [
      ["https://example.com/image.png", True],
      ["/local/path.jpg", False],
      ["https://192.0.0.0:80/file.jpg", True]
  ]
  for url, expected in urls:
      assert bool(th_init.__file_url_reg__.fullmatch(url)) == expected
  ```

These tests ensure compliance with Threads API constraints, such as text limits and media formats.

To run a specific test file:

```bash
pytest tests/test.py -v
```

### Coverage Reports

Generate coverage reports to check test thoroughness:

```bash
pytest --cov=threadspipepy --cov-report=html
```

This measures coverage for the `threadspipepy` module (e.g., `threadspipe.py`), focusing on methods like `__split_post__`, `__is_base64__`, and `__handle_media__`. Aim for high coverage (>80%) on core logic. Open `htmlcov/index.html` in a browser to view detailed reports.

### Linting and Formatting

Run linting to enforce style and catch issues:

```bash
pylint threadspipepy/
black --check .
```

- `pylint` checks for PEP 8 compliance, unused imports, and potential bugs in files like `threadspipe.py` and `cli.py`.
- `black` ensures consistent formatting.

Fix issues automatically with `black .` if needed.

## GitHub Actions Workflows

The project uses GitHub Actions for CI/CD, defined in `.github/workflows/python-package.yml`. This workflow triggers on pushes and pull requests, running on Python 3.8–3.11.

- **Pytest Execution**: Runs `pytest tests/` to validate all unit tests, including those for initialization (`test_user_id_nd_access_token_type_error`), parameter updates (`test_update_param_method`), and media handling.
- **Coverage Integration**: Implicitly tracks coverage during pytest runs, reporting results to ensure key paths (e.g., error raising for invalid Base64 or URLs) are tested.
- **Linting**: Executes linting tools to scan the codebase for style violations and errors before allowing merges.

View workflow status via the repository's Actions tab. Failed builds block PR merges, so always run local equivalents first. The workflow badge in the README confirms passing status.

:::tip
If a workflow fails, check the logs for specific test or lint errors. Common issues include unhandled edge cases in post splitting or media validation.
:::

## Writing Tests

Extend `tests/test.py` for new features. Use simple function-based tests with `pytest.raises` for errors and assertions for outputs. Tests currently focus on internal methods without external dependencies.

### Tips for Mocking API Calls

The existing tests do not mock API interactions (e.g., Threads or GitHub endpoints in `pipe` or `__upload_to_github__`), relying on direct instantiation. To test API logic without live calls:

- Use `unittest.mock` to patch requests:

  ```python
  from unittest.mock import patch, MagicMock

  @patch('requests.post')
  def test_pipe_method(mock_post):
      mock_post.return_value.json.return_value = {'id': 'post123'}
      th = ThreadsPipe(user_id='test', access_token='test')
      result = th.pipe(text='Hello')
      assert result['id'] == 'post123'
      mock_post.assert_called_once()
  ```

- For more advanced HTTP mocking, install `responses` (`pip install responses`) and use it in fixtures:

  ```python
  import responses

  def test_api_call():
      with responses.RequestsMock() as rsps:
          rsps.add(rsps.POST, 'https://graph.threads.net/v1.0/me/threads', json={'id': '123'})
          th = ThreadsPipe(user_id='test', access_token='test')
          th.pipe(text='Test')
      assert len(rsps.calls) == 1
  ```

This isolates tests from network flakiness and rate limits.

### Testing Local File Uploads

Local files (e.g., paths like `/path/to/image.jpg`) trigger GitHub temporary uploads before Threads posting. Current tests validate paths via URL checks but don't simulate uploads. To test:

- Create temporary files using `tempfile`:

  ```python
  import tempfile
  import os

  def test_local_upload():
      with tempfile.NamedTemporaryFile(suffix='.jpg', delete=False) as tmp:
          tmp.write(b'fake image data')
          tmp_path = tmp.name

      th = ThreadsPipe(user_id='test', access_token='test', gh_username='test', gh_repo_name='test', gh_bearer_token='test')
      # Mock the upload and deletion methods if needed
      with patch.object(th, '_ThreadsPipe__upload_to_github') as mock_upload:
          mock_upload.return_value = 'https://github.com/uploaded.jpg'
          result = th.pipe(media=[tmp_path], text='Test with local file')
          mock_upload.assert_called_once_with(tmp_path)
      os.unlink(tmp_path)  # Clean up
  ```

- Mock `__upload_to_github__` and `__delete_uploaded_files__` to verify file handling without actual GitHub interactions. Ensure tests cover Base64 detection for embedded media and deletion on post failures.

Add new tests to `test.py` and run `pytest` to verify.

## Contributing

Contributions are welcome! The project follows standard open-source practices to keep the codebase stable and aligned with Threads API updates.

### Getting Started

1. **Fork the Repository**: Create a fork on GitHub to work in your own copy.
2. **Clone and Set Up**: 
   ```bash
   git clone https://github.com/yourusername/ThreadsPipe-py.git
   cd ThreadsPipe-py
   pip install -e .[dev]
   ```
3. **Create a Branch**: Use descriptive names for feature branches:
   ```bash
   git checkout -b feature/add-new-media-support
   ```

### Making Changes

- Implement features or fixes in `threadspipe.py`, `cli.py`, or add tests in `tests/`.
- Update docstrings and examples in `README.md`.
- Run tests, coverage, and linting locally to ensure everything passes.
- Reference relevant sections of `tests/test.py` for patterns (e.g., splitting long posts or validating media).

### Submitting Pull Requests (PRs)

1. Commit changes with clear messages (e.g., "Fix: Add dots to split posts when tags exceed limit").
2. Push to your fork:
   ```bash
   git push origin feature/your-branch
   ```
3. Open a PR on the main repository via GitHub. Describe the changes, reference any issues, and link to tests added.
4. Ensure the GitHub Actions workflow passes (pytest, coverage, linting).

PRs should be focused: one feature or fix per PR. Reviewers will check for API compatibility and test coverage.

### Updating the Changelog

All changes must update `CHANGELOG.md` in reverse-chronological order. Follow the existing format:

- Add a new section like `## [Unreleased] - YYYY-MM-DD` at the top.
- Use subsections: **Added** for new features (e.g., "Added support for quoting posts"), **Fix** for bugs (e.g., "Fix: Improved base64 detection for binary files").
- Reference specific methods or files (e.g., "Updated `__split_post__` to handle tags better").
- For releases, bump the version in `pyproject.toml` and move the unreleased section to the new version.

Example entry:
```
## [0.5.0] - 2025-04-01

### Added
- Support for mocking API calls in tests.

### Fix
- Enhanced URL validation for IPv6 addresses.
```

This ensures users can track evolution, from initial posting (v0.1.0) to advanced features like reposts (v0.3.0).

:::caution
Do not commit directly to `main`. Always use branches and PRs. Semantic versioning guides releases: patches for fixes, minors for features.
:::

For questions, open an issue on GitHub. Thanks for contributing to ThreadsPipe!