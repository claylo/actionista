# Actionista

A Claude Code plugin for GitHub Actions assistance — helps create, review, and optimize workflows with up-to-date action versions and best practices.

## Features

- **Version Awareness**: Tracks 100+ popular GitHub Actions with their latest versions
- **Pattern Knowledge**: Matrix builds, caching, reusable workflows, concurrency, and more
- **Security Guidance**: Secrets management, OIDC, runner hardening, supply chain security
- **Active Analysis**: Reviews existing workflows and suggests improvements
- **ASCII Flowcharts**: Generates visual workflow representations

## Installation

```bash
claude plugin install claylo/actionista
```

## Usage

### Automatic Assistance

The plugin automatically activates when you work with:

- GitHub Actions workflow files (`.github/workflows/*.yml`)
- Any YAML file containing workflow syntax
- Questions about CI/CD, GitHub Actions, or specific actions

**Example queries that trigger the skill:**

```
"How do I set up caching in my Node.js workflow?"
"What's the latest version of actions/checkout?"
"Help me create a matrix build for multiple Node versions"
"How do I use OIDC with AWS?"
```

### Commands

#### `/review-workflow`

Review and analyze GitHub Actions workflow files for issues and improvements.

```bash
# Review a specific workflow
/review-workflow .github/workflows/ci.yml

# Review all workflows in the project
/review-workflow --all

# Review and auto-fix safe issues (version bumps, add caching)
/review-workflow --fix

# Validate workflow syntax only
/review-workflow --validate
```

### Workflow Analyzer Agent

The workflow analyzer agent activates automatically when you create or modify workflow files. It can:

- Identify outdated action versions
- Suggest performance optimizations (caching, parallelization)
- Find security issues (permissions, script injection)
- Recommend best practices
- Generate ASCII flowcharts of job dependencies

## Actions Index

The plugin maintains an index of 100+ popular GitHub Actions organized by category:

| Category | Examples |
|----------|----------|
| **Core GitHub** | checkout, cache, upload-artifact, download-artifact |
| **Language Setup** | setup-node, setup-python, setup-go, setup-java |
| **Cloud Providers** | aws-actions/*, azure/*, google-github-actions/* |
| **Docker** | docker/build-push-action, docker/login-action |
| **Security** | github/codeql-action, aquasecurity/trivy-action |
| **Release** | release-please-action, goreleaser-action |
| **Testing** | codecov-action, cypress-io/github-action |

The index is updated daily via GitHub Actions and submitted as a PR for review.

## Skill Content

The plugin includes comprehensive documentation on:

### Patterns
- Matrix builds with include/exclude
- Dependency caching strategies
- Artifact handling
- Reusable workflows
- Concurrency control
- Environment management
- Composite actions

### Security
- Secrets management and OIDC
- Security hardening
- Supply chain protection

### References
- Complete workflow syntax
- Expression language and contexts
- Trigger events
- Runner configuration
- Permissions and OIDC
- Troubleshooting guide

### Examples
- Node.js CI workflow
- Rust CI workflow
- AWS deployment with OIDC
- Release automation with release-please

## Development

### Update the Actions Index

The index is updated automatically via GitHub Actions, but you can run it manually:

```bash
cd /path/to/actionista
scripts/update-index
```

Requires `gh` (authenticated), `yq`, and `jq`.

### Test Locally

```bash
# Test with plugin directory
claude --plugin-dir /path/to/actionista

# Or install for a project
cd your-project
claude plugin install /path/to/actionista --scope project
```

## MCP Server

The plugin includes a built-in MCP server that provides tools for querying the actions index:

| Tool | Description |
|------|-------------|
| `lookup_action` | Look up version info for a specific action |
| `list_actions` | List tracked actions, optionally filtered by category |
| `check_workflow` | Report outdated action versions in a workflow file |

The MCP server starts automatically when the plugin is loaded in Claude Code.

## Contributing

Contributions welcome! Areas of interest:

- Adding more actions to `tracked-actions.yaml`
- Improving pattern documentation
- Adding more workflow examples
- Building the MCP server

## License

MIT

---

Built with [Claude Code](https://claude.ai/code)
