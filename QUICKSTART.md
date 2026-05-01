# Nark Quickstart Guide

Get from zero to a clean codebase in under 10 minutes.

---

## 1. Install and scan

```bash
npx nark --tsconfig ./tsconfig.json
```

That's it. Nark ships profiles for 170+ npm packages. It checks your TypeScript code against those profiles and reports missing error handling.

### Why pass `--tsconfig`?

Nark uses the TypeScript compiler API to understand your code. It needs your `tsconfig.json` to know which files to analyze, how your paths resolve, and what your module boundaries look like. Without it, Nark can't see your project the way TypeScript sees it.

If your project has multiple tsconfigs (common in monorepos), point at the one that covers the code you want to scan:

```bash
# Scan the web app
npx nark --tsconfig apps/web/tsconfig.json

# Scan the API server
npx nark --tsconfig packages/api/tsconfig.json
```

If you just run `npx nark` with no flags, it defaults to `./tsconfig.json` in the current directory.

### Initialize for easier repeat scans

```bash
npx nark init
```

This creates a `.nark/` directory with your project config so future scans remember your settings.

---

## 2. Read the output

By default, Nark shows a compact summary:

```
Nark v1.x.x — profile coverage scanner

Scanning... 247 files analyzed in 3.2s

 RESULTS
 ───────────────────────────────────
  Packages detected:  8 of 170+
  Packages with profiles: 5
  Violations: 12 (9 errors, 3 warnings)

  axios        — 4 errors
  prisma       — 3 errors, 2 warnings
  stripe       — 2 errors, 1 warning
```

### Verbose mode

Add `--verbose` to see every violation with file, line number, and what's wrong:

```bash
npx nark --tsconfig ./tsconfig.json --verbose
```

Verbose output shows:
- The exact file and line where the violation occurs
- Which postcondition is unmet (e.g., `missing-try-catch`, `record-not-found`)
- A description of what could go wrong
- The severity (ERROR or WARNING)

```
 ERROR  src/payments/charge.ts:42
  stripe.charges.create() — missing-try-catch
  This call can throw on network timeout, card declined, or rate limit.
  Without a try-catch, the error propagates as an unhandled rejection.

 WARNING  src/api/users.ts:18
  prisma.user.findUnique() — record-not-found
  No null check after query. A deleted user would cause a runtime crash.
```

Use compact mode (default) for a quick health check. Use verbose when you're ready to fix things.

### Other useful flags

| Flag | What it does |
|------|-------------|
| `--verbose` | Show every violation with file, line, description |
| `--report-only` | Always exit 0 (useful in CI when you want to see results without failing the build) |
| `--fail-threshold warning` | Fail on warnings too, not just errors |
| `--include-tests` | Also scan test files (excluded by default) |
| `--sarif` | Output in SARIF format (for GitHub code scanning, VS Code, etc.) |

---

## 3. Fix violations

You have two options: fix manually, or use nark-fix to automate it.

### Option A: Fix manually

Read the verbose output, add try-catch blocks, null checks, or error type guards where Nark flags missing handling. Run `npx nark` again to verify your fixes resolved the violations.

### Option B: Use nark-fix (recommended)

nark-fix is a Claude Code skill that reads every violation, triages for false positives, proposes a batched fix plan, and applies fixes after your approval.

**Install the skill:**

```bash
git clone https://github.com/nark-sh/nark-fix ~/.claude/skills/nark-fix
```

**Prerequisites:**

```bash
# Authenticate with nark.sh (free tier works for scanning)
npx nark auth login

# Configure MCP for dashboard integration (optional but recommended)
# Follow setup at https://app.nark.sh/developer
```

**Run it:**

In any Claude Code session, type:

```
/nark-fix
```

Or for a preview without changes:

```
/nark-fix --dry-run
```

**What happens:**

1. Nark-fix runs a baseline scan
2. It reads every flagged file and triages each violation (true positive vs. false positive)
3. It presents a fix plan grouped by package — you see exactly which files will change and how
4. You approve (or customize) the plan
5. It applies fixes in atomic commits, rescans after each batch to verify, and reports the delta
6. At the end, it generates an Impact Report showing the real-world risks you just eliminated

**Useful flags:**

| Flag | What it does |
|------|-------------|
| `--dry-run` | Show the fix plan without changing any code |
| `--warnings` | Fix warnings too (default: errors only) |
| `--package axios` | Fix only one package at a time |
| `--auto` | Fully autonomous — no approval prompts, runs to completion |
| `--batch` | Scan once, fix all, rescan once (faster for large codebases) |

---

## 4. Suppress false positives

Sometimes Nark flags code that's intentionally written that way. Suppress those with `.narkrc.json`:

```json
{
  "ignore": [
    {
      "package": "next",
      "postconditionId": "redirect-inside-try-catch",
      "reason": "Next.js redirect() throws intentionally — caught by framework, not user code"
    }
  ]
}
```

Place `.narkrc.json` next to your `tsconfig.json`. It's meant to be committed — your team sees exactly what's suppressed and why.

---

## 5. Use the dashboard

Connect your repo at [app.nark.sh](https://app.nark.sh) to get:

- **Scan history** — See violation counts over time. Every scan (manual, CI, or nark-fix) is recorded so you can track whether your codebase is getting safer or accumulating risk.
- **Fix run badges** — When nark-fix resolves violations, the dashboard shows a "Fix Run" marker on the scan timeline so you know exactly when fixes were applied and by whom.
- **Triage labels** — True positive and false positive labels carry forward across scans via fingerprint matching. Triage once, and the label sticks even as your code changes.
- **Package breakdown** — See which packages have the most violations and where your biggest risk areas are.

The dashboard turns Nark from a one-time scan into a continuous quality signal for your team.

---

## 6. Add to CI

Gate PRs on new violations only (don't block on existing debt):

```bash
npx nark ci --tsconfig ./tsconfig.json
```

This runs a diff-aware scan that compares the current branch against the baseline commit. Only violations introduced in the PR are reported. Existing violations in main are ignored.

```yaml
# .github/workflows/nark.yml
name: Nark
on: [pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # needed for diff
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npx nark ci --tsconfig ./tsconfig.json
```

---

## Quick reference

```bash
# First scan
npx nark --tsconfig ./tsconfig.json

# Detailed output
npx nark --tsconfig ./tsconfig.json --verbose

# Initialize project config
npx nark init

# See what packages are covered
npx nark show supported-packages

# Check your auth status
npx nark show deployment

# Authenticate
npx nark auth login

# CI mode (diff-aware, PRs only)
npx nark ci --tsconfig ./tsconfig.json

# SARIF output (GitHub code scanning)
npx nark --tsconfig ./tsconfig.json --sarif-output nark.sarif

# Fix violations with Claude Code
/nark-fix --dry-run          # preview
/nark-fix                    # fix errors
/nark-fix --warnings --auto  # fix everything, hands-free
```
