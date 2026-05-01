# nark-fix — Nark Fix — Claude Code Skill

> **WRITES CODE AND CREATES COMMITS.** This skill modifies your source files and commits changes to git. It always shows you a full fix plan and waits for your approval before touching anything.

A Claude Code skill that scans your TypeScript codebase for Nark profile violations, analyzes patterns across all packages, proposes a batched fix plan, and — after your approval — applies fixes with atomic commits.

## What it does

1. Runs a full baseline scan (read-only)
2. Groups all violations by package and detects cross-cutting patterns
3. Reads every affected file before proposing any changes
4. Presents a complete fix plan and **waits for your approval**
5. Applies fixes in batches, commits each batch, and rescans after each to verify progress
6. Uploads before/after results to the dashboard so you can track improvement over time

## Install

```bash
gh repo clone nark-sh/nark-fix ~/.claude/skills/nark-fix
```

Or manually:

```bash
git clone https://github.com/nark-sh/nark-fix ~/.claude/skills/nark-fix
```

## Prerequisites

1. **A Nark account** — [app.nark.sh](https://app.nark.sh) (Solo plan or higher for API access)
2. **Authenticated** — Run `npx nark auth login` to authenticate your machine
3. **MCP configured** — Follow the setup guide at [app.nark.sh/developer](https://app.nark.sh/developer)

## Usage

In any Claude Code session, type:

```
/nark-fix
```

Or say: **"nark fix"**, **"fix Nark profile violations"**, **"fix nark errors"**

### Options

| Command | Description |
|---------|-------------|
| `/nark-fix` | Scan + fix all ERROR violations (approval required) |
| `/nark-fix --warnings` | Also fix WARNING violations |
| `/nark-fix --dry-run` | Scan + show plan, but make no changes |
| `/nark-fix --audit` | Scan + triage, produce TP/FP report with impact analysis, no fixes |
| `/nark-fix --package axios` | Fix only violations for one package |
| `/nark-fix --skip-upload` | Run locally only, no dashboard upload |

### Dry run first

If you're unsure what changes will be made, always run `--dry-run` first:

```
/nark-fix --dry-run
```

This shows the full fix plan — which files will be touched, what pattern each fix follows, and how many violations each batch resolves — without writing a single line of code.

### Audit mode

To get a TP/FP analysis report without any fixes:

```
/nark-fix --audit
```

This scans, triages each violation as True Positive / False Positive / Borderline, and produces a structured report at `.nark/audit-report.md` with impact assessments and recommendations. No code changes, no commits, no dashboard uploads.

## How it finds your API key

The skill checks in this order:
1. `NARK_API_KEY` environment variable (if set)
2. `~/.claude.json` → `projects.<cwd>.mcpServers.nark.headers.Authorization`
3. `~/.claude.json` → top-level `mcpServers.nark.headers.Authorization`
4. `~/.claude/claude_desktop_config.json` → same path

If MCP is already configured for your project, you're already set.

## How fixes are applied

- Fixes match your existing code style (indentation, quote style, import paths)
- Errors are never silently swallowed — fixes always log or rethrow
- Package-specific error type guards are used when available (e.g. `axios.isAxiosError()`)
- Each batch is a separate atomic commit with a descriptive message
- Pre-commit hook failures are handled: the hook error is fixed and a new commit is created (never `--amend`)

## Contributing

Issues and PRs welcome at [github.com/nark-sh/nark-fix](https://github.com/nark-sh/nark-fix).
