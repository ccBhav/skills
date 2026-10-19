# JavaScript SDK idioms

For SDKs written in plain JavaScript (no TypeScript build step), and for the plain-JS callers of any SDK. Surface design is the same as [typescript.md](typescript.md) (client options, `RequestOptions`, `.withResponse()`, auto-paging, error classes, packaging), so read that for shape. This file covers what changes when the compiler isn't there to help.

## Types without TypeScript

JS callers still get autocomplete if the SDK ships type declarations. Author types in JSDoc and generate `.d.ts`:

```js
/**
 * @typedef {object} InvoiceCreateParams
 * @property {string} customerId
 * @property {number} amount  Amount in minor units (cents).
 * @property {'usd' | 'eur' | (string & {})} [currency]
 */

/**
 * Create an invoice.
 * @param {InvoiceCreateParams} params
 * @param {RequestOptions} [options]
 * @returns {Promise<Invoice>}
 */
create(params, options) { ... }
```

- `jsconfig.json` / `tsconfig.json` with `allowJs`, `checkJs`, `strict`: typecheck the SDK's own source in CI (`tsc --noEmit`).
- Emit declarations with `tsc --allowJs --declaration --emitDeclarationOnly --outDir types`, and point `exports[...].types` at them.
- Run `api-extractor` (or diff the emitted `.d.ts`) to track the public surface, the same as for a TS SDK.
- Every exported function, class and option documented with JSDoc. For plain-JS callers the doc comment is the whole type system.

## Validate at runtime

A TS caller gets a compile error for `client.invoices.create({ customerID: 'x' })`. A JS caller gets a confusing 400 from the server, or nothing at all. Close that gap at the boundary:

- Check required arguments and their primitive types before sending, and throw `TypeError` naming the parameter: `invoices.create(): "customerId" is required`.
- Warn on unknown keys in options objects (the logger, once per key), and suggest the nearest valid name: `unknown option "timout" (did you mean "timeout"?)`. This is how typos show up in JS.
- Keep validation shallow: required fields, primitive types, obviously wrong shapes. Deep schema validation duplicates the server and drifts from it.
- Validate in one helper shared by every method so each error message has the same shape.

## Enums and constants

Export frozen constant objects so JS callers have something to autocomplete and don't need magic strings:

```js
export const InvoiceStatus = Object.freeze({ Draft: 'draft', Open: 'open', Paid: 'paid' });
```

Response values stay plain strings, and unknown values pass through. This is the same open-enum rule as TS.

## Classes and privacy

- Use `#private` fields and methods for internals. Underscore-prefixed members become public API the first time a caller depends on one.
- Don't monkey-patch globals or extend built-in prototypes. Polyfill nothing at import time.
- Use `Symbol.asyncIterator` for auto-paging so `for await` works. Return a thenable (or a real `Promise` subclass) so `await list()` gives the first page.

## Module formats

- Plain JS can ship ESM and CJS from one source without a bundler: write ESM, then generate the CJS build with a small build step (`tsup`, `esbuild`). Hand-maintaining two copies drifts.
- If you ship both formats, watch for the dual-package hazard: `instanceof` fails across copies when one app loads the ESM build and another loads the CJS one. Keep error classes and other stateful modules in one format and re-export them from the other, or check `err.name` in tests.
- Set `"type": "module"` or use `.mjs`/`.cjs` extensions explicitly. Ambiguous files break in some loaders.

## Tooling

- ESLint with `eslint-plugin-jsdoc` to enforce doc comments on exports.
- `node:test` + `node:assert` is enough for unit tests. Use `msw` or an injected `fetch` for the transport (see [testing.md](testing.md)).
- Test the emitted `.d.ts` with a small TS consumer file (`tsd` or `tsc --noEmit` on `examples/*.ts`) so types and runtime stay in step.
