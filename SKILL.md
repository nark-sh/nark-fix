# bc-fix Skill

**Trigger:** `/bc-fix`, "bc fix", "fix behavioral contract violations", "resolve bc violations", "fix bc errors"
**Version:** 2.0.0

Agentic loop that scans a repo for behavioral contract violations, triages false positives, pushes TP/FP/resolve feedback to the dashboard via MCP tools, applies DRY fixes package-by-package, and loops until the repo is clean. Each triage decision and fix is recorded in the dashboard for corpus/verify-cli improvement.

---

## Invocation

```
/bc-fix                    # Scan + analyze + fix all ERRORs
/bc-fix --warnings         # Also fix WARNINGs (default: ERRORs only)
/bc-fix --dry-run          # Scan + show plan, but make no changes
/bc-fix --package <name>   # Fix only violations for one package (e.g. axios)
/bc-fix --skip-upload      # Run locally only, no dashboard upload or MCP feedback
/bc-fix --local            # Point at http://localhost:3000 instead of production (for saas development)
/bc-fix --auto             # Fully autonomous: skip all approval gates and prompts, run every package to completion
/bc-fix --batch            # Scan once → fix all → rescan once. No intermediate scans. For automated use.
/bc-fix --batch --auto     # Fully autonomous batch mode (no approval gate + no intermediate scans)
/bc-fix --resume            # Explicitly resume from .bc-fix-state.json (same as auto-detection)
/bc-fix --fresh             # Ignore any existing state file and start over
```

---

## Phase 0 — Bootstrap

### Step 0.0 — Check for existing run state

Before anything else, check for `.bc-fix-state.json` in the current repo root:

```bash
if [ -f ".bc-fix-state.json" ]; then
  STATE=$(cat .bc-fix-state.json)
fi
```

If the file exists and `--fresh` flag was NOT passed:
- Parse `state.branch` from the JSON
- Get current branch: `git branch --show-current`
- If branches **do not match**: print "State file is from branch '<state.branch>', you are on '<current>'. Starting fresh." and continue with normal flow (the old file will be overwritten at Phase 1).
- If branches **match** and `state.phase == "fix_loop"`:
  - Print:
    ```
    Resuming previous bc-fix session.
      Baseline scan:       <state.baselineScanId>
      Packages complete:   <comma-separated list of packages where state.packages[name].status == "complete">
      Packages remaining:  <comma-separated list of packages where state.packages[name].status == "pending">
    ```
  - Load into working variables:
    - `$BASELINE_SCAN_ID` = `state.baselineScanId`
    - `$REPO_ID` = `state.repoId`
    - `$VIOLATION_ID_MAP` = `state.violationIdMap`
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

**Note on gitignore:** When first writing `.bc-fix-state.json` (Step 1.2.5 below), check whether it is already in the target repo's `.gitignore`. If not, append both `.bc-fix-state.json` and `.bc-fix-continuation.md` to `.gitignore`.

### Step 0.1 — Resolve API key

Check in order:
1. `$BC_API_KEY` environment variable
2. `cat ~/.claude.json 2>/dev/null` → parse `projects.<cwd>.mcpServers["behavioral-contracts"].headers.Authorization` → strip `"Bearer "` prefix
3. `cat ~/.claude.json 2>/dev/null` → parse top-level `mcpServers["behavioral-contracts"].headers.Authorization`
4. `cat ~/.claude/claude_desktop_config.json 2>/dev/null` → same path

If no key found and `--skip-upload` not set: tell the user "API key not found. Set BC_API_KEY or configure MCP with: claude mcp add --transport http behavioral-contracts <url> --header 'Authorization: Bearer <key>' --scope user". Stop.

Store as `$API_KEY`.

### Step 0.2 — Resolve base URL

Resolve in priority order:
1. `--local` flag passed at invocation → use `http://localhost:3000`
2. `.bc-scan` config file `baseUrl` field
3. `$BC_BASE_URL` environment variable
4. Default: `https://app.behavioral-contracts.com`

When `--local` is set, also resolve the API key from the local dev `.env` or `.env.local` file in the saas app if the standard key lookup (Step 0.1) fails — local dev keys may differ from production keys.

Store as `$BASE_URL`.

### Step 0.3 — Resolve repository ID

Same as bc-scan skill Step 3. Store as `$REPO_ID`.

### Step 0.3.5 — Auto-activate if repo is accessible but not activated (unless --skip-upload)

After resolving `$REPO_ID`:

If `$REPO_ID` is empty or not found (the repo hasn't been activated on the website yet), but the GitHub App is installed (bc-fix can determine this because the scan in Phase 1 will fail with a "not found" error), attempt auto-activation:

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
    -d "{\"githubFullName\":\"$GITHUB_FULL_NAME\",\"source\":\"bc-fix\"}")

  # Parse new repositoryId from response
  NEW_REPO_ID=$(echo "$ACTIVATE_RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('repository',{}).get('id',''))" 2>/dev/null || echo "")

  if [ -n "$NEW_REPO_ID" ]; then
    echo "Repository auto-activated on behavioral-contracts.com — ID: $NEW_REPO_ID"
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

The key design intent: bc-fix detects it's running on a repo that the GitHub App has access to (because the API key is valid) but hasn't been connected to the dashboard yet. It auto-connects it by calling the repositories POST endpoint with the `githubFullName` extracted from the git remote. The `source: "bc-fix"` field is passed so the backend can distinguish auto-activations from manual ones (optional logging, no behavior change needed on the API side).

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
$FP_NOTES              = []   # [{violationId, note}] — scanner shortcomings from FP triage
$RESOLVED_VIOLATIONS   = []   # [{violationId, resolutionDetail}] — fixed violations
$SCANNER_ISSUES        = []   # Deduplicated scanner/corpus improvement opportunities
$TP_COUNT              = 0    # Total true positives across all packages
$FP_COUNT              = 0    # Total false positives across all packages
$BORDERLINE_COUNT      = 0    # Total borderline cases across all packages
$PACKAGES_FIXED        = []   # [{packageName, fixed, fp}] — per-package TP/FP tally
```

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

**Note:** `list_packages_with_errors` and `list_errors_for_package` are MCP tools registered on the `behavioral-contracts` server. Call them via the MCP tool interface directly (not curl).

### Step 1.2.5 — Write initial state file

After building `$VIOLATION_ID_MAP`, write `.bc-fix-state.json` to the repo root:

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
  "violationIdMap": "<$VIOLATION_ID_MAP object>"
}
```

Also check `.gitignore` — if `.bc-fix-state.json` is not listed, append:
```
.bc-fix-state.json
.bc-fix-continuation.md
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

Update `.bc-fix-state.json`: set `phase` to `"fix_loop"` and add triage results:

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

## Phase 3 — Present Fix Plan (Approval Gate)

Before writing any code, present the full plan to the user. Format:

```
Behavioral Contracts — Fix Plan
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

**4.2a — Update state file:** Read `.bc-fix-state.json`, set `packages["<this package>"].status = "complete"` and `packages["<this package>"].commit = "<git rev-parse --short HEAD>"`, then overwrite the file.

**4.2b — Write continuation file:** Overwrite `.bc-fix-continuation.md` with:

```markdown
# bc-fix continuation

If this session ran out of context, simply re-run:

    /bc-fix --auto --local --warnings

bc-fix will auto-detect `.bc-fix-state.json` and resume from where it stopped.

---

If the state file was lost, use this manual prompt:

I was running bc-fix and ran out of context. Here's where I left off:

- Branch: <CURRENT_BRANCH>
- Baseline scan: <BASELINE_SCAN_ID>
- Completed packages: <package> (commit <sha>), ...
- Remaining packages: <package> (<N> violations), ...

Triage summary (do not re-triage — use these results):
<one line per package: "package: N TPs, M FPs, K borderline">

Please continue the fix loop for remaining packages, skipping completed ones.
```

Update this file after every batch commit so it always reflects current progress.

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

For each violation fixed in this batch, call `resolve_violation` MCP tool:
```
Arguments: {
  violationId: <dashboard violation ID from $VIOLATION_ID_MAP>,
  resolutionDetail: "<what was done: e.g. 'Wrapped axios.get call in try-catch with axios.isAxiosError type guard. Error is logged then rethrown to preserve caller handling.'>"
}
```

Call individually (not batch) to provide rich per-violation detail. If the dashboard violation ID is not in `$VIOLATION_ID_MAP`, skip and log.

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

  **4.2a — Update state file:** Read `.bc-fix-state.json`, set `packages["<this package>"].status = "complete"` and `packages["<this package>"].commit = "<git rev-parse --short HEAD>"`, then overwrite the file.

  **4.2b — Write continuation file:** Overwrite `.bc-fix-continuation.md` with:

  ```markdown
  # bc-fix continuation

  If this session ran out of context, simply re-run:

      /bc-fix --auto --local --warnings --batch

  bc-fix will auto-detect `.bc-fix-state.json` and resume from where it stopped.
  Note: batch mode is active — rescans are deferred to the final verify step.

  ---

  If the state file was lost, use this manual prompt:

  I was running bc-fix in batch mode and ran out of context. Here's where I left off:

  - Branch: <CURRENT_BRANCH>
  - Baseline scan: <BASELINE_SCAN_ID>
  - Completed packages: <package> (commit <sha>), ...
  - Remaining packages: <package> (<N> violations), ...

  Triage summary (do not re-triage — use these results):
  <one line per package: "package: N TPs, M FPs, K borderline">

  Please continue the fix loop for remaining packages, skipping completed ones.
  When all packages are fixed, run the final batch scan (Step 4.3).
  ```

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

For every violation that was fixed across ALL batches, call `resolve_violation` MCP tool.
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
Behavioral Contracts — Fix Complete
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

Write a `bc-triage-report.md` file to the repo root. This file is meant to be shared with other developers and the corpus team. It should be comprehensive and standalone.

**File path:** `<repo root>/bc-triage-report.md`

**Content structure:**

```markdown
# Behavioral Contracts Triage Report
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

*Generated by Behavioral Contracts bc-fix workflow · Scan <BASELINE_SCAN_ID>*
```

**Rules:**
- Include real before/after code examples, not placeholders — copy actual code from the fixed files
- For FP groups, always include "Root cause" explaining the scanner limitation
- For TP groups, include the hotspot file breakdown table
- If `--dry-run` was used and no fixes were applied, note that in the summary
- If `--skip-upload` was used, omit the "What Was Sent to the Dashboard" section

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

The following MCP tools are called on the `behavioral-contracts` server throughout this skill. All require the server to be registered and reachable. If unreachable, fallback to `--skip-upload` behavior (fixes still happen, no feedback is pushed).

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
| `resolve_violation` | Step 4.4 — per fixed violation | `violationId`, `resolutionDetail` |
| `submit_fix_session` | Step 5.6 — aggregate session record | `repositoryId`, `scanBeforeId`, `scanAfterId`, `violationsFixed`, `fpCount`, `borderlineCount`, `packagesFixed`, `scannerIssues`, `branch`, `commitBefore`, `commitAfter` |

---

## What NOT to Do

- Never fix violations one-at-a-time before seeing all violations
- Never present a fix plan without triaging for false positives first (Phase 2.5)
- Never skip the triage feedback push — this data improves the corpus
- Never use `catch (e: any)` — use `unknown` + type narrowing
- Never silently swallow errors — always log or rethrow
- Never create a shared utility if only one package benefits
- Never skip the approval gate (Phase 3)
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

- **bc-scan skill:** `~/.claude/skills/bc-scan/SKILL.md`
- **MCP setup:** `claude mcp add --transport http behavioral-contracts <url> --header 'Authorization: Bearer <key>' --scope user`
- **MCP tools list:** Call `mcp__behavioral-contracts__ping` to verify connectivity

---

## Reusable Iteration Prompt

```
I want to use /bc-fix to fix behavioral contract violations in this repo, one package at a time.

Please:
1. Run /bc-fix to get a baseline scan (or /bc-fix --package <name> to target one package)
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
