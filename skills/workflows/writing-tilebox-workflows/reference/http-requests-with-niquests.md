# HTTP Requests With niquests

Prefer `niquests` for new non-storage HTTP calls in workflow code unless the project already standardizes on another client.

Use `obstore` for object storage. Use `niquests` for web APIs, metadata endpoints, service calls, webhooks, small downloads that are not object-store access, and streaming HTTP where an object-store abstraction is not appropriate.

## Session Pattern

Use sessions for repeated calls:

```python
import niquests

with niquests.Session(base_url="https://api.example.com") as session:
    response = session.get("/items", timeout=30)
    response.raise_for_status()
    data = response.json()
```

Async sessions are available when the surrounding code is already async:

```python
async with niquests.AsyncSession(base_url="https://api.example.com") as session:
    response = await session.get("/items", timeout=30)
```

## Rules

- Always set timeouts.
- Use sessions for connection pooling.
- Stream large request/response bodies instead of loading blindly.
- Log request identifiers and relevant status codes, not secrets.
- Use workflow task retries for transient service failures when idempotent.
- Add rate limiting when calling provider APIs at scale.
- Keep auth in environment/secrets, not task parameters.

## Avoid

- Bare HTTP calls for S3/GCS/Azure/R2 object storage; use `obstore`.
- Unbounded retries inside a task plus Tilebox retries outside the task.
- Downloading large files through HTTP if a COG/object-store range-read path exists.
- Hidden global sessions with mutable per-job auth state.

References: https://niquests.readthedocs.io/
