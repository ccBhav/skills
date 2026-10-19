# skills

My personal [Claude Code](https://claude.com/claude-code) skills.

## Skills

| Skill | What it does |
|-------|--------------|
| [`bulletproof`](skills/bulletproof/SKILL.md) | Applies and enforces the Bulletproof six-layer domain architecture (entities, storage, gateway, useCase, transport, setup) with SOLID boundaries. |
| [`cc-pr-review`](skills/cc-pr-review/SKILL.md) | Reviews a `constructionclock-backend` PR with two independent reviewers plus a comparer, then rewrites the PR description in the team format. |
| [`lore`](skills/lore/SKILL.md) | Tells the "lore" of a function: its last 5 changes, what it does, and a diagram of its callers. |
| [`pr-description`](skills/pr-description/SKILL.md) | Writes a PR description in the team format, commits with it, and opens a PR into `stage`. |
| [`sdk-design-patterns`](skills/sdk-design-patterns/SKILL.md) | Designs or reviews TypeScript/JavaScript/Go client SDKs: ergonomics, typed errors, versioning and deprecation, OpenAPI code generation, testing. |
| [`summerize-app`](skills/summerize-app/SKILL.md) | Writes a `SUMMARY.md` outlining what an app does, its endpoints, and how its components interact. |

## Install

### With the `skills` CLI

Install everything:

```bash
npx skills add ccBhav/skills
```

Install one skill:

```bash
npx skills add ccBhav/skills --skill lore
```

### As a Claude Code plugin

```
/plugin marketplace add ccBhav/skills
/plugin install ccbhav-skills@ccbhav-skills
```

### Manually

Copy a skill folder into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project):

```bash
git clone https://github.com/ccBhav/skills.git
cp -R skills/skills/lore ~/.claude/skills/
```

## Adding a skill

1. Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter.
2. Add `./skills/<name>` to the `skills` list in [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json).
3. Add a row to the table above.
