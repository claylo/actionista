# Cross-Agent Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make actionista work across Claude Code, Cursor, Codex, OpenCode, Gemini CLI, and Zed — giving every agent the best tier it supports (skills, MCP, CLI).

**Architecture:** Extract shared handler functions from the MCP server into a sourced library. Build a CLI wrapper that sources the same library. Create per-agent discovery configs and install docs. Unify version management via a single `VERSION` file.

**Tech Stack:** Bash (actionista-lib, actionista-mcp, actionista CLI, tests), JSON (manifests, MCP configs), JavaScript (OpenCode plugin), Markdown (AGENTS.md, GEMINI.md, install docs)

**Spec:** `record/specs/2026-03-19-cross-agent-support-design.md`

---

## File Structure

```
actionista/
├── VERSION                             # NEW — "0.3.0", single source of truth
├── .claude-plugin/plugin.json          # MODIFY — read version from VERSION
├── .cursor-plugin/plugin.json          # NEW — Cursor plugin manifest
├── .cursor/mcp.json                    # NEW — Cursor MCP config (relative path)
├── .codex/INSTALL.md                   # NEW — Codex install instructions
├── .opencode/
│   ├── INSTALL.md                      # NEW — OpenCode install instructions
│   └── plugins/actionista.js           # NEW — OpenCode plugin hook
├── .mcp.json                           # existing (Claude Code, unchanged)
├── gemini-extension.json               # NEW — Gemini CLI manifest
├── GEMINI.md                           # NEW — Gemini context with @ refs
├── AGENTS.md                           # NEW — Codex/Zed agent context
├── package.json                        # NEW — OpenCode npm version
├── scripts/
│   ├── actionista-lib                  # NEW — shared handler functions + helpers
│   ├── actionista-mcp                  # MODIFY — JSON-RPC only, sources lib
│   └── actionista                      # NEW — CLI wrapper, sources lib
├── tests/
│   ├── run-all.sh                      # NEW — test runner
│   ├── structural/
│   │   ├── test-config-files.sh        # NEW — manifests exist + valid JSON
│   │   ├── test-versions-sync.sh       # NEW — all versions match VERSION
│   │   └── test-scripts-syntax.sh      # NEW — bash -n on all scripts
│   ├── mcp/
│   │   ├── test-initialize.sh          # NEW — JSON-RPC initialize handshake
│   │   ├── test-lookup.sh              # NEW — lookup_action tool
│   │   ├── test-check-workflow.sh      # NEW — check_workflow tool
│   │   └── test-list.sh               # NEW — list_actions tool
│   └── cli/
│       ├── test-lookup.sh              # NEW — actionista lookup
│       ├── test-check.sh              # NEW — actionista check
│       └── test-list.sh               # NEW — actionista list
├── docs/install/                       # NEW — per-agent install guides
│   ├── claude-code.md
│   ├── cursor.md
│   ├── codex.md
│   ├── opencode.md
│   ├── gemini.md
│   └── zed.md
└── .github/workflows/
    └── release-on-update.yml           # MODIFY — bump all manifests from VERSION
```

---

## Task 1: VERSION File and Test Harness

**Files:**
- Create: `VERSION`
- Create: `tests/run-all.sh`
- Create: `tests/structural/test-versions-sync.sh`
- Create: `tests/structural/test-config-files.sh`
- Create: `tests/structural/test-scripts-syntax.sh`

### Rationale

The VERSION file is the foundation everything else builds on. Writing the structural tests first means every subsequent task gets validated automatically. The version sync test will fail initially (only VERSION exists, no other manifests yet) — that's expected and correct.

- [ ] **Step 1: Create VERSION file**

```bash
echo "0.3.0" > VERSION
```

- [ ] **Step 2: Create test runner**

Create `tests/run-all.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

passed=0
failed=0

run_test() {
    local test_file="$1"
    local name
    name=$(basename "$test_file" .sh)

    printf "  %-40s " "$name"
    local output
    if output=$(bash "$test_file" "$REPO_ROOT" 2>&1); then
        echo "PASS"
        passed=$((passed + 1))
    else
        echo "FAIL"
        echo "$output" | sed 's/^/    /'
        failed=$((failed + 1))
    fi
}

echo "=== Structural Tests ==="
for t in "$SCRIPT_DIR"/structural/test-*.sh; do
    [ -f "$t" ] && run_test "$t"
done

echo ""
echo "=== MCP Tests ==="
for t in "$SCRIPT_DIR"/mcp/test-*.sh; do
    [ -f "$t" ] && run_test "$t"
done

echo ""
echo "=== CLI Tests ==="
for t in "$SCRIPT_DIR"/cli/test-*.sh; do
    [ -f "$t" ] && run_test "$t"
done

echo ""
echo "=== Results: $passed passed, $failed failed ==="
[ "$failed" -eq 0 ]
```

- [ ] **Step 3: Create structural test — scripts syntax**

Create `tests/structural/test-scripts-syntax.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"
errors=0

for script in "$REPO_ROOT"/scripts/actionista*; do
    [ -f "$script" ] || continue
    if ! bash -n "$script" 2>/dev/null; then
        echo "Syntax error: $script"
        errors=$((errors + 1))
    fi
done

# Check OpenCode plugin if it exists
if [ -f "$REPO_ROOT/.opencode/plugins/actionista.js" ]; then
    if ! node --check "$REPO_ROOT/.opencode/plugins/actionista.js" 2>/dev/null; then
        echo "Syntax error: .opencode/plugins/actionista.js"
        errors=$((errors + 1))
    fi
fi

[ "$errors" -eq 0 ]
```

- [ ] **Step 4: Create structural test — config files**

Create `tests/structural/test-config-files.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"
errors=0

check_json() {
    local file="$1"
    if [ ! -f "$file" ]; then
        echo "Missing: $file"
        errors=$((errors + 1))
        return
    fi
    if ! jq empty "$file" 2>/dev/null; then
        echo "Invalid JSON: $file"
        errors=$((errors + 1))
    fi
}

# Required JSON manifests
check_json "$REPO_ROOT/.claude-plugin/plugin.json"
check_json "$REPO_ROOT/.cursor-plugin/plugin.json"
check_json "$REPO_ROOT/.mcp.json"
check_json "$REPO_ROOT/.cursor/mcp.json"
check_json "$REPO_ROOT/gemini-extension.json"
check_json "$REPO_ROOT/package.json"
check_json "$REPO_ROOT/skills/github-actions/index.json"

# Required non-JSON files
for f in VERSION AGENTS.md GEMINI.md; do
    if [ ! -f "$REPO_ROOT/$f" ]; then
        echo "Missing: $f"
        errors=$((errors + 1))
    fi
done

# Required scripts must be executable
for s in scripts/actionista-lib scripts/actionista-mcp scripts/actionista; do
    if [ ! -x "$REPO_ROOT/$s" ]; then
        echo "Not executable: $s"
        errors=$((errors + 1))
    fi
done

[ "$errors" -eq 0 ]
```

- [ ] **Step 5: Create structural test — version sync**

Create `tests/structural/test-versions-sync.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"
errors=0

if [ ! -f "$REPO_ROOT/VERSION" ]; then
    echo "Missing VERSION file"
    exit 1
fi

expected=$(cat "$REPO_ROOT/VERSION" | tr -d '[:space:]')

check_version() {
    local file="$1"
    local query="$2"
    local actual

    if [ ! -f "$file" ]; then
        echo "Missing: $file (expected version $expected)"
        errors=$((errors + 1))
        return
    fi

    actual=$(jq -r "$query" "$file" 2>/dev/null)
    if [ "$actual" != "$expected" ]; then
        echo "Version mismatch: $file has '$actual', expected '$expected'"
        errors=$((errors + 1))
    fi
}

# JSON manifests with .version field
check_version "$REPO_ROOT/.claude-plugin/plugin.json" '.version'
check_version "$REPO_ROOT/.cursor-plugin/plugin.json" '.version'
check_version "$REPO_ROOT/gemini-extension.json" '.version'
check_version "$REPO_ROOT/package.json" '.version'

# MCP server version (run initialize, extract from response)
mcp_version=$(echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
    | bash "$REPO_ROOT/scripts/actionista-mcp" \
    | jq -r '.result.serverInfo.version' 2>/dev/null)
if [ "$mcp_version" != "$expected" ]; then
    echo "Version mismatch: actionista-mcp reports '$mcp_version', expected '$expected'"
    errors=$((errors + 1))
fi

[ "$errors" -eq 0 ]
```

- [ ] **Step 6: Make test runner executable and run structural tests**

```bash
chmod +x tests/run-all.sh tests/structural/test-*.sh
bash tests/structural/test-scripts-syntax.sh .
```

Expected: PASS (existing scripts have valid syntax).

The config and version tests will fail — that's correct, the files they check don't exist yet.

- [ ] **Step 7: Commit**

```bash
git add VERSION tests/
git commit -m "feat: add VERSION file and structural test harness"
```

---

## Task 2: Extract actionista-lib

**Files:**
- Create: `scripts/actionista-lib`
- Create: `tests/mcp/test-initialize.sh`
- Create: `tests/mcp/test-lookup.sh`

### Rationale

This is the critical extraction. `actionista-lib` gets the shared functions (tool defs, handlers, helpers). The MCP server will source it in the next task. We write MCP tests now that test the *full MCP server* — they'll validate the refactor in Task 3.

- [ ] **Step 1: Write MCP test — initialize**

Create `tests/mcp/test-initialize.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"

response=$(echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
    | bash "$REPO_ROOT/scripts/actionista-mcp")

# Verify response structure
version=$(echo "$response" | jq -r '.result.serverInfo.version')
name=$(echo "$response" | jq -r '.result.serverInfo.name')
protocol=$(echo "$response" | jq -r '.result.protocolVersion')

[ "$name" = "actionista" ] || { echo "Bad server name: $name"; exit 1; }
[ -n "$version" ] || { echo "Missing version"; exit 1; }
[ "$protocol" = "2024-11-05" ] || { echo "Bad protocol: $protocol"; exit 1; }
```

- [ ] **Step 2: Write MCP test — lookup**

Create `tests/mcp/test-lookup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"

# Send initialize then lookup
response=$(printf '%s\n%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
    '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"lookup_action","arguments":{"action":"actions/checkout"}}}' \
    | bash "$REPO_ROOT/scripts/actionista-mcp" | tail -1)

# Verify lookup result
text=$(echo "$response" | jq -r '.result.content[0].text')
action=$(echo "$text" | jq -r '.action')
latest=$(echo "$text" | jq -r '.latest // empty')
sha=$(echo "$text" | jq -r '.sha // empty')
pinned=$(echo "$text" | jq -r '.pinned // empty')

[ "$action" = "actions/checkout" ] || { echo "Bad action: $action"; exit 1; }
[ -n "$latest" ] || { echo "Missing latest version"; exit 1; }
[ -n "$sha" ] || { echo "Missing SHA"; exit 1; }
[ -n "$pinned" ] || { echo "Missing pinned reference"; exit 1; }
```

- [ ] **Step 3: Run MCP tests against current (pre-refactor) server**

```bash
chmod +x tests/mcp/test-*.sh
bash tests/mcp/test-initialize.sh .
bash tests/mcp/test-lookup.sh .
```

Expected: PASS — these validate the current MCP server works before we refactor.

- [ ] **Step 4: Create actionista-lib**

Create `scripts/actionista-lib` — extract everything from `actionista-mcp` except `handle_request()` and `main()`:

```bash
#!/usr/bin/env bash
# actionista-lib — shared handler functions for actionista MCP server and CLI
#
# Source this file, don't execute it directly.
# Provides: tools_json, handle_lookup_action, handle_list_actions, handle_check_workflow
#
# Expects SCRIPT_DIR and INDEX_PATH to be set by the caller, OR sets defaults.

# Set defaults if caller hasn't set them
: "${SCRIPT_DIR:="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"}"
: "${INDEX_PATH:="$SCRIPT_DIR/../skills/github-actions/index.json"}"

# ============================================================================
# Tool definitions
# ============================================================================

tools_json() {
    cat <<'TOOLS'
[
  {
    "name": "lookup_action",
    "description": "Look up version info for a GitHub Action. Returns latest version, full version tag, commit SHA for pinning, description, category, deprecated versions, and input parameters. Includes a ready-to-paste 'pinned' reference.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "action": {
          "type": "string",
          "description": "Action name, e.g. 'actions/checkout' or 'docker/build-push-action'"
        }
      },
      "required": ["action"]
    }
  },
  {
    "name": "list_actions",
    "description": "List all tracked GitHub Actions in the index. Optionally filter by category.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "category": {
          "type": "string",
          "description": "Filter by category (e.g. 'core', 'docker', 'security', 'cloud-aws'). Omit to list all."
        }
      }
    }
  },
  {
    "name": "check_workflow",
    "description": "Check a GitHub Actions workflow file for outdated action versions. Compares uses: directives against the current index.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "file": {
          "type": "string",
          "description": "Path to the workflow YAML file to check"
        }
      },
      "required": ["file"]
    }
  }
]
TOOLS
}

# ============================================================================
# Tool handlers
# ============================================================================

handle_lookup_action() {
    local action="$1"

    if [[ ! -f "$INDEX_PATH" ]]; then
        echo '{"error": "Index file not found. Run scripts/update-index first."}'
        return
    fi

    local result
    result=$(jq --arg a "$action" '.actions[$a] // empty' "$INDEX_PATH")

    if [[ -z "$result" ]]; then
        # Try fuzzy match
        local matches
        matches=$(jq -r --arg a "$action" '.actions | keys[] | select(test($a; "i"))' "$INDEX_PATH" | head -5)
        if [[ -n "$matches" ]]; then
            jq -n --arg a "$action" --arg m "$matches" \
                '{error: ("Action not found: " + $a), suggestions: ($m | split("\n"))}'
        else
            jq -n --arg a "$action" '{error: ("Action not found: " + $a)}'
        fi
    else
        jq -n --arg a "$action" --argjson info "$result" \
            '{action: $a} + $info + (if $info.sha then {pinned: ($a + "@" + $info.sha + " # " + $info.latestFull)} else {} end)'
    fi
}

handle_list_actions() {
    local category="${1:-}"

    if [[ ! -f "$INDEX_PATH" ]]; then
        echo '{"error": "Index file not found. Run scripts/update-index first."}'
        return
    fi

    if [[ -n "$category" ]]; then
        jq --arg c "$category" '
            [.actions | to_entries[] | select(.value.category == $c) | {name: .key, latest: .value.latest, description: .value.description}] as $matched |
            if ($matched | length) == 0 then
                {error: ("No actions found in category: " + $c), available_categories: ([.actions[].category] | unique | sort)}
            else
                {category: $c, actions: $matched}
            end' "$INDEX_PATH"
    else
        jq '{
            total: (.actions | length),
            categories: [.actions | to_entries | group_by(.value.category)[] | {
                category: .[0].value.category,
                count: length,
                actions: [.[].key]
            }] | sort_by(.category)
        }' "$INDEX_PATH"
    fi
}

handle_check_workflow() {
    local file="$1"

    if [[ ! -f "$file" ]]; then
        jq -n --arg f "$file" '{error: ("File not found: " + $f)}'
        return
    fi

    if [[ ! -f "$INDEX_PATH" ]]; then
        echo '{"error": "Index file not found. Run scripts/update-index first."}'
        return
    fi

    # Extract uses: directives from workflow
    local uses_lines
    uses_lines=$(grep -v '^[[:space:]]*#' "$file" | grep -oE 'uses:[[:space:]]*[^ ]+' | sed 's/uses:[[:space:]]*//' | sort -u)

    if [[ -z "$uses_lines" ]]; then
        jq -n --arg f "$file" '{file: $f, message: "No action references found", issues: []}'
        return
    fi

    local issues="[]"
    local checked=0
    local outdated=0

    while IFS= read -r usage; do
        # Parse owner/repo@version (handle subpaths like github/codeql-action/init@v3)
        local action_with_version="$usage"
        local action version

        if [[ "$action_with_version" == *"@"* ]]; then
            version="${action_with_version##*@}"
            action="${action_with_version%@*}"
        else
            continue
        fi

        # Look up in index
        local index_entry
        index_entry=$(jq -r --arg a "$action" '.actions[$a] // empty' "$INDEX_PATH")

        if [[ -z "$index_entry" ]]; then
            continue
        fi

        checked=$((checked + 1))
        local latest
        latest=$(echo "$index_entry" | jq -r '.latest')

        # If version is a SHA, check against the index SHA
        local index_sha is_sha="false"
        index_sha=$(echo "$index_entry" | jq -r '.sha // empty')
        if [[ ${#version} -ge 40 && "$version" =~ ^[0-9a-f]+$ ]]; then
            is_sha="true"
            if [[ "$version" == "$index_sha" ]]; then
                continue  # SHA-pinned and up to date
            fi
        fi

        if [[ "$version" != "$latest" || "$is_sha" == "true" ]]; then
            local latest_full deprecated sha_val
            latest_full=$(echo "$index_entry" | jq -r '.latestFull')
            sha_val="$index_sha"
            deprecated=$(echo "$index_entry" | jq -r --arg v "$version" '.deprecated // [] | if map(. == $v) | any then "true" else "false" end')

            # Find applicable migration data
            local cur_major_num lat_major_num
            cur_major_num=$(echo "$version" | sed -n 's/^v\([0-9]*\).*/\1/p')
            lat_major_num=$(echo "$latest" | sed -n 's/^v\([0-9]*\).*/\1/p')

            local migration="null"
            if [[ -n "$cur_major_num" && -n "$lat_major_num" ]]; then
                migration=$(echo "$index_entry" | jq --arg from_num "$cur_major_num" --arg to_num "$lat_major_num" '
                    .migrations // {} | to_entries
                    | map(select(
                        (.key | split("→") | .[0] | ltrimstr("v") | tonumber) >= ($from_num | tonumber)
                        and (.key | split("→") | .[1] | ltrimstr("v") | tonumber) <= ($to_num | tonumber)
                    ))
                    | if length == 0 then null
                      elif length == 1 then .[0].value
                      else {
                        steps: [.[] | {migration: .key} + .value],
                        inputChanges: {
                            added: [.[].value.inputChanges.added // [] | .[]] | unique,
                            removed: [.[].value.inputChanges.removed // [] | .[]] | unique
                        }
                      }
                      end')
            fi

            issues=$(echo "$issues" | jq \
                --arg a "$action" --arg cur "$version" --arg lat "$latest" \
                --arg latf "$latest_full" --arg sha "$sha_val" --arg dep "$deprecated" \
                --argjson mig "$migration" \
                '. + [{
                    action: $a, current: $cur, latest: $lat, latestFull: $latf,
                    deprecated: ($dep == "true")
                } + (if $sha != "" then {sha: $sha, pinned: ($a + "@" + $sha + " # " + $latf)} else {} end)
                + (if $mig != null then {migration: $mig} else {} end)]')
            outdated=$((outdated + 1))
        fi
    done <<< "$uses_lines"

    jq -n --arg f "$file" --argjson checked "$checked" --argjson outdated "$outdated" --argjson issues "$issues" \
        '{file: $f, checked: $checked, outdated: $outdated, issues: $issues}'
}
```

- [ ] **Step 5: Make actionista-lib executable (for source compatibility)**

```bash
chmod +x scripts/actionista-lib
```

- [ ] **Step 6: Commit**

```bash
git add scripts/actionista-lib tests/mcp/
git commit -m "feat: extract actionista-lib and add MCP tests"
```

---

## Task 3: Refactor actionista-mcp to Source Lib

**Files:**
- Modify: `scripts/actionista-mcp`

### Rationale

Replace the handler functions in `actionista-mcp` with a `source` of `actionista-lib`. Keep ONLY the JSON-RPC dispatch (`handle_request`) and the main loop. MCP tests from Task 2 validate the refactor.

- [ ] **Step 1: Rewrite actionista-mcp**

Replace `scripts/actionista-mcp` with:

```bash
#!/usr/bin/env bash
# actionista-mcp — MCP server for GitHub Actions version lookups
#
# Speaks JSON-RPC 2.0 over stdio. Provides tools:
#   - lookup_action: Get version info for a specific action
#   - list_actions: List tracked actions by category
#   - check_workflow: Report outdated versions in a workflow file
#
# Requirements: jq, yq (for check_workflow only)

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
INDEX_PATH="$SCRIPT_DIR/../skills/github-actions/index.json"

# Load shared handler functions
source "$SCRIPT_DIR/actionista-lib"

# ============================================================================
# JSON-RPC dispatch
# ============================================================================

handle_request() {
    local request="$1"

    local id method params
    id=$(echo "$request" | jq -c '.id // empty')
    method=$(echo "$request" | jq -r '.method // empty')
    params=$(echo "$request" | jq -c '.params // {}')

    # Notifications (no id) — don't respond
    if [[ -z "$id" || "$id" == "null" ]]; then
        return
    fi

    case "$method" in
        initialize)
            local version
            version=$(cat "$SCRIPT_DIR/../VERSION" 2>/dev/null || echo "0.0.0")
            jq -n --argjson id "$id" --arg v "$version" '{
                jsonrpc: "2.0",
                id: $id,
                result: {
                    protocolVersion: "2024-11-05",
                    capabilities: {tools: {}},
                    serverInfo: {name: "actionista", version: $v}
                }
            }'
            ;;

        tools/list)
            local tools
            tools=$(tools_json)
            jq -n --argjson id "$id" --argjson tools "$tools" '{
                jsonrpc: "2.0",
                id: $id,
                result: {tools: $tools}
            }'
            ;;

        tools/call)
            local tool_name tool_args result
            tool_name=$(echo "$params" | jq -r '.name // empty')
            tool_args=$(echo "$params" | jq -c '.arguments // {}')

            case "$tool_name" in
                lookup_action)
                    local action
                    action=$(echo "$tool_args" | jq -r '.action // empty')
                    if [[ -z "$action" ]]; then
                        result='{"error": "Missing required parameter: action"}'
                    else
                        result=$(handle_lookup_action "$action")
                    fi
                    ;;
                list_actions)
                    local category
                    category=$(echo "$tool_args" | jq -r '.category // empty')
                    result=$(handle_list_actions "$category")
                    ;;
                check_workflow)
                    local file
                    file=$(echo "$tool_args" | jq -r '.file // empty')
                    if [[ -z "$file" ]]; then
                        result='{"error": "Missing required parameter: file"}'
                    else
                        result=$(handle_check_workflow "$file")
                    fi
                    ;;
                *)
                    jq -n --argjson id "$id" --arg name "$tool_name" '{
                        jsonrpc: "2.0",
                        id: $id,
                        error: {code: -32601, message: ("Unknown tool: " + $name)}
                    }'
                    return
                    ;;
            esac

            jq -n --argjson id "$id" --arg text "$result" '{
                jsonrpc: "2.0",
                id: $id,
                result: {content: [{type: "text", text: $text}]}
            }'
            ;;

        *)
            jq -n --argjson id "$id" --arg m "$method" '{
                jsonrpc: "2.0",
                id: $id,
                error: {code: -32601, message: ("Method not found: " + $m)}
            }'
            ;;
    esac
}

# ============================================================================
# Main loop — read JSON-RPC from stdin, write responses to stdout
# ============================================================================

main() {
    if ! command -v jq &>/dev/null; then
        echo '{"jsonrpc":"2.0","error":{"code":-32603,"message":"jq is required but not installed"}}' >&2
        exit 1
    fi

    while IFS= read -r line; do
        [[ -z "$line" ]] && continue
        if ! echo "$line" | jq -e '.' &>/dev/null; then
            continue
        fi

        local response
        response=$(handle_request "$line")

        if [[ -n "$response" ]]; then
            echo "$response"
        fi
    done
}

main
```

Key change: The `initialize` response now reads version from `VERSION` file instead of a hardcoded string. This eliminates the version drift between the MCP server and manifests.

- [ ] **Step 2: Run MCP tests to verify refactor**

```bash
bash tests/mcp/test-initialize.sh .
bash tests/mcp/test-lookup.sh .
```

Expected: PASS — same behavior, different internal structure.

- [ ] **Step 3: Commit**

```bash
git add scripts/actionista-mcp
git commit -m "refactor: actionista-mcp sources shared lib, reads version from VERSION"
```

---

## Task 4: CLI Wrapper

**Files:**
- Create: `scripts/actionista`
- Create: `tests/cli/test-lookup.sh`
- Create: `tests/cli/test-check.sh`
- Create: `tests/cli/test-list.sh`

### Rationale

The CLI gives Codex, Gemini CLI, and any Bash-capable agent access to actionista's data. It sources the same `actionista-lib` as the MCP server. JSON output by default (for agents), human-readable on TTY.

- [ ] **Step 1: Write CLI tests**

Create `tests/cli/test-lookup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"
CLI="$REPO_ROOT/scripts/actionista"

# Test lookup
result=$("$CLI" lookup actions/checkout)
action=$(echo "$result" | jq -r '.action')
latest=$(echo "$result" | jq -r '.latest // empty')
sha=$(echo "$result" | jq -r '.sha // empty')
pinned=$(echo "$result" | jq -r '.pinned // empty')

[ "$action" = "actions/checkout" ] || { echo "Bad action: $action"; exit 1; }
[ -n "$latest" ] || { echo "Missing latest"; exit 1; }
[ -n "$sha" ] || { echo "Missing sha"; exit 1; }
[ -n "$pinned" ] || { echo "Missing pinned"; exit 1; }

# Test not found
result=$("$CLI" lookup nonexistent/action)
error=$(echo "$result" | jq -r '.error // empty')
[ -n "$error" ] || { echo "Expected error for nonexistent action"; exit 1; }
```

Create `tests/cli/test-check.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"
CLI="$REPO_ROOT/scripts/actionista"

# Check a workflow that exists in the repo
result=$("$CLI" check "$REPO_ROOT/.github/workflows/update-index.yml")
file=$(echo "$result" | jq -r '.file')
checked=$(echo "$result" | jq -r '.checked')

[ -n "$file" ] || { echo "Missing file field"; exit 1; }
[ "$checked" -ge 0 ] 2>/dev/null || { echo "Bad checked count: $checked"; exit 1; }

# Test missing file
result=$("$CLI" check /nonexistent/workflow.yml)
error=$(echo "$result" | jq -r '.error // empty')
[ -n "$error" ] || { echo "Expected error for missing file"; exit 1; }
```

Create `tests/cli/test-list.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"
CLI="$REPO_ROOT/scripts/actionista"

# List all
result=$("$CLI" list)
total=$(echo "$result" | jq -r '.total')
[ "$total" -gt 0 ] 2>/dev/null || { echo "Bad total: $total"; exit 1; }

# List by category
result=$("$CLI" list core)
category=$(echo "$result" | jq -r '.category // empty')
[ "$category" = "core" ] || { echo "Bad category: $category"; exit 1; }

# List bad category
result=$("$CLI" list nonexistent-category)
error=$(echo "$result" | jq -r '.error // empty')
[ -n "$error" ] || { echo "Expected error for bad category"; exit 1; }
```

- [ ] **Step 2: Create CLI wrapper**

Create `scripts/actionista`:

```bash
#!/usr/bin/env bash
# actionista — CLI for GitHub Actions version lookups
#
# Usage:
#   actionista lookup <action>    — version, SHA, pinned reference
#   actionista check <file>       — find outdated actions in a workflow
#   actionista list [category]    — browse tracked actions
#   actionista version            — print version
#   actionista help               — show this help

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
INDEX_PATH="$SCRIPT_DIR/../skills/github-actions/index.json"

# Load shared handler functions
source "$SCRIPT_DIR/actionista-lib"

VERSION=$(cat "$SCRIPT_DIR/../VERSION" 2>/dev/null || echo "0.0.0")

# TTY detection: human-readable output when stdout is a terminal
IS_TTY=false
[ -t 1 ] && IS_TTY=true

# Format JSON for human consumption when on a TTY
format_output() {
    if [ "$IS_TTY" = true ]; then
        jq -C '.' 2>/dev/null || cat
    else
        cat
    fi
}

usage() {
    cat <<EOF
actionista $VERSION — GitHub Actions version assistant

Usage:
  actionista lookup <action>    Look up version info for a GitHub Action
  actionista check <file>       Check a workflow file for outdated actions
  actionista list [category]    List tracked actions (optionally by category)
  actionista version            Print version
  actionista help               Show this help

Options:
  --json                        Force JSON output (even on TTY)

Examples:
  actionista lookup actions/checkout
  actionista check .github/workflows/ci.yml
  actionista list docker
EOF
}

# Handle --json flag anywhere in args
for arg in "$@"; do
    [ "$arg" = "--json" ] && IS_TTY=false
done
# Remove --json from positional args
args=()
for arg in "$@"; do
    [ "$arg" != "--json" ] && args+=("$arg")
done
set -- "${args[@]+"${args[@]}"}"

case "${1:-help}" in
    lookup)
        [ -n "${2:-}" ] || { echo '{"error": "Usage: actionista lookup <action>"}'; exit 1; }
        handle_lookup_action "$2" | format_output
        ;;
    check)
        [ -n "${2:-}" ] || { echo '{"error": "Usage: actionista check <file>"}'; exit 1; }
        handle_check_workflow "$2" | format_output
        ;;
    list)
        handle_list_actions "${2:-}" | format_output
        ;;
    version)
        echo "$VERSION"
        ;;
    help|--help|-h)
        usage
        ;;
    *)
        echo "Unknown command: $1" >&2
        usage >&2
        exit 1
        ;;
esac
```

- [ ] **Step 3: Make executable and run CLI tests**

```bash
chmod +x scripts/actionista tests/cli/test-*.sh
bash tests/cli/test-lookup.sh .
bash tests/cli/test-check.sh .
bash tests/cli/test-list.sh .
```

Expected: PASS

- [ ] **Step 4: Run full test suite**

```bash
bash tests/run-all.sh
```

Expected: Structural config/version tests still fail (missing per-agent files). MCP and CLI tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/actionista tests/cli/
git commit -m "feat: add CLI wrapper for Codex/Gemini/Bash agents"
```

---

## Task 5: MCP check_workflow and list Tests

**Files:**
- Create: `tests/mcp/test-check-workflow.sh`
- Create: `tests/mcp/test-list.sh`

### Rationale

Complete the MCP test coverage. These were deferred from Task 2 to keep that task focused on the extraction prerequisite tests.

- [ ] **Step 1: Write MCP check_workflow test**

Create `tests/mcp/test-check-workflow.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"

# Use a real workflow file from the repo
workflow="$REPO_ROOT/.github/workflows/update-index.yml"
[ -f "$workflow" ] || { echo "Missing test workflow: $workflow"; exit 1; }

response=$(printf '%s\n%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
    "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"check_workflow\",\"arguments\":{\"file\":\"$workflow\"}}}" \
    | bash "$REPO_ROOT/scripts/actionista-mcp" | tail -1)

text=$(echo "$response" | jq -r '.result.content[0].text')
file_field=$(echo "$text" | jq -r '.file')
checked=$(echo "$text" | jq -r '.checked')

[ -n "$file_field" ] || { echo "Missing file field"; exit 1; }
[ "$checked" -ge 0 ] 2>/dev/null || { echo "Bad checked count: $checked"; exit 1; }
```

- [ ] **Step 2: Write MCP list test**

Create `tests/mcp/test-list.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO_ROOT="${1:-.}"

# List all actions
response=$(printf '%s\n%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
    '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_actions","arguments":{}}}' \
    | bash "$REPO_ROOT/scripts/actionista-mcp" | tail -1)

text=$(echo "$response" | jq -r '.result.content[0].text')
total=$(echo "$text" | jq -r '.total')

[ "$total" -gt 0 ] 2>/dev/null || { echo "Bad total: $total"; exit 1; }

# List by category
response=$(printf '%s\n%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
    '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_actions","arguments":{"category":"core"}}}' \
    | bash "$REPO_ROOT/scripts/actionista-mcp" | tail -1)

text=$(echo "$response" | jq -r '.result.content[0].text')
category=$(echo "$text" | jq -r '.category // empty')

[ "$category" = "core" ] || { echo "Bad category: $category"; exit 1; }
```

- [ ] **Step 3: Run and verify**

```bash
chmod +x tests/mcp/test-check-workflow.sh tests/mcp/test-list.sh
bash tests/mcp/test-check-workflow.sh .
bash tests/mcp/test-list.sh .
```

Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add tests/mcp/
git commit -m "test: add MCP check_workflow and list tests"
```

---

## Task 6: Cursor Configs

**Files:**
- Create: `.cursor-plugin/plugin.json`
- Create: `.cursor/mcp.json`

- [ ] **Step 1: Create Cursor plugin manifest**

Create `.cursor-plugin/plugin.json`:

```json
{
  "name": "actionista",
  "displayName": "Actionista",
  "version": "0.3.0",
  "description": "GitHub Actions assistant — version tracking, SHA pinning, workflow review",
  "author": {
    "name": "Clay Loveless",
    "email": "clay@loveless.net",
    "url": "https://github.com/claylo"
  },
  "homepage": "https://github.com/claylo/actionista",
  "repository": "https://github.com/claylo/actionista",
  "license": "MIT",
  "skills": "./skills/",
  "agents": "./agents/"
}
```

- [ ] **Step 2: Create Cursor MCP config**

Create `.cursor/mcp.json`:

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

Note: Uses relative path (not `${CLAUDE_PLUGIN_ROOT}` which is Claude-specific).

- [ ] **Step 3: Commit**

```bash
git add .cursor-plugin/ .cursor/
git commit -m "feat: add Cursor plugin and MCP configs"
```

---

## Task 7: Codex and Zed Configs

**Files:**
- Create: `AGENTS.md`
- Create: `.codex/INSTALL.md`

- [ ] **Step 1: Create AGENTS.md**

This file is read by both Codex and Zed as agent context:

```markdown
# Actionista — GitHub Actions Assistant

You have access to `scripts/actionista` for GitHub Actions version lookups, SHA pinning, and workflow analysis.

## CLI Commands

- `scripts/actionista lookup <action>` — Get version, SHA, and ready-to-paste pinned reference
- `scripts/actionista check <file>` — Find outdated actions in a workflow file with migration info
- `scripts/actionista list [category]` — Browse all tracked actions (120+), optionally filtered by category

## Skills

Read `skills/github-actions/SKILL.md` for comprehensive GitHub Actions patterns, security best practices, and workflow examples.

Key skill directories:
- `skills/github-actions/patterns/` — Matrix builds, caching, artifacts, reusable workflows
- `skills/github-actions/security/` — Hardening, secrets management, supply chain security
- `skills/github-actions/references/` — Triggers, permissions, expressions, runners
- `skills/github-actions/examples/` — Complete workflow examples (Node CI, Rust CI, AWS deploy)

## Workflow Review

When reviewing or writing GitHub Actions workflows:
1. Run `scripts/actionista check <file>` to find outdated actions
2. Use `scripts/actionista lookup <action>` for current versions with SHA pins
3. Consult `skills/github-actions/security/supply-chain.md` for pinning best practices
```

- [ ] **Step 2: Create Codex install guide**

Create `.codex/INSTALL.md`:

```markdown
# Installing Actionista for Codex

1. Clone the repository:
   ```bash
   git clone https://github.com/claylo/actionista.git ~/.codex/actionista
   ```

2. The CLI is immediately available:
   ```bash
   ~/.codex/actionista/scripts/actionista lookup actions/checkout
   ```

3. Codex reads `AGENTS.md` at the repo root for context about available commands and skills.

4. To use in any project, reference the actionista path:
   ```bash
   ~/.codex/actionista/scripts/actionista check .github/workflows/ci.yml
   ```
```

- [ ] **Step 3: Commit**

```bash
git add AGENTS.md .codex/
git commit -m "feat: add AGENTS.md and Codex install guide"
```

---

## Task 8: OpenCode Config

**Files:**
- Create: `package.json`
- Create: `.opencode/plugins/actionista.js`
- Create: `.opencode/INSTALL.md`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "actionista",
  "version": "0.3.0",
  "description": "GitHub Actions assistant — version tracking, SHA pinning, workflow review",
  "type": "module",
  "main": ".opencode/plugins/actionista.js",
  "author": "Clay Loveless <clay@loveless.net>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/claylo/actionista.git"
  }
}
```

- [ ] **Step 2: Create OpenCode plugin**

Create `.opencode/plugins/actionista.js`:

```javascript
// actionista — OpenCode plugin hook
// Injects skill paths and CLI context into OpenCode sessions.

export default {
  name: "actionista",

  config(config) {
    // Add skills directory to OpenCode's skill discovery
    const skillsPath = new URL("../../skills", import.meta.url).pathname;
    config.skills = config.skills || {};
    config.skills.paths = config.skills.paths || [];
    if (!config.skills.paths.includes(skillsPath)) {
      config.skills.paths.push(skillsPath);
    }
    return config;
  },

  experimental: {
    chat: {
      system: {
        transform(system) {
          const cliPath = new URL("../../scripts/actionista", import.meta.url).pathname;
          return system + `\n\nYou have access to the actionista CLI at ${cliPath}:\n` +
            `- ${cliPath} lookup <action> — version, SHA, pinned reference\n` +
            `- ${cliPath} check <file> — find outdated actions in a workflow\n` +
            `- ${cliPath} list [category] — browse tracked actions\n`;
        }
      }
    }
  }
};
```

- [ ] **Step 3: Create OpenCode install guide**

Create `.opencode/INSTALL.md`:

```markdown
# Installing Actionista for OpenCode

## Via opencode.json

Add to your project's `opencode.json`:

```json
{
  "plugin": ["actionista@git+https://github.com/claylo/actionista.git"]
}
```

## Manual MCP Setup

If the plugin's MCP registration doesn't work automatically, add to your `opencode.json`:

```json
{
  "mcp": {
    "actionista": {
      "type": "stdio",
      "command": "bash",
      "args": ["<path-to-actionista>/scripts/actionista-mcp"]
    }
  }
}
```

## CLI Fallback

The CLI works without MCP:

```bash
<path-to-actionista>/scripts/actionista lookup actions/checkout
<path-to-actionista>/scripts/actionista check .github/workflows/ci.yml
```
```

- [ ] **Step 4: Commit**

```bash
git add package.json .opencode/
git commit -m "feat: add OpenCode plugin, package.json, and install guide"
```

---

## Task 9: Gemini CLI Config

**Files:**
- Create: `gemini-extension.json`
- Create: `GEMINI.md`

- [ ] **Step 1: Create Gemini extension manifest**

Create `gemini-extension.json`:

```json
{
  "name": "actionista",
  "description": "GitHub Actions assistant — version tracking, SHA pinning, workflow review",
  "version": "0.3.0",
  "contextFileName": "GEMINI.md"
}
```

- [ ] **Step 2: Create GEMINI.md**

```markdown
# Actionista — GitHub Actions Assistant

You have access to the actionista CLI for GitHub Actions version lookups.

## CLI Commands

Run these via Bash:

- `scripts/actionista lookup <action>` — Get version, SHA, and ready-to-paste pinned reference
- `scripts/actionista check <file>` — Find outdated actions in a workflow file with migration info
- `scripts/actionista list [category]` — Browse all tracked actions (120+)

## Skills Reference

@skills/github-actions/SKILL.md — Main skill file with patterns, security, and version index
@skills/github-actions/security/supply-chain.md — SHA pinning and supply chain security
@skills/github-actions/patterns/caching.md — Dependency caching patterns
@skills/github-actions/patterns/matrix-builds.md — Matrix build strategies
@skills/github-actions/references/triggers.md — Workflow trigger events

## Workflow Review Process

When reviewing or writing GitHub Actions workflows:
1. Run `scripts/actionista check <file>` to find outdated actions
2. Use `scripts/actionista lookup <action>` for current versions with SHA pins
3. Consult the security skills for best practices
```

- [ ] **Step 3: Commit**

```bash
git add gemini-extension.json GEMINI.md
git commit -m "feat: add Gemini CLI extension and context file"
```

---

## Task 10: Zed and Claude Code Install Docs

**Files:**
- Create: `docs/install/zed.md`
- Create: `docs/install/claude-code.md`
- Create: `docs/install/cursor.md`
- Create: `docs/install/codex.md`
- Create: `docs/install/opencode.md`
- Create: `docs/install/gemini.md`

- [ ] **Step 1: Create Zed install doc**

Create `docs/install/zed.md`:

```markdown
# Actionista for Zed

Zed supports MCP servers and reads `AGENTS.md` for agent context.

## Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/claylo/actionista.git ~/actionista
   ```

2. Add the MCP server to `~/.config/zed/settings.json`:
   ```json
   {
     "context_servers": {
       "actionista": {
         "command": {
           "path": "bash",
           "args": ["~/actionista/scripts/actionista-mcp"]
         }
       }
     }
   }
   ```

3. Zed reads `AGENTS.md` from the repo root for agent context. When working in a project that uses actionista, symlink or copy it:
   ```bash
   ln -s ~/actionista/AGENTS.md ./AGENTS.md
   ```

## Available Tools (via MCP)

- `lookup_action` — version info, SHA, pinned reference
- `list_actions` — browse tracked actions by category
- `check_workflow` — find outdated actions in workflow files

## Available Skills

Copy or symlink `skills/github-actions/` into your project for Zed's skill discovery.
```

- [ ] **Step 2: Create remaining install docs**

Create `docs/install/claude-code.md`:

```markdown
# Actionista for Claude Code

## Via Marketplace (recommended)

```bash
/plugin marketplace add claylo/claylo-marketplace
/plugin install actionista
```

## Manual Installation

```bash
git clone https://github.com/claylo/actionista.git ~/.claude/plugins/actionista
```

Actionista auto-registers its MCP server, skills, and agents via `.claude-plugin/plugin.json` and `.mcp.json`.

## What You Get

- **MCP tools:** `lookup_action`, `list_actions`, `check_workflow` — available as native tools
- **Skills:** `github-actions` skill triggers on workflow-related queries
- **Agent:** `workflow-analyzer` for reviewing workflow files
- **Slash command:** `/review-workflow` for on-demand workflow analysis
```

Create `docs/install/cursor.md`:

```markdown
# Actionista for Cursor

1. Clone the repository:
   ```bash
   git clone https://github.com/claylo/actionista.git ~/.cursor/plugins/actionista
   ```

2. Cursor discovers the plugin via `.cursor-plugin/plugin.json` and registers the MCP server from `.cursor/mcp.json`.

## What You Get

- **MCP tools:** `lookup_action`, `list_actions`, `check_workflow`
- **Skills:** GitHub Actions patterns, security, and version reference
- **Agent:** `workflow-analyzer` for workflow review
```

Create `docs/install/codex.md`:

```markdown
# Actionista for Codex

See `.codex/INSTALL.md` for setup instructions.

Codex uses the CLI wrapper (`scripts/actionista`) and reads `AGENTS.md` for context.
```

Create `docs/install/opencode.md`:

```markdown
# Actionista for OpenCode

See `.opencode/INSTALL.md` for setup instructions.

OpenCode uses the npm plugin system and optionally the MCP server.
```

Create `docs/install/gemini.md`:

```markdown
# Actionista for Gemini CLI

1. Clone the repository:
   ```bash
   git clone https://github.com/claylo/actionista.git ~/actionista
   ```

2. Register as a Gemini extension — add to your Gemini CLI config:
   ```bash
   gemini extensions add ~/actionista
   ```

3. Gemini reads `GEMINI.md` for context and `@` file references to skill content.

## What You Get

- **CLI commands** via Bash: `scripts/actionista lookup|check|list`
- **Skills** via `@` references in GEMINI.md
```

- [ ] **Step 3: Commit**

```bash
git add docs/install/
git commit -m "docs: add per-agent install guides"
```

---

## Task 11: Update Claude Plugin Version

**Files:**
- Modify: `.claude-plugin/plugin.json`

### Rationale

Reconcile the version drift: plugin.json says 0.2.0, everything else says 0.3.0. This is the handoff's noted landmine.

- [ ] **Step 1: Update plugin.json version**

In `.claude-plugin/plugin.json`, change:
```json
"version": "0.2.0"
```
to:
```json
"version": "0.3.0"
```

- [ ] **Step 2: Run version sync test**

```bash
bash tests/structural/test-versions-sync.sh .
```

Expected: PASS — all manifests now report 0.3.0.

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: reconcile plugin.json version to 0.3.0"
```

---

## Task 12: Update Release Workflow

**Files:**
- Modify: `.github/workflows/release-on-update.yml`

### Rationale

The release workflow needs to: (1) read version from `VERSION`, (2) bump `VERSION`, (3) update ALL manifests (not just Claude's plugin.json), (4) remove `[skip release]` — path filters already prevent re-triggering.

**Spec divergence note:** The spec (section "Release Pipeline", item 4) says to bump `scripts/actionista-mcp` (serverInfo version string). Task 3's refactor changed the MCP server to read version from `VERSION` at runtime (`cat "$SCRIPT_DIR/../VERSION"`), so the hardcoded version string no longer exists and the release workflow does NOT need to patch `actionista-mcp`. This is an intentional improvement over the spec.

- [ ] **Step 1: Rewrite release-on-update.yml**

Replace `.github/workflows/release-on-update.yml` with:

```yaml
# Release on Index Update
# When an index update merges to main, bump patch version, tag, release,
# and notify claylo-marketplace to update its listing.

name: Release on Update

on:
  push:
    branches: [main]
    paths:
      - 'skills/github-actions/index.json'

permissions:
  contents: write

jobs:
  release:
    name: Tag and Release
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          fetch-depth: 0

      - name: Compute next version
        id: version
        run: |
          current=$(cat VERSION)
          IFS='.' read -r major minor patch <<< "$current"
          next="${major}.${minor}.$((patch + 1))"
          echo "current=$current" >> $GITHUB_OUTPUT
          echo "next=$next" >> $GITHUB_OUTPUT
          echo "tag=v${next}" >> $GITHUB_OUTPUT
          echo "Bumping ${current} → ${next}"

      - name: Bump all versions
        run: |
          next="${{ steps.version.outputs.next }}"

          # VERSION file (source of truth)
          echo "$next" > VERSION

          # JSON manifests
          for manifest in .claude-plugin/plugin.json .cursor-plugin/plugin.json package.json gemini-extension.json; do
            if [ -f "$manifest" ]; then
              jq --arg v "$next" '.version = $v' "$manifest" > tmp.json
              mv tmp.json "$manifest"
            fi
          done

      - name: Commit version bump
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add VERSION .claude-plugin/plugin.json .cursor-plugin/plugin.json package.json gemini-extension.json
          git commit -m "chore: bump version to ${{ steps.version.outputs.next }}"
          git push

      - name: Build release notes
        id: notes
        run: |
          prev_tag=$(git tag --sort=-v:refname | head -1)
          {
            echo "notes<<NOTES_EOF"
            echo "Automated index update — action versions refreshed from GitHub."
            echo ""
            echo "**Version:** ${{ steps.version.outputs.next }}"
            echo ""
            if [ -n "$prev_tag" ]; then
              changed=$(git diff "${prev_tag}"..HEAD -- skills/github-actions/index.json | grep '^[+-].*"latest' | head -20)
              if [ -n "$changed" ]; then
                echo "### Version changes"
                echo '```diff'
                echo "$changed"
                echo '```'
              fi
            fi
            echo ""
            echo "[Full changelog →](https://github.com/claylo/actionista/blob/main/CHANGELOG.md)"
            echo "NOTES_EOF"
          } >> $GITHUB_OUTPUT

      - name: Create release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git tag "${{ steps.version.outputs.tag }}"
          git push origin "${{ steps.version.outputs.tag }}"
          gh release create "${{ steps.version.outputs.tag }}" \
            --title "${{ steps.version.outputs.tag }}: Index update" \
            --notes "${{ steps.notes.outputs.notes }}"

      - name: Notify marketplace
        env:
          GH_TOKEN: ${{ secrets.CROSS_REPO_PAT }}
        run: |
          gh api repos/claylo/claylo-marketplace/dispatches \
            -f event_type=plugin-update \
            -f 'client_payload[plugin]=actionista' \
            -f 'client_payload[version]=${{ steps.version.outputs.next }}'
```

Key changes from current:
- Reads version from `VERSION` instead of `plugin.json`
- Bumps `VERSION` + all JSON manifests in one step
- No `[skip release]` guard or commit message flag (path filter prevents loops)

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/release-on-update.yml
git commit -m "feat: release workflow bumps VERSION and all manifests"
```

---

## Task 13: Run Full Test Suite and Fix

**Files:**
- Possibly modify any files with issues

- [ ] **Step 1: Run full test suite**

```bash
bash tests/run-all.sh
```

Expected: ALL PASS. Every structural, MCP, and CLI test should pass now.

- [ ] **Step 2: Fix any failures**

If any tests fail, diagnose and fix. Common issues:
- Permission bits on new scripts (`chmod +x`)
- JSON syntax errors in new manifests
- Path resolution in tests

- [ ] **Step 3: Run tests again to confirm**

```bash
bash tests/run-all.sh
```

Expected: All pass.

- [ ] **Step 4: Commit fixes (if any)**

Stage only the specific files that were fixed (do NOT use `git add -A`), then:

```bash
git commit -m "fix: address test failures"
```

---

## Task 14: Update .gitignore

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: Add entries for new agent tooling**

Ensure `.gitignore` includes entries for agent-specific local state that shouldn't be committed:

```
# Cursor local settings
.cursor/*.local.*

# OpenCode local
.opencode/*.local.*
```

The `.cursor/mcp.json` and `.opencode/plugins/actionista.js` are intentionally tracked (they're config templates, not local state).

- [ ] **Step 2: Commit**

```bash
git add .gitignore
git commit -m "chore: update gitignore for multi-agent configs"
```

---

## Summary

| Task | Description | New Files | Modified Files |
|------|-------------|-----------|----------------|
| 1 | VERSION + test harness | 5 | 0 |
| 2 | Extract actionista-lib | 3 | 0 |
| 3 | Refactor actionista-mcp | 0 | 1 |
| 4 | CLI wrapper | 4 | 0 |
| 5 | MCP test completion | 2 | 0 |
| 6 | Cursor configs | 2 | 0 |
| 7 | Codex/Zed configs | 2 | 0 |
| 8 | OpenCode configs | 3 | 0 |
| 9 | Gemini CLI configs | 2 | 0 |
| 10 | Install docs | 6 | 0 |
| 11 | Claude plugin version | 0 | 1 |
| 12 | Release workflow | 0 | 1 |
| 13 | Full test run + fixes | 0 | varies |
| 14 | Gitignore update | 0 | 1 |

**Total: ~29 new files, 4 modified files, 14 commits**
