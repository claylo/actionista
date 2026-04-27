# Troubleshooting Reference

Common GitHub Actions errors and solutions.

## Authentication Errors

### "Resource not accessible by integration"

**Cause**: GITHUB_TOKEN lacks required permissions.

**Solution**:
```yaml
permissions:
  contents: write       # For pushing
  pull-requests: write  # For PR comments
  issues: write         # For issue comments
```

### "Bad credentials"

**Cause**: Invalid or expired token.

**Solutions**:
- Check secret is set correctly in repository settings
- Verify secret name matches `${{ secrets.NAME }}`
- Regenerate token if expired

### "Permission denied (publickey)"

**Cause**: SSH key not configured for git operations.

**Solutions**:
```yaml
- uses: actions/checkout@v6
  with:
    ssh-key: ${{ secrets.DEPLOY_KEY }}

# Or use token
- uses: actions/checkout@v6
  with:
    token: ${{ secrets.PAT }}
```

## Workflow Trigger Failures

### Downstream workflow does not run after push/tag/release

**Cause**: The upstream workflow used `GITHUB_TOKEN` to push commits, create tags, or create releases. Events created by `GITHUB_TOKEN` **do not trigger other workflows**. This is a deliberate GitHub limitation to prevent infinite loops. There is no error and no log entry — the downstream workflow simply never fires.

**Symptoms**:
- Workflow A pushes to `main`, Workflow B has `on: push: branches: [main]` — B never runs
- Workflow A creates a release, Workflow B has `on: release: types: [published]` — B never runs
- Everything works when you push manually or via the GitHub UI

**Solutions**:

Use a Personal Access Token (PAT) or GitHub App token for the operation that needs to trigger downstream workflows:

```yaml
# Push with a PAT so downstream workflows fire
- name: Commit and push
  env:
    GH_TOKEN: ${{ secrets.MY_PAT }}
  run: |
    git remote set-url origin "https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository }}.git"
    git push

# Or create a release with a PAT
- name: Create release
  env:
    GH_TOKEN: ${{ secrets.MY_PAT }}
  run: |
    gh release create v1.0.0 --title "v1.0.0" --notes "Release notes"
```

**Alternatives**:
- Use `workflow_run` to chain workflows explicitly (but this requires the upstream workflow name, not the event)
- Use a GitHub App installation token (more scoped than a PAT)
- Use a deploy key with write access (for push-only scenarios)

**Important**: When using a PAT to trigger downstream workflows, add loop prevention to avoid infinite chains (e.g., `[skip ci]` in commit messages, `if: "!contains(...)"` conditions).

## Checkout Errors

### "Shallow clone detected"

**Cause**: Fetch depth is 1 (default).

**Solution**:
```yaml
- uses: actions/checkout@v6
  with:
    fetch-depth: 0  # Full history
```

### "Submodule not found"

**Solution**:
```yaml
- uses: actions/checkout@v6
  with:
    submodules: true          # Checkout submodules
    # or
    submodules: recursive     # Nested submodules
    token: ${{ secrets.PAT }} # If private submodules
```

### "Reference is not a tree"

**Cause**: Trying to checkout non-existent ref.

**Solutions**:
- Verify ref exists
- Use `fetch-depth: 0` for tags
- Check branch name spelling

## Cache Errors

### "Cache not found"

**Causes**:
- First run (no cache yet)
- Key doesn't match
- Cache expired (7 days for unused caches)

**Solutions**:
```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-  # Fallback to partial match
```

### "Cache size exceeds limit"

**Cause**: Cache exceeds 10 GB limit.

**Solutions**:
- Cache only essential directories
- Use more specific paths
- Clear old caches via API

## Artifact Errors

### "Artifact not found"

**Causes**:
- Artifact name mismatch
- Artifact from different workflow run
- Artifact expired

**Solution**:
```yaml
# Upload
- uses: actions/upload-artifact@v4
  with:
    name: my-artifact  # Must match

# Download
- uses: actions/download-artifact@v4
  with:
    name: my-artifact  # Exact same name
```

### "No files found"

**Cause**: Path pattern doesn't match any files.

**Solutions**:
```yaml
- uses: actions/upload-artifact@v4
  with:
    path: |
      dist/
      !dist/**/*.map
    if-no-files-found: error  # Fail if nothing found
```

## Docker Errors

### "Cannot connect to Docker daemon"

**Cause**: Docker not available or not started.

**Solutions**:
- Use Linux runner (`ubuntu-latest`)
- For macOS, Docker requires setup
- For Windows, use Windows containers or WSL

### "Image not found"

**Solutions**:
```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- run: docker pull ghcr.io/owner/image:tag
```

### "Rate limit exceeded"

**Cause**: Docker Hub rate limits.

**Solutions**:
```yaml
# Login to increase limits
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

# Or use ghcr.io instead
```

## Node.js Errors

### "npm ERR! code ERESOLVE"

**Cause**: Dependency conflict.

**Solutions**:
```yaml
- run: npm ci --legacy-peer-deps
# or
- run: npm ci --force
```

### "ENOENT: package-lock.json"

**Cause**: Lock file missing.

**Solutions**:
```yaml
# Use npm install instead of npm ci
- run: npm install

# Or commit package-lock.json
```

### "Node.js version not found"

**Solution**:
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'        # Use available version
    check-latest: true        # Get latest matching
```

## Python Errors

### "Python version not found"

**Solution**:
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    check-latest: true
```

### "pip: command not found"

**Solution**:
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
# Python action sets up pip
```

## Matrix Errors

### "Matrix value undefined"

**Cause**: Accessing matrix value outside matrix job.

**Solution**: Matrix values only available in the job with the matrix.

### "Matrix too large"

**Cause**: Exceeds 256 job limit.

**Solution**: Reduce combinations or use `include`/`exclude`.

## Conditional Errors

### "Unexpected value 'null'"

**Cause**: Expression returns null.

**Solution**:
```yaml
# Add default
if: ${{ inputs.value || 'default' }}

# Check for existence
if: ${{ inputs.value != '' }}
```

### "Unrecognized named-value"

**Cause**: Invalid context or typo.

**Solutions**:
- Check context name: `github`, `env`, `secrets`, `steps`
- Verify step ID matches
- Check for typos

## Output Errors

### "Output too large"

**Cause**: Exceeds 1 MB per output.

**Solutions**:
- Use artifacts for large data
- Compress before outputting
- Write to file and upload

### "set-output deprecated"

**Solution**:
```yaml
# Old (deprecated)
echo "::set-output name=version::1.0.0"

# New
echo "version=1.0.0" >> $GITHUB_OUTPUT
```

## Timeout Errors

### "The job running exceeded the maximum time"

**Cause**: Job exceeded 6 hours (default) or set timeout.

**Solutions**:
```yaml
jobs:
  build:
    timeout-minutes: 120  # Increase timeout
```

Or optimize job to run faster.

## Concurrency Errors

### "Skipped due to concurrency"

**Expected behavior**: Newer run cancelled older pending run.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # This causes skipping
```

## Workflow Errors

### "Workflow does not exist or is disabled"

**Causes**:
- Workflow file not on default branch
- Workflow disabled in settings
- YAML syntax error preventing load

**Solutions**:
- Merge workflow to default branch
- Enable in Actions settings
- Validate YAML syntax

### "Invalid workflow file"

**Solutions**:
- Validate YAML syntax
- Check required fields (`on`, `jobs`)
- Verify action versions exist

## Debugging

### Enable Debug Logging

Set repository secret `ACTIONS_STEP_DEBUG` to `true`.

Or re-run with debug:
1. Click "Re-run jobs"
2. Check "Enable debug logging"

### Print Debug Info

```yaml
- run: |
    echo "Event: ${{ github.event_name }}"
    echo "Ref: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Actor: ${{ github.actor }}"

- run: |
    echo "Runner OS: ${{ runner.os }}"
    echo "Runner Arch: ${{ runner.arch }}"

- run: env | sort  # All environment variables
```

### Check Available Disk

```yaml
- run: df -h
```

### Check Available Memory

```yaml
- run: free -h  # Linux
```

### List Installed Software

```yaml
- run: |
    node --version
    npm --version
    python --version
    docker --version
```

## Common Gotchas

### Expression in `if`

```yaml
# These are equivalent
if: github.ref == 'refs/heads/main'
if: ${{ github.ref == 'refs/heads/main' }}

# But these are NOT
if: success()       # Works
if: ${{ success() }} # Works
```

### String Comparison

```yaml
# Case-insensitive
if: github.actor == 'Username'  # Matches 'username' too

# Use contains for substring
if: contains(github.event.head_commit.message, '[skip ci]')
```

### Boolean Inputs

```yaml
# Inputs are strings!
inputs:
  debug:
    type: boolean

# Must compare correctly
if: inputs.debug == 'true'  # String comparison
if: inputs.debug == true    # Also works
if: ${{ inputs.debug }}     # Truthy check works
```

### Multiline Commands

```yaml
# Use | for multiline
- run: |
    echo "First"
    echo "Second"

# Don't mix styles
- run: echo "First" &&
       echo "Second"  # Can be fragile
```

### Quoting in YAML

```yaml
# These are different
run: echo "${{ github.sha }}"   # Variable expanded
run: 'echo "${{ github.sha }}"' # Still expanded (expressions always expand)

# Special characters need quotes
if: "!contains(message, '[skip]')"  # ! needs quotes
```
