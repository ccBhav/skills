---
name: sdk-design-patterns
description: Design or review a client SDK / client library for an API (REST, gRPC, WebSocket) in TypeScript, JavaScript or Go. Use when building a new SDK, wrapping an API in a client package, generating SDKs from an OpenAPI spec (Stainless, Speakeasy, Fern, openapi-generator, oapi-codegen), designing SDK error types, versioning or deprecating SDK methods, testing an SDK, or auditing an existing SDK's ergonomics.
---

# SDK Design Patterns

An SDK is a product whose users are developers. Its job is the **pit of success**: the obvious call is the correct one, and the safe behaviour (timeouts, retries, typed errors) happens without being asked for. Every public symbol is a versioning commitment, so the public surface stays as small as the use cases allow.

Two branches: **Design** (building or restructuring an SDK) and **Review** (auditing an existing one). Both judge against the principles below.

## Principles

**Progressive disclosure.** The common task is one line with zero config beyond credentials; power lives in optional arguments, never in required setup.

```ts
const client = new Acme();                       // reads ACME_API_KEY
const inv = await client.invoices.create({ customerId, amount: 1000 });
```

- Credentials fall back to a documented env var; base URL, timeout, retries, HTTP client are optional.
- Per-request overrides (timeout, retries, headers, idempotency key, abort/cancel) ride on an optional last argument.

**Resource-oriented surface.** `client.<resource>.<verb>()`, nested for sub-resources (`client.invoices.lineItems.list(invoiceId)`). Verbs are consistent across resources: `create / retrieve / update / list / delete`, plus named actions (`invoices.finalize`). One spelling per concept across the whole SDK.

**Types are the docs.** Every request and response is a named, exported type; IDE autocomplete should be enough to write a correct call. Server enums are **open** (unknown values pass through) so a new server value is never a breaking change for the SDK.

**Safe defaults.**
- Timeout on every request (e.g. 60s), overridable.
- Automatic retries (e.g. 2) with exponential backoff + jitter, on connection errors, 408, 409 lock conflicts, 429 and 5xx; honour `Retry-After`. Retry non-idempotent requests only when an idempotency key is attached — generate one automatically for retried POSTs if the API supports it.
- `User-Agent: <sdk>/<version> <lang>/<runtime-version>` so the API owner can see SDK adoption.

**Pagination as iteration.** `list()` returns something the caller can loop over that fetches pages lazily, plus explicit single-page access (`.page()`, `hasNextPage`, cursor) for callers who batch.

**Typed errors.** A small hierarchy keyed on what callers branch on (status class, retryability), each carrying status, error code, message, request ID and the raw body. Details: [references/errors-and-versioning.md](references/errors-and-versioning.md).

**Escape hatches.** Every layer of abstraction has a way out:
- raw HTTP response (headers, status) alongside the parsed value;
- injectable transport (`fetch` / `*http.Client`) for proxies, tracing, tests;
- a generic `client.request(method, path, ...)` for endpoints the SDK doesn't cover yet;
- hooks/middleware for logging and telemetry.

**Name the pattern.** Most SDK parts are classic design patterns (Facade client, Iterator pager, Chain of Responsibility middleware). Use the names and the matching code smells in design docs and review findings: [references/design-patterns.md](references/design-patterns.md).

**Streaming and realtime** (SSE, WebSocket, gRPC streams) surface as the language's native async iteration with explicit close/cancel, and typed events in a discriminated union.

## Design

1. **Pin the source of truth.** Find the API contract (OpenAPI spec, `.proto`, or docs). Decide generated, hand-written or hybrid using [references/codegen.md](references/codegen.md). Done when the choice is made and the contract's gaps (missing operationIds, untyped schemas, undocumented errors) are listed.
2. **Write the README first.** Draft quickstart code for the five most common tasks before any implementation. Done when each is ≤ ~5 lines, needs no config beyond the API key, and reads naturally out loud.
3. **Shape the surface** in the target language: [references/typescript.md](references/typescript.md), [references/javascript.md](references/javascript.md) (plain-JS source or JS callers; read alongside typescript.md) or [references/go.md](references/go.md). Done when every resource, verb, option and return type in the README compiles against the planned types.
4. **Errors and versioning**: [references/errors-and-versioning.md](references/errors-and-versioning.md). Done when the error hierarchy, the API-version pinning and the deprecation path are written down.
5. **Testing and release**: [references/testing.md](references/testing.md). Done when each test layer has a tool and a CI trigger.
6. **Self-review** the design against [references/review-checklist.md](references/review-checklist.md). Done when every item is pass or consciously waived with a reason.

## Review

1. Map the **public surface**: package entry points / exports (`index.ts`, `package.json` `exports`), or exported Go identifiers (`go doc -all ./...`). Everything else is internal and cheaper to change.
2. Write the quickstart for the 3 most common tasks using only the SDK's public API. Friction here (required config, casts, manual pagination, string-matching errors) is the headline finding.
3. Apply every item in [references/review-checklist.md](references/review-checklist.md), reading the language reference for idiom checks.
4. Report findings with `file:line`, grouped by severity:
   - **breaking-risk**: things that will force a major version or already broke one (closed enums, leaked internals, required params added);
   - **correctness**: unsafe retries, missing timeouts, swallowed errors, leaked secrets in logs;
   - **ergonomics**: friction on the common path;
   - **nit**.

   Where a finding matches a code smell in [references/design-patterns.md](references/design-patterns.md), name it and link the fix.

   For each, show the current call site and the proposed one. Done when every checklist item is marked pass / fail / n.a. with evidence.
