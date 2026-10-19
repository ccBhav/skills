---
name: cc-pr-review
description: "Use when Bhavek asks for a PR review on roofbundle/constructionclock-backend (e.g. \"review PR 5296\"). Asks for the PR, runs two independent reviewers in parallel (Matt Pocock's two-axis code-review, and a github-pr-review inline-comment pass), has a third agent compare their findings, then rewrites the PR description in the team's pr-description format."
---

Combines three skills into one PR review of `roofbundle/constructionclock-backend`:

- **pr-description** (`~/.claude/skills/pr-description`): the team's PR description format, used to rewrite the PR's description.
- **code-review** (mattpocock/skills `skills/engineering/code-review`, MIT, © 2026 Matt Pocock): the two-axis Standards + Spec review. This is Reviewer A.
- **github-pr-review** (dev-agent-skills): a line-by-line review drafted as inline comments with code suggestions and an event type. This is Reviewer B.

A third agent, the Comparer, checks both reviews against the diff and reports where they agree and where they differ.

Everything outward-facing needs Bhavek's explicit yes in chat first. That covers editing the PR description, posting comments, approving, requesting changes and merging. Sub-agents never write to GitHub or Jira.

## Environment facts
- **GitHub access:** on the Mac, use `gh` (logged in as `ccBhav`). In a cloud session `gh api` returns 403. There, read GitHub through the built-in browser (`mcp__remote-devices__Claude_Browser__*`): load one PR page, then use `javascript_tool` with a same-origin `fetch()` for the `.diff` and other pages.
- **Local checkout:** `~/workspaces/Claude-backend/constructionclock-backend`. Use it only to read standards files, because it may be stale. The diff always comes from GitHub. Never touch other folders under `~/workspaces`.
- **Jira:** ticket keys look like `CCA-1234` / `CWA-1234` / `CAA-1234`. Fetch them with the Atlassian connector (`getJiraIssue`; load it via ToolSearch). Links use `https://roofbundle.atlassian.net/browse/<KEY>`.
- **Scratchpad:** put every intermediate file in the session scratchpad directory, referred to below as `$S`. Sub-agents may not reach the Mac's home directory, so copy anything they need into `$S`.

## Process

### 1. Ask for the PR and pin the diff
If Bhavek didn't give a PR number, ask for it. Then save:
- `$S/pr.diff`: from `gh pr diff <N> -R roofbundle/constructionclock-backend`, or `/pull/<N>.diff` in the browser. It is merge-base relative.
- `$S/pr-meta.md`: title, base ← head, author, current description, check status, and the latest commit SHA. Get these from `gh pr view <N> -R … --json title,baseRefName,headRefName,author,body,statusCheckRollup,commits`.
- `$S/commits.txt`: the commit list (oneline).

Stop here if the PR isn't found or the diff is empty. If the diff is over ~3000 lines, ask whether to review all of it or scope it to certain paths.

### 2. Gather the spec
Look in this order:
1. Ticket keys in the PR title, head branch, description or commits. Fetch each one with its description and acceptance criteria. For a stage/prod backport (e.g. `stage-CCA-xxxx`), the spec is the original ticket.
2. A spec path or link Bhavek passed.
3. A matching file under `docs/plans/`, `docs/` or `instructions/` in the local checkout.
4. Otherwise, ask Bhavek. If there is no spec, Reviewer A skips its Spec axis.

Save it to `$S/spec.md`, including the ticket key(s) and Jira URL(s).

### 3. Gather the standards
Find every file in the local checkout that documents how code should be written: `CODING_STANDARDS.md`, `CONTRIBUTING.md`, `CLAUDE.md`, `AGENTS.md`, README conventions, `instructions/`, `.claude/`, `eslint.config.cjs`, the Prettier config, and `.github/workflows/*.yml` (to see what tooling already enforces). Copy the relevant contents into `$S/standards.md`.

### 4. Spawn Reviewer A and Reviewer B in parallel
Use the Agent tool (general-purpose) and send both calls in **one message**. Tell each one to Read the `$S` files and write its full report to the named output file, and to return the same report. Neither may call GitHub or post anything.

**Reviewer A: code-review (Standards + Spec).** Prompt:
> You are running Matt Pocock's two-axis code review on PR #<N>. Read `$S/pr.diff`, `$S/commits.txt`, `$S/pr-meta.md`, `$S/standards.md` and `$S/spec.md`. Keep the two axes in separate sections and never merge or rerank findings across them.
>
> **## Standards**: report, per file/hunk, (a) every place the diff violates a documented standard, citing the standard (file + rule); and (b) any smell from the baseline below, naming it and quoting the hunk. Documented-standard breaches can be hard violations; baseline smells are always judgement calls ("possible Feature Envy"). A documented repo standard overrides the baseline. Skip anything ESLint/Prettier already enforces.
>
> **## Spec**: report (a) requirements that are missing or partial; (b) behaviour that wasn't asked for (scope creep); and (c) requirements that look implemented but look wrong. Quote the spec line for each finding. For a backport, flag drift from the dev version if the description claims parity. If `$S/spec.md` says there is no spec, write "no spec available".
>
> Give every finding an id (`A1`, `A2`, …), the `file:line`, and a severity of hard or judgement. Under 500 words. Write the report to `$S/review-a.md`.
>
> Smell baseline (Fowler, *Refactoring* ch.3), what it is → fix: **Mysterious Name** (name hides intent → rename) · **Duplicated Code** (same shape in several hunks → extract) · **Feature Envy** (method uses another object's data more than its own → move it) · **Data Clumps** (fields/params that travel together → one type) · **Primitive Obsession** (string/number standing in for a domain concept → small type) · **Repeated Switches** (same switch on same type recurs → polymorphism or a shared map) · **Shotgun Surgery** (one change scattered across files → gather) · **Divergent Change** (one module changed for unrelated reasons → split) · **Speculative Generality** (hooks for needs the spec lacks → inline/delete) · **Message Chains** (`a.b().c().d()` → hide the walk) · **Middle Man** (mostly delegates → call the target) · **Refused Bequest** (ignores what it inherits → composition).

**Reviewer B: github-pr-review (inline comments).** Prompt:
> You are drafting a GitHub pull-request review for PR #<N>, following the github-pr-review skill, but **draft only: do not run `gh` or post anything**. Read `$S/pr.diff`, `$S/pr-meta.md`, `$S/spec.md` and `$S/standards.md`. Review the diff line by line for correctness, edge cases, error handling, security (injection, auth, exposed secrets), performance (N+1 queries, needless loops), async/await misuse, missing tests and backward compatibility.
>
> For each issue, draft one inline comment with an id (`B1`, `B2`, …), its `path`, `line` (and `start_line` for a range) in the new file, `side` (RIGHT for added/changed lines, LEFT for deleted ones), a severity (blocking / non-blocking / question), and the comment body. Where a fix is concrete, include a ```suggestion block that replaces the whole line range with complete, correct code. Only comment on lines that appear in the diff.
>
> End with a recommended event type and a one-paragraph overall review message. Use REQUEST_CHANGES for bugs, security or failing tests; APPROVE when only non-blocking suggestions remain; COMMENT for questions or neutral feedback. Under 600 words. Write the review to `$S/review-b.md`.

### 5. Spawn the Comparer
Once both reviewers have finished, spawn one more general-purpose agent. Prompt:
> Compare two independent reviews of PR #<N>. Read `$S/review-a.md` (two-axis Standards/Spec review), `$S/review-b.md` (inline-comment review), `$S/pr.diff` and `$S/spec.md`. Check every finding against the diff yourself: does the quoted code exist at that location, and is the claim actually true?
>
> Report in these sections:
> 1. **Agreed**: issues both reviewers raised (pair the ids, e.g. A3 ↔ B1), with the strongest wording of the issue.
> 2. **Only Reviewer A**: and **Only Reviewer B**: each finding with your verdict: *confirmed*, *plausible* or *false positive* (give the reason).
> 3. **Conflicts**: places where the reviewers disagree, or where B's suggested code would break a standard or spec point A raised. Say which is right.
> 4. **Blind spots**: anything clearly wrong in the diff that neither reviewer caught (keep this short and only include what you are confident about).
> 5. **Verdict**: the recommended GitHub event type, the must-fix list (confirmed blocking items only), and the nice-to-haves.
>
> Keep A's Standards and Spec labels on its findings; don't collapse the axes. Under 700 words. Write the report to `$S/comparison.md`.

### 6. Report to Bhavek
Start with one header line: PR #, title, base ← head, ticket link(s) and check status. Then give the Comparer's report, lightly cleaned. Then give a one-line pointer to the full reviews in `$S/review-a.md` and `$S/review-b.md`. End with one line of totals: findings from each reviewer, how many are agreed, and how many are confirmed blocking.

### 7. Rewrite the PR description (pr-description format)
Draft a new description in exactly this structure, using the diff, the commits, the spec and the Comparer's verdict. Save it to `$S/pr-report.md`:

```
# What was the problem/requirement? (What/Why)
[2-3 sentences: the problem and why it matters]

## Was this change documented? Is there a Jira issue for it?
Jira: [<KEY>](https://roofbundle.atlassian.net/browse/<KEY>)
[Also mention docs/FILENAME.md if internal docs exist]

# What was the solution? (How)
[Approach, key files/functions changed, architecture decisions]

## Is this a breaking change?
[Yes/No; if yes, what breaks and the migration path]

# How was this change tested?
[Concrete tests in the diff, plus manual testing described in the PR]

# Code Review Notes
[From the Comparer: confirmed must-fix items, notable suggestions, positives]

## Files reviewed
- `path/to/file.js`: what changed and any notes

# Checklist
- [ ] I have run the `npm run tests`
- [ ] I have updated swagger docs, or doc update is unneeded

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

Rules: the Jira link must be a real clickable URL, so if no key was found, ask rather than omit it. Reference real file paths and function names. Keep anything important from the existing description that the new one would lose (for example deploy notes or screenshots).

Show the draft, and ask with AskUserQuestion whether to replace the PR description with it. Only on a yes, run `gh pr edit <N> -R roofbundle/constructionclock-backend --body-file $S/pr-report.md`, or paste it into the description editor in the browser for a cloud session. Never commit, push or open a new PR from this skill.

### 8. Offer to post the review
In one line, offer to post Reviewer B's inline comments, filtered to what the Comparer marked *confirmed* or *plausible*, with the Comparer's event type. On a yes, follow the github-pr-review workflow: show exactly what will be posted (each file:line, the comment text, the event type and the overall message) and get an explicit yes via AskUserQuestion. Then create a **pending** review with `gh api repos/roofbundle/constructionclock-backend/pulls/<N>/reviews` (using the latest commit SHA as `commit_id`, with `-f` for strings, `-F` for line numbers, and quoted `'comments[][…]'` params), and submit it via `…/reviews/<REVIEW_ID>/events`.

## Why two reviewers and a comparer
The two-axis review catches wrong-thing-built (Spec) and convention breaks (Standards), but it works at hunk level. The inline review works line by line and catches bugs, security and edge cases, but it ignores the ticket. Each one misses things the other sees. Running them independently, with no shared context, and then having a third agent check both against the diff, turns two opinions into a verified list. It also filters out false positives before anything reaches the PR author.
