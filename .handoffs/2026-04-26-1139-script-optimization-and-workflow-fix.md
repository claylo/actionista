# Handoff: Script Optimization and Workflow Fix

**Date:** 2026-04-26
**Branch:** `feat/tracked-actions-expansion`
**State:** Yellow

> Yellow: all changes are local, uncommitted. `commit.txt` is ready for `gtxt`. Supersedes the 0317 handoff from earlier today — same branch, additional work.

## Where things stand

Four files modified on `feat/tracked-actions-expansion`, none committed. `tracked-actions.yaml` is at 264 actions (covered in prior handoff). This session added: `release-on-update.yml` now bumps version across all three manifests, `update-index` skips migration recomputation for unchanged actions, and `discover-actions` uses curl globbing instead of config files. Full fresh run of `discover-actions` completes in ~19 seconds with no rate limiting.

## Decisions made

- **Version bump in all manifests.** `release-on-update.yml` now updates `plugin.json`, `tile.json`, and `skills/actionista/SKILL.md` (frontmatter `metadata.version`) in one step. Previously only `plugin.json` was bumped, leaving the other two stale.
- **Skip migration recomputation for unchanged actions.** `update-index` now compares `latestFull` (exact version tag) from the current GraphQL fetch against the existing `actions-index.json`. If identical, existing migration data is carried forward without a second-pass fetch. On a typical daily run (~2-5 version bumps out of 264), this eliminates ~8 GraphQL batches.
- **Curl globbing over config files.** `discover-actions` rewritten to use curl's native `[1-N]` range globbing for listing pages and `{a,b,c}` brace-list globbing for detail pages, with `#1` output variables for per-request filenames. Replaced the `-K` config file approach entirely. `--parallel-max` reduced from 50 to 20 to avoid GitHub 429s.

## What's next

1. **Commit and push.** `commit.txt` covers all four modified files. Run `gtxt && git pm`.
2. **Verify `release-on-update.yml` sed pattern.** The SKILL.md version bump uses `sed -i "s/^  version: .*/  version: ${NEXT}/"` — this matches the current two-space indent under `metadata:`. If SKILL.md frontmatter structure changes, the sed will silently no-op. Consider adding a post-bump verification step.
3. **Run `update-index` post-merge.** First run against 264 actions will be the baseline. Expect ~14 GraphQL batches in pass 1 (version fetch) and near-zero in pass 2 (migrations) since all actions will be new to the index.
4. **Cross-agent support** remains unimplemented — spec at `record/specs/2026-03-19-cross-agent-support-design.md`.

## Landmines

- **First `update-index` run after merge will recompute ALL migrations.** The optimization compares against the existing `actions-index.json`, which currently tracks ~121 actions. The ~143 newly-added actions have no existing `latestFull` to compare against, so they'll all queue for migration diffs. Subsequent runs will benefit from the cache.
- **SKILL.md sed relies on exact indentation.** The pattern `^  version: .*` (two leading spaces) matches line 13 of `SKILL.md` under the `metadata:` key. If the frontmatter is reformatted (e.g., by a YAML tool that changes indentation), the sed silently fails and the commit proceeds with a stale SKILL.md version.
- **`discover-actions` brace-list has an implicit ARG_MAX ceiling.** The 600-slug brace list is ~12KB, well under macOS's 1MB `ARG_MAX`. At ~3,000 slugs it would hit the limit. Not a near-term concern.
- **The 0317 handoff from earlier today is now stale.** It describes the same branch but before the workflow, update-index, and discover-actions changes. This handoff supersedes it.
