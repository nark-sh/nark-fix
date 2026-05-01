# nark-fix Skill

**Trigger:** `/nark-fix`, "nark fix", "fix Nark profile violations", "resolve nark violations", "fix nark errors"
**Version:** 2.2.0

Agentic loop that scans a repo for Nark profile violations, triages false positives, pushes TP/FP/resolve feedback to the dashboard via MCP tools, applies DRY fixes package-by-package, and loops until the repo is clean. Each triage decision and fix is recorded in the dashboard for corpus/verify-cli improvement.

**Goal: reach zero violations so the PR gate passes.** When a violation cannot be fixed through code (scanner limitation / Rules of Hooks / intentional design), use `.nark/config.json` ignore entries as the resolution path — not leaving violations unaddressed.

---

## Invocation

```
/nark-fix                    # Scan + analyze + fix all ERRORs
/nark-fix --warnings         # Also fix WARNINGs (default: ERRORs only)
/nark-fix --dry-run          # Scan + show plan, but make no changes
/nark-fix --audit            # Scan + triage → produce TP/FP report with impact analysis, no fixes
/nark-fix --package <name>   # Fix only violations for one package (e.g. axios)
/nark-fix --skip-upload      # Run locally only, no dashboard upload or MCP feedback
/nark-fix --local            # Point at http://localhost:3000 instead of production (for saas development)
/nark-fix --auto             # Fully autonomous: skip all approval gates and prompts, run every package to completion, auto-compact between packages so it can run indefinitely without user intervention
/nark-fix --batch            # Scan once → fix all → rescan once. No intermediate scans. For automated use.
/nark-fix --batch --auto     # Fully autonomous batch mode (no approval gate + no intermediate scans + auto-compact)
/nark-fix --resume            # Explicitly resume from .nark/fix-state.json (same as auto-detection)
/nark-fix --fresh             # Ignore any existing state file and start over
```

---

## Hands-Free / Auto-Compact Operation

`--auto` is the **run-forever** mode. It skips every approval prompt and runs all packages to completion. It also auto-compacts context between packages so a large repo with many packages never needs user intervention to restart.

### How it works

nark-fix already writes `.nark/fix-state.json` after every package batch commit. This file contains everything needed to resume: branch, baseline scan ID, per-package status (pending/complete), and the full violation ID map. Phase 0 reads it automatically on startup and jumps back into the fix loop for remaining packages.

Auto-compact uses this mechanism deliberately:

1. After each package batch commit, state is written to `.nark/fix-state.json`
2. If the auto-compact trigger fires, the skill calls `/compact` to compress context
3. On resume, Phase 0 detects the state file, skips Phase 1 (scan) and Phase 2/2.5 (triage), and goes directly to Phase 3 for remaining packages — all without user input

### Auto-compact trigger (--auto mode only)

After each package batch commit, check both conditions:

- **Package cadence:** Every 3rd completed package (`packages_completed_this_session % 3 === 0`)
- **Violation volume:** Whenever `total_violations_fixed_this_session > 20`

If either condition is true:

1. Confirm `.nark/fix-state.json` is current (it should be — Step 4.2a writes it immediately after commit)
2. Print:
   ```
   [AUTO-COMPACT] <N> packages complete — compacting context. State saved to .nark/fix-state.json.
   Resume will pick up at: <list of remaining pending packages>
   ```
3. Issue `/compact`
4. On resume: Phase 0 detects `.nark/fix-state.json`, prints the resume banner, loads state, and continues with remaining packages — no user action needed

### Incremental triage report

The final `.nark/triage-report.md` (Phase 5.5) is built incrementally: each package's section is appended to the file immediately after that package's batch commit. This means if a compact + resume happens mid-run, the report already has all completed packages and only needs the remaining packages appended at Phase 5.5. The report file is gitignored alongside `.nark/fix-state.json`.

### Recommended invocation for hands-free runs

```bash
/nark-fix --auto             # Standard mode: scan, triage, fix each package with intermediate rescans
/nark-fix --auto --batch     # Batch mode: fix all packages, then one final scan (faster, fewer API calls)
```

Both will run start to finish without requiring any user input or restarts, even on repos with 20+ packages.

---

## Handling Persistent False Positives (Goal: Zero Violations for PR Gate)

The goal of nark-fix is to reach **zero violations** so the PR gate passes. When a violation cannot be eliminated through code changes, there are two resolution mechanisms:

1. **`.narkrc.json`** — commit-tracked suppression by postconditionId (and optionally file glob). The correct tool for persistent FPs that the whole team should share. No source file changes needed.
2. **Dashboard FALSE_POSITIVE label** — marks a violation as reviewed on the dashboard. Labels carry forward scan-to-scan via fingerprint matching.

**Use `.narkrc.json` first** — it's commit-tracked (visible in PR diffs), shared across all team members, and works with the local CLI.

> **IMPORTANT — `.nark-suppressions.json` is NOT a suppression mechanism.**
> That file is telemetry-only (enriches FP signal sent to nark.sh for corpus improvement). Writing entries to it does NOT suppress violations from scan output. The only config-based suppression the analyzer reads is `.narkrc.json`.

> **IMPORTANT — `.narkrc.json` location.**
> The analyzer sets `projectRoot = path.dirname(tsconfig)`. Place `.narkrc.json` in that directory (e.g. `apps/web/.narkrc.json` when `--tsconfig apps/web/tsconfig.json`), NOT at the git repo root.

### `.narkrc.json` format

```json
{
  "ignore": [
    {
      "package": "next",
      "postconditionId": "redirect-inside-try-catch",
      "reason": "Next.js redirect() throws NEXT_REDIRECT intentionally — caught by framework error boundary, not user try-catch. Wrapping would swallow the redirect."
    },
    {
      "file": "src/generated/**",
      "package": "prisma",
      "postconditionId": "missing-error-handling",
      "reason": "Generated code — do not modify."
    }
  ]
}
```

Fields per rule (all optional except `reason`):
- `package` — package name to match (or `"*"` for any)
- `postconditionId` — the rule ID to suppress (or `"*"` for any)
- `file` — glob pattern relative to projectRoot (matches violation file path)
- `reason` — **required**, min 10 chars — audit trail for code review

At least one of `file`, `package`, or `postconditionId` must be specified.

### When to use suppression vs. code fix

| Situation | Resolution |
|-----------|-----------|
| Error state is genuinely unhandled | Fix the code (add try-catch, expose `error`, add `onError`) |
| Error is handled by graceful degradation the scanner can't detect | Fix the code if trivial (1–2 lines); else suppress via `.narkrc.json` |
| Hook must be called unconditionally (Rules of Hooks), usage is null-guarded | Add to `.narkrc.json` |
| Architectural pattern is intentional (optional FormProvider, optional context) | Add to `.narkrc.json` |
| Violation is in generated/vendored code | Add to `.narkrc.json` with `file` glob |
| After a code fix the scanner still flags the same site due to a language constraint | Add to `.narkrc.json` for the residual |
| Next.js redirect()/notFound() inside try-catch FP | Add to `.narkrc.json` — these are intentional control-flow throws |

### Suppression workflow in nark-fix

In Phase 2.5, when a violation is labeled LIKELY FALSE POSITIVE and a code fix is not appropriate:

1. Determine `postconditionId` from the violation (the `contract_clause` field)
2. Determine the correct `.narkrc.json` path: `path.dirname(<tsconfig path>)/.narkrc.json`
3. Read or create `.narkrc.json` at that location
4. Add an `ignore` entry with `package`, `postconditionId`, and a meaningful `reason`
5. `git add <path>/.narkrc.json` and include it in the batch commit
6. After the commit, trigger a rescan — suppressed violations will not appear in the next scan results
7. Also call `batch_review_violations` with `action: "FALSE_POSITIVE"` for the dashboard label (belt-and-suspenders)

### Dashboard FALSE_POSITIVE label

Use in addition to `.narkrc.json` (belt-and-suspenders), OR alone when you can't commit (e.g. a temp workaround). Labels carry forward via fingerprint matching scan-to-scan, but are not visible in PR diffs. Prefer `.narkrc.json` for the team-visible audit trail.

---

## Phase 0 — Bootstrap

### Step 0.0-pre — Check for skill updates

Before anything else, check if the local nark-fix skill is behind the remote. This is a non-blocking check — if it fails (no network, no git), skip silently.

```bash
SKILL_DIR="$HOME/.claude/skills/nark-fix"
if [ -d "$SKILL_DIR/.git" ]; then
  LOCAL_SHA=$(git -C "$SKILL_DIR" rev-parse HEAD 2>/dev/null)
  REMOTE_SHA=$(git -C "$SKILL_DIR" ls-remote origin HEAD 2>/dev/null | cut -f1)
  if [ -n "$REMOTE_SHA" ] && [ "$LOCAL_SHA" != "$REMOTE_SHA" ]; then
    echo "⚡ nark-fix skill update available. Run: cd ~/.claude/skills/nark-fix && git pull"
  fi
fi
```

If the local and remote SHAs differ, print the update notice and continue. Do NOT auto-pull — the user may be on a custom branch or have local modifications. Just inform them.

If `ls-remote` takes more than 3 seconds (slow network), skip the check entirely — do not block the fix session.

### Step 0.0 — Check for existing run state

Before anything else, check for `.nark/fix-state.json` in the current repo root:

```bash
if [ -f ".nark/fix-state.json" ]; then
  STATE=$(cat .nark/fix-state.json)
fi
```

If the file exists and `--fresh` flag was NOT passed:
- Parse `state.branch` from the JSON
- Get current branch: `git branch --show-current`
- If branches **do not match**: print "State file is from branch '<state.branch>', you are on '<current>'. Starting fresh." and continue with normal flow (the old file will be overwritten at Phase 1).
- If branches **match** and `state.phase == "fix_loop"`:
  - Print:
    ```
    Resuming previous nark-fix session.
      Baseline scan:       <state.baselineScanId>
      Packages complete:   <comma-separated list of packages where state.packages[name].status == "complete">
      Packages remaining:  <comma-separated list of packages where state.packages[name].status == "pending">
    ```
  - Load into working variables:
    - `$BASELINE_SCAN_ID` = `state.baselineScanId`
    - `$REPO_ID` = `state.repoId`
    - `$VIOLATION_ID_MAP` = `state.violationIdMap`
    - `$QUEUE_ID_MAP` = `state.queueIdMap` (may be empty if session was pre-queue-fix)
    - `$FP_NOTES` = `state.fpNotes`
    - Triage results = `state.triage`
    - Flags (warnings/auto/local/batch/skipUpload) = from `state.flags` (override with any flags passed at invocation)
  - **Skip Phase 1 entirely** (scan + violation map already done).
  - **Skip Phase 2 and Phase 2.5 entirely** (triage already done).
  - Jump to Phase 3 — but filter the fix plan to only include packages where `state.packages[name].status == "pending"`.
- If branches **match** and `state.phase == "triage"`:
  - Print: "Resuming: scan complete, re-running triage."
  - Load `$BASELINE_SCAN_ID`, `$REPO_ID`, `$VIOLATION_ID_MAP` from state.
  - Skip Phase 1. Proceed from Phase 2 normally.

**`--fresh` flag:** If `--fresh` is passed, ignore any existing state file and run the full flow from scratch (overwriting the state file at Phase 1).

**Note on gitignore:** When first writing `.nark/fix-state.json` (Step 1.2.5 below), check whether it is already in the target repo's `.gitignore`. If not, append `.nark/fix-state.json`, `.nark/fix-continuation.md`, and `.nark/impact-report.md` to `.gitignore`.

### Step 0.1 — Resolve API key

Check in order:
1. `$NARK_API_KEY` environment variable
2. `cat ~/.nark/credentials 2>/dev/null` → parse `.token` field (this is where `npx nark auth login` stores the key)
3. `cat ~/.claude.json 2>/dev/null` → parse `projects.<cwd>.mcpServers["nark"].headers.Authorization` → strip `"Bearer "` prefix
4. `cat ~/.claude.json 2>/dev/null` → parse top-level `mcpServers["nark"].headers.Authorization`
5. `cat ~/.claude/claude_desktop_config.json 2>/dev/null` → same path

If no key found and `--skip-upload` not set: tell the user "API key not found. Run `npx nark auth login` or set NARK_API_KEY". Stop.

Store as `$API_KEY`.

### Step 0.2 — Resolve base URL

Resolve in priority order:
1. `--local` flag passed at invocation → use `http://localhost:3000`
2. `.nark/config.json` file `baseUrl` field
3. `$BC_BASE_URL` environment variable
4. Default: `https://app.nark.sh`

When `--local` is set, also resolve the API key from the local dev `.env` or `.env.local` file in the saas app if the standard key lookup (Step 0.1) fails — local dev keys may differ from production keys.

Store as `$BASE_URL`.

### Step 0.3 — Resolve repository ID

Resolve the repository ID from the MCP `list_repositories` tool, matching by GitHub remote URL. Store as `$REPO_ID`.

### Step 0.3.5 — Auto-activate if repo is accessible but not activated (unless --skip-upload)

After resolving `$REPO_ID`:

If `$REPO_ID` is empty or not found (the repo hasn't been activated on the website yet), but the GitHub App is installed (nark-fix can determine this because the scan in Phase 1 will fail with a "not found" error), attempt auto-activation:

```bash
# Detect owner/repo from git remote
REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "")
# Extract owner/repo from SSH or HTTPS remote URL
# SSH:   git@github.com:owner/repo.git
# HTTPS: https://github.com/owner/repo.git
GITHUB_FULL_NAME=$(echo "$REMOTE_URL" | sed -E 's|.*github\.com[:/]||' | sed 's|\.git$||')

if [ -n "$GITHUB_FULL_NAME" ]; then
  # Try to auto-activate by calling POST /api/repositories
  ACTIVATE_RESPONSE=$(curl -s -X POST "$BASE_URL/api/repositories" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"githubFullName\":\"$GITHUB_FULL_NAME\",\"source\":\"nark-fix\"}")

  # Parse new repositoryId from response
  NEW_REPO_ID=$(echo "$ACTIVATE_RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('repository',{}).get('id',''))" 2>/dev/null || echo "")

  if [ -n "$NEW_REPO_ID" ]; then
    echo "Repository auto-activated on nark.sh — ID: $NEW_REPO_ID"
    REPO_ID=$NEW_REPO_ID
  else
    echo "Could not auto-activate: $(echo "$ACTIVATE_RESPONSE" | head -c 200)"
    echo "Visit $BASE_URL to activate this repository manually."
    # Continue with --skip-upload behavior
    SKIP_UPLOAD=true
  fi
fi
```

This runs silently when it succeeds. Only shown to user if activation fails.

The key design intent: nark-fix detects it's running on a repo that the GitHub App has access to (because the API key is valid) but hasn't been connected to the dashboard yet. It auto-connects it by calling the repositories POST endpoint with the `githubFullName` extracted from the git remote. The `source: "nark-fix"` field is passed so the backend can distinguish auto-activations from manual ones (optional logging, no behavior change needed on the API side).

Note: The existing `POST /api/repositories` endpoint already handles this case — it creates a new repository record given a valid `githubFullName` for a repo the installation can access. No backend changes needed for auto-activate.

### Step 0.4 — Capture current branch and initial commit

```bash
CURRENT_BRANCH=$(git branch --show-current 2>/dev/null || echo "local")
COMMIT_BEFORE=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
```

### Step 0.5 — Initialize feedback state

Create an in-memory (working notes) structure to track across the full session:

```
$VIOLATION_ID_MAP       = {}   # "{filePath}:{lineNumber}:{packageName}:{postconditionId}" → dashboardViolationId
$QUEUE_ID_MAP          = {}   # dashboardViolationId → queueId (from queue_fix response)
$FP_NOTES              = []   # [{violationId, note}] — scanner shortcomings from FP triage
$RESOLVED_VIOLATIONS   = []   # [{violationId, resolutionDetail}] — fixed violations
$SCANNER_ISSUES        = []   # Deduplicated scanner/corpus improvement opportunities
$TP_COUNT              = 0    # Total true positives across all packages
$FP_COUNT              = 0    # Total false positives across all packages
$BORDERLINE_COUNT      = 0    # Total borderline cases across all packages
$PACKAGES_FIXED        = []   # [{packageName, fixed, fp}] — per-package TP/FP tally
```

### Step 0.6 — Resolve tsconfig path

The nark CLI defaults to `./tsconfig.json`, but many projects have an empty root tsconfig that delegates to `tsconfig.app.json`, `tsconfig.build.json`, etc. nark-fix must find a tsconfig with actual source files.

**Resolution order:**

1. If the user passed `--tsconfig <path>` at invocation, use that path. Done.
2. Check `./tsconfig.json`:
   - Read the file
   - If it has an `"include"` array with entries (e.g. `["src"]`), OR has `"files"` with entries, use it. Done.
   - If it's empty, only has `"references"`, or only has `"compilerOptions"` with no `"include"`/`"files"`, it's a root-level project-references config — keep looking.
3. Look for common alternatives in priority order:
   ```
   ./tsconfig.app.json
   ./tsconfig.build.json
   ./tsconfig.lib.json
   ./apps/web/tsconfig.json
   ./src/tsconfig.json
   ```
   For each candidate that exists, check if it has `"include"` or `"files"` with entries. Use the first one that does.
4. If none found, also try:
   ```bash
   find . -maxdepth 3 -name 'tsconfig*.json' -not -path '*/node_modules/*' 2>/dev/null
   ```
   Read each result and pick the first with `"include"` containing source paths.
5. If still none found, fall back to `./tsconfig.json` and let the nark CLI handle the error.

**Print the result:**
```
tsconfig: <resolved path> <"(auto-detected)" if not user-specified, or "(user-specified)" if --tsconfig was passed>
```

If auto-detected and the path is NOT `./tsconfig.json`, also print:
```
Note: ./tsconfig.json was empty — using <path> which includes source files.
```

Store as `$TSCONFIG_PATH` for use in Phase 1 scan command and Phase 2.6 report header.

---

## Phase 1 — Triggering cloud baseline scan...

### Step 1.1 — Trigger cloud baseline scan

Get current branch:
```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "local")
```

Trigger a cloud scan via the MCP endpoint:
```bash
curl -s -X POST $BASE_URL/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/call\",\"params\":{\"name\":\"trigger_scan\",\"arguments\":{\"repositoryId\":\"$REPO_ID\",\"branch\":\"$CURRENT_BRANCH\"}},\"id\":1}"
```

**Hung scan handling:** If the response returns `success: false` with an `existingScan` that is `QUEUED` or `RUNNING`, call `cancel_scan` to clear it, then re-trigger:

```bash
# Cancel the hung scan
curl -s -X POST $BASE_URL/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/call\",\"params\":{\"name\":\"cancel_scan\",\"arguments\":{\"scanId\":\"$EXISTING_SCAN_ID\"}},\"id\":1}"

# Re-trigger now that the slot is clear
curl -s -X POST $BASE_URL/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/call\",\"params\":{\"name\":\"trigger_scan\",\"arguments\":{\"repositoryId\":\"$REPO_ID\",\"branch\":\"$CURRENT_BRANCH\"}},\"id\":1}"
```

Parse `.result.scan.id` from the response as `$BASELINE_SCAN_ID`. If the request fails or returns no scan id, show the error and stop.

Poll for completion using the `poll_scan_status` MCP tool (handles polling internally):
```bash
curl -s -X POST $BASE_URL/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/call\",\"params\":{\"name\":\"poll_scan_status\",\"arguments\":{\"scanId\":\"$BASELINE_SCAN_ID\"}},\"id\":1}"
```

Parse `.result.status` — if `"FAILED"`, show error and stop.

Use the cloud scan results as the baseline. Fetch violations via MCP tools:
- Call `list_packages_with_errors` with `{ repositoryId: $REPO_ID }`
- For each package, call `list_errors_for_package` with `{ repositoryId: $REPO_ID, packageName: "<package>" }`

Store `$BASELINE_SCAN_ID` for use in subsequent steps.

### Step 1.2 — Build violation ID map (unless --skip-upload)

Call `list_packages_with_errors` MCP tool:
```
Arguments: { repositoryId: $REPO_ID }
```

For each package returned, call `list_errors_for_package` MCP tool:
```
Arguments: { repositoryId: $REPO_ID, packageName: "<package>" }
```

For each violation returned by the dashboard, build the normalized key using actual violation fields (`filePath`, `lineNumber`) and the package name from the enclosing `list_errors_for_package` call:
```
key = "{filePath}:{lineNumber}:{packageName}:{postconditionId}"
```
Build `$VIOLATION_ID_MAP[key] = dashboardViolation.id`.

If MCP tools are unavailable (server offline), log "Dashboard feedback will be skipped — MCP server unreachable" and set `SKIP_UPLOAD=true`. Continue with fixes — offline mode still works.

**Note:** `list_packages_with_errors` and `list_errors_for_package` are MCP tools registered on the `nark` server. Call them via the MCP tool interface directly (not curl).

### Step 1.2.5 — Write initial state file

After building `$VIOLATION_ID_MAP`, write `.nark/fix-state.json` to the repo root:

```json
{
  "version": "1",
  "startedAt": "<ISO timestamp>",
  "branch": "<CURRENT_BRANCH>",
  "repoId": "<REPO_ID>",
  "baselineScanId": "<BASELINE_SCAN_ID>",
  "flags": {
    "warnings": "<bool — was --warnings passed>",
    "auto": "<bool — was --auto passed>",
    "local": "<bool — was --local passed>",
    "batch": "<bool — was --batch passed>",
    "skipUpload": "<bool — was --skip-upload passed>"
  },
  "phase": "triage",
  "packages": {
    "<packageName>": { "status": "pending", "violationCount": "<N>" }
  },
  "violationIdMap": "<$VIOLATION_ID_MAP object>",
  "queueIdMap": {}
}
```

// `queueIdMap` is populated during Phase 2.5.4 — maps dashboardViolationId → queueId

Also check `.gitignore` — if `.nark/fix-state.json` is not listed, append:
```
.nark/fix-state.json
.nark/fix-continuation.md
.nark/impact-report.md
.nark/audit-report.md
.nark/audit-report.html
.nark/scanner-issues.md
```

### Step 1.3 — Check if already clean

If zero violations from the MCP data (or zero ERRORs when running without `--warnings`):

```
Nothing to fix — repo is already clean.
Scan ID: <BASELINE_SCAN_ID>
```

Stop.

---

## Phase 2 — Full Violation Analysis

**Read all violations before touching any code.**

### Step 2.1 — Group violations by package

From the violations array (grouped by the `packageName` used in each `list_errors_for_package` call — violations themselves have no `package` field), list per package group:
- Package name
- Violation count (ERRORs / WARNINGs)
- Postcondition IDs involved
- Affected files (deduplicated)
- Unique function signatures

### Step 2.1.5 — Consult resolution history (unless --skip-upload)

For each package with violations, call `get_resolution_history` MCP tool:
```
Arguments: { packageName: "<package>" }
```

If prior resolutions exist, extract patterns:
- What fix approach was used before (try-catch wrapper, utility function, type guard pattern, etc.)
- Whether prior fixes in this codebase followed a specific code style

Include in the Phase 3 plan: "Prior fix pattern for `<package>`: <description>". This avoids inventing a new approach when the codebase already has an established one.

### Step 2.2 — Detect cross-cutting patterns

Analyze across ALL package groups simultaneously. Identify:

**Universal pattern** (applies to 2+ packages):
- Same `postconditionId` appears in 3+ packages → candidate for a shared wrapper
- All violations are "unprotected await" style → a single `withErrorHandling<T>()` generic could cover them all

**Package-specific pattern** (applies to 1 package only):
- Violations require package-specific error type checking (e.g. `axios.isAxiosError()`)
- Violations involve a builder/chain pattern specific to one library

**Hybrid** (shared wrapper + package-specific error type):
- Multiple packages share the same try-catch structure, but each needs its own error type guard

**Rule of thumb:**
- 3+ packages with same postcondition → propose universal utility first
- Per-package silo is correct when error handling is fundamentally different per package

### Step 2.3 — Read affected files

Read each unique file that contains violations. Understand:
- What error handling pattern (if any) already exists in the file
- Whether a shared utility file already exists (e.g. `lib/errors.ts`, `utils/error-handling.ts`)
- The code style and patterns the developer uses

---

## Phase 2.5 — False Positive Triage

**Triage every violation before writing the fix plan.** Do not count a violation as "needs fixing" until you've read the code at the flagged location.

### Step 2.5.1 — Read flagged context

For each violation in the scan output:
1. Read the flagged line ±15 lines of surrounding context
2. Check whether the concern is ALREADY satisfied:

| Postcondition | Signs of a false positive |
|---|---|
| `record-not-found` | `if (!result)` / null check within ~10 lines after the call |
| `missing-try-catch` | call is inside an outer `try { }` block or `.catch()` chain |
| `connection-error` | caller already catches connection error type |
| `$disconnect` | call is inside `.finally()` or a shutdown/cleanup handler |

### Step 2.5.2 — Label each violation and its sub-violations

Assign one label **per postcondition** — primary violation AND each sub-violation independently:
- **TRUE POSITIVE** — the concern is real and not already addressed; fix needed
- **LIKELY FALSE POSITIVE** — the concern appears already handled; scanner missed existing code
- **BORDERLINE** — ambiguous; explain why; lean toward fixing unless clearly redundant

**Sub-violations must be labeled separately.** A single call site can have a FP primary (`file-not-found`) and a TP sub-violation (`permission-denied`). Collapsing them to one label loses the signal. Track sub-violation labels as:
```
{violationId, subViolationIndex, postconditionId, label}
```
where `subViolationIndex` is the 0-based position in the `subViolations[]` array.

For each LIKELY FALSE POSITIVE (primary or sub), capture:
- The exact line(s) that already satisfy the concern
- **Why the scanner got it wrong** — what pattern did verify-cli fail to detect? What does the corpus contract not cover?

### Step 2.5.2.5 — Update session counters

After labeling all violations (primary + sub), update the session-level counters:

- Increment `$TP_COUNT` by the number of TRUE POSITIVE primary violations
- Increment `$FP_COUNT` by the number of LIKELY FALSE POSITIVE primary violations
- Increment `$BORDERLINE_COUNT` by the number of BORDERLINE primary violations
- For each package group processed, append to `$PACKAGES_FIXED`:
  ```
  { packageName: "<pkg>", fixed: <TP count for this pkg>, fp: <FP count for this pkg> }
  ```
  (borderline violations count toward `fixed` since they will be fixed)

### Step 2.5.3 — Present triage results to user

Show the triage table **before** the fix plan:

```
Triage Results
─────────────────────────────────────────────────────────────
  TRUE POSITIVE (N):
    • <file>:<line> — <postconditionId>                     [primary]
    • <file>:<line> — <postconditionId> (sub[0])            [sub-violation]

  LIKELY FALSE POSITIVE (N):
    • <file>:<line> — <postconditionId>                     [primary]
      Already handled: line <X> satisfies this
      Scanner issue: <why verify-cli/corpus missed this>
    • <file>:<line> — <postconditionId> (sub[1])            [sub-violation]
      Already handled: <explanation>
      Scanner issue: <explanation>

  BORDERLINE (N):
    • <file>:<line> — <postconditionId>
      Reason: <why it's ambiguous>
─────────────────────────────────────────────────────────────
Proceeding to fix plan for <M> actionable violations (<TP> true positives + <B> borderline).
Note: sub-violation labels will produce MIXED badges on the dashboard when they differ from primary.
```

### Step 2.5.4 — Push triage feedback to dashboard (unless --skip-upload)

After completing triage, push labels in two passes: **primary violations first, then sub-violations**.

#### Pass A — Primary violation labels

1. **TRUE POSITIVES** — call `batch_review_violations` MCP tool:
   ```
   Arguments: {
     violationIds: [list of TP dashboard violation IDs from $VIOLATION_ID_MAP],
     action: "TRUE_POSITIVE"
   }
   ```

1a. **Queue TRUE POSITIVES for agent** — after batch_review_violations TRUE_POSITIVE succeeds, call `queue_fix` once per TP violation:
    ```
    Arguments: {
      violationId: <dashboard violation ID from $VIOLATION_ID_MAP>,
      note: "Queued by nark-fix for automated repair"
    }
    ```
    Parse the returned `queueId` from the response. Store in `$QUEUE_ID_MAP[dashboardViolationId] = queueId`.
    If queue_fix fails or MCP is unreachable, log "queue_fix unavailable — skipping queue" and continue without queueId. The fix loop proceeds normally.

2. **LIKELY FALSE POSITIVES** — call `batch_review_violations` MCP tool:
   ```
   Arguments: {
     violationIds: [list of FP dashboard violation IDs],
     action: "FALSE_POSITIVE"
   }
   ```

3. **BORDERLINE** — call `batch_review_violations` MCP tool:
   ```
   Arguments: {
     violationIds: [list of borderline dashboard violation IDs],
     action: "FLAG"
   }
   ```

4. **For each LIKELY FALSE POSITIVE** — call `add_violation_note` MCP tool to record the scanner shortcoming:
   ```
   Arguments: {
     violationId: <dashboard violation ID>,
     note: "Scanner FP: <what existing code already satisfies this postcondition>. Improvement opportunity: <specific change needed in verify-cli or corpus to detect this pattern — e.g. 'verify-cli does not trace try-catch blocks in parent scopes', or 'corpus contract for <package> does not recognize the .catch() chaining pattern as satisfying missing-try-catch'>"
   }
   ```

   Add each note to `$SCANNER_ISSUES` for the Phase 5 report (deduplicate by improvement description).

#### Pass B — Sub-violation labels (per-postcondition)

For every sub-violation that has a **different label from its parent**, call `batch_review_violations` individually with `subViolationIndex`:

```
Arguments: {
  violationIds: [<parent dashboard violation ID>],
  action: "<TRUE_POSITIVE | FALSE_POSITIVE | FLAG>",
  subViolationIndex: <0-based index in subViolations array>
}
```

Call once per sub-violation that needs a label. Sub-violations with the **same label as the primary** do not need a separate call — they inherit. Only call when the label differs.

**Example:** Primary is FALSE_POSITIVE, but sub[0] (`permission-denied`) is TRUE_POSITIVE:
```
batch_review_violations({ violationIds: ["id"], action: "TRUE_POSITIVE", subViolationIndex: 0 })
```
This produces a MIXED badge on the dashboard instead of collapsing everything to FALSE_POSITIVE.

5. If a dashboard violation ID is not found in `$VIOLATION_ID_MAP` for a given violation (match failure), skip the feedback call for that violation and log: "No dashboard ID found for <file>:<line>:<package> — skipping feedback".

### Step 2.5.5 — Update state file with triage results

Update `.nark/fix-state.json`: set `phase` to `"fix_loop"` and add triage results:

```json
{
  "...existing fields...": "...",
  "phase": "fix_loop",
  "triage": {
    "truePositives": ["<file>:<line>:<package>:<postconditionId>"],
    "falsePositives": ["<file>:<line>:<package>:<postconditionId>"],
    "borderline": ["<file>:<line>:<package>:<postconditionId>"]
  },
  "fpNotes": [ { "violationId": "<id>", "note": "<scanner shortcoming>" } ]
}
```

Read the existing file, merge in the new fields, and overwrite.

---

## Phase 2.6 — Audit Report (only when `--audit`)

**If `--audit` was NOT passed, skip this section entirely and proceed to Phase 3.**

When `--audit` is passed, the skill produces a structured TP/FP analysis report and stops — no fix planning, no code changes, no commits, no dashboard upload.

### Step 2.6.1 — Generate audit report

After Phase 2.5 triage is complete, write `.nark/audit-report.md` with the following structure:

```markdown
# Nark Audit Report

**Project:** <repo name from git remote or directory name>
**Date:** <ISO date>
**Nark Version:** <from scan output>
**Git Commit:** <COMMIT_BEFORE>
**tsconfig:** <tsconfig path used for the scan>
**Total Violations:** <N> (<TP_COUNT> True Positives, <FP_COUNT> False Positives, <BORDERLINE_COUNT> Borderline)

---

## Summary

| Package | PostconditionId | Count | True Positive | False Positive | Borderline |
|---------|----------------|-------|---------------|----------------|------------|
| <package> | <postconditionId> | <N> | <tp> | <fp> | <bl> |
| ... | ... | ... | ... | ... | ... |
| **TOTAL** | | **<N>** | **<TP>** | **<FP>** | **<BL>** |

---

## Impact Summary

| Risk Level | Description | Count |
|-----------|-------------|-------|
| **Critical** | <description> | <N> |
| **High** | <description> | <N> |
| **Medium** | <description> | <N> |
| **Low** | <description> | <N> |

---

## TRUE POSITIVES (<TP_COUNT>)

Order by severity: Critical first, then High, Medium, Low.

### <N>. <package>:<postconditionId> — <count> TP (<IMPACT LEVEL>)

**What happens if left unfixed:** <impact description — e.g. "Users see blank screens with no error feedback when API calls fail", "Component crashes on invalid external data">

**Affected locations:**
- `<file>:<line>` — <brief description>
- ...

... (repeat for each TP group, ordered by impact)

---

## BORDERLINE (<BORDERLINE_COUNT>)

### <N>. <package>:<postconditionId>

- **<file>:<line>**: <why it's ambiguous>

---

## Recommendations

<Prioritized list of recommended actions, grouped by effort/impact. Suggest systemic fixes (e.g. global error handlers) over per-violation fixes where applicable. Number each priority.>

---

## Suggested Suppressions (.narkrc.json)

Ready-to-use ignore entries for confirmed false positives. Copy into your `.narkrc.json`:

\```json
{
  "ignore": [
    {
      "package": "<package>",
      "postconditionId": "<postconditionId>",
      "file": "<optional glob if FP is file-specific>",
      "reason": "<concise explanation of why this is a false positive>"
    }
  ]
}
\```

Note: When FPs are mixed with TPs at the same postconditionId level, they cannot be blanket-suppressed. Those require per-file suppressions or scanner improvements.

---

## Appendix A: Scanner/Corpus Improvement Opportunities

<Compact table summarizing scanner issues discovered during triage. Reference .nark/scanner-issues.md for full detail.>

| Package | Issue |
|---------|-------|
| <pkg> | <brief description of what the scanner got wrong> |

---

## Appendix B: False Positive Detail (<FP_COUNT>)

<Full detail on each FP group — moved to appendix so the reader hits actionable findings first.>

### <N>. <package>:<postconditionId> — <count> FP

- **<file>:<line>**: <explanation of why this is a false positive — what existing code already handles the concern>
- **Scanner issue**: <what the scanner/corpus missed — e.g. "doesn't detect try-catch in parent scope", "confuses NextResponse.redirect() with redirect() from next/navigation">

... (repeat for each FP group)
```

### Step 2.6.2 — Impact classification

For each TRUE POSITIVE group, assign an impact level:

| Impact Level | Criteria |
|-------------|----------|
| **High** | Unhandled error crashes the application/server, data loss, security issue |
| **Medium** | Silent failure with degraded UX (blank screens, missing feedback, stale data) |
| **Low** | Edge-case failure unlikely to occur in practice, cosmetic, or already partially mitigated |

### Step 2.6.3 — Write .narkrc.json suppressions for confirmed FPs

After generating the report, also persist the FP findings as actionable suppressions:

1. **Read or create `.narkrc.json`** at `path.dirname(<tsconfig path>)`
2. For each LIKELY FALSE POSITIVE group, add an `ignore` entry with:
   - `package` — the package name
   - `postconditionId` — the rule ID
   - `file` — glob pattern if the FP is file-specific (omit if package-wide)
   - `reason` — concise explanation (min 10 chars) from the triage analysis
3. **Deduplicate** — don't add entries that already exist in `.narkrc.json`
4. Write the updated file

This ensures the triage work is not wasted — future scans (and `--dry-run` / full fix runs) will not re-flag these confirmed FPs.

### Step 2.6.4 — Write scanner improvement notes

Write `.nark/scanner-issues.md` with deduplicated scanner/corpus improvement opportunities discovered during triage:

```markdown
# Scanner Improvement Notes

Generated by `nark-fix --audit` on <date>

| Package | PostconditionId | Issue | Suggestion |
|---------|----------------|-------|------------|
| <pkg> | <rule> | <what the scanner got wrong> | <how to fix in verify-cli or corpus> |
```

This file helps the nark team improve the corpus. It is gitignored.

### Step 2.6.5 — Generate HTML report and open in browser

Convert the markdown audit report to a styled HTML file so it can be opened in a browser and easily printed to PDF.

```bash
npx marked -i .nark/audit-report.md -o .nark/audit-report.html 2>/dev/null
```

If `marked` is not available or fails, fall back to a raw approach:

```bash
# Wrap markdown in minimal HTML with GitHub-flavored styling
cat > .nark/audit-report.html << 'HTMLEOF'
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Nark Audit Report</title>
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 900px; margin: 40px auto; padding: 0 20px; color: #1a1a1a; line-height: 1.6; }
  h1 { border-bottom: 2px solid #e1e4e8; padding-bottom: 8px; }
  h2 { border-bottom: 1px solid #e1e4e8; padding-bottom: 6px; margin-top: 2em; }
  table { border-collapse: collapse; width: 100%; margin: 1em 0; }
  th, td { border: 1px solid #d0d7de; padding: 6px 12px; text-align: left; font-size: 13px; }
  th { background: #f6f8fa; font-weight: 600; }
  code { background: #f6f8fa; padding: 2px 6px; border-radius: 3px; font-size: 13px; }
  pre { background: #f6f8fa; padding: 16px; border-radius: 6px; overflow-x: auto; }
  pre code { background: none; padding: 0; }
  hr { border: none; border-top: 1px solid #e1e4e8; margin: 2em 0; }
  strong { color: #0d1117; }
  @media print { body { max-width: 100%; margin: 0; } }
</style>
</head>
<body>
HTMLEOF

# Append converted markdown (use marked if available, otherwise include raw markdown in <pre>)
if command -v npx &>/dev/null && npx marked --version &>/dev/null 2>&1; then
  npx marked -i .nark/audit-report.md >> .nark/audit-report.html
else
  echo "<pre>" >> .nark/audit-report.html
  cat .nark/audit-report.md >> .nark/audit-report.html
  echo "</pre>" >> .nark/audit-report.html
fi

echo "</body></html>" >> .nark/audit-report.html
```

Then open it:
```bash
open .nark/audit-report.html 2>/dev/null || xdg-open .nark/audit-report.html 2>/dev/null || echo "Open .nark/audit-report.html in your browser"
```

The user can then `Cmd+P` → "Save as PDF" to share it.

### Step 2.6.6 — Print summary and stop

Print a summary to the user:

```
Nark Audit Report
═══════════════════════════════════════════════════════════

Violations analyzed:  <N>
True Positives:       <TP_COUNT> (Critical: <C>, High: <H>, Medium: <M>, Low: <L>)
False Positives:      <FP_COUNT>
Borderline:           <BORDERLINE_COUNT>

Outputs:
  Report (HTML):     .nark/audit-report.html  ← opened in browser (Cmd+P to save as PDF)
  Report (Markdown): .nark/audit-report.md
  Suppressions:      .narkrc.json (<N> entries added)
  Scanner issues:    .nark/scanner-issues.md
```

**Stop here.** Do not proceed to Phase 3. No fixes, no commits, no dashboard upload.

---

## Phase 3 — Present Fix Plan (Approval Gate)

Before writing any code, present the full plan to the user. Format:

```
Nark — Fix Plan
═══════════════════════════════════════════════════════════

Branch:    <$CURRENT_BRANCH>
Baseline:  <N> violations (<E> errors, <W> warnings) — <M> actionable after triage
Approach:  <Universal helper + per-package / Per-package only / Universal only>

─── BATCH 1: Universal (covers N packages, M violations) ─────────────
  Create: <file path for shared utility>
  Purpose: <one-line description>
  Packages covered: <list>
  Estimated violations resolved: M
  Prior pattern: <from resolution history if available>

─── BATCH 2: <package-name> (<N> violations) ──────────────────────────
  Strategy: <e.g. "Wrap 3 async functions with shared utility + AxiosError handler">
  Files touched: <list>
  Approach: <code sketch>

─── BATCH 3: <package-name> (<N> violations) ──────────────────────────
  ... (repeat for each package)

─── After each batch ────────────────────────────────────────────────
  Commit → rescan → upload delta → push resolve feedback to dashboard

```

If `--batch` mode is active: add this note below the plan header:
```
Note: --batch mode active. Intermediate rescans are disabled.
All fixes will be committed sequentially, then ONE final scan will verify everything.
```

If `--auto` was passed: print the plan and immediately proceed without asking. Do not use AskUserQuestion.

Otherwise, use the AskUserQuestion tool with a dropdown:

```
AskUserQuestion(
  header: "Fix Plan Approval",
  question: "Ready to apply these fixes?",
  options: [
    { label: "Yes — proceed", description: "Apply all batches as planned" },
    { label: "No — abort", description: "Cancel without making any changes" },
    { label: "Customize", description: "Modify the plan before proceeding" }
  ],
  multiSelect: false
)
```

**WAIT for user response before proceeding.**

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
3. If creating a new shared utility file, place it where the developer's other utilities live

**Quality rules for fixes:**
- Do not add boilerplate comments like `// Handle error` — write meaningful catch logic
- Do not swallow errors silently (`catch (e) {}`) — always log or rethrow
- Prefer narrowed error types over `catch (e: any)`
- If the package exports error type guards (e.g. `axios.isAxiosError`), use them
- If no specific error type exists, use `catch (error: unknown)` + `instanceof Error` narrowing

**Track what was done per violation:**
For each violation being fixed, note the resolutionDetail:
```
"Added try-catch wrapping <function> call with <error type> guard. <package> violations resolved by: <pattern description>"
```
Store in `$RESOLVED_VIOLATIONS` as `{violationId, resolutionDetail}`.

### Step 4.2 — Commit batch

```bash
git add <specific files changed>
git commit -m "$(cat <<'EOF'
fix(bc): resolve <package or universal> violations

<1-2 sentence description of what changed and why>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

If pre-commit hook fails: fix the hook error, re-stage, create a NEW commit (never amend).

After committing, perform two housekeeping writes:

**4.2a — Update state file:** Read `.nark/fix-state.json`, set `packages["<this package>"].status = "complete"` and `packages["<this package>"].commit = "<git rev-parse --short HEAD>"`, then overwrite the file.

**4.2b — Write continuation file:** Overwrite `.nark/fix-continuation.md` with:

```markdown
# nark-fix continuation

If this session ran out of context, simply re-run:

    /nark-fix --auto --local --warnings

nark-fix will auto-detect `.nark/fix-state.json` and resume from where it stopped.

---

If the state file was lost, use this manual prompt:

I was running nark-fix and ran out of context. Here's where I left off:

- Branch: <CURRENT_BRANCH>
- Baseline scan: <BASELINE_SCAN_ID>
- Completed packages: <package> (commit <sha>), ...
- Remaining packages: <package> (<N> violations), ...

Triage summary (do not re-triage — use these results):
<one line per package: "package: N TPs, M FPs, K borderline">

Please continue the fix loop for remaining packages, skipping completed ones.
```

Update this file after every batch commit so it always reflects current progress.

**4.2c — Append to incremental triage report:** After each package batch commit, append that package's section to `.nark/triage-report.md` (create if it doesn't exist). Use the same format as Phase 5.5 but for this package only. This ensures partial results survive a compact + resume cycle. Mark the file as gitignored alongside `.nark/fix-state.json` if not already.

**4.2d — Auto-compact (--auto mode only):** After the state file, continuation file, and triage report are current, check auto-compact trigger:

- Condition A: `packages_completed_this_session % 3 === 0` (every 3rd package)
- Condition B: `total_violations_fixed_this_session > 20`

If either condition is true:
1. Print:
   ```
   [AUTO-COMPACT] <N> packages complete — compacting context. State saved to .nark/fix-state.json.
   Remaining: <list of pending package names>
   ```
2. Issue `/compact` to compress context
3. On resume: Phase 0 detects `.nark/fix-state.json`, prints the resume banner, and continues with remaining pending packages — no user action needed

### Step 4.3 — Rescan and upload

Trigger a new cloud scan and poll to completion (same procedure as Step 1.1 — use `trigger_scan` then `poll_scan_status` via `/api/mcp`).

Fetch results via MCP tools (unless `--skip-upload`). Show delta with severity breakdown:

```
Batch <N> complete
  Before: <X> violations (<E_before> ERRORs, <W_before> WARNINGs)
  After:  <Y> violations (<E_after> ERRORs, <W_after> WARNINGs)
  Resolved: <X-Y> net
  Scan ID: <new scan ID>
```

**If severity changed but count unchanged** (e.g. 3 ERRORs → 3 WARNINGs), make this explicit:

```
Batch <N> complete
  ✅ ERROR violations eliminated: <list postconditionIds>
  ⚠️  New WARNINGs surfaced:     <list postconditionIds>
  Scan ID: <new scan ID>
```

### Step 4.4 — Push resolve feedback to dashboard (unless --skip-upload)

For each violation fixed in this batch, push resolve feedback using the appropriate tool:

**When queueId is available** (stored in `$QUEUE_ID_MAP` for this violation) — call `mark_fixed` MCP tool:
```
Arguments: {
  queueId: <$QUEUE_ID_MAP[violationId]>,
  resolution: "<what was done: e.g. 'Wrapped axios.get call in try-catch with axios.isAxiosError type guard. Error is logged then rethrown to preserve caller handling.'>",
  resolveViolation: true
}
```
`mark_fixed` with `resolveViolation: true` closes the queue item AND writes the RESOLVE review in one call. Do NOT also call `resolve_violation` — that would double-write.

**Fallback when no queueId** (mid-session additions, violations added after initial triage) — call `resolve_violation` MCP tool:
```
Arguments: {
  violationId: <dashboard violation ID from $VIOLATION_ID_MAP>,
  resolutionDetail: "<what was done>"
}
```

Call individually (not batch) to provide rich per-violation detail. If the dashboard violation ID is not in `$VIOLATION_ID_MAP`, skip and log.

**For violations added to .narkrc.json or genuinely unfixable** — when queueId is available, call `mark_fixed` with `resolveViolation: false`:
```
Arguments: {
  queueId: <$QUEUE_ID_MAP[violationId]>,
  resolution: "Cannot fix: <reason>. Added ignore rule to .narkrc.json.",
  resolveViolation: false
}
```
This closes the queue item without writing a RESOLVE review (the suppression is the resolution path).

### Step 4.5 — Check for regressions or unexpected violations

If the new scan reveals violations NOT in the original baseline:

- If `--auto` is set: automatically append them to the fix queue, log "Auto-adding <N> newly uncovered violations to next batch", and continue.
- Otherwise: use AskUserQuestion(
    header: "New Violations Detected",
    question: "Found <N> new violations not in the original baseline — likely uncovered by the fix. Add to next batch?",
    options: [
      { label: "Yes — add to plan", description: "Append new violations to the fix queue and continue" },
      { label: "No — skip", description: "Leave them for a future run" }
    ],
    multiSelect: false
  )

### Step 4.6 — Loop

If violations remain and more batches exist: proceed to next batch (no additional approval needed once the plan is approved).

If new violations were discovered in 4.5 and user said yes (or `--auto` is set): append them to the plan and continue.

---

## Phase 4 (--batch mode) — Fix All Packages, Then One Rescan

When `--batch` flag is set, replace the standard Phase 4 loop with this sequence.

**DO NOT rescan after each package batch.** Instead:

### Step 4.1 — Apply all fixes

For each package batch in the approved plan:
- Apply all code fixes (same quality rules as standard mode)
- Commit each batch:
  ```bash
  git add <specific files changed>
  git commit -m "$(cat <<'EOF'
  fix(bc): resolve <package> violations

  <1-2 sentence description>

  Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
  EOF
  )"
  ```
- Do NOT trigger a cloud scan after this commit
- Do NOT push resolve feedback yet (no new scan ID)
- After each commit, perform these housekeeping writes (batch mode variants):

  **4.2a — Update state file:** Read `.nark/fix-state.json`, set `packages["<this package>"].status = "complete"` and `packages["<this package>"].commit = "<git rev-parse --short HEAD>"`, then overwrite the file.

  **4.2b — Write continuation file:** Overwrite `.nark/fix-continuation.md` with:

  ```markdown
  # nark-fix continuation

  If this session ran out of context, simply re-run:

      /nark-fix --auto --local --warnings --batch

  nark-fix will auto-detect `.nark/fix-state.json` and resume from where it stopped.
  Note: batch mode is active — rescans are deferred to the final verify step.

  ---

  If the state file was lost, use this manual prompt:

  I was running nark-fix in batch mode and ran out of context. Here's where I left off:

  - Branch: <CURRENT_BRANCH>
  - Baseline scan: <BASELINE_SCAN_ID>
  - Completed packages: <package> (commit <sha>), ...
  - Remaining packages: <package> (<N> violations), ...

  Triage summary (do not re-triage — use these results):
  <one line per package: "package: N TPs, M FPs, K borderline">

  Please continue the fix loop for remaining packages, skipping completed ones.
  When all packages are fixed, run the final batch scan (Step 4.3).
  ```

  **4.2c — Append to incremental triage report:** Append this package's section to `.nark/triage-report.md` (create if it doesn't exist, gitignore alongside `.nark/fix-state.json`). Same format as Phase 5.5 but for this package only.

  **4.2d — Auto-compact (--auto mode only):** After state file, continuation file, and triage report are current:
  - Trigger if: `packages_completed_this_session % 3 === 0` OR `total_violations_fixed_this_session > 20`
  - Print: `[AUTO-COMPACT] <N> packages complete — compacting. State saved to .nark/fix-state.json. Remaining: <list>`
  - Issue `/compact`. On resume, Phase 0 reads state and continues from next pending package automatically.

- Continue to next package batch

Print after each commit:
```
✓ <package>  (<N> violations fixed, committed — scan deferred to final verify)
```

### Step 4.2 — Push all commits

```bash
git push
```

### Step 4.3 — One final scan

Trigger ONE cloud scan (same as Step 1.1 — use `trigger_scan` then `poll_scan_status` via `/api/mcp`). Wait for completion.

### Step 4.4 — Push all resolve feedback

For every violation that was fixed across ALL batches, push resolve feedback using the appropriate tool:

**When queueId is available** (stored in `$QUEUE_ID_MAP` for this violation) — call `mark_fixed` MCP tool:
```
Arguments: {
  queueId: <$QUEUE_ID_MAP[violationId]>,
  resolution: "<what was done>",
  resolveViolation: true
}
```
`mark_fixed` with `resolveViolation: true` closes the queue item AND writes the RESOLVE review in one call. Do NOT also call `resolve_violation` — that would double-write.

**Fallback when no queueId** (mid-session additions, violations added after initial triage) — call `resolve_violation` MCP tool:
```
Arguments: {
  violationId: <dashboard violation ID from $VIOLATION_ID_MAP>,
  resolutionDetail: "<what was done>"
}
```

Call individually (not batch) to provide rich per-violation detail. If the dashboard violation ID is not in `$VIOLATION_ID_MAP`, skip and log.

**For violations added to .narkrc.json or genuinely unfixable** — when queueId is available, call `mark_fixed` with `resolveViolation: false`:
```
Arguments: {
  queueId: <$QUEUE_ID_MAP[violationId]>,
  resolution: "Cannot fix: <reason>. Added ignore rule to .narkrc.json.",
  resolveViolation: false
}
```
This closes the queue item without writing a RESOLVE review (the suppression is the resolution path).

Call `batch_review_violations` for the initial triage labels (TP/FP/FLAG) if not yet pushed.
Use the new final scan ID for any scan-scoped feedback.

### Step 4.5 — Delta report

```
Batch complete
Before: <N> violations
After:  <M> violations
Fixed:  <N-M>
Remaining: <M> (list if > 0)
Scans triggered: 2 (baseline + verify) vs <N+1> in standard mode
```

---

## Phase 5 — Final Verification

### Step 5.1 — Push

After all batches are complete:
```bash
git push
```

(Only push if git remote exists.)

### Step 5.2 — Final scan

Trigger one final cloud scan (same as Step 1.1 — use `trigger_scan` then `poll_scan_status` via `/api/mcp`) and wait for completion.

### Step 5.3 — Report

```
Nark — Fix Complete
═══════════════════════════════════════════════════════════

Branch:    <$CURRENT_BRANCH>
Started:   <BASELINE_SCAN_ID> — <N> violations (<E> errors, <W> warnings)
Final:     <FINAL_SCAN_ID>   — <N> violations (<E> errors, <W> warnings)

Resolved:  <count> violations (TP fixes)
Remaining: <count> (if any)
Commits:   <count>

Dashboard: $BASE_URL/repositories/<REPO_ID>
```

If remaining > 0, show what's left and why it wasn't fixed.

If false positives were identified in Phase 2.5, list them:
```
False positives identified (not fixed — scanner limitation):
  <file>:<line> — <postconditionId>
  Already handled by: <description of existing code>
```

### Step 5.4 — Scanner/Corpus Improvement Report

Collect `$SCANNER_ISSUES` (deduplicated FP notes from Step 2.5.4) and present:

```
─── Scanner/Corpus Improvement Opportunities ──────────────────────
  (Logged to dashboard for verify-cli/corpus team review)

  verify-cli issues:
    • <issue description> (<N> occurrences)
      Example: "does not trace try-catch blocks in parent function scopes"
      Example: "does not recognize .catch() chaining as satisfying missing-try-catch"

  Corpus issues:
    • <issue description> (<N> occurrences)
      Example: "@stripe/stripe-js contract missing .checkout.sessions.create() in tracked functions"
      Example: "prisma contract does not handle generic catch blocks as valid for record-not-found"
```

These notes are already stored in the dashboard on the individual violations. This surface here is for your awareness to prioritize corpus/verify-cli work.

If zero issues: "No scanner false positives detected — corpus coverage looks good for this codebase."

### Step 5.5 — Triage Report

Write a `.nark/triage-report.md` file to the repo root. This file is meant to be shared with other developers and the corpus team. It should be comprehensive and standalone.

**File path:** `<repo root>/.nark/triage-report.md`

**Content structure:**

```markdown
# Nark Triage Report
**Repository:** <githubFullName or local path>
**Branch:** <branch> @ <commitSha>
**Scan ID:** <BASELINE_SCAN_ID> → <FINAL_SCAN_ID>
**Date:** <today>

---

## Summary

| Package | Violations | Verdict | Action |
|---------|-----------|---------|--------|
| `<pkg>` | N | ✅ TRUE POSITIVE / ❌ FALSE POSITIVE / Mixed | Code fixes applied / No changes — scanner limitation / N fixed, M FP |
| **Total** | **N** | | |

---

## <Package> — <N> TRUE POSITIVES / FALSE POSITIVES / Mixed

### What the scanner found
<Describe the postconditionIds flagged and what they mean in this codebase>

### Hotspot breakdown (if TPs)
| File | Violations |
...

### Fix patterns applied (if TPs)
<Before/after code examples showing exactly what was changed>

### Why these are false positives (if FPs)
<Explain what existing code already satisfies the postcondition>
<Explain what pattern the scanner failed to recognize>

### Borderline cases (if any)
| File | Issue |
...

---

## Scanner/Corpus Improvement Opportunities

<For each FP group, describe the specific corpus or verify-cli change needed>

---

## What Was Sent to the Dashboard

| Action | Count | Package |
...

---

## Next Steps (if any violations remain)

<List remaining violations and what to do about them>

---

*Generated by Nark nark-fix workflow · Scan <BASELINE_SCAN_ID>*
```

**Rules:**
- Include real before/after code examples, not placeholders — copy actual code from the fixed files
- For FP groups, always include "Root cause" explaining the scanner limitation
- For TP groups, include the hotspot file breakdown table
- If `--dry-run` was used and no fixes were applied, note that in the summary
- If `--skip-upload` was used, omit the "What Was Sent to the Dashboard" section

### Step 5.5.5 — Impact Report

After the triage report, generate an Impact Report that shows developers the real-world consequences of the violations that were fixed. This makes the value of nark-fix concrete and visceral.

**When to generate:** Always, unless `--dry-run` was used and no fixes were applied.

**Data sources:** Use the per-violation data already accumulated during the fix loop:
- `packageName` — which package the call belongs to
- `postconditionId` — what kind of error handling was missing
- `filePath` and function context — what the code does
- `severity` — ERROR or WARNING
- `$TP_COUNT`, `$FP_COUNT` — session counters
- `$RESOLVED_VIOLATIONS` — list of `{violationId, resolutionDetail}`

#### Step 5.5.5a — Build scenario lines

For each fixed TRUE POSITIVE violation, generate a 1-sentence "What Could Have Gone Wrong" scenario. Use the `postconditionId`, `packageName`, and the function/file context to make it specific and concrete.

**Pattern:** `"Without this fix, <failure mode> on <package>.<function>() would <user-visible consequence>"`

**Scenario generation rules:**
- Reference the actual function name and package from the violation
- Describe a realistic failure mode based on `postconditionId`:

| postconditionId pattern | Failure mode to describe |
|---|---|
| `missing-try-catch` / `missing-error-handling` | Network timeout, DNS failure, connection refused, or API error would propagate as unhandled rejection |
| `record-not-found` | A missing/deleted record would cause a null reference crash instead of a 404 |
| `connection-error` | Database/service connection loss would crash the process instead of retrying or failing gracefully |
| `rate-limit` / `api-error-rate-limit` | A 429 response would be treated as a generic error instead of backing off |
| `permission-denied` / `authorization` | An auth failure would show a cryptic 500 instead of a clear 403 |
| `validation-error` | Invalid input would crash the handler instead of returning a 400 with field errors |
| `timeout` | A slow upstream would hang the request indefinitely instead of failing fast |
| `disconnect` / `cleanup` | Resource leak — connection/handle not released on error path |

- Keep it to ONE sentence, max ~120 chars
- Be specific to the codebase — mention the file purpose if clear from the path (e.g., "payment handler", "user signup", "background sync job")

**Examples:**
```
- stripe.charges.create() in src/payments/charge.ts — Without this fix, a network timeout would return a generic 500 to the user instead of showing "payment processing failed, please retry"
- axios.get() in src/jobs/sync-inventory.ts — Without this fix, a DNS failure would crash the background job silently, leaving inventory stale for hours
- prisma.user.findUnique() in src/api/users/[id].ts — Without this fix, a deleted user would crash with "Cannot read properties of null" instead of returning 404
```

#### Step 5.5.5b — Compute risk score

```
RISK_SCORE = (errors_fixed * 3) + (warnings_fixed * 1)
```

Where `errors_fixed` = count of fixed violations with severity ERROR, `warnings_fixed` = count with severity WARNING.

Risk level label:
- 0: "No risk reduced" (nothing was fixed)
- 1–5: "Low risk eliminated"
- 6–15: "Moderate risk eliminated"
- 16–30: "High risk eliminated"
- 31+: "Critical risk eliminated"

#### Step 5.5.5c — Print to terminal

After the triage report output, print:

```
Nark — Impact Report
═══════════════════════════════════════════════════════════

Violations fixed: <TP_COUNT>
  Errors:   <errors_fixed>
  Warnings: <warnings_fixed>

Risk score: <RISK_SCORE> — <risk level label>

What Could Have Gone Wrong
───────────────────────────────────────────────────────────
  1. <package>.<function>() in <short file path>
     <scenario sentence>

  2. <package>.<function>() in <short file path>
     <scenario sentence>

  ...
═══════════════════════════════════════════════════════════
Full report saved to .nark/impact-report.md
```

If more than 10 scenarios, show the first 10 in the terminal and note: "... and <N> more (see .nark/impact-report.md for full list)"

#### Step 5.5.5d — Write .nark/impact-report.md

Write the full impact report to `<repo root>/.nark/impact-report.md`. Ensure `.nark/impact-report.md` is in `.gitignore` (add if not present).

**Content:**

```markdown
# Nark Impact Report
**Repository:** <githubFullName or local path>
**Branch:** <branch> @ <commitSha>
**Date:** <today>

---

## Summary

| Metric | Value |
|--------|-------|
| Violations fixed | <TP_COUNT> |
| Errors fixed | <errors_fixed> |
| Warnings fixed | <warnings_fixed> |
| Risk score | <RISK_SCORE> — <risk level label> |

---

## What Could Have Gone Wrong

Each fix below prevented a potential production incident. Without these fixes, unhandled error paths would have caused crashes, data loss, or degraded user experience.

| # | Package | Function | File | Severity | What Could Have Gone Wrong |
|---|---------|----------|------|----------|---------------------------|
| 1 | <pkg> | <func>() | <file> | ERROR | <scenario> |
| 2 | <pkg> | <func>() | <file> | WARNING | <scenario> |
| ... | | | | | |

---

## Risk Breakdown by Package

| Package | Errors Fixed | Warnings Fixed | Package Risk Score |
|---------|-------------|---------------|-------------------|
| <pkg> | <N> | <M> | <N*3 + M*1> |
| ... | | | |
| **Total** | **<E>** | **<W>** | **<RISK_SCORE>** |

---

*Generated by nark-fix · Scan <BASELINE_SCAN_ID> → <FINAL_SCAN_ID>*
```

### Step 5.6 — Submit fix session to dashboard (unless --skip-upload or --dry-run)

After the triage report is written, capture the final commit SHA and submit an aggregate fix session record so the History tab shows a Fix Run badge on the after-scan row.

```bash
COMMIT_AFTER=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
```

Call `submit_fix_session` MCP tool:
```bash
curl -s -X POST $BASE_URL/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"jsonrpc\":\"2.0\",
    \"method\":\"tools/call\",
    \"params\":{
      \"name\":\"submit_fix_session\",
      \"arguments\":{
        \"repositoryId\":\"$REPO_ID\",
        \"scanBeforeId\":\"$BASELINE_SCAN_ID\",
        \"scanAfterId\":\"$FINAL_SCAN_ID\",
        \"violationsFixed\":$TP_COUNT,
        \"fpCount\":$FP_COUNT,
        \"borderlineCount\":$BORDERLINE_COUNT,
        \"packagesFixed\":$PACKAGES_FIXED,
        \"scannerIssues\":$SCANNER_ISSUES,
        \"branch\":\"$CURRENT_BRANCH\",
        \"commitBefore\":\"$COMMIT_BEFORE\",
        \"commitAfter\":\"$COMMIT_AFTER\"
      }
    },
    \"id\":1
  }"
```

On success, print:
```
Fix session submitted → dashboard History tab will show Fix Run badge on scan $FINAL_SCAN_ID
```

On failure (tool unavailable or error), log warning and continue — this is non-blocking.

**Note on $PACKAGES_FIXED format:** Must be a JSON array string, e.g.:
```
'[{"packageName":"axios","fixed":3,"fp":1},{"packageName":"prisma","fixed":2,"fp":0}]'
```
Build this from the per-package tallies accumulated in Step 2.5.2.5. If no packages were processed (all FPs), use `'[]'`.

**Note on $SCANNER_ISSUES format:** Must be a JSON array string of plain strings, e.g.:
```
'["verify-cli does not recognise Promise.allSettled() as error boundary","ai corpus api-error-rate-limit fires even with catch block"]'
```
Build from the deduplicated `$SCANNER_ISSUES` list. If empty, use `'[]'`.

---

## MCP Tool Reference

The following MCP tools are called on the `nark` server throughout this skill. All require the server to be registered and reachable. If unreachable, fallback to `--skip-upload` behavior (fixes still happen, no feedback is pushed).

All MCP tools are called via `POST $BASE_URL/api/mcp` with JSON-RPC body:
```
{"jsonrpc":"2.0","method":"tools/call","params":{"name":"<tool>","arguments":{...}},"id":1}
```

| Tool | When called | Key arguments |
|------|-------------|---------------|
| `trigger_scan` | Step 1.1 — trigger cloud scan | `repositoryId`, `branch` |
| `cancel_scan` | Step 1.1 — cancel a hung QUEUED/RUNNING scan before re-triggering | `scanId` |
| `poll_scan_status` | Step 1.1 — wait for scan to finish | `scanId` — polls internally, returns when done |
| `list_packages_with_errors` | Step 1.2 — fetch packages after scan | `repositoryId` |
| `list_errors_for_package` | Step 1.2 — per package | `repositoryId`, `packageName` |
| `get_resolution_history` | Phase 2.1.5 — per package | `packageName` |
| `batch_review_violations` | Phase 2.5.4 — after triage | `violationIds[]`, `action` (TRUE_POSITIVE/FALSE_POSITIVE/FLAG), optional `subViolationIndex` (0-based, for per-postcondition labels) |
| `add_violation_note` | Phase 2.5.4 — per FP | `violationId`, `note` (scanner shortcoming description) |
| `resolve_violation` | Step 4.4 — per fixed violation (fallback when no queueId) | `violationId`, `resolutionDetail` |
| `queue_fix` | Phase 2.5.4 Pass A — after TP batch_review | `violationId`, `note` — returns `queueId` |
| `mark_fixed` | Step 4.4 — per fixed violation when queueId available | `queueId`, `resolution`, `resolveViolation` (bool) — closes queue item; if resolveViolation: true also writes RESOLVE review |
| `get_fix_queue` | (future / agent use) — retrieve pending queue items | none required — returns pending queue items for the repo |
| `submit_fix_session` | Step 5.6 — aggregate session record | `repositoryId`, `scanBeforeId`, `scanAfterId`, `violationsFixed`, `fpCount`, `borderlineCount`, `packagesFixed`, `scannerIssues`, `branch`, `commitBefore`, `commitAfter` |

---

## What NOT to Do

- Never fix violations one-at-a-time before seeing all violations
- Never present a fix plan without triaging for false positives first (Phase 2.5)
- Never skip the triage feedback push — this data improves the corpus
- Never use `catch (e: any)` — use `unknown` + type narrowing
- Never silently swallow errors — always log or rethrow
- Never create a shared utility if only one package benefits
- Never skip the approval gate (Phase 3) unless `--auto` is passed — that flag explicitly enables hands-free operation
- Never commit with `git add -A` — stage specific files only
- Never amend commits
- Never force-push

---

## Edge Cases

**MCP server offline:** Log warning, set `SKIP_FEEDBACK=true`, continue with local fixes. All fix logic still executes.

**All violations are in test files:** Note this in the plan, then use AskUserQuestion(
  header: "Test File Violations",
  question: "All violations are in test files. Fix those too?",
  options: [
    { label: "Yes — fix test files too", description: "Include test file violations in the fix plan" },
    { label: "No — skip test files", description: "Leave test file violations unfixed" }
  ],
  multiSelect: false
)

**Violations in generated files:** Skip them. Note: "Skipped <N> violations in generated files."

**Pre-existing error handler that's close but not compliant:** Improve it rather than replacing it.

**Large file with many violations:** Fix all violations in that file in one edit pass.

**Violation ID not found in map:** Skip dashboard feedback for that violation, log it, continue.

---

## Reference

- **MCP setup:** `claude mcp add --transport http nark <url> --header 'Authorization: Bearer <key>' --scope user`
- **MCP tools list:** Call `mcp__nark__ping` to verify connectivity

---

## Reusable Iteration Prompt

```
I want to use /nark-fix to fix Nark profile violations in this repo, one package at a time.

Please:
1. Run /nark-fix to get a baseline scan (or /nark-fix --package <name> to target one package)
2. Pick the package with the most violations (or I'll specify with --package <name>)
3. For each violation, triage before fixing:
   - Read the flagged line ±15 lines of context
   - Assess: TRUE POSITIVE, LIKELY FALSE POSITIVE, or BORDERLINE
   - Note the rationale and any scanner shortcoming for FPs
4. Push TP/FP/FLAG feedback to dashboard before writing any code
5. Propose fixes only for true positives and borderline cases
6. Apply fixes, commit, rescan — push resolve feedback after each batch commit
7. After each package is done, show:
   - Violations fixed (with severity before/after)
   - False positives identified (and scanner issue noted)
   - Warnings surfaced that need extra infrastructure (backlog candidates)
   - Suggested next package to tackle
8. At the end, show the scanner/corpus improvement report

Packages to work through (in priority order):
- <package> (<N> violations — <context>)
- ...
```
