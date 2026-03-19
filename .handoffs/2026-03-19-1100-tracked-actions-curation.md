# Tracked Actions Curation — Research Complete, Not Yet Applied

**Date:** 2026-03-19
**Status:** Research done, implementation deferred

## Drop List (9 actions — archived)
- `actions/create-release` — archived, use `softprops/action-gh-release`
- `actions/upload-release-asset` — archived, use `softprops/action-gh-release`
- `hashicorp/terraform-github-actions` — archived, use `hashicorp/setup-terraform`
- `google-github-actions/release-please-action` — archived, replace with `googleapis/release-please-action`
- `marvinpinto/action-automatic-releases` — archived
- `chartboost/ruff-action` — archived, replace with `astral-sh/ruff-action`
- `mattallty/jest-github-action` — 3+ years stale
- `8398a7/action-slack` — archived, use `slackapi/slack-github-action`
- `tibdex/github-app-token` — archived, replace with `actions/create-github-app-token`

## Ref Update
- `github/super-linter` → `super-linter/super-linter` (repo transferred, 10.3k stars)

## High-Priority Additions (~60 actions)
Top additions by stars:
- `appleboy/ssh-action` (6,011), `JamesIves/github-pages-deploy-action` (4,558)
- `googleapis/release-please-action` (2,330), `ncipollo/release-action` (1,652)
- `google-github-actions/run-gemini-cli` (1,868), `tauri-apps/tauri-action` (1,471)
- `step-security/harden-runner` (1,002), `actions/attest-build-provenance` (911)
- `oven-sh/setup-bun` (677), `denoland/setup-deno` (323)

## New Categories Needed
- Supply chain security (SLSA, SBOM, cosign, harden-runner, scorecard)
- Test reporting (dorny/test-reporter, EnricoMi/publish-unit-test-result-action)
- Benchmarking (benchmark-action/github-action-benchmark)
- AI assistants (anthropics/claude-code-action, google/run-gemini-cli)

## Selection Criteria (proposed, not yet documented)
1. Not archived, updated within 18 months
2. No star floor — track what's worth tracking
3. Official org actions auto-include (actions/, github/, docker/)
4. Fills a real category
5. Prune archived/stale regularly

## Full research in conversation context — rerun research agents if needed.
