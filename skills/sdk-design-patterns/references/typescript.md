# TypeScript SDK idioms

## Client and configuration

```ts
export interface ClientOptions {
  apiKey?: string;            // default: process.env.ACME_API_KEY
  baseURL?: string;           // default: process.env.ACME_BASE_URL ?? 'https://api.acme.com'
  timeout?: number;           // ms, default 60_000
  maxRetries?: number;        // default 2
  fetch?: typeof fetch;       // injectable transport
  defaultHeaders?: Record<string, string>;
  logger?: Logger;            // or a logLevel; never log auth headers
}

export class Acme {
  invoices: Invoices;
  constructor(opts: ClientOptions = {}) { /* throw a clear error if no apiKey */ }
  withOptions(opts: Partial<ClientOptions>): Acme { /* derived client, shared transport */ }
}
```

- Throw at construction if credentials are missing, naming the env var — not on first request.
- Read `process.env` behind a guard so the SDK still loads in browsers and edge runtimes.
- Browser use with a secret key is usually a mistake: require an explicit opt-in flag (`dangerouslyAllowBrowser: true`).

## Requests

- Method signature: `(pathParams..., body/query?, requestOptions?)`. Path params positional (they're always required and few); body/query a single object.
- `RequestOptions`: `{ timeout?, maxRetries?, signal?: AbortSignal, headers?, idempotencyKey? }`.
- Return a promise subclass that also exposes the raw response without a second call:
  ```ts
  const inv = await client.invoices.retrieve(id);
  const { data, response } = await client.invoices.retrieve(id).withResponse();
  ```
- Use `fetch` (Node ≥18, Bun, Deno, browsers, edge). No axios/node-fetch dependency.

## Types

- Export every param and response type, namespaced by resource: `Acme.InvoiceCreateParams`, `Acme.Invoice`.
- String literal unions, not `enum` (enums don't tree-shake and break structural typing).
- **Open unions** for server-controlled values: `status: 'draft' | 'open' | 'paid' | (string & {})` keeps autocomplete and survives new values.
- Discriminated unions for polymorphic payloads (`type` field), so `switch` narrows.
- Optional vs nullable are different: `field?: T` = may be absent; `field: T | null` = present and null. Match the spec exactly.
- Dates: keep ISO strings in the types unless you parse consistently everywhere; half-parsed dates are worse than none.
- No `any` in the public surface; `unknown` for genuinely opaque blobs.

## Pagination

```ts
for await (const inv of client.invoices.list({ limit: 100 })) { ... }   // auto-paging

const page = await client.invoices.list();                               // one page
page.data; page.hasNextPage(); await page.getNextPage();
```

Implement as a `PagePromise` that is both awaitable (first page) and async-iterable (all items).

## Errors

Classes extending a base `AcmeError extends Error`, with `name` set so they survive bundling and cross-realm checks. Export them from the root and as statics on the client (`Acme.NotFoundError`) so callers can `instanceof` without extra imports. See [errors-and-versioning.md](errors-and-versioning.md).

## Streaming

- SSE / chunked: return an object that is `AsyncIterable<Event>` with `.controller.abort()` and helpers like `.finalMessage()`.
- WebSocket: typed `on(event, handler)` with a discriminated event union, `send()` typed per message, automatic reconnect with backoff as an opt-in, and `close()`.

## Packaging

- `package.json` `exports` map with `types`, `import`, `require`; ship ESM + CJS (or ESM-only if your users allow it) and test both.
- `"sideEffects": false`; no top-level work at import time.
- Target the oldest supported Node LTS; declare it in `engines`.
- Zero or near-zero runtime dependencies. Each one is a supply-chain and bundle-size cost your users inherit.
- Public API report with `@microsoft/api-extractor` (checked in) so any surface change shows up in review.
- Mark deprecations with JSDoc `@deprecated <what to use instead>` — editors strike through the call.
