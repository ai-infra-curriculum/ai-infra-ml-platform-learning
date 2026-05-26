# Exercise 04: Build a Python SDK over the Generated Client

## Objective

Start with the openapi-generator-produced low-level Python client. Wrap it
with a thin "high-level" SDK that has the ergonomic features generators don't
add automatically.

## Requirements

The high-level SDK must include:

1. **Retry with backoff** on idempotent operations + 5xx responses
2. **Pagination iterator**: `for job in client.list_jobs(): ...` transparently fetches subsequent pages
3. **Auth helper**: reads API key from `ML_PLATFORM_API_KEY` env var; injects header
4. **Idempotency-key auto-generation**: opt-out per call
5. **Streaming progress**: `client.submit_and_wait(spec, on_status=lambda s: ...)`
6. **Typed exceptions**: `PermissionError`, `QuotaExceededError`, `JobFailedError`

## Deliverables

- `pyproject.toml` packaging the SDK
- Tests using `httpx.MockTransport` (no live server needed)
- README with quickstart + auth + common workflows

## Acceptance

- `pip install -e .` works
- All tests pass
- `from ml_platform import Client; Client().jobs.submit({...})` reads like Python, not Java
