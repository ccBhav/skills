# Filling in the plan page

`assets/page-shell.html` is a complete, working page. Copy it to `docs/plans/<slug>/plan.html` and replace every `FILL:` placeholder with the plan's content. The theme, layout, tabs, demo wiring and comment buttons work as they are; leave the wiring at the bottom alone unless you have a reason.

The page and `plan.md` carry the same content. Write `plan.md` first, then carry it over, so the two never disagree.

## Structure

| Part | What goes in it |
|------|-----------------|
| `<title>` | `<Feature name> Plan`, 2–4 words is ideal |
| `header.plan-head` | `h1` title, the meta chips (version, status, Jira link, updated date), the brief |
| `#glossary` | One `<dt>`/`<dd>` pair per term |
| `#outline` | The overview mermaid diagram, then one `h3` + `ol.steps` per phase |
| `#use-cases` | Three `article.case`: `happy`, `extreme`, `error` |
| `#code` | One block per sketch: `h3`, a sentence, `.code-tabs`, and `.demo` when one makes sense |
| `#decisions` | One line per decision, with the short why |
| `#open`, `#changelog` | As in `plan.md` |
| `#appendix` | One `<details class="appendix-item" id="a1">` per appendix item, grouped under `h3`s (Data models, Code, API examples, Other) — only the groups the plan has |

Keep the `.plan-section` class and the `.section-head` with its `Comment` button on every section — that's what readers click to comment on that section. Add more sections the same way if the plan needs them (e.g. `#data` for a schema change, `#rollout`), and add them to the `nav.toc` list.

Keep the Jira chip even when there's no ticket: set it to `Jira <strong>not linked</strong>` (no link) so readers know one hasn't been attached yet, rather than wondering if it was forgotten. When a ticket is linked later, swap it back to the link.

## Appendix

Each item is a collapsed `<details>` so the appendix doesn't swamp the page. Give each one an `id` (`a1`, `a2`…) and link to it from the body with `<a href="#a1">A1</a>` — the shell opens a collapsed item automatically when a link to it is followed.

Inside an item: a one-sentence intro, then the detail. For the same model or logic in several languages, reuse the `.code-tabs` block. Its tabs don't have to be JS/TS/Go: set each button's `data-lang` and label (e.g. `TypeScript`, `Go`, `SQL`) and give each `<pre>` the matching `data-lang`. If a JS pane isn't linked to a `SKETCHES` function, put the code straight into the `<pre>` like the others. For code in a single language, use a bare `<pre class="code">`. Field tables use `table.fields`.

## Diagrams

Put mermaid source inside `<div class="diagram"><pre class="mermaid">…</pre></div>`. It's the same source as in `plan.md`, so copy it across. The shell loads mermaid and picks a light or dark theme to match the reader.

Inside `<pre class="mermaid">`, escape `<` as `&lt;` and `&` as `&amp;` — the browser parses the HTML before mermaid sees it.

## Code sketches

Each sketch has three tabs: JavaScript, TypeScript and Go.

- **JavaScript** goes in the `SKETCHES` object in the first `<script>`, as a plain named function. The shell fills the JS tab from `fn.toString()`, so the tab shows exactly the code the demo runs. `toString()` doesn't include comments written above the function, so put anything readers need to see inside the function body.
- **TypeScript** and **Go** go straight into their `<pre data-lang="ts">` / `<pre data-lang="go">` panes as text. Escape `<`, `>` and `&` (`&lt;`, `&gt;`, `&amp;`).
- Match `data-sketch="<name>"` on the `.code-tabs` to the key in `SKETCHES`.

All three should do the same thing, written the way each language is usually written. In Go that means returning `(value, error)` instead of throwing. In TypeScript it means real types rather than `any`. Keep each one short enough to read without scrolling, around 30 lines at most. If the real thing is longer, sketch the interesting core and say what's left out.

## Interactive demos

A demo lets a reader type an input and see what the sketch returns. That's often the fastest way to understand a rule like "how are batches split" or "which discount applies".

Add a demo when the sketch computes something from its inputs: validation, pricing, mapping, splitting, state transitions, parsing. Skip it when the sketch is mostly I/O or wiring (a DB write, an HTTP handler, a queue consumer) and there's nothing to show apart from mocks.

To add one:

1. Put `<div class="demo" data-demo="<name>"><h4>Try it</h4></div>` after the sketch's `.code-tabs`.
2. Add an entry to `DEMOS` with:
   - `fields`: `{ name, label, type, value }`. `type` is `number`, `text`, `json` (an object or array, shown in a textarea) or `list` (comma-separated, arrives as an array of strings). `value` is the starting value.
   - `presets`: `happy`, `extreme` and `error`, each `{ label, values }`, using the same inputs as the use cases so the demo and the use cases line up. Labels say what the input is ("Extreme: 5M rows").
   - `run(args)`: call the `SKETCHES` function and return what to show. You can return a string or any JSON-able value. A thrown error is shown in red as the error result, so the error preset should make the sketch throw (or return its error value) the way the use case describes.
   - `note`: usually "Runs the JavaScript version. The TypeScript and Go versions follow the same logic."
3. Try every preset in your head against the sketch. A demo whose output doesn't match the use case is worse than no demo.

For a state machine or a multi-step flow, a demo can take a `list` of events and return the state after each one. That's still one `run` call.

## Comments

Publish with `capabilities: {comments: {composer_only: true}}`. With that:
- each section's **Comment** button opens claude.ai's own comment box anchored to that section
- readers can also use claude.ai's comment mode to comment anywhere on the page
- comments are stored with the Artifact and shown by claude.ai, not by the page, so there's nothing to build

The buttons stay hidden when the page is opened outside claude.ai or by someone who can't comment.

Comments stay attached to the page across republishes, so always update the same Artifact. Pass its `url` from `plan.md`'s front matter, and leave out `capabilities` on a republish so the setting carries over.
