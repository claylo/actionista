# Handoff: v1.0.0 Skill-Only Refactor

**Date:** 2026-04-25
**Branch:** main (merged from chore/update-workflows-and-actions)
**State:** Green

## Where things stand

actionista v1.0.0 is released. The MCP server, CLI, and slash command have been removed — the plugin is now a single skill (`skills/actionista/`) with a bundled workflow-analyzer agent. All 7 repo workflows are hardened with SHA pins, permissions, concurrency, and current action versions. The actions index tracks 121 actions and updates daily via cron.

## Decisions made

- **Dropped MCP server** — Claude can read `actions-index.json` directly via the skill; the JSON-RPC wrapper added overhead and failure modes for no benefit.
- **Skill directory is `skills/actionista/`** not `skills/github-actions/` — the description establishes the GitHub Actions context; the directory name matches the plugin name for `actionista:actionista` namespacing.
- **Agents and scripts live inside the skill directory** — makes `skills/actionista/` standalone-installable via `npx skills add claylo/actionista` without needing the plugin wrapper.
- **Flattened patterns/ and security/ into references/** — single directory with prefixed filenames (`patterns-caching.md`, `security-secrets.md`) reduces nesting without losing organization.
- **Renamed index.json to actions-index.json** — avoids ambiguity with npm/node conventions at repo root.
- **SKILL.md uses `!` backtick syntax** for dynamic context injection — checks for `actionlint` and `act` at skill load time, suggests `brew install` if missing.

## What's next

1. **Tracked actions curation** — prior handoff (`.handoffs/2026-03-19-1100-tracked-actions-curation.md`) identified 9 actions to drop and ~60 to add. Still pending.
2. **Cross-agent support** — spec approved at `record/specs/2026-03-19-cross-agent-support-design.md`, not yet implemented. Would let Cursor/Windsurf/etc. use the skill.
3. **Version drift** — `plugin.json` says 0.2.2 due to auto-bumping from index update workflow. Needs manual reconciliation to 1.0.0 before the next index update triggers a release.

## Landmines

- **Auto-release workflow bumps patch version on every index update commit to main.** The `release-on-update.yml` workflow watches `skills/actionista/actions-index.json` — any push to main touching that file triggers a version bump in `plugin.json` and a GitHub release. The version in `plugin.json` will drift from the 1.0.0 intent unless manually set before the next daily cron run (6 AM UTC).
- **tessl content score caps at 85%.** We ran 5 iterations trimming content per tessl suggestions — each round flagged a new section as "verbose." The conciseness criterion appears to have an asymptotic ceiling for skills with substantial inline examples. Don't chase it past 94/100 overall.
