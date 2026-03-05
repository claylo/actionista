# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2025-01-04

### Added

- Initial release of Actionista Claude Code plugin
- **Skill: github-actions**
  - Comprehensive GitHub Actions workflow knowledge
  - Pattern documentation (matrix, caching, artifacts, reusable workflows, concurrency, environments, composite actions)
  - Security guidance (secrets, hardening, supply chain)
  - Reference documentation (workflow syntax, expressions, triggers, runners, permissions, troubleshooting)
  - Example workflows (Node.js CI, Rust CI, AWS deployment, release-please)
  - Actions index with 100+ tracked actions and current versions
- **Agent: workflow-analyzer**
  - Proactive workflow file analysis
  - Outdated version detection
  - Security issue identification
  - Performance optimization suggestions
  - ASCII flowchart generation
- **Command: /review-workflow**
  - On-demand workflow analysis
  - Support for --all, --fix, --validate flags
- **GitHub Action: update-index.yml**
  - Daily automated index updates
  - PR-based review workflow
- **Script: update-index**
  - Fetches latest versions via GitHub GraphQL API (batched queries)
  - Requires gh, yq, jq

### Infrastructure

- Plugin manifest with full metadata
- MCP server configuration
- Comprehensive README with usage examples
- MIT License

## [Unreleased]

## [0.2.0] - 2026-03-05

### Added

- **MCP Server**: Bash-based stdio JSON-RPC server with tools:
  - `lookup_action` — look up version info for a specific action
  - `list_actions` — list tracked actions by category
  - `check_workflow` — report outdated action versions in workflow files

### Fixed

- Removed `aws-actions/amazon-eks-run-job` from tracked actions (repo does not exist)
- Removed `alexellis/arkade-get` from tracked actions (no release tags)
- Updated all `actions/checkout` references from v4 to v6 across skill documentation
- Fixed README to reference bash `scripts/update-index` instead of Node.js script

### Changed

- `.mcp.json` now uses built-in bash MCP server instead of external binary
- Bumped plugin version to 0.2.0
