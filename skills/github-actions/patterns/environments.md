# Environments

Deployment environments with protection rules and secrets.

## Basic Environment

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: ./deploy.sh
```

## Environment with URL

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: ./deploy.sh
```

The URL appears in the GitHub UI and PR deployments.

## Dynamic URL

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - id: deploy
        run: |
          URL=$(./deploy.sh)
          echo "url=$URL" >> $GITHUB_OUTPUT
```

## Protection Rules

Configure in Settings → Environments:

### Required Reviewers

- Specify users/teams who must approve
- Deployment waits for approval
- Reviewers notified via GitHub

### Wait Timer

- Delay before deployment proceeds
- 0-43200 minutes (30 days)
- Gives time to cancel

### Branch Restrictions

- Limit which branches can deploy
- Patterns like `main`, `release/*`
- Protects production from feature branches

### Environment Secrets

- Secrets scoped to environment
- Only available in jobs targeting that environment
- Override repository secrets

## Environment Secrets

### Using Environment Secrets

```yaml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}  # Staging API key

  deploy-production:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}  # Production API key
```

Same secret name, different values per environment.

## Multi-Stage Deployment

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
      - run: ./deploy.sh production
```

## Approval Workflow

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - run: ./deploy.sh
```

With required reviewers configured:
1. Tests pass
2. Deployment job waits for approval
3. Reviewer approves in GitHub UI
4. Deployment proceeds

## Preview Environments

```yaml
name: Preview

on:
  pull_request:

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    environment:
      name: preview-${{ github.event.pull_request.number }}
      url: https://pr-${{ github.event.pull_request.number }}.preview.example.com
    steps:
      - uses: actions/checkout@v6
      - run: ./deploy-preview.sh ${{ github.event.pull_request.number }}
```

## Cleanup on PR Close

```yaml
name: Cleanup Preview

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - run: ./cleanup-preview.sh ${{ github.event.pull_request.number }}
```

## Environment Variables

### From Environment Configuration

Configure in Settings → Environments → Environment variables:

```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ vars.DEPLOY_URL }}"
```

### Combining with Secrets

```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_URL: ${{ vars.DEPLOY_URL }}       # Variable
          API_KEY: ${{ secrets.API_KEY }}          # Secret
```

## Concurrency with Environments

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: production-deploy
      cancel-in-progress: false  # Never cancel deployments
```

## OIDC with Environments

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/ProductionRole
          aws-region: us-east-1
```

OIDC token includes `environment` claim for fine-grained access control.

## Best Practices

### 1. Separate Environments

```yaml
# staging and production should be separate
environment: staging   # Different secrets, different rules
environment: production
```

### 2. Require Reviewers for Production

Always require approval for production deployments.

### 3. Use Branch Restrictions

Only allow `main` to deploy to production.

### 4. Limit Secret Scope

Use environment secrets, not repository secrets, for sensitive credentials.

### 5. Include URLs

```yaml
environment:
  name: production
  url: https://example.com  # Shows in GitHub UI
```

### 6. Clean Up Preview Environments

Always clean up when PR closes.

## Creating Environments

### Via GitHub UI

Settings → Environments → New environment

### Via GitHub API

```bash
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/repo/environments/production
```

### Via Terraform

```hcl
resource "github_repository_environment" "production" {
  repository  = github_repository.repo.name
  environment = "production"

  reviewers {
    users = [data.github_user.reviewer.id]
  }

  deployment_branch_policy {
    protected_branches     = true
    custom_branch_policies = false
  }
}
```

## Deployment Status

GitHub tracks deployments:

- **Pending**: Waiting for approval or timer
- **In progress**: Currently deploying
- **Success**: Deployment completed
- **Failure**: Deployment failed
- **Inactive**: Superseded by newer deployment

View in repository → Deployments.
