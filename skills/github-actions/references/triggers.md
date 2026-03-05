# Trigger Events Reference

Complete reference for GitHub Actions workflow trigger events.

## Repository Events

### `push`

Triggered when commits are pushed:

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    branches-ignore:
      - 'feature/**'
    tags:
      - 'v*'
    tags-ignore:
      - '*-beta'
    paths:
      - 'src/**'
      - '!src/**/*.md'
    paths-ignore:
      - 'docs/**'
      - '*.md'
```

### `pull_request`

Triggered on PR activity:

```yaml
on:
  pull_request:
    types:
      - opened
      - synchronize      # New commits pushed
      - reopened
      - closed
      - ready_for_review
      - labeled
      - unlabeled
      - edited           # Title/body changed
      - review_requested
      - review_request_removed
      - auto_merge_enabled
      - auto_merge_disabled
    branches:
      - main
      - 'release/**'
    paths:
      - 'src/**'
```

Default types: `opened`, `synchronize`, `reopened`

### `pull_request_target`

Like `pull_request` but runs in the context of the base branch (has write access to secrets):

```yaml
on:
  pull_request_target:
    types: [opened, synchronize]
    branches: [main]
```

**Security warning**: Be careful with `pull_request_target` — it has write access. Never run untrusted code from the PR.

### `pull_request_review`

Triggered on PR review activity:

```yaml
on:
  pull_request_review:
    types:
      - submitted
      - edited
      - dismissed
```

### `pull_request_review_comment`

Triggered on PR review comments:

```yaml
on:
  pull_request_review_comment:
    types:
      - created
      - edited
      - deleted
```

### `issue_comment`

Triggered on issue/PR comments:

```yaml
on:
  issue_comment:
    types:
      - created
      - edited
      - deleted
```

Note: Also triggers on PR comments since PRs are issues.

### `issues`

Triggered on issue activity:

```yaml
on:
  issues:
    types:
      - opened
      - edited
      - deleted
      - transferred
      - pinned
      - unpinned
      - closed
      - reopened
      - assigned
      - unassigned
      - labeled
      - unlabeled
      - milestoned
      - demilestoned
```

### `create`

Triggered when branch or tag is created:

```yaml
on:
  create:
    # No filters available - use if condition
```

### `delete`

Triggered when branch or tag is deleted:

```yaml
on:
  delete:
    # No filters available - use if condition
```

### `fork`

Triggered when repository is forked:

```yaml
on: fork
```

### `release`

Triggered on release activity:

```yaml
on:
  release:
    types:
      - published       # Most common for deployment
      - unpublished
      - created
      - edited
      - deleted
      - prereleased
      - released
```

### `watch`

Triggered when repository is starred:

```yaml
on:
  watch:
    types: [started]    # Only available type
```

## Workflow Events

### `workflow_dispatch`

Manual trigger with optional inputs:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

      version:
        description: 'Version to deploy'
        required: false
        type: string

      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean
        default: false

      log_level:
        description: 'Log level'
        required: true
        type: environment  # Select from environments
```

Input types: `string`, `boolean`, `choice`, `number`, `environment`

Access in workflow:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
      - run: echo "Debug mode: ${{ inputs.debug }}"
```

### `workflow_call`

Make workflow reusable:

```yaml
on:
  workflow_call:
    inputs:
      config-path:
        required: true
        type: string
      runner:
        required: false
        type: string
        default: 'ubuntu-latest'
    secrets:
      npm-token:
        required: true
      deploy-key:
        required: false
    outputs:
      artifact-url:
        description: "URL of the published artifact"
        value: ${{ jobs.build.outputs.artifact-url }}
```

Call from another workflow:
```yaml
jobs:
  call-workflow:
    uses: owner/repo/.github/workflows/reusable.yml@main
    with:
      config-path: ./config.json
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### `workflow_run`

Triggered when another workflow completes:

```yaml
on:
  workflow_run:
    workflows:
      - "Build"
      - "Test"
    types:
      - completed      # Only available type
    branches:
      - main
```

Access info about triggering workflow:
```yaml
jobs:
  on-success:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - run: echo "Workflow ${{ github.event.workflow_run.name }} succeeded"
```

## Scheduled Events

### `schedule`

Run on a schedule (cron syntax):

```yaml
on:
  schedule:
    - cron: '0 0 * * *'       # Daily at midnight UTC
    - cron: '0 */6 * * *'     # Every 6 hours
    - cron: '30 4 * * 1-5'    # 4:30 AM UTC weekdays
    - cron: '0 9 1 * *'       # 9 AM UTC first of month
```

Cron syntax: `minute hour day-of-month month day-of-week`

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of the week (0 - 6, 0 = Sunday)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

**Notes:**
- Scheduled workflows only run on the default branch
- Minimum interval is 5 minutes
- Times are in UTC
- May be delayed during high-load periods

## External Events

### `repository_dispatch`

Triggered via API:

```yaml
on:
  repository_dispatch:
    types:
      - webhook
      - custom-event
```

Trigger via API:
```bash
curl -X POST \
  -H "Accept: application/vnd.github.v3+json" \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/repo/dispatches \
  -d '{"event_type":"custom-event","client_payload":{"key":"value"}}'
```

Access payload:
```yaml
steps:
  - run: echo "${{ github.event.client_payload.key }}"
```

## Other Events

### `deployment`

```yaml
on:
  deployment:
```

### `deployment_status`

```yaml
on:
  deployment_status:
```

### `discussion`

```yaml
on:
  discussion:
    types:
      - created
      - edited
      - deleted
      - pinned
      - unpinned
      - labeled
      - unlabeled
      - category_changed
      - answered
      - unanswered
```

### `discussion_comment`

```yaml
on:
  discussion_comment:
    types:
      - created
      - edited
      - deleted
```

### `label`

```yaml
on:
  label:
    types:
      - created
      - edited
      - deleted
```

### `milestone`

```yaml
on:
  milestone:
    types:
      - created
      - closed
      - opened
      - edited
      - deleted
```

### `page_build`

```yaml
on: page_build
```

### `project`

```yaml
on:
  project:
    types:
      - created
      - closed
      - reopened
      - edited
      - deleted
```

### `project_card`

```yaml
on:
  project_card:
    types:
      - created
      - moved
      - converted
      - edited
      - deleted
```

### `project_column`

```yaml
on:
  project_column:
    types:
      - created
      - updated
      - moved
      - deleted
```

### `public`

Triggered when repository visibility changes to public:

```yaml
on: public
```

### `registry_package`

```yaml
on:
  registry_package:
    types:
      - published
      - updated
```

### `status`

Triggered on commit status changes:

```yaml
on: status
```

### `check_run`

```yaml
on:
  check_run:
    types:
      - created
      - rerequested
      - completed
      - requested_action
```

### `check_suite`

```yaml
on:
  check_suite:
    types:
      - completed
```

## Filtering Patterns

### Branch/Tag Patterns

```yaml
branches:
  - main                    # Exact match
  - 'release/*'             # Single level: release/v1
  - 'release/**'            # Multi level: release/v1/patch
  - '!release/**-alpha'     # Exclude alpha releases
  - 'feature/[a-z]*'        # Character ranges

tags:
  - 'v*'                    # All version tags
  - 'v[0-9]+.[0-9]+.[0-9]+' # Semantic versions
  - '!*-beta'               # Exclude beta tags
```

### Path Patterns

```yaml
paths:
  - 'src/**'                # All files in src/
  - '*.js'                  # JS files in root
  - '**/*.ts'               # TS files anywhere
  - 'docs/**/*.md'          # Markdown in docs/
  - '!**/*.test.js'         # Exclude test files
  - '!docs/**'              # Exclude docs/
```

## Common Patterns

### CI on Push and PR

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

### Deploy on Release

```yaml
on:
  release:
    types: [published]
```

### Nightly Build

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:  # Allow manual trigger too
```

### PR Preview

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'docs/**'
      - 'src/**'
```

### Dependent Workflows

```yaml
# deploy.yml
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: github.event.workflow_run.conclusion == 'success'
```
