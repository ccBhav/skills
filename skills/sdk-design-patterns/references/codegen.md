# Code generation

## Generated, hand-written or hybrid

| Approach | Pick when |
|---|---|
| **Generated** | API has many endpoints, changes often, or needs several languages; you own a good OpenAPI spec. |
| **Hand-written** | Small API (< ~15 operations), unusual protocol, or the SDK is mostly client-side logic (caching, local state). |
| **Hybrid** | Default for serious SDKs: generated core (types, resources, transport) + hand-written helpers (webhook verification, polling `waitUntil`, upload helpers, streaming accumulators) in files the generator never overwrites. |

## Generators

| Tool | Notes |
|---|---|
| **Stainless** | Commercial. Idiomatic TS/Go/Python/etc. SDKs in the style of the OpenAI and Anthropic SDKs; config file for resource naming, pagination, custom code. |
| **Speakeasy** | Commercial. Many languages, Terraform providers, strong CI/GitHub integration; overlays to patch the spec. |
| **Fern** | Commercial with OSS core. SDKs + docs from OpenAPI or Fern definition. |
| **openapi-generator** | Free, many languages, quality varies by generator; often needs custom templates to feel idiomatic. |
| **oapi-codegen** (Go) | Free; good types and `net/http` client/server stubs. Usually wrapped by a hand-written ergonomic layer. |
| **openapi-typescript + openapi-fetch** (TS) | Free; types-only + tiny typed fetch. Good for internal SDKs; you add retries, pagination and errors yourself. |

Verify current pricing and feature sets before recommending a commercial generator; these move quickly.

## Spec hygiene

Generated SDK quality is capped by the spec. Before generating:
- Every operation has an `operationId` that reads as a method name (`invoices_create` → `invoices.create`); tags or path prefixes group them into resources.
- Every schema is a named component (inline anonymous schemas become `InlineResponse2003`).
- Error responses (4xx/5xx) are documented with their body schema.
- Enums listed; nullable vs optional correct; `format` set for dates, ints, binaries.
- Pagination described consistently (same params and response envelope everywhere; vendor extensions if the generator supports them).
- Lint with Spectral or Redocly in CI; fail the build on new warnings.
- Patch spec issues with overlays (OpenAPI Overlay spec) rather than forking the spec.

## Pipeline

```
spec change (API repo)
  → lint spec
  → regenerate SDK(s) on a branch
  → open PR with the diff + public-API report diff
  → tests (unit, recorded, contract)  — see testing.md
  → merge
  → release: version bump from conventional commits or changesets, changelog, publish
```

- TS: publish to npm from CI with provenance (`npm publish --provenance`), via changesets or release-please.
- Go: releasing is a git tag (`v1.4.0`, or `v2.0.0` with the `/v2` module path); the module proxy picks it up. GoReleaser is unnecessary for libraries.
- Generated code is committed (reviewable, `go get`-able) and marked with a `// Code generated ... DO NOT EDIT.` header so linters and reviewers skip it.
- A breaking spec change should fail the pipeline unless the PR is labelled for a major release.
