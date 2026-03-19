# Cross-Agent Support for Actionista

**Date:** 2026-03-19
**Status:** Approved
**Author:** Clay Loveless + Claude

## Summary

Make actionista work across all major AI coding agents: Claude Code, Cursor, Codex, OpenCode, Gemini CLI, and Zed. The plugin's core assets (skills, MCP server, index data) are already agent-agnostic. This design adds per-agent discovery configs and a CLI wrapper so every agent gets maximum capability for its platform.

## Architecture: Three Layers

Every agent gets the best tier it supports:

1. **Skills layer** (all agents) — `skills/github-actions/SKILL.md`, patterns, security docs, `index.json`. Universal markdown + JSON. Every agent can read these.
2. **MCP layer** (Claude, Cursor, Zed, OpenCode) — `scripts/actionista-mcp` provides `lookup_action`, `list_actions`, `check_workflow` as native tools via JSON-RPC stdio.
3. **CLI layer** (Codex, Gemini, anything with Bash) — `scripts/actionista` wraps the MCP handler functions as shell subcommands: `actionista lookup <action>`, `actionista check <file>`, `actionista list [category]`.

## Agent Capability Matrix

| Agent | Skills | MCP | CLI | Config Files |
|-------|--------|-----|-----|-------------|
| Claude Code | yes | yes | — | `.claude-plugin/plugin.json`, `.mcp.json` |
| Cursor | yes | yes | — | `.cursor-plugin/plugin.json`, `.cursor/mcp.json` |
| Zed | via AGENTS.md | yes | fallback | `docs/install/zed.md`, `AGENTS.md` |
| OpenCode | yes | manual MCP | fallback | `.opencode/plugins/actionista.js`, `package.json` |
| Codex | yes | — | yes | `.codex/INSTALL.md`, `AGENTS.md` |
| Gemini CLI | via GEMINI.md | — | yes | `gemini-extension.json`, `GEMINI.md` |

## Version Source of Truth

A single `VERSION` file at repo root (e.g., `0.3.0`) is the authoritative version. The release workflow reads from it and writes to all manifests. This prevents the version-drift problem seen in other multi-manifest plugins.

## Prerequisites

Before building the cross-agent configs, two refactors are needed:

1. **Extract shared handler library** — Move `handle_lookup_action`, `handle_list_actions`, `handle_check_workflow` and helpers from `scripts/actionista-mcp` into `scripts/actionista-lib`. Both the MCP server and CLI wrapper source this file. The MCP server keeps only the JSON-RPC dispatch loop.
2. **Reconcile versions** — All manifests must agree on the current version before adding new ones.

## File Layout

```
actionista/
├── VERSION                             # NEW — single source of truth for version
├── .claude-plugin/plugin.json          # existing, bump version
├── .cursor-plugin/plugin.json          # NEW
├── .cursor/mcp.json                    # NEW — Cursor MCP config (separate from .mcp.json)
├── .codex/INSTALL.md                   # NEW
├── .opencode/
│   ├── INSTALL.md                      # NEW
│   └── plugins/actionista.js           # NEW
├── .mcp.json                           # existing — Claude Code plugin MCP config
├── gemini-extension.json               # NEW
├── GEMINI.md                           # NEW — context file with @ refs
├── AGENTS.md                           # NEW — Codex/Zed agent context
├── package.json                        # NEW — OpenCode npm plugin version
├── scripts/
│   ├── actionista-lib                  # NEW — shared handler functions
│   ├── actionista-mcp                  # REFACTOR — JSON-RPC dispatch, sources lib
│   └── actionista                      # NEW — CLI wrapper, sources lib
├── agents/workflow-analyzer.md         # existing
├── skills/github-actions/              # existing — shared across all agents
├── docs/
│   └── install/                        # NEW — per-agent install guides
│       ├── claude-code.md
│       ├── cursor.md
│       ├── codex.md
│       ├── opencode.md
│       ├── gemini.md
│       └── zed.md
├── tests/                              # NEW
│   ├── run-all.sh
│   ├── structural/
│   │   ├── test-config-files.sh
│   │   ├── test-versions-sync.sh
│   │   └── test-scripts-syntax.sh
│   ├── mcp/
│   │   ├── test-initialize.sh
│   │   ├── test-lookup.sh
│   │   ├── test-check-workflow.sh
│   │   └── test-list.sh
│   └── cli/
│       ├── test-lookup.sh
│       ├── test-check.sh
│       └── test-list.sh
└── .github/workflows/
    └── release-on-update.yml           # MODIFY — bump all manifests from VERSION
```

## Per-Agent Config Details

### Claude Code (existing)

- `.claude-plugin/plugin.json` — plugin manifest with version
- `.mcp.json` — MCP server config using `${CLAUDE_PLUGIN_ROOT}`
- Skills auto-discovered from `skills/` directory
- Agent `workflow-analyzer` auto-discovered from `agents/`

### Cursor

`.cursor-plugin/plugin.json`:
```json
{
  "name": "actionista",
  "displayName": "Actionista",
  "version": "0.3.0",
  "description": "GitHub Actions assistant — version tracking, SHA pinning, workflow review",
  "author": { "name": "Clay Loveless" },
  "homepage": "https://github.com/claylo/actionista",
  "repository": "https://github.com/claylo/actionista",
  "license": "MIT",
  "skills": "./skills/",
  "agents": "./agents/"
}
```

Cursor uses its own MCP config at `.cursor/mcp.json` (NOT the root `.mcp.json` which is Claude Code-specific and uses `${CLAUDE_PLUGIN_ROOT}`). The `.cursor/mcp.json` uses a relative path:
```json
{
  "mcpServers": {
    "actionista": {
      "command": "bash",
      "args": ["./scripts/actionista-mcp"]
    }
  }
}
```

### Codex

`.codex/INSTALL.md` — instructions to:
1. Clone repo to `~/.codex/actionista`
2. Symlink: `ln -s ~/.codex/actionista/skills ~/.agents/skills/actionista`
3. Codex discovers skills via native skill discovery

`AGENTS.md` at repo root provides context for Codex and Zed (both read this file as an agent context convention):
```markdown
# Actionista — GitHub Actions Assistant

You have access to `scripts/actionista` for GitHub Actions version lookups:
- `scripts/actionista lookup <action>` — version, SHA, pinned reference
- `scripts/actionista check <file>` — find outdated actions in a workflow
- `scripts/actionista list [category]` — browse tracked actions

Read `skills/github-actions/SKILL.md` for patterns and best practices.
```

### OpenCode

`.opencode/plugins/actionista.js` — follows the superpowers plugin pattern:
- `config` hook: injects `skills/` path into OpenCode's skill discovery
- `experimental.chat.system.transform`: injects context about the CLI wrapper

Note: OpenCode's plugin config hook can modify `config.skills.paths` but MCP server registration via the plugin is unverified. MCP may require manual user config in `opencode.json`. The install docs will cover both paths.

Install via `opencode.json`:
```json
{ "plugin": ["actionista@git+https://github.com/claylo/actionista.git"] }
```

`package.json` at root:
```json
{
  "name": "actionista",
  "version": "0.3.0",
  "type": "module",
  "main": ".opencode/plugins/actionista.js"
}
```

### Gemini CLI

`gemini-extension.json`:
```json
{
  "name": "actionista",
  "description": "GitHub Actions assistant — version tracking, SHA pinning, workflow review",
  "version": "0.3.0",
  "contextFileName": "GEMINI.md"
}
```

`GEMINI.md` uses `@` file references to load skill content and documents the CLI wrapper.

### Zed

No plugin packaging required. Documented in `docs/install/zed.md`:
1. Add MCP server to `~/.config/zed/settings.json` under `context_servers`
2. `AGENTS.md` provides agent context (Zed reads this)

Future: proper Zed extension via their Rust/WASM extension system.

## CLI Wrapper: `scripts/actionista`

Subcommands:
- `actionista lookup <action>` — returns JSON with version, SHA, pinned reference
- `actionista check <file>` — reports outdated actions with upgrade info
- `actionista list [category]` — lists tracked actions, optionally filtered

Output is JSON by default (for agent consumption). When stdout is a TTY, output is human-readable.

Implementation: sources `scripts/actionista-lib` (the shared handler library extracted from the MCP server). Approximately 30-50 lines of bash dispatching subcommands to `handle_lookup_action`, `handle_list_actions`, `handle_check_workflow`.

## Release Pipeline

`release-on-update.yml` modification — on index update merge to main:

1. Read current version from `VERSION` file (single source of truth)
2. Compute next patch version
3. Write new version to `VERSION`
4. Bump version in ALL manifests (read from `VERSION`, write to each):
   - `.claude-plugin/plugin.json`
   - `.cursor-plugin/plugin.json`
   - `gemini-extension.json`
   - `package.json`
   - `scripts/actionista-mcp` (serverInfo version string)
5. Commit version bump (no skip flag — this commit IS the release signal for agents tracking via git)
6. Tag and create GitHub release
7. Dispatch to `claylo-marketplace` (Claude Code)

No re-trigger risk: `release-on-update.yml` only fires on `skills/github-actions/index.json` path changes. The version bump commit touches manifests, not the index.

Cursor and OpenCode pull directly from the git repo, so the tag is sufficient. No additional marketplace notifications needed for now.

## Test Strategy

Three tiers, modeled on superpowers' approach:

### Tier 1: Structural (fast, no deps)

- `test-config-files.sh` — all manifest files exist and are valid JSON
- `test-versions-sync.sh` — all manifests report the same version
- `test-scripts-syntax.sh` — `bash -n` on all scripts, `node --check` on OpenCode plugin

### Tier 2: MCP + CLI (fast, no API calls)

- MCP tests: send JSON-RPC messages to `actionista-mcp` via stdin, verify response structure
- CLI tests: run `actionista lookup/check/list`, verify output format and content
- Both use the local `index.json` — no network calls

### Tier 3: Skill Triggering (expensive, opt-in)

- Future: `claude -p` with naive prompts to verify the github-actions skill triggers
- Requires Claude Code CLI and costs tokens
- Run manually or in CI with budget controls

## Version

This work ships as actionista v0.3.0.
