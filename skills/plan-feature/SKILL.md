---
name: plan-feature
description: Plan a software feature end to end — ask what the feature is, pull in the Jira ticket if there is one (or link one later), grill the user one question at a time to settle every decision, then write a versioned plan (title, brief, version, glossary, outline, diagrams, JS/TS/Go snippets, happy/extreme/error use cases) to docs/plans/<slug>/ and publish it as a shareable, commentable Artifact page with interactive demos. Use whenever the user wants to plan, spec, scope, design or break down a feature, ticket or epic ("plan this feature", "let's plan PROJ-123", "write a tech plan / RFC / design doc for…", "help me think through how to build X"), or wants to update an existing plan in docs/plans or link a Jira ticket to one ("add CC-482 to the export plan") — even if they don't say "plan".
---

# Plan a feature

You're producing a plan someone else can pick up and build from, and that teammates can read and comment on without being in this conversation. Three outputs, all in one folder:

```
docs/plans/<slug>/
├── plan.md        # the plan, the source of truth (lives in git)
├── decisions.md   # every grilling question, the options, the decision, and why
└── plan.html      # the same plan as a page, published as a shared Artifact
```

`<slug>` is a short kebab-case name for the feature (`bulk-invoice-export`). Don't put the Jira key in it: the ticket often comes later, and the folder name should never change once the plan exists, because links to it would break. The ticket lives in the front matter instead.

Write for a smart new joiner: short sentences, plain words, a diagram wherever something moves between parts, and every bit of jargon in the glossary. If a sentence needs re-reading, rewrite it.

## 1. Find out what we're planning

Check `docs/plans/` first. If a plan for this feature already exists, jump to **Updating an existing plan** below.

Otherwise ask, in one message:
- What's the feature? (a sentence or two is fine)
- Is there a Jira ticket yet? (key or URL — "not yet" is fine, it can be linked later)

No ticket is a normal case, not a blocker: set `jira: none` and carry on. Mention once, at hand-over, that they can link one later.

If they give a ticket and a Jira/Atlassian connector is available, fetch it: summary, description, acceptance criteria, comments, linked issues. With the Atlassian connector that's usually `getAccessibleAtlassianResources` for the cloud id, then `getJiraIssue`. If there's no connector or the fetch fails, ask them to paste the ticket text — don't stall on it.

Then read the relevant code before asking anything else. Find the modules, entities, endpoints and tests this feature will touch, and existing patterns it should follow. Every question you can answer from the code is one you don't have to ask.

## 2. Grill

Use the `grilling` skill (invoke it if it's available). If it isn't, the rules are: interview relentlessly until you share an understanding; walk each branch of the design tree, resolving dependencies one by one; ask **one question at a time** and give your recommended answer with each; if the code can answer it, look instead of asking.

Branches worth walking (skip any that clearly don't apply, follow any that open up):
- **Why** — the problem, who has it, what "done" looks like
- **Scope** — what's in, what's explicitly out (non-goals)
- **Words** — terms that mean something specific here (these become the glossary)
- **Data** — new or changed entities, fields, migrations
- **Interfaces** — endpoints, events, function signatures, UI touchpoints
- **Happy path** — the normal flow, step by step
- **Extremes** — biggest/smallest/emptiest inputs, volume, concurrency, slow dependencies
- **Errors** — what fails, what the user sees, what gets retried or rolled back
- **Rollout** — flags, migrations order, backwards compatibility
- **Testing** — what proves it works

Offer choices where they exist — "A) … B) … — I'd pick B because …" is faster to answer than an open question.

**Record as you go.** Create `decisions.md` after the first answer and append to it after every answer, so nothing is lost if the session dies. Use this shape for each entry:

```markdown
<a id="d3"></a>
### D3. Should exports run synchronously or as a background job?
- **Options:** A) synchronous in the request · B) background job + email link
- **Recommended:** B — exports over 5k rows time out the request today
- **Decision:** B
- **Why:** matches how CSV reports already work (`src/reports/jobs.ts`)
- **Asked:** 2026-10-19 · **Plan version:** v0.1
```

If the user says "you decide" or the code answered it, still log it (mark **Decided by:** Claude / code) — the reader needs to know it was a choice.

Stop grilling when you could write the outline without guessing. Say so, and give them the chance to add anything.

## 3. Write `plan.md`

Use the template in `references/plan-template.md`. The order matters — it's the order a reader needs things in:

1. **Title**, **brief** (3–5 plain sentences: what, why, for whom), **version** (start at `v0.1`), status, Jira link, date, Artifact link (filled in after publishing)
2. **Glossary** — every term a newcomer would trip on, one line each, in plain words
3. **Plan outline** — phases → steps; each step says what changes, where (file paths), and how you'll know it's done. Put a mermaid diagram at the top showing how the parts connect, and others wherever a flow or sequence is easier to see than read.
4. **Use cases** — a happy path, an extreme path and an error path. Each gives the scenario, the input, the step-by-step (a sequence diagram helps), and the expected result.
5. **Code sketches** — the key pieces in **JavaScript, TypeScript and Go**: same logic, idiomatic in each (Go returns `error`, TS has types, JS has JSDoc). Keep them short and focused on the interesting part; they're sketches, not the implementation.
6. **Key decisions** — one line per decision, linking to `decisions.md#d3` etc.
7. **Open questions & risks**
8. **Changelog** — `v0.1 — 2026-10-19 — first draft`
9. **Appendix** — the detail a builder needs but a reader skimming the plan doesn't: full data models, longer code, request/response examples, config, migration SQL, links to related docs and tickets. See below.

### The appendix

The body of the plan should stay readable in one sitting. So whenever something is long, exact, or only matters to whoever builds it, put it in the appendix and point to it from the body ("full schema in [A1](#a1-export-job-model)"). Pointing works both ways: each appendix item says which plan step or use case it supports.

Number the items `A1`, `A2`… under these groups, and include only the groups this plan needs:
- **Data models**: entities and types, with fields, types, constraints and a one-line meaning per field. When the same model exists in several languages, show it as TypeScript type + Go struct (+ SQL/Prisma/etc. if there's a schema change). A mermaid `erDiagram` helps when there are several related entities.
- **Code**: anything longer than a sketch, such as a full handler, a migration script or a test outline. Use JS/TS/Go tabs when it's the same logic across languages; otherwise use whatever language it really is (SQL, YAML, JSON, shell).
- **API examples**: real-looking requests and responses for the happy, extreme and error paths.
- **Other**: config, feature-flag names, environment variables, dashboards, links to related tickets, docs or prior art in the codebase.

Every item still follows the plain-language rule: a one-sentence intro saying what it is and why it's here, before the detail.


Diagrams: use mermaid (```` ```mermaid ````) — GitHub renders it in the .md and the page renders it too, so you write each one once. Prefer `flowchart` for structure, `sequenceDiagram` for use cases, `stateDiagram-v2` for lifecycles. Keep node labels short.

## 4. Build and publish the page

Follow the Artifact tool's own rules (load the `artifact-design` skill before writing, and the `artifact-capabilities` skill before declaring capabilities). Start from `assets/page-shell.html` — copy it to `docs/plans/<slug>/plan.html` and fill it in. It already has the theme, layout, mermaid, the JS/TS/Go tabs, the demo widget, and per-section comment buttons wired up; `references/page-guide.md` explains each part and how to fill it.

The page carries the same content as `plan.md`, plus **interactive demos**: for each code sketch where it makes sense, a small form where the reader types an input (a number, a list, a JSON blob) and sees what the code returns. Add preset buttons for the happy, extreme and error inputs from the use cases so readers can see all three in one click. The demo runs the real JavaScript sketch — the shell renders the JS tab from the same function the demo calls, so what's shown is what runs. Go and TS can't run in the browser; say so under the demo ("runs the JavaScript version — the Go and TS versions follow the same logic"). Skip a demo when the code is pure I/O (a DB write, an HTTP call) and there's nothing meaningful to compute — a fake demo is worse than none.

Publish with:
- `icon: "plan"`, and a one-sentence `description` (the brief's first sentence)
- `capabilities: {comments: {composer_only: true}}` — this gives each section a "Comment" button that opens claude.ai's own comment box on that section. No extra permission prompts for readers.

On the page, the appendix is one section with a collapsible block per item so it doesn't swamp the plan (see `references/page-guide.md`).

Then write the Artifact URL into `plan.md`'s front matter (`artifact:`) so later runs update the same page and keep its comment threads.

## 5. Hand over

Tell the user, briefly:
- where the files are
- if there's no Jira ticket: that they can link one any time ("add CC-482 to the <feature> plan")
- the Artifact link — it's **private until they share it** (Share button in claude.ai). Teammates in their organization can then comment on any section; people outside it need to be invited.
- the decisions count and any open questions still needing an answer

Don't commit unless they ask.

## Linking a Jira ticket later

When the user wants to attach a ticket to a plan that already exists ("here's the ticket for the export plan: CC-482"):

1. Find the plan in `docs/plans/` — match on the name they use; if more than one could fit, ask which.
2. Fetch the ticket as in step 1 (connector, or ask them to paste it).
3. Compare the ticket with the plan. Look for acceptance criteria the plan doesn't cover, scope the ticket includes or excludes differently, and terms the ticket uses that the glossary doesn't. If the ticket matches the plan, say so and skip to 4. If it differs, grill **only** on those differences, logging each as a new decision as usual.
4. Set `jira:` and `jira_url:` in `plan.md`'s front matter and the header line, and change the page's Jira chip from "not linked" to the ticket link.
5. Bump the minor version with a changelog line: `v0.3 — <date> — Linked CC-482` (plus anything the ticket changed).
6. Republish to the same Artifact, as in **Updating an existing plan** step 5. Leave the folder name as it is.
7. Offer — don't just do it — to add a comment on the Jira ticket linking the plan page. Posting to Jira is visible to the whole team, so it needs their yes.

## Updating an existing plan

When `docs/plans/<slug>/plan.md` already exists:

1. Read `plan.md` and `decisions.md`.
2. If `plan.md` has an `artifact:` URL, read the page's comments (the `ArtifactComments` tool — load it with ToolSearch). Unresolved comments are feedback: treat each one as a question to work through.
3. Ask what's changed (new info, scope change, review comments). Grill **only** on what's changed or what the comments raise — don't re-ask settled decisions unless something has invalidated them. New decisions continue the numbering (D12, D13…); a reversed decision gets a new entry that says **Supersedes:** D4, and D4 gets **Superseded by:** D12.
4. Bump the version — minor (`v0.1 → v0.2`) for additions and clarifications, major (`v0.x → v1.0`) when the plan is agreed/approved or the approach changes fundamentally. Add a changelog line saying what changed.
5. Update `plan.md`, `decisions.md` and `plan.html`, then republish to the **same** Artifact (pass its `url`; read it first if this session hasn't). Omit `capabilities` on the republish so the comment setting carries over.
6. Close the loop on comments. Claude can only reply to and resolve threads that someone has sent to Claude (Send to Claude, or @claude in the thread). On those, reply with a one-liner saying what changed in which version, then resolve them. For the rest, list them for the user — which ones you addressed and which are still open — so they can resolve them in the page or send them to Claude.
