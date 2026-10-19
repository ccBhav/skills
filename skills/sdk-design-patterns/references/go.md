# Go SDK idioms

## Client and configuration

Functional options for the client; they extend without breaking callers.

```go
client := acme.NewClient()                                   // reads ACME_API_KEY
client := acme.NewClient(
    option.WithAPIKey(key),
    option.WithHTTPClient(httpClient),                       // injectable transport
    option.WithMaxRetries(3),
)
```

- `NewClient` doesn't fail on a missing key; the first request returns a clear error naming the env var. (Constructors that return `(*Client, error)` are fine too — pick one and be consistent.)
- Put options in an `option` subpackage so they work both on the client and per request.
- Services as struct fields: `client.Invoices.New(ctx, params)`, `client.Invoices.LineItems.List(ctx, invoiceID, params)`.

## Requests

```go
inv, err := client.Invoices.New(ctx, acme.InvoiceNewParams{
    CustomerID: "cus_123",
    Amount:     1000,
}, option.WithIdempotencyKey(k))
```

- `context.Context` first, always. Timeouts and cancellation come from the context; the client default timeout applies only when the context has no deadline.
- Signature: `(ctx, pathParams..., params, ...option.RequestOption)`.
- Params as a named struct even with one field — adding fields later is non-breaking, adding positional args is.
- Raw response access via an option: `option.WithResponseInto(&httpResp)`.
- Never panic on API input; never call `log.Fatal`. Return errors.

## Optional fields

Go can't tell "unset" from zero. Pick one convention and use it everywhere:
- pointer fields with helpers (`acme.String("x")`, `acme.Int(5)`), or
- a generic `param.Opt[T]` / `Field[T]` wrapper with `.IsSet()`.

Make sure JSON encoding omits unset fields and sends explicit zero values when set (`omitempty` alone drops a deliberate `0` or `false`).

## Types

- Exported structs for every request/response; JSON tags match the wire format.
- Server enums as a named string type with constants: `type InvoiceStatus string; const InvoiceStatusPaid InvoiceStatus = "paid"`. Unknown values decode fine — that's the open-enum property for free.
- Polymorphic payloads: an interface with an unexported marker method plus concrete types, or a struct with a `Type` field and `AsX()` accessors. Keep raw JSON available for unknown variants.
- `time.Time` for timestamps, decoded consistently.

## Pagination

Go 1.23+ iterators:

```go
for inv, err := range client.Invoices.ListAutoPaging(ctx, params) {
    if err != nil { return err }
    ...
}
page, err := client.Invoices.List(ctx, params)   // one page; page.GetNextPage(ctx)
```

For older Go support, an iterator struct with `Next() bool`, `Current() T`, `Err() error` (the `bufio.Scanner` shape).

## Errors

```go
var apiErr *acme.Error
if errors.As(err, &apiErr) && apiErr.StatusCode == http.StatusNotFound { ... }
```

- One exported `*Error` type carrying `StatusCode`, `Code`, `Message`, `RequestID`, raw body, `Request`/`Response`. Status-class helpers or sentinel `errors.Is` targets (`acme.ErrNotFound`) if callers branch often.
- Wrap transport errors with `%w` so `errors.Is(err, context.DeadlineExceeded)` still works.
See [errors-and-versioning.md](errors-and-versioning.md).

## Streaming

`iter.Seq2[Event, error]` or a `Stream[T]` with `Next/Current/Err/Close`; always `Close()` — document `defer stream.Close()`. Cancellation via context.

## Module and versioning

- Major version ≥2 lives in the module path: `github.com/acme/acme-go/v2`. Plan for it; `v0` signals instability honestly.
- Support the last two Go minor releases (Go's own support policy); set `go` in `go.mod` accordingly.
- Deprecate with a paragraph starting `// Deprecated: use X instead.` — `staticcheck` and gopls flag callers.
- Catch accidental breaks with `gorelease` or `golang.org/x/exp/cmd/apidiff` in CI.
- Minimal dependencies; stdlib `net/http` and `encoding/json` are enough.
