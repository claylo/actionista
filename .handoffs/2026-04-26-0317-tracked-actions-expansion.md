# Handoff: Tracked Actions Expansion + Marketplace Survey Script

**Date:** 2026-04-26
**Branch:** `feat/tracked-actions-expansion`
**State:** Yellow

> Yellow: all changes are local, uncommitted. `commit.txt` is ready for `gtxt`. The `actions-index.json` will be stale until the next `update-index` run rebuilds it against the expanded tracked set.

## Where things stand

`tracked-actions.yaml` expanded from 121 entries (with 11 cross-category duplicates) to 264 unique actions across 20 categories. A new `scripts/discover-actions` script surveys the GitHub Marketplace for popular actions we're not tracking. The script runs in ~0.4s with cache and ~3s fresh (600 detail fetches via HTTP/2 multiplexed curl). Nothing is committed yet.

## Decisions made

- **Top-100 marketplace coverage is mandatory.** If an action is in the top 100 by marketplace popularity, we track it regardless of how niche it seems. Popularity is real-world signal.
- **Keep our category taxonomy, note GitHub's.** GitHub uses ~30 flat category slugs (`continuous-integration`, `deployment`, `utilities`, etc.). Our 20 categories are finer-grained and more useful for discovery. The `discover-actions` script maps between the two for reference.
- **Re-added two previously-dropped actions.** `8398a7/action-slack` and `marvinpinto/action-automatic-releases` were dropped in the prior handoff as archived, but both appear in the marketplace top 100. Added them back — the index will reflect their actual state.
- **New `build-tools` category.** Houses build systems (cmake, gradle, maven), compilation caching (ccache, sccache), and cross-compilation tools. These didn't fit `setup-languages` or `package-managers`.
- **`ai-assistants` expanded from 1 to 13.** AI code review is now a dominant marketplace category. Tracks Claude, CodeRabbit, Shippie, and others.

## What's next

1. **Commit and push.** `commit.txt` is ready. Run `gtxt && git pm` on `feat/tracked-actions-expansion`.
2. **Run `update-index` after merge.** The daily cron (6 AM UTC) will pick it up automatically, or trigger manually. First run will be slow — 264 actions in batches of 20 = 14 GraphQL batches.
3. **Backfill discover-actions cache.** Run `scripts/discover-actions --fresh --pages=30` after the rate limit fully clears. The current cache has ~100 stale rate-limit responses from earlier development iterations.
4. **Cross-agent support** is still unimplemented — spec at `record/specs/2026-03-19-cross-agent-support-design.md`.

## Landmines

- **`plugin.json` version drift.** The `release-on-update.yml` workflow bumps patch version on every index update commit to main. After merge, the first index update will auto-release from whatever version `plugin.json` currently holds. Verify it's set correctly before the 6 AM UTC cron fires.
- **Rate limiting on GitHub Marketplace API.** The marketplace endpoints (not the GraphQL API) rate-limit aggressively. The early serial-fetch and xargs attempts during this session triggered 429s that poisoned ~130 cache files. A `--fresh` run is needed to replace them. Space out retries if 429s recur.
- **`discover-actions` uses `grep -l '"payload"'` to filter valid cache files.** Any cached file that lacks the string `"payload"` (rate-limit HTML, empty responses, 429 JSON) is excluded from consolidation. If GitHub changes their response shape, this filter would silently drop valid data.
- **`externalUsesPathPrefix` is the golden field.** The marketplace detail API returns `.payload.action.slug` (marketplace URL slug, e.g., `github-pages-action`) which differs from the repo name (`actions-gh-pages`). The correct owner/repo comes from `.payload.action.externalUsesPathPrefix` with the trailing `@` stripped. The listing endpoint does NOT include owner/repo — only the detail page has it.
