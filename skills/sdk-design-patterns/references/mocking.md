# Shipping a mock alongside the SDK

Consumers need to unit-test their own code without calling your API. Without help, each one hand-writes `jest.mock('@acme/sdk')` stubs that return made-up shapes, drift from the real types, and never throw your real error classes. Ship the test double yourself: a companion **mock package** that is versioned and released with the SDK.

```ts
import { createMockAcme } from '@acme/sdk-mock';

const acme = createMockAcme({ seed: 1 });
acme.mock.on('invoices.create').reply(201, (req) => acme.fake.invoice({ amount: req.body.amount }));
acme.mock.on('invoices.retrieve').replyError(404, { code: 'invoice_not_found' });

await billInvoice(acme.client, 'inv_123');   // the caller's code under test, using a real client

expect(acme.mock.calls('invoices.create')).toHaveLength(1);
```

## Design

**A real client over a fake transport.** `createMockAcme()` returns a genuine SDK client built with an injected `fetch`, not a hand-written stub object. Request serialization, response parsing, pagination and error mapping all run for real, so `instanceof Acme.NotFoundError` works in consumer tests exactly as in production. The mock only decides which HTTP response comes back.

**Separate package, same release.** Publish `@acme/sdk-mock` (Go: an `acmetest` package, see below). Then faker and test helpers never land in production bundles. Release it from the same CI run as the SDK with the same version number, and declare the SDK as a peer dependency (`"@acme/sdk": "1.4.0"`) so the two can't drift. Consumers install it as a devDependency.

**Framework-agnostic.** No Jest, Vitest or Mocha imports. Expose plain data (`calls()`, `reset()`) that works with any assertion library.

**Fast and strict by default.**
- Retries off (`maxRetries: 0`) and no real timers, so error tests don't wait on backoff. Callers can turn retries back on to test their own retry handling.
- An operation without a handler gets a factory-generated success response (see defaults below). A request that matches no SDK operation fails with a clear error naming the method and path. It never falls through to the network.

## Handlers

Key handlers by **operation name** (`'invoices.create'`), matching the SDK method, not by HTTP path. Consumers think in SDK calls, and operation names survive API path changes.

| Call | Behaviour |
|---|---|
| `.reply(status, body \| (req) => body)` | Fixed or computed response; `req` exposes parsed `params`, `body`, `query`, `headers` |
| `.replyOnce(...)` | Applies once, then falls back to the next handler (queue them to script a sequence) |
| `.replyError(status, { code, message, requestId? })` | An error body in the API's real error shape (e.g. RFC 9457), so the SDK builds its real error class |
| `.replyPages([[...], [...]])` | Successive pages with real cursors and `hasNextPage`, so the consumer's auto-paging loop runs for real |
| `.networkError()` / `.timeout()` | The SDK's connection and timeout errors |
| `.delay(ms)` | Latency, for testing consumer timeouts and loading states |
| `.stream([events])` | SSE / streaming responses as a sequence of typed events |

Plus `mock.calls(op)` (parsed requests, in order), `mock.reset()` and `mock.pending()` (handlers that were set up but never called, which usually means a broken test).

Optional **stateful mode** (`createMockAcme({ stateful: true })`): an in-memory store so `create` → `retrieve` → `list` → `delete` behave consistently. Useful for integration-style tests of consumer workflows. Keep it opt-in, since stateless handlers are easier to reason about in unit tests.

## Fake data with faker

Use [`@faker-js/faker`](https://www.npmjs.com/package/@faker-js/faker) for factory data. v10 is ESM-only and needs Node `^20.19 || ^22.13 || >=24`, which loads ESM from `require()`. Check that this fits the SDK's supported Node range, or pin faker v9.

```ts
import { Faker, en } from '@faker-js/faker';
import type { Invoice, Customer } from '@acme/sdk';
import { InvoiceStatus } from '@acme/sdk';

export function createFactories({ seed = 1 } = {}) {
  // Own instance per mock, never the global `faker`: parallel tests can't disturb each other's sequence.
  const f = new Faker({ locale: [en], seed });
  f.setDefaultRefDate('2026-01-01T00:00:00Z');        // dates don't move with the wall clock

  const id = (prefix: string) => `${prefix}_${f.string.alphanumeric(14)}`;

  const customer = (o: Partial<Customer> = {}): Customer => ({
    id: id('cus'),
    email: f.internet.email(),
    name: f.company.name(),
    createdAt: f.date.past().toISOString(),
    ...o,
  });

  const invoice = (o: Partial<Invoice> = {}): Invoice => ({
    id: id('inv'),
    customerId: o.customerId ?? customer().id,
    amount: f.number.int({ min: 100, max: 100_000 }),   // minor units, like the real API
    currency: f.finance.currencyCode().toLowerCase(),
    status: f.helpers.arrayElement(Object.values(InvoiceStatus)),
    memo: f.helpers.maybe(() => f.lorem.sentence(), { probability: 0.5 }) ?? null,
    createdAt: f.date.recent().toISOString(),
    ...o,
  });

  return { seed: (s: number) => f.seed(s), customer, invoice,
           invoices: (n: number, o?: Partial<Invoice>) => f.helpers.multiple(() => invoice(o), { count: n }) };
}
```

Factory rules:
- **Typed against the SDK's own response types.** A factory that stops compiling when the API changes is the point: the mock can't drift silently.
- **Deterministic.** Seeded instance plus fixed reference date, so a failing test reproduces exactly. Expose the seed in failure output.
- **Realistic shape, not just realistic values.** Real ID prefixes, money in the same units as the API, enum values taken from the SDK's constants, nullable fields sometimes `null` (`helpers.maybe`) so consumers' null handling gets exercised.
- **Overrides win.** Any field the test cares about is passed in; everything else is noise. Support nested overrides for nested objects.
- **Related data stays consistent.** `invoice({ customerId })` links to a customer the test also created, and stateful mode enforces it.

### Generating factories from the spec

If the SDK is generated (see [codegen.md](codegen.md)), generate the factories and the operation table from the same OpenAPI spec:

| Schema | faker call |
|---|---|
| `format: email` / `uuid` / `uri` / `date-time` | `internet.email()` / `string.uuid()` / `internet.url()` / `date.recent().toISOString()` |
| `enum` | `helpers.arrayElement(values)` |
| `integer` / `number` with `minimum`/`maximum` | `number.int({ min, max })` / `number.float({ min, max })` |
| `string` with `pattern` | `helpers.fromRegExp(pattern)` |
| property name heuristics (`name`, `phone`, `city`, `company`) | `person.fullName()`, `phone.number()`, `location.city()`, `company.name()` |
| `nullable` / not `required` | wrap in `helpers.maybe(...)` |
| `example` present | use it: the API author chose it |

Hand-written overrides go in a file the generator doesn't touch. A CI check fails if any SDK operation lacks a default handler or any response type lacks a factory.

## Other entry points

- **MSW handlers.** Some consumers construct the client deep inside their code and can't inject it. Also export `handlers` for [MSW](https://mswjs.io/) (`setupServer(...acmeHandlers({ seed: 1 }))`), built on the same factories and operation table.
- **Fixtures for snapshot tests.** `fake.invoice()` output with a fixed seed is stable, so it can safely appear in consumer snapshots.

## Go: `acmetest`

Faker is JS-only. For Go, ship an `acmetest` package in the SDK module (like `net/http/httptest`) using [gofakeit](https://github.com/brianvoe/gofakeit) for data:

```go
func TestBill(t *testing.T) {
    m := acmetest.New(t, acmetest.WithSeed(1))       // httptest.Server; closed via t.Cleanup
    m.On("invoices.create").Reply(201, m.Fake.Invoice(acmetest.InvoiceOverrides{Amount: acme.Int(1000)}))
    m.On("invoices.retrieve").ReplyError(404, "invoice_not_found")

    err := Bill(context.Background(), m.Client, "inv_123")   // m.Client is a real *acme.Client

    var apiErr *acme.Error
    require.ErrorAs(t, err, &apiErr)
    require.Len(t, m.Calls("invoices.create"), 1)
}
```

The same rules apply: a real client against a fake server, seeded data (`gofakeit.New(seed)`), handlers keyed by operation, and unmatched requests failing the test through `t.Fatalf`.
