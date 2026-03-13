# bc-fix Skill

**Trigger:** `/bc-fix`, "bc fix", "fix behavioral contract violations", "resolve bc violations", "fix bc errors"
**Version:** 1.0.0

Agentic loop that scans a repo for behavioral contract violations, analyzes patterns across all packages, proposes a DRY fix plan, gets approval, applies fixes in batches, rescans after each batch, and loops until the repo is clean.

---

## Invocation

```
/bc-fix                    # Scan + analyze + fix all ERRORs
/bc-fix --warnings         # Also fix WARNINGs (default: ERRORs only)
/bc-fix --dry-run          # Scan + show plan, but make no changes
/bc-fix --package <name>   # Fix only violations for one package (e.g. axios)
/bc-fix --skip-upload      # Run locally only, no dashboard upload
```

---

## Phase 0 — Bootstrap (reuse bc-scan auth + config)

### Step 0.1 — Resolve API key

Check in order:
1. `$BC_API_KEY` environment variable
2. `cat ~/.claude.json 2>/dev/null` → parse `projects.<cwd>.mcpServers["behavioral-contracts"].headers.Authorization` → strip `"Bearer "` prefix
3. `cat ~/.claude.json 2>/dev/null` → parse top-level `mcpServers["behavioral-contracts"].headers.Authorization`
4. `cat ~/.claude/claude_desktop_config.json 2>/dev/null` → same path

If no key found and `--skip-upload` not set: tell the user "API key not found. Set BC_API_KEY or configure MCP." Stop.

Store as `$API_KEY`.

### Step 0.2 — Find tsconfig

Check `.bc-scan` config file first:
```bash
cat .bc-scan 2>/dev/null
```
If it contains `"tsconfig": "<path>"`, use that. Otherwise discover same as bc-scan skill (Step 2).

### Step 0.3 — Resolve repository ID

Same as bc-scan skill Step 3. Store as `$REPO_ID`.

---

## Phase 1 — Baseline Scan

### Step 1.1 — Run verify-cli

```bash
npx @behavioral-contracts/verify-cli@latest \
  --tsconfig <tsconfig_path> \
  --output /tmp/bc-fix-baseline.json \
  --include-drafts \
  --no-terminal
```

If output file not created, show error and stop.

### Step 1.2 — Upload baseline (unless --skip-upload)

Build and POST the upload payload same as bc-scan skill Steps 7. This records the "before" state in the dashboard.

Store `$BASELINE_SCAN_ID` from response.

### Step 1.3 — Check if already clean

Parse `/tmp/bc-fix-baseline.json`. If zero violations (or zero ERRORs when running without `--warnings`):

```
Nothing to fix — repo is already clean.
Scan ID: <BASELINE_SCAN_ID>
```

Stop.

---

## Phase 2 — Full Violation Analysis

**Read all violations before touching any code.** Never fix one violation in isolation before seeing the full picture.

### Step 2.1 — Group violations by package

From the violations array, group by `violation.package`. For each package group, list:
- Package name
- Violation count (ERRORs / WARNINGs)
- Postcondition IDs involved
- Affected files (deduplicated)
- Unique function signatures

### Step 2.2 — Detect cross-cutting patterns

Analyze across ALL package groups simultaneously. Identify:

**Universal pattern** (applies to 2+ packages):
- Same `postconditionId` appears in 3+ packages → candidate for a shared wrapper
- All violations are "unprotected await" style → a single `withErrorHandling<T>()` generic could cover them all
- All violations are "missing try-catch" with no package-specific error type checks needed → pure DRY opportunity

**Package-specific pattern** (applies to 1 package only):
- Violations require package-specific error type checking (e.g. `axios.isAxiosError()`, `error instanceof Prisma.PrismaClientKnownRequestError`)
- Violations involve a builder/chain pattern specific to one library
- The fix logic differs meaningfully between packages

**Hybrid** (shared wrapper + package-specific error type):
- Multiple packages share the same try-catch structure, but each needs its own error type guard in the catch block
- Build one wrapper with an optional `errorHandler` callback that each package provides

**Rule of thumb for deciding:**
- 3+ packages with same postcondition → propose universal utility first
- If the catch block needs package-specific types, use a typed callback pattern: `withErrorHandling(fn, (err) => handleAxiosError(err))`
- Per-package silo is correct when: the package has a unique retry/circuit-breaker pattern, or the error handling is fundamentally different

### Step 2.3 — Read affected files

Read each unique file that contains violations. Understand:
- What error handling pattern (if any) already exists in the file
- Whether a shared utility file already exists (e.g. `lib/errors.ts`, `utils/error-handling.ts`)
- The code style and patterns the developer uses (async/await vs promises, classes vs functions, etc.)

---

## Phase 3 — Present Fix Plan (Approval Gate)

Before writing any code, present the full plan to the user. Format:

```
Behavioral Contracts — Fix Plan
═══════════════════════════════════════════════════════════

Baseline:  <N> violations (<E> errors, <W> warnings)
Approach:  <Universal helper + per-package / Per-package only / Universal only>

─── BATCH 1: Universal (covers N packages, M violations) ─────────────
  Create: <file path for shared utility, e.g. src/lib/error-handling.ts>
  Purpose: <one-line description of what the utility does>
  Packages covered: axios, prisma, stripe (and any others)
  Estimated violations resolved: M

─── BATCH 2: <package-name> (<N> violations) ──────────────────────────
  Strategy: <e.g. "Wrap 3 async functions with shared utility + AxiosError handler">
  Files touched: <list>
  Approach: <code sketch of what the fix looks like>

─── BATCH 3: <package-name> (<N> violations) ──────────────────────────
  ... (repeat for each package)

─── Rescan after each batch ─────────────────────────────────────────
  Upload results after each commit to track progress

Proceed? (yes / no / customize)
  - "yes" — execute as shown
  - "no" — cancel
  - "customize" — adjust any batch before proceeding
```

**WAIT for user response before proceeding.**

If user says "customize": ask which batch to change and what to do differently, then re-show the updated plan and wait for confirmation.

If `--dry-run` was passed: show plan but do not ask to proceed. Stop after displaying.

---

## Phase 4 — Fix Loop

Execute batches in order. For each batch:

### Step 4.1 — Apply fixes

For each file in the batch:
1. Read the file
2. Apply the fix following the developer's existing code style:
   - Match their indentation, quote style, semicolons
   - Use their existing import paths and patterns
   - If they use a custom logger, use it in the catch block
   - If they re-throw errors, preserve that pattern
   - Prefer editing existing error handlers over adding new ones
3. If creating a new shared utility file, place it where the developer's other utilities live (detect from imports in affected files)

**Quality rules for fixes:**
- Do not add boilerplate comments like `// Handle error` — write meaningful catch logic
- Do not swallow errors silently (`catch (e) {}`) — always log or rethrow
- Prefer narrowed error types over `catch (e: any)`
- If the package exports error type guards (e.g. `axios.isAxiosError`), use them
- If no specific error type exists, use `catch (error: unknown)` + `instanceof Error` narrowing

### Step 4.2 — Commit batch

After all files in the batch are fixed:

```bash
git add <specific files changed>
git commit -m "$(cat <<'EOF'
fix(bc): resolve <package or universal> violations

<1-2 sentence description of what changed and why>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

If pre-commit hook fails: fix the hook error (lint/format), re-stage, create a NEW commit (never amend).

### Step 4.3 — Rescan and upload

Run verify-cli again:

```bash
npx @behavioral-contracts/verify-cli@latest \
  --tsconfig <tsconfig_path> \
  --output /tmp/bc-fix-progress.json \
  --include-drafts \
  --no-terminal
```

Upload results (unless `--skip-upload`). Show delta:

```
Batch <N> complete
  Before: <X> violations   After: <Y> violations   Resolved: <X-Y>
  Scan ID: <new scan ID>
```

### Step 4.4 — Check for regressions or unexpected violations

If the new scan reveals violations NOT in the original baseline:
- These are likely pre-existing violations now uncovered by the fix (rare but possible)
- Report them: "Found 2 new violations not in baseline — likely uncovered by fix. Add to next batch? (yes/no)"

### Step 4.5 — Loop

If violations remain and more batches exist: proceed to next batch (no additional approval needed once the plan is approved).

If new violations were discovered in 4.4 and user said yes: append them to the plan and continue.

---

## Phase 5 — Final Verification

### Step 5.1 — Push

After all batches are complete:
```bash
git push
```

(Only push if git remote exists. If no remote, skip and note: "No remote configured — push manually when ready.")

### Step 5.2 — Final scan

Run one last scan. Upload as the "after" state.

### Step 5.3 — Report

```
Behavioral Contracts — Fix Complete
═══════════════════════════════════════════════════════════

Started:   <BASELINE_SCAN_ID> — <N> violations (<E> errors, <W> warnings)
Final:     <FINAL_SCAN_ID>   — <N> violations (<E> errors, <W> warnings)

Resolved:  <count>
Remaining: <count> (if any)
Commits:   <count>

Dashboard: https://app.behavioral-contracts.com/repositories/<REPO_ID>
```

If remaining > 0, show what's left and why it wasn't fixed:
```
Remaining violations (not auto-fixed):
  [WARN] <package>.<function> — <message>
         <file>:<line>
         Reason: <e.g. "requires manual review — complex async flow">
```

If remaining = 0:
```
All behavioral contract violations resolved.
```

### Step 5.4 — Cleanup

```bash
rm -f /tmp/bc-fix-baseline.json /tmp/bc-fix-progress.json
```

---

## What NOT to Do

- Never fix violations one-at-a-time before seeing all violations — always analyze the full picture first
- Never use `catch (e: any)` — use `unknown` + type narrowing
- Never silently swallow errors — always log or rethrow
- Never create a shared utility if only one package benefits — that's just indirection
- Never skip the approval gate (Phase 3) even if `--dry-run` is not set
- Never commit with `git add -A` — stage specific files only
- Never amend commits — always create new commits
- Never force-push

---

## Edge Cases

**No TypeScript project / no tsconfig found:** Stop with message "No tsconfig.json found. Pass --tsconfig <path> or run from the project root."

**All violations are in test files:** Note this in the plan. Ask: "All violations are in test files. Fix those too? (yes/no)"

**Violations in generated files:** Skip them. Note: "Skipped <N> violations in generated files (dist/, .next/, generated/)."

**Pre-existing error handler that's close but not compliant:** Improve it rather than replacing it. Respect the developer's intent.

**Large file with many violations:** Fix all violations in that file in one edit pass, not one-by-one.

---

## Reference

- **bc-scan skill:** `~/.claude/skills/bc-scan/SKILL.md` — upload and scan steps
- **Setup:** `~/.claude/skills/bc-scan/docs/setup.md`
- **Troubleshooting:** `~/.claude/skills/bc-scan/docs/troubleshooting.md`
