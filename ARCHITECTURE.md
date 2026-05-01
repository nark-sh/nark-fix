# Architecture Decisions

This file is for humans. It is not referenced by SKILL.md and is not loaded into context.

---

## Why SKILL.md is a single file (2026-05-01)

**Decision:** Keep the entire skill specification in one monolithic SKILL.md rather than splitting into sub-files per phase.

**Size at time of decision:** 1,456 lines / 62KB / ~15-18K tokens.

**Reasons to keep it monolithic:**

1. Claude Code's skill loader reads exactly one file (SKILL.md). There's no `#include` mechanism. Splitting would require explicit `Read` tool calls at runtime to load sub-files, adding latency and risking the agent skipping reads.

2. The skill is a sequential workflow (Phase 0 through 5) where phases are interdependent. Phase 4 references state structures defined in Phase 0. Phase 5 references counters accumulated across Phases 2.5 and 4. Splitting by phase would force cross-references that are harder to follow than a linear document.

3. The SKILL.md is a fixed context cost. The real context pressure comes from the fix loop itself — reading source files, accumulating violation data, running scans. The auto-compact mechanism (Step 4.2d) handles that growing cost. The skill spec doesn't grow.

4. 62KB / ~15-18K tokens is within workable bounds for Opus, which has a 200K context window.

**If splitting becomes necessary in the future:**

The highest-value sections to extract (saves ~135 lines / ~3-4K tokens):

- MCP Tool Reference table (~40 lines) — only needed when debugging MCP connectivity
- `.narkrc.json` format and suppression docs (~50 lines) — only needed during FP suppression in Phase 2.5
- Reusable Iteration Prompt (~25 lines) — user-facing prompt template, not used by the agent
- Edge Cases (~20 lines) — rarely triggered paths

Each extracted file would need a one-line instruction in SKILL.md like: "Before starting Phase 2.5 suppression, read `docs/narkrc-format.md`."

**Revisit if:** SKILL.md exceeds ~2,000 lines or ~80KB, or if agents consistently run out of context before completing Phase 4 fix loops.
