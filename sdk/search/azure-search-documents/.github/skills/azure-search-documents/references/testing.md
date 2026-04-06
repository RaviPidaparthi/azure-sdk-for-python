# Testing Guide

Tests use `pytest` plus the [Test Proxy](https://github.com/Azure/azure-sdk-tools/tree/main/tools/test-proxy/Azure.Sdk.Tools.TestProxy) for recording and playback.

## Test Types

- **Unit tests** (no `_live` suffix): Pure logic tests. No HTTP calls, no recordings, no service needed.
- **Live tests** (`_live` suffix): Make HTTP calls to Azure Search. Run in two modes:
  - **Playback** (default): Replays previously recorded HTTP interactions. Fast, offline.
  - **Live**: Set `AZURE_TEST_RUN_LIVE=true`. Runs against a real service and captures new recordings.

New live tests will fail in playback until you generate recordings by running them live.

## Running Tests

**Unit tests only** (no HTTP, no recordings):
```powershell
python -m pytest tests/ --ignore-glob="*_live*"
```

**Live tests in playback** (replays existing recordings):
```powershell
python -m pytest tests/ -k "live"
```

**Live tests against real service** (captures new recordings):
1. Set up environment: `.\.github\skills\azure-search-documents\scripts\env.ps1`
2. Run:
   ```powershell
   $env:AZURE_TEST_RUN_LIVE = "true"
   pytest tests/ -k "live"
   ```
3. Push recordings: `test-proxy push -a assets.json`
4. Include updated `assets.json` in your PR.

If test files are moved, recordings must be re-captured because the test proxy derives recording paths from file locations.

## File Naming Convention

| Pattern | Example | Description |
|---------|---------|-------------|
| `test_<name>.py` | `test_search_client.py` | Sync unit test |
| `test_<name>_async.py` | `test_search_client_async.py` | Async unit test |
| `test_<name>_live.py` | `test_search_client_basic_live.py` | Sync live test (playback or real service) |
| `test_<name>_live_async.py` | `test_search_client_basic_live_async.py` | Async live test (playback or real service) |

All test files live flat in `tests/`, no subfolders.

## Writing New Tests

Always create both sync and async versions together.
