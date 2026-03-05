# Expressions Reference

Complete reference for GitHub Actions expressions `${{ }}`.

## Syntax

```yaml
${{ <expression> }}
```

Expressions can be used in:
- `if` conditionals
- Environment variables
- Action inputs
- Job/step outputs
- Most workflow values

## Contexts

### `github` Context

Event and repository information:

```yaml
github.event_name           # push, pull_request, workflow_dispatch, etc.
github.ref                  # refs/heads/main, refs/tags/v1.0.0
github.ref_name             # main, v1.0.0 (branch/tag name only)
github.sha                  # Full commit SHA
github.head_ref             # PR source branch
github.base_ref             # PR target branch
github.actor                # User who triggered the workflow
github.repository           # owner/repo
github.repository_owner     # owner
github.workspace            # /home/runner/work/repo/repo
github.run_id               # Unique workflow run ID
github.run_number           # Sequential run number
github.run_attempt          # Retry attempt number
github.workflow             # Workflow name
github.job                  # Current job ID
github.token                # GITHUB_TOKEN value

# Event payload
github.event                # Full event payload object
github.event.pull_request.number
github.event.pull_request.title
github.event.pull_request.body
github.event.pull_request.head.sha
github.event.pull_request.base.ref
github.event.head_commit.message
github.event.inputs.my_input
```

### `env` Context

Environment variables:

```yaml
env.MY_VAR                  # Custom environment variable
env.CI                      # true
env.GITHUB_ACTIONS          # true
```

### `vars` Context

Repository/organization variables:

```yaml
vars.MY_VARIABLE            # Variable from settings
vars.ENVIRONMENT            # etc.
```

### `secrets` Context

Secret values:

```yaml
secrets.GITHUB_TOKEN        # Automatic token
secrets.MY_SECRET           # Custom secret
secrets.NPM_TOKEN
secrets.AWS_ACCESS_KEY_ID
```

### `job` Context

Current job info:

```yaml
job.status                  # success, failure, cancelled
job.container.id            # Container ID if using container
job.container.network       # Container network
job.services                # Service container info
```

### `jobs` Context

For reusable workflows:

```yaml
jobs.<job_id>.result        # success, failure, cancelled, skipped
jobs.<job_id>.outputs.<name>
```

### `steps` Context

Step outputs and status:

```yaml
steps.<step_id>.outputs.<output_name>
steps.<step_id>.outcome     # success, failure, cancelled, skipped
steps.<step_id>.conclusion  # Same as outcome after continue-on-error
```

### `needs` Context

Dependent job info:

```yaml
needs.<job_id>.result       # success, failure, cancelled, skipped
needs.<job_id>.outputs.<name>
```

### `matrix` Context

Matrix values:

```yaml
matrix.os                   # ubuntu-latest, macos-latest, etc.
matrix.node                 # 18, 20, 22, etc.
matrix.include.coverage     # Extra properties from include
```

### `strategy` Context

Strategy info:

```yaml
strategy.fail-fast          # true/false
strategy.job-index          # 0, 1, 2... (matrix position)
strategy.job-total          # Total jobs in matrix
strategy.max-parallel       # Parallel limit
```

### `runner` Context

Runner info:

```yaml
runner.name                 # Runner name
runner.os                   # Linux, Windows, macOS
runner.arch                 # X64, ARM64
runner.temp                 # Temp directory path
runner.tool_cache           # Tool cache path
runner.debug                # Debug mode enabled (1 or empty)
```

### `inputs` Context

Workflow dispatch/call inputs:

```yaml
inputs.environment          # Dispatch input value
inputs.debug                # Boolean input
```

## Operators

### Comparison

```yaml
==    # Equal (case-insensitive for strings)
!=    # Not equal
<     # Less than
<=    # Less than or equal
>     # Greater than
>=    # Greater than or equal
```

### Logical

```yaml
&&    # And
||    # Or
!     # Not
```

### Examples

```yaml
# Logical
${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
${{ github.event_name == 'push' || github.event_name == 'pull_request' }}
${{ !cancelled() }}

# Comparison
${{ github.run_attempt > 1 }}
${{ matrix.node >= 20 }}
```

## Functions

### String Functions

```yaml
# contains(search, item)
${{ contains(github.event.head_commit.message, '[skip ci]') }}
${{ contains(github.event.pull_request.labels.*.name, 'urgent') }}

# startsWith(searchString, searchValue)
${{ startsWith(github.ref, 'refs/tags/') }}
${{ startsWith(github.head_ref, 'feature/') }}

# endsWith(searchString, searchValue)
${{ endsWith(github.repository, '-private') }}

# format(string, replaceValue0, ...)
${{ format('Hello {0} {1}!', 'World', github.actor) }}
${{ format('{0}-{1}', runner.os, runner.arch) }}

# join(array, separator)
${{ join(matrix.*.node, ', ') }}
${{ join(github.event.pull_request.labels.*.name, ' ') }}
```

### JSON Functions

```yaml
# toJSON(value) - Convert to JSON string
${{ toJSON(github.event) }}
${{ toJSON(matrix) }}

# fromJSON(value) - Parse JSON string
strategy:
  matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}

# Example: dynamic matrix
jobs:
  setup:
    outputs:
      matrix: ${{ steps.set-matrix.outputs.result }}
    steps:
      - id: set-matrix
        run: |
          echo "result=[\"a\",\"b\",\"c\"]" >> $GITHUB_OUTPUT

  build:
    needs: setup
    strategy:
      matrix:
        value: ${{ fromJSON(needs.setup.outputs.matrix) }}
```

### Hash Functions

```yaml
# hashFiles(path, ...) - SHA-256 hash of files
${{ hashFiles('**/package-lock.json') }}
${{ hashFiles('**/Cargo.lock', '**/Cargo.toml') }}
${{ hashFiles('**/*.go', 'go.sum') }}

# Common cache key patterns
key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
key: rust-${{ runner.os }}-${{ hashFiles('**/Cargo.lock') }}
key: go-${{ runner.os }}-${{ hashFiles('**/go.sum') }}
```

### Status Check Functions

```yaml
# success() - All previous steps succeeded
if: success()

# failure() - Any previous step failed
if: failure()

# always() - Always run (even on cancel)
if: always()

# cancelled() - Workflow was cancelled
if: cancelled()

# Combinations
if: always() && !cancelled()
if: success() || failure()
if: failure() && steps.deploy.outcome == 'failure'
```

## Object Filters

### `*` Syntax

Filter arrays:

```yaml
# Get all label names
${{ github.event.pull_request.labels.*.name }}

# Check if specific label exists
${{ contains(github.event.pull_request.labels.*.name, 'bug') }}

# Matrix filter
${{ matrix.*.os }}
```

## Common Patterns

### Conditional on Branch

```yaml
if: github.ref == 'refs/heads/main'
if: github.ref_name == 'main'
if: startsWith(github.ref, 'refs/tags/')
```

### Conditional on Event

```yaml
if: github.event_name == 'push'
if: github.event_name == 'pull_request'
if: github.event_name == 'workflow_dispatch'
if: github.event_name == 'schedule'
```

### Conditional on Actor

```yaml
if: github.actor == 'dependabot[bot]'
if: github.actor != 'dependabot[bot]'
```

### Conditional on Labels

```yaml
if: contains(github.event.pull_request.labels.*.name, 'deploy')
if: "!contains(github.event.pull_request.labels.*.name, 'skip-ci')"
```

### Conditional on Commit Message

```yaml
if: "!contains(github.event.head_commit.message, '[skip ci]')"
if: contains(github.event.head_commit.message, '[deploy]')
```

### Conditional on Previous Steps

```yaml
if: steps.cache.outputs.cache-hit != 'true'
if: steps.check.outputs.changed == 'true'
if: steps.build.outcome == 'success'
```

### Conditional on Job Dependencies

```yaml
if: needs.build.result == 'success'
if: always() && needs.test.result == 'failure'
```

### Using Matrix Values

```yaml
if: matrix.os == 'ubuntu-latest'
if: matrix.node == 20 && matrix.coverage
```

### Environment Variables from Expressions

```yaml
env:
  DEPLOY_ENV: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
  VERSION: ${{ github.event.release.tag_name || github.sha }}
  IS_MAIN: ${{ github.ref == 'refs/heads/main' }}
```

### Ternary-like Pattern

```yaml
# condition && true_value || false_value
${{ github.event_name == 'push' && 'Push event' || 'Other event' }}

# Nested for multiple conditions
${{ github.ref == 'refs/heads/main' && 'production' ||
    github.ref == 'refs/heads/develop' && 'staging' ||
    'development' }}
```

## Escaping and Literals

### Literal Strings

```yaml
# Use single quotes for literals
if: github.event.head_commit.message == 'Release v1.0'

# Escape special characters
if: contains(github.event.head_commit.message, '''single''')
```

### Multi-line Expressions

```yaml
if: >-
  github.event_name == 'push' &&
  github.ref == 'refs/heads/main' &&
  !contains(github.event.head_commit.message, '[skip ci]')
```

## Type Coercion

GitHub Actions performs automatic type coercion:

| Type | Falsy | Truthy |
|------|-------|--------|
| null | null | N/A |
| boolean | false | true |
| number | 0 | any other |
| string | '' | any non-empty |
| array | N/A | always |
| object | N/A | always |

```yaml
# All falsy
if: ${{ null }}
if: ${{ false }}
if: ${{ 0 }}
if: ${{ '' }}

# All truthy
if: ${{ true }}
if: ${{ 1 }}
if: ${{ 'any string' }}
if: ${{ fromJSON('[]') }}
```
