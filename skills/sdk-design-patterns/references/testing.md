# Testing an SDK

Test layers, cheapest first. Each catches a failure the others miss.

| Layer | Catches | TypeScript | Go |
|---|---|---|---|
| **Unit (mock transport)** | request building: path, method, query, headers, body serialization; retry/backoff logic; error mapping | inject a fake `fetch`; `msw` | `httptest.Server` or a custom `http.RoundTripper` |
| **Recorded HTTP (VCR)** | response decoding against real payloads | Polly.js, `nock` back mode | `go-vcr` (`gopkg.in/dnaeon/go-vcr.v4`) |
| **Contract** | drift between SDK and spec | run against a Prism mock server generated from the OpenAPI spec | same, Prism in CI |
| **Live smoke** | auth, environments, real behaviour | small suite against a sandbox, nightly + pre-release | same |
| **Docs/examples** | README and examples rot | typecheck every snippet (extract to `examples/`, `tsc --noEmit`) | `Example*` test functions (compiled and run by `go test`) |
| **Public API surface** | accidental breaking changes | `api-extractor` report diff | `gorelease` / `apidiff` |

## Retries and timeouts

Test them deterministically: fake timers (`vi.useFakeTimers()` / Jest fake timers) or an injectable clock/sleep in Go. Assert:
- 429 with `Retry-After` waits that long;
- 5xx retries `maxRetries` times then throws `InternalServerError` with the last request ID;
- POST without idempotency key is not retried (or gets one automatically, per your design);
- timeout throws `APITimeoutError`, and abort throws the abort error, not a retry.

## Recordings

- Scrub secrets in a before-save hook: `Authorization`, cookies, API keys in query strings, PII in bodies. Review cassettes in PRs like code.
- Match on method + path + normalized body; ignore volatile headers.
- Re-record on a schedule (e.g. weekly CI job against sandbox) so cassettes don't fossilize an old API.

## Matrix

- TS: every supported Node LTS, plus Bun / Deno / a browser bundle / edge runtime if you claim support. Test the CJS and ESM entry points separately.
- Go: the last two Go minor releases; `go test -race`.
- Run the matrix on PRs touching transport or packaging; the fast suite on everything else.
