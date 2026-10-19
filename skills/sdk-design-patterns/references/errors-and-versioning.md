# Errors and versioning

## Error hierarchy

Shape it by what callers branch on, not by every status code.

```
AcmeError                       base; everything the SDK throws/returns
├── APIConnectionError          no response: DNS, TLS, reset
│   └── APITimeoutError
├── APIUserAbortError           caller cancelled (TS AbortSignal; Go uses ctx.Err())
└── APIError                    got an HTTP response with an error status
    ├── BadRequestError         400
    ├── AuthenticationError     401
    ├── PermissionDeniedError   403
    ├── NotFoundError           404
    ├── ConflictError           409
    ├── UnprocessableEntityError 422
    ├── RateLimitError          429  (+ retryAfter)
    └── InternalServerError     ≥500
```

In Go, collapse this to one `*Error` type plus helpers; the TS class tree is idiomatic there, not in Go.

Every `APIError` carries:
- `status`, `code` (machine-readable API error code), `message` (human, from the server)
- `requestId` (from the response header) — the single most useful field for support tickets; include it in the message string
- `headers` and the raw parsed `body`
- for validation errors: field-level details if the API provides them

Error messages: `404 invoice_not_found: No invoice with id inv_123 (request_id: req_abc)`. Never include the API key or full request body.

### RFC 9457 problem details

RFC 9457 obsoletes RFC 7807; the `application/problem+json` shape is the same. When the API returns it, map:

| Problem field | SDK error field |
|---|---|
| `type` (URI) | `code` (or `type`), stable for branching |
| `title` | short summary |
| `status` | `status` |
| `detail` | `message` |
| `instance` | `instance` / link to the occurrence |
| extension members | typed fields if documented, else kept in `body` |

Parse problem+json and the API's legacy error shape both; fall back to the raw text body when neither parses. An SDK error must never fail to construct because the body was unexpected.

### Retryability

Decide retry in one place (status + error type + idempotency), and expose `err.retryable` or the equivalent so callers with their own retry loops agree with the SDK. Rate-limit errors expose `retryAfter`.

## Versioning

**SDK version** follows semver. **API version** is a separate axis:
- If the API is versioned by header or date (`Acme-Version: 2026-10-01`), each SDK release pins one API version and sends it on every request; callers can override per client.
- Bumping the pinned API version is a breaking SDK change if response shapes change.

### What counts as breaking

| Change | Breaking? |
|---|---|
| Remove/rename a method, type, field or option | Yes |
| Add a required parameter | Yes |
| Narrow an accepted param type | Yes |
| Change a default (timeout, retries, pagination size) | Behavioural break; treat as major or flag loudly |
| Add a value to a **closed** enum/union in a response | Yes (breaks exhaustive switches) — hence open enums |
| Add an optional param / new method / new response field | No |
| Make a required param optional | No |
| Add a value to an **open** enum | No |

### Deprecation workflow

1. Ship the replacement first, in a minor release.
2. Mark the old symbol: JSDoc `@deprecated Use invoices.finalize()` / Go `// Deprecated: Use Invoices.Finalize.` Say what to use, not just that it's old.
3. Optionally warn once at runtime (behind a logger, never `console.warn` spam in hot paths).
4. Changelog entry under **Deprecated**; migration snippet with before/after.
5. Keep it for at least one minor release and a stated period (e.g. 6 months).
6. Remove in the next major, with a migration guide; a codemod if the change touches many call sites.

When the API itself sunsets an endpoint (`Deprecation` / `Sunset` headers, RFC 8594/9745), surface them through the logger so callers see it before the endpoint disappears.
