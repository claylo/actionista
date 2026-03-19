# Handoff: SHA Pinning, Workflow Fixes, and Cross-Agent Design

**Date:** 2026-03-19
**Branch:** main
**State:** Yellow

> Yellow: all scripts work locally, index has full SHA data for 118 actions, but nothing is committed yet. The cross-agent work is spec-only.

## Where things stand

Three things were done this session: (1) SHA commit pinning data added to the index and MCP server, (2) broken update-index workflow fixed and all workflows SHA-pinned, (3) a full design spec written for cross-agent support (Claude Code, Cursor, Codex, OpenCode, Gemini CLI, Zed). The first two are code-complete with 8 files changed. The third is a spec awaiting implementation.

## Prior handoff

`.handoffs/2026-03-19-1100-tracked-actions-curation.md` contains research for a tracked-actions curation pass: 9 actions to drop (archived), ~60 to add, new categories needed. That work is deferred but the research is captured.

## Decisions made

- **SHA data in index** — Every action entry now has a `sha` field (full 40-char commit SHA) and `lookup_action` returns a ready-to-paste `pinned` field (`action@sha # version`).
- **Update workflow simplified** — Removed Claude code action dependency. Commits directly to main. The prior approach got 20 permission denials per run and never created PRs (broken since March 6).
- **Batch size reduced to 20** — SHA fields made GraphQL queries too large for 50-repo batches (502 errors).
- **`best_release` function** — New tag selection logic that prefers specific versions (`v3.0.0`) over rolling major tags (`v3`). Fixed 14 actions.
- **Cross-agent architecture: three layers** — Skills (all agents), MCP (Claude/Cursor/Zed/OpenCode), CLI wrapper (Codex/Gemini/anything). Spec at `record/specs/2026-03-19-cross-agent-support-design.md`.
- **VERSION file as single source of truth** — Prevents manifest version drift across 5+ config files.
- **No `[skip release]`** — Version bump commits are the release signal for agents tracking via git. Path filters on `release-on-update.yml` prevent re-triggering.

## What's next

1. **Commit the current changes** — 8 files: workflows, scripts, index.json, tracked-actions.yaml. This is the SHA pinning + workflow fix work.
2. **Write implementation plan** — Invoke `writing-plans` skill against the cross-agent spec at `record/specs/2026-03-19-cross-agent-support-design.md`.
3. **Implement cross-agent support** — Key deliverables: extract `scripts/actionista-lib`, build `scripts/actionista` CLI, create per-agent configs, add `VERSION` file, update release pipeline, write tests.
4. **Tracked-actions curation** — Apply the research from the prior handoff: drop 9 archived, add ~60, run `update-index` to populate.
5. **Remove `[skip release]`** from `release-on-update.yml` during implementation.

## Landmines

- **Uncommitted changes include `index.json` (335 lines changed)** — The index now has SHA data for all 118 actions, plus version corrections from the `best_release` fix. Don't regenerate the index without the updated `scripts/update-index` or you'll lose the SHA fields.
- **MCP server version is 0.3.0 but `plugin.json` is still 0.2.0** — Version drift exists right now. The cross-agent spec introduces a `VERSION` file to fix this; reconcile during implementation.
- **GraphQL batch failures are transient** — Running `update-index` 2-3 times fills gaps from failed batches. The batch size (20) is a pragmatic workaround for GitHub's query complexity limits.
- **`actionista-mcp` calls `main` unconditionally** — The CLI wrapper can't source this file as-is. Must extract handlers into `scripts/actionista-lib` first (noted in spec as a prerequisite).
- **Cursor uses `.cursor/mcp.json`, NOT the root `.mcp.json`** — The `.mcp.json` with `${CLAUDE_PLUGIN_ROOT}` is Claude Code-specific. Caught in spec review.
