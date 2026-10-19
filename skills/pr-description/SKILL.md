---
name: pr-description
description: Generate a well-structured PR description for the current branch, commit it as the commit message, and open a PR into stage
---

Analyze the current git branch, generate a PR description using the team's standard
format, use that report as the commit message, then push the branch and open a pull
request into `stage`.

## Process

1. **Gather context**
   - Check the current branch name and extract the Jira issue key (e.g., `CCA-7853`,
     `CWA-2364`). Branch names are usually `<KEY>-slug` or `<KEY>_slug`.
   - Build the Jira URL as `https://roofbundle.atlassian.net/browse/<KEY>` — this link
     is **required** in the report. If no key can be extracted from the branch name,
     check the existing commit messages on the branch; if still not found, ask the user
     for the ticket key rather than omitting the link.
   - Get the base branch (`stage` for this repo)
   - Run `git log <base>...HEAD --oneline` to see all commits on this branch
   - Run `git diff <base>...HEAD --stat` to see which files changed
   - Look for documentation files (docs/*.md) that describe the changes

2. **Analyze changes**
   - Read the commits to understand what was built
   - Identify the problem being solved (use commit messages and docs)
   - Identify the solution/approach taken
   - Check if any breaking changes exist (API changes, schema changes, renamed functions, removed features)
   - Note test coverage (look for test files in the diff)
   - Look for swagger/docs updates

3. **Code review** — For each modified file, examine for:
   - **Correctness** — Logic errors, edge cases, error handling, null checks
   - **Security** — SQL injection, XSS, command injection, exposed secrets, unsafe operations
   - **Performance** — N+1 queries, inefficient loops, unnecessary operations, memory leaks
   - **Code quality** — Duplication, complexity, naming, readability, adherence to patterns
   - **Best practices** — Proper error handling, async/await patterns, validation at boundaries
   - **Tests** — Unit test coverage, integration test coverage, edge case coverage
   - **Backward compatibility** — Will this break existing clients or APIs?

4. **Output format** — Use this exact structure:

```
# What was the problem/requirement? (What/Why)
[2-3 sentences explaining the problem or requirement being addressed]

## Was this change documented? Is there a Jira issue for it?
Jira: [<KEY>](https://roofbundle.atlassian.net/browse/<KEY>)
[Also mention docs/FILENAME.md if internal docs exist]

# What was the solution? (How)
[Explain the approach taken, key files/functions changed, architecture decisions]

## Is this a breaking change?
[Yes/No - if yes, clearly describe what breaks and migration path if any]

# How was this change tested?
[Describe unit tests, integration tests, manual testing, or test scenarios]

# Code Review Notes
[Key observations from code review - positive findings, concerns to address, suggestions for improvement]

## Files reviewed
- `path/to/file.js` — What changed and any notes
- `path/to/file.js` — What changed and any notes

# Checklist
- [ ] I have run the `npm run tests`
- [ ] I have updated swagger docs, or doc update is unneeded
```

5. **Quality guidelines**
   - Be specific: reference file paths and function names
   - For problems: focus on "why" not just "what"
   - For solutions: explain the approach and trade-offs if relevant
   - For breaking changes: be explicit and clear about impact
   - For testing: be concrete (not generic)
   - The Jira link must be a real clickable URL, not a bare ticket key

6. **Show the report and confirm** — Print the finished report to the user and ask them
   to confirm before anything is committed or pushed. Do not skip this: everything below
   this point is outward-facing.

7. **Commit using the report as the commit message**
   - Write the report to a scratch file (e.g. `<scratchpad>/pr-report.md`) so the exact
     text is reused for both the commit and the PR body — never retype it.
   - Stage the work: `git add -A` (review `git status` first; do not sweep in unrelated
     files such as `package-lock.json` churn unless the user wants them).
   - The commit message is: a conventional one-line subject
     (`<KEY> <type>(<scope>) <summary>`, e.g. `CCA-9081 feat(payworks) per-company
     credentials encrypted at rest`), a blank line, then the full report verbatim as the
     body. Commit with `git commit -F <file>` where the file holds subject + blank line +
     report.
   - End the commit message with the `Co-Authored-By` trailer.
   - If the branch already has commits and the working tree is clean, do **not** rewrite
     history — the report still becomes the PR body. Only amend if the user asks for it.

8. **Push and open the PR into `stage`**
   - Never do this on `stage` itself. If `git branch --show-current` is `stage`, stop and
     tell the user to branch first.
   - `git push -u origin <current-branch>`
   - `gh pr create --base stage --head <current-branch> --title "<the same subject line>"
     --body-file <scratchpad>/pr-report.md`
   - Append the standard footer to the PR body:
     `🤖 Generated with [Claude Code](https://claude.com/claude-code)`
   - If a PR for this branch already exists, update it instead:
     `gh pr edit <number> --body-file <scratchpad>/pr-report.md`
   - Report the PR URL back to the user.
