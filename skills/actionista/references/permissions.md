# Permissions Reference

Complete reference for GITHUB_TOKEN permissions and OIDC.

## GITHUB_TOKEN

Automatic token provided to every workflow run with repository access.

### Default Permissions

For **public repositories**: Read-only for all scopes

For **private repositories**: Configurable in Settings → Actions → General

### Permission Scopes

| Scope | Read | Write | Description |
|-------|------|-------|-------------|
| `actions` | ✓ | ✓ | Manage Actions artifacts and caches |
| `checks` | ✓ | ✓ | Create/update check runs |
| `contents` | ✓ | ✓ | Repository contents, commits, branches |
| `deployments` | ✓ | ✓ | Deployment environments |
| `discussions` | ✓ | ✓ | GitHub Discussions |
| `id-token` | — | ✓ | Request OIDC token |
| `issues` | ✓ | ✓ | Issues and comments |
| `packages` | ✓ | ✓ | GitHub Packages |
| `pages` | ✓ | ✓ | GitHub Pages |
| `pull-requests` | ✓ | ✓ | PRs and comments |
| `repository-projects` | ✓ | ✓ | Projects (classic) |
| `security-events` | ✓ | ✓ | Code scanning alerts |
| `statuses` | ✓ | ✓ | Commit statuses |

### Setting Permissions

#### Workflow Level

```yaml
permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
```

#### Job Level

```yaml
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read

  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # For OIDC
```

### Permission Values

```yaml
permissions:
  contents: read    # Read-only access
  contents: write   # Read and write access
  contents: none    # No access
```

### Shorthand Permissions

```yaml
permissions: read-all   # Read access to all scopes
permissions: write-all  # Write access to all scopes (avoid!)
permissions: {}         # No permissions
```

## Principle of Least Privilege

Always request minimal permissions:

```yaml
# Bad: Too permissive
permissions: write-all

# Good: Only what's needed
permissions:
  contents: read
  packages: write
```

### Common Permission Sets

#### CI/Build

```yaml
permissions:
  contents: read
```

#### Publish Package

```yaml
permissions:
  contents: read
  packages: write
```

#### Create Release

```yaml
permissions:
  contents: write
```

#### Update PR

```yaml
permissions:
  contents: read
  pull-requests: write
```

#### Deploy with OIDC

```yaml
permissions:
  contents: read
  id-token: write
```

#### Code Scanning

```yaml
permissions:
  contents: read
  security-events: write
```

#### GitHub Pages

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

## OIDC (OpenID Connect)

Use OIDC for cloud authentication without long-lived secrets.

### How It Works

1. Workflow requests OIDC token from GitHub
2. Token includes claims about workflow (repo, branch, actor)
3. Cloud provider verifies token with GitHub
4. Cloud provider issues short-lived credentials

### Enable OIDC

```yaml
permissions:
  id-token: write   # Required for OIDC
  contents: read
```

### AWS with OIDC

#### AWS Setup

1. Create OIDC identity provider in IAM
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. Create IAM role with trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:org/repo:*"
        }
      }
    }
  ]
}
```

#### Workflow Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/GitHubActionsRole
          aws-region: us-east-1

      - run: aws s3 ls
```

### Google Cloud with OIDC

#### GCP Setup

1. Create Workload Identity Pool
2. Add GitHub as identity provider
3. Create service account and grant access

```bash
# Create workload identity pool
gcloud iam workload-identity-pools create "github-pool" \
  --project="PROJECT_ID" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Add GitHub provider
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Grant service account access
gcloud iam service-accounts add-iam-policy-binding "SA@PROJECT.iam.gserviceaccount.com" \
  --project="PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/org/repo"
```

#### Workflow Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'SA@PROJECT.iam.gserviceaccount.com'

      - uses: google-github-actions/setup-gcloud@v2

      - run: gcloud compute instances list
```

### Azure with OIDC

#### Azure Setup

1. Create app registration
2. Add federated credential for GitHub

```bash
# Create app registration
az ad app create --display-name "github-actions"

# Get app ID
APP_ID=$(az ad app list --display-name "github-actions" --query '[0].appId' -o tsv)

# Create federated credential
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:org/repo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

#### Workflow Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - run: az account show
```

## OIDC Token Claims

The OIDC token includes these claims:

| Claim | Description | Example |
|-------|-------------|---------|
| `iss` | Issuer | `https://token.actions.githubusercontent.com` |
| `sub` | Subject | `repo:org/repo:ref:refs/heads/main` |
| `aud` | Audience | `sts.amazonaws.com` |
| `ref` | Git ref | `refs/heads/main` |
| `sha` | Commit SHA | `abc123...` |
| `repository` | Repository | `org/repo` |
| `repository_owner` | Owner | `org` |
| `actor` | Actor | `username` |
| `event_name` | Event | `push` |
| `workflow` | Workflow name | `Deploy` |
| `job_workflow_ref` | Workflow ref | `org/repo/.github/workflows/deploy.yml@refs/heads/main` |
| `environment` | Environment | `production` |

Use these for fine-grained access control in trust policies.

## Environment Protection

Combine with environments for deployment gates:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/ProductionRole
          aws-region: us-east-1
```

Environment protection rules:
- Required reviewers
- Wait timer
- Deployment branches
- Environment secrets

## Best Practices

### Always Declare Permissions

```yaml
# Explicit permissions are clearer and more secure
permissions:
  contents: read
```

### Use OIDC for Cloud Access

```yaml
# Avoid long-lived credentials
permissions:
  id-token: write  # Enable OIDC

# Instead of
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_KEY }}  # Avoid this
```

### Scope by Environment

```yaml
# Different permissions for different environments
jobs:
  staging:
    environment: staging
    permissions:
      id-token: write

  production:
    environment: production
    needs: staging
    permissions:
      id-token: write
```

### Fine-Grained Trust Policies

```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:org/repo:environment:production"
    }
  }
}
```
