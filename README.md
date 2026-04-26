# Actionista

A Claude Code plugin for GitHub Actions — creates, reviews, and optimizes workflows with current action versions, SHA pinning, and best practices.

## Installation

### Via marketplace (recommended)

```bash
/plugin marketplace add claylo/claylo-marketplace
/plugin install actionista@claylo-marketplace
```

### Standalone skill

The `skills/actionista/` directory is a self-contained [Agent Skill](https://agentskills.io). Copy it into `~/.claude/skills/` or `.claude/skills/` to use without the plugin wrapper.

You may also install `actionista` using [npx skills](https://github.com/vercel-labs/skills).

```bash
npx skills add claylo/actionista
```

## What it does

- Tracks 120+ GitHub Actions with latest versions, SHAs, and migration diffs (updated daily)
- Creates workflow YAML with correct permissions, concurrency, caching, and matrix builds
- Reviews existing workflows for outdated actions, security issues, and performance problems
- Detects local tooling (`actionlint`, `act`) and suggests installation if missing
- Includes a workflow-analyzer subagent for proactive review after workflow edits

## How it works

The skill activates automatically when you work with GitHub Actions — workflow files, CI/CD questions, or any action by name. No commands to remember.

```
"Set up a CI workflow for this Node.js project"
"What's the latest version of actions/checkout?"
"Review my workflows for security issues"
"Add caching to this build"
```

## Actions index

The plugin maintains `actions-index.json` with version data for 121 actions across 19 categories:

| Category | Examples |
|----------|----------|
| Core | checkout, cache, upload-artifact, github-script |
| Languages | setup-node, setup-python, setup-go, setup-java, rust-toolchain |
| Cloud | aws-actions/\*, azure/\*, google-github-actions/\* |
| Docker | build-push-action, login-action, metadata-action |
| Security | codeql-action, trivy-action, trufflehog |
| Release | release-please-action, goreleaser-action, action-gh-release |

Each entry includes the latest version, full version tag, commit SHA for pinning, input parameters, deprecated versions, and migration data for major version bumps.

The index updates daily via GitHub Actions. Run it manually:

```bash
skills/actionista/scripts/update-index
```

Requires `gh` (authenticated), `yq`, and `jq`.

## Skill contents

All reference material lives in `skills/actionista/references/`:

| Topic | Files |
|-------|-------|
| Workflow syntax, expressions, triggers | `workflow-syntax.md`, `expressions.md`, `triggers.md` |
| Runners, permissions, troubleshooting | `runners.md`, `permissions.md`, `troubleshooting.md` |
| Patterns | `patterns-*.md` (matrix, caching, artifacts, reusable workflows, concurrency, environments, composite actions) |
| Security | `security-*.md` (secrets/OIDC, hardening, supply chain) |
| Schema | `github-action.schema.json` |

Working templates in `examples/`: Node.js CI, Rust CI, AWS deployment with OIDC, release-please.

## Contributing

Contributions welcome:

- Add actions to `skills/actionista/tracked-actions.yaml`
- Improve reference docs or add examples
- Report issues with version detection or migration data

## License

MIT
