# SDK review checklist

Mark each item **pass / fail / n.a.** with evidence (`file:line` or the call you wrote).

## Getting started
- [ ] Install + first successful call in ≤ 5 lines, with credentials from an env var.
- [ ] Missing credentials produce an error that names the env var / option.
- [ ] README quickstart compiles against the current version.

## Surface
- [ ] Resources and verbs are consistent (`create/retrieve/update/list/delete`) across the SDK.
- [ ] Path params positional, body/query a single object, request options last and optional.
- [ ] Every param and response type is exported and named; no `any` / `interface{}` in public signatures.
- [ ] Server-controlled enums are open; unknown values decode without error.
- [ ] Optional vs nullable vs zero-value modelled correctly (Go: explicit zero values are sendable).
- [ ] Internals are not exported (TS: `exports` map; Go: `internal/` packages).

## Transport defaults
- [ ] Every request has a timeout; overridable per client and per request.
- [ ] Retries with backoff + jitter on connection errors, 408/409/429/5xx; `Retry-After` honoured.
- [ ] Non-idempotent requests retried only with an idempotency key.
- [ ] Cancellation works (TS `AbortSignal`; Go `context.Context` first arg).
- [ ] Transport is injectable (`fetch` / `*http.Client`).
- [ ] `User-Agent` identifies SDK + version + runtime.
- [ ] Auth headers and secrets never appear in logs or error messages.

## Pagination and streaming
- [ ] `list` supports auto-paging iteration and single-page access.
- [ ] Streams use native async iteration and expose close/cancel; resources are released on early exit.

## Errors
- [ ] Hierarchy distinguishes connection, timeout, abort and API errors; API errors split by the statuses callers branch on.
- [ ] Every API error carries status, code, message, request ID, headers, raw body.
- [ ] Unexpected error bodies (HTML, empty, non-JSON) still produce a proper SDK error.
- [ ] Go: errors wrap with `%w`; `errors.As` / `errors.Is` work. TS: `instanceof` works and `name` is set.

## Escape hatches
- [ ] Raw response (status, headers) reachable for any call.
- [ ] Generic request method for endpoints the SDK lacks.
- [ ] Hooks / middleware or logger for observability.

## Versioning
- [ ] Semver followed; API version pinned per SDK release and overridable.
- [ ] Deprecations say what to use instead and follow the documented removal window.
- [ ] Public API surface tracked in CI (`api-extractor` / `gorelease`).
- [ ] Changelog with Added / Changed / Deprecated / Removed / Fixed.

## Packaging and tests
- [ ] TS: ESM + CJS (or deliberate ESM-only) both tested; `sideEffects: false`; minimal deps; `engines` set.
- [ ] JS: types shipped (`.d.ts` from JSDoc or TS); required args validated at runtime with errors that name the parameter; unknown option keys warned.
- [ ] Go: module path carries major version ≥2; supports last two Go releases; minimal deps.
- [ ] Test layers present: unit with mock transport, recorded HTTP, contract or live smoke, compiled examples.
- [ ] Retry/timeout tests are deterministic (fake timers / injected clock).
