# Classic patterns behind SDK design

Most SDK building blocks are a well-known design pattern. Use the name in design docs and review comments ("the pager is an Iterator, the transport is a Chain of Responsibility"), and send readers to [refactoring.guru](https://refactoring.guru/) for the full explanation and examples in [TypeScript](https://refactoring.guru/design-patterns/typescript) and [Go](https://refactoring.guru/design-patterns/go).

## Patterns

| SDK concern | Pattern | How it shows up |
|---|---|---|
| The client itself | [Facade](https://refactoring.guru/design-patterns/facade) | One `Acme` object in front of the transport, auth, retries and serialization. Callers never touch the parts. |
| Pagination | [Iterator](https://refactoring.guru/design-patterns/iterator) | `for await` / `range` over items while pages are fetched lazily behind it. |
| Middleware / hooks | [Chain of Responsibility](https://refactoring.guru/design-patterns/chain-of-responsibility) | Each layer (auth, retry, logging, tracing) handles the request and passes it on. Go `http.RoundTripper` chains, TS `fetch` wrappers. |
| Injectable transport | [Adapter](https://refactoring.guru/design-patterns/adapter) | Wrap `fetch`, `*http.Client` or a test fake behind one internal interface. |
| Retry / backoff policy | [Strategy](https://refactoring.guru/design-patterns/strategy) | Backoff, auth and serialization are swappable objects, not `if` branches inside the client. |
| Adding behaviour without changing the core | [Decorator](https://refactoring.guru/design-patterns/decorator) | `withOptions()` derived clients, caching or instrumentation wrappers that keep the same interface. |
| Complex request construction | [Builder](https://refactoring.guru/design-patterns/builder) | Only for genuinely multi-step requests (query builders, multipart uploads). For ordinary calls a params object is better. |
| Streaming / realtime events | [Observer](https://refactoring.guru/design-patterns/observer) | `on('message', handler)` for WebSocket and SSE event emitters. |
| Lazy or remote resources | [Proxy](https://refactoring.guru/design-patterns/proxy) | Lazily loaded sub-clients, and resource handles that fetch on first access. |
| Constructing typed results | [Factory Method](https://refactoring.guru/design-patterns/factory-method) | Building the right error subclass from a status code, or the right event type from a `type` field. |

Prefer the lightest form that does the job. A `fetch` wrapper function is already a Chain of Responsibility, and a functional option is already a Builder step. Pattern names are vocabulary to explain a design, not a list of classes to add.

## Smells and refactorings for reviews

When a review finding matches a [code smell](https://refactoring.guru/refactoring/smells), name it and link the [refactoring technique](https://refactoring.guru/refactoring/techniques) that fixes it:

| Smell in an SDK | Fix |
|---|---|
| [Long Parameter List](https://refactoring.guru/smells/long-parameter-list): `create(customerId, amount, currency, dueDate, memo)` | [Introduce Parameter Object](https://refactoring.guru/introduce-parameter-object): a single params object or struct |
| [Primitive Obsession](https://refactoring.guru/smells/primitive-obsession): IDs, money and statuses as bare strings and numbers | Named types (`InvoiceStatus`), and branded or typed IDs where mixing them up is a real risk |
| [Data Clumps](https://refactoring.guru/smells/data-clumps): `timeout, retries, headers` repeated on every method | One `RequestOptions` type |
| [Switch Statements](https://refactoring.guru/smells/switch-statements): status-code `switch` in several places | One error factory (Factory Method above) |
| [Message Chains](https://refactoring.guru/smells/message-chains): `client.http().transport().retry()` | [Hide Delegate](https://refactoring.guru/hide-delegate): expose the option at the top |
| [Middle Man](https://refactoring.guru/smells/middle-man): resource classes that only forward to the client | [Remove Middle Man](https://refactoring.guru/remove-middle-man), unless the class exists for namespacing |
| [Speculative Generality](https://refactoring.guru/smells/speculative-generality): plugin systems and config nobody asked for | Delete it. Every public hook is a versioning commitment |
| [Shotgun Surgery](https://refactoring.guru/smells/shotgun-surgery): adding an endpoint touches many files | Code generation, or one resource definition per endpoint |
| [Divergent Change](https://refactoring.guru/smells/divergent-change): the client class changes for every feature | Split transport, auth and resources |
| Returning `{ error, data }` or error strings | [Replace Error Code with Exception](https://refactoring.guru/replace-error-code-with-exception) in TS/JS. In Go, return a typed `*Error` |
