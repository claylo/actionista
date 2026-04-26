# Handoff: Index Update Crash Fix

**Date:** 2026-04-26
**Branch:** `main`
**State:** Yellow

> Yellow: index restored to 121 actions, script fixed and pushed, but the index hasn't been rebuilt to 264 yet. `release-on-update` and `notify-marketplace` workflow chain is fixed but untested end-to-end.

## Where things stand

The 264-action expansion from the prior session (PR #8) caused `update-index` to crash on its first daily cron run. The script accumulated all 264 actions' JSON in a shell variable, then passed it to `jq` via `--argjson` — exceeding Linux's `ARG_MAX` (~2MB). The shell redirect truncated `actions-index.json` to zero bytes before `jq` even attempted to exec, and the workflow committed the empty file. Three commits on `main` fix this: stdin piping for the final `jq` call, atomic write via temp file + `mv`, safety guards (zero-action and >10% loss), and `CROSS_REPO_PAT` for release creation so `notify-marketplace` actually triggers.

## Decisions made

- **Pipe `$all_entries` via stdin instead of `--argjson`.** The `--argjson` flag passes data as a command-line argument, subject to `ARG_MAX`. Stdin has no such limit. All other `$all_entries` uses already piped via `echo | jq`.
- **Atomic write pattern.** Write to `${INDEX_PATH}.tmp`, then `mv` into place. Prevents the redirect-truncation footgun if `jq` (or anything else) fails after the shell opens the output file.
- **Safety guards before write.** Refuse to write if action count is 0 or drops >10% vs existing index. Belt-and-suspenders — the `ARG_MAX` fix is the real cause, but these catch future failure modes.
- **`CROSS_REPO_PAT` for release creation.** `GITHUB_TOKEN` events don't trigger other workflows. `release-on-update` was creating releases that `notify-marketplace` (on `release: [published]`) never saw.

## What's next

1. **Run `update-index` locally or via `workflow_dispatch`.** The index is at 121 actions; `tracked-actions.yaml` has 264. Next successful run will add ~143 actions and recompute all their migrations.
2. **Verify `notify-marketplace` fires.** After the next index update triggers a release, confirm the marketplace dispatch actually runs. Check that `CROSS_REPO_PAT` has `Contents:write` on the actionista repo (not just claylo-marketplace).
3. **Cross-agent support** remains unimplemented — spec at `record/specs/2026-03-19-cross-agent-support-design.md`.

## Landmines

- **`CROSS_REPO_PAT` scope.** If it's a fine-grained PAT scoped only to `claylo-marketplace`, the `gh release create` in `release-on-update.yml` will fail with a 403. It needs `Contents:write` on `claylo/actionista` too.
- **First full run will be slow.** 264 actions = 14 batches in pass 1, then ~88 migration diffs in pass 2 (5 batches). All migrations are recomputed since 143 actions are new to the index. Subsequent runs will carry forward unchanged migrations.
- **Prior handoffs are stale.** The 1139 and 0317 handoffs from earlier today describe `feat/tracked-actions-expansion` before it merged and before the crash. This handoff is the current state of `main`.
