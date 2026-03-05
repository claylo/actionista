---
name: review-workflow
description: Review and analyze GitHub Actions workflow files for issues, outdated actions, and improvements
argument-hint: "[path] [--all] [--fix] [--validate]"
allowed-tools: Read, Glob, Grep, Write, Edit
---

# Review Workflow Command

Analyze GitHub Actions workflow files for issues and improvements.

## Usage

```
/review-workflow                           # Review all workflows
/review-workflow .github/workflows/ci.yml  # Review specific file
/review-workflow --all                     # Review all workflows
/review-workflow --fix                     # Review and auto-fix safe issues
/review-workflow --validate                # Validate YAML syntax only
```

## Arguments

- `path` (optional): Specific workflow file to review
- `--all`: Review all workflows in `.github/workflows/`
- `--fix`: Automatically apply safe fixes (version updates, add caching)
- `--validate`: Only validate YAML syntax, don't analyze

## Instructions

When this command is invoked:

1. **Determine scope**
   - If path provided: analyze that specific file
   - If `--all` or no args: find all `.github/workflows/*.yml` files

2. **Validate YAML** (always, or `--validate` only)
   - Check YAML syntax is valid
   - Check required fields exist (`on`, `jobs`)
   - Report syntax errors

3. **If not `--validate` only, perform full analysis:**
   - Load the actionista skill for comprehensive analysis
   - Use the workflow-analyzer agent for detailed review
   - Generate findings report

4. **If `--fix` flag:**
   - Apply safe automatic fixes:
     - Update action versions to latest major
     - Add `cache:` to setup-* actions
     - Add `timeout-minutes` to jobs
     - Add `concurrency` block
   - Show diff of changes made
   - List changes that require manual review

5. **Present results**

   Format output as:

   ```
   ┌─────────────────────────────────────────────┐
   │  Actionista Workflow Review                  │
   └─────────────────────────────────────────────┘

   Analyzing: .github/workflows/ci.yml

   ## Summary
   - Jobs: 4
   - Steps: 15
   - Actions: 8
   - Issues: 3 (1 critical, 2 warnings)

   ## Issues

   🔴 CRITICAL: Outdated checkout action
   Location: jobs.build.steps[0]
   Current: actions/checkout@v3
   Latest: actions/checkout@v6

   🟠 WARNING: Missing dependency caching
   Location: jobs.build.steps[1]
   Add: cache: 'npm' to setup-node

   ## Visualization
   {ASCII flowchart of job dependencies}

   ## Recommended Actions
   1. Update actions/checkout from v3 to v6
   2. Add caching to setup-node step
   ```

## Examples

### Review Single Workflow
```
/review-workflow .github/workflows/ci.yml
```

### Review All and Auto-Fix
```
/review-workflow --all --fix
```

### Validate Only
```
/review-workflow --validate
```

## Notes

- The command uses the actionista skill's `index.json` for current action versions
- Critical issues (security, breaking changes) are highlighted first
- Auto-fix only applies safe, non-breaking changes
- Always review auto-fix changes before committing
