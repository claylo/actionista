# Secrets Management

Securely manage sensitive data in GitHub Actions.

## Types of Secrets

| Type | Scope | Priority |
|------|-------|----------|
| Environment | Single environment | Highest |
| Repository | Single repo | Medium |
| Organization | All/selected repos | Lowest |

Higher priority secrets override lower.

## Using Secrets

```yaml
steps:
  - run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}
      TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Built-in Secrets

### GITHUB_TOKEN

Automatic token for repository access:

```yaml
- uses: actions/checkout@v6
  with:
    token: ${{ secrets.GITHUB_TOKEN }}

- run: gh pr create --title "Update"
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Permissions controlled via `permissions` key.

## Secret Best Practices

### Never Echo Secrets

```yaml
# BAD - leaks secret
- run: echo ${{ secrets.API_KEY }}

# GOOD - use environment variable
- run: ./script.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

### Mask Dynamic Secrets

```yaml
- run: |
    TOKEN=$(get-token)
    echo "::add-mask::$TOKEN"
    echo "token=$TOKEN" >> $GITHUB_OUTPUT
```

### Avoid Command Line

```yaml
# BAD - visible in process list
- run: curl -H "Authorization: ${{ secrets.TOKEN }}" ...

# GOOD - use environment variable
- run: curl -H "Authorization: $TOKEN" ...
  env:
    TOKEN: ${{ secrets.API_TOKEN }}
```

## OIDC (OpenID Connect)

Prefer OIDC over long-lived credentials.

### AWS

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/GitHubActions
          aws-region: us-east-1
```

AWS Setup:
1. Create OIDC identity provider in IAM
2. Create role with trust policy
3. No access keys needed!

### Google Cloud

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/PROJECT/locations/global/workloadIdentityPools/github/providers/github
          service_account: sa@project.iam.gserviceaccount.com
```

### Azure

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
```

## Environment Secrets

Scope secrets to deployment environments:

```yaml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}  # Staging key

  deploy-production:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}  # Production key
```

## Secret Rotation

### Automated Rotation

```yaml
name: Rotate Secrets

on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly

jobs:
  rotate:
    runs-on: ubuntu-latest
    steps:
      - name: Generate new secret
        id: generate
        run: |
          NEW_SECRET=$(openssl rand -hex 32)
          echo "::add-mask::$NEW_SECRET"
          echo "secret=$NEW_SECRET" >> $GITHUB_OUTPUT

      - name: Update secret
        env:
          GH_TOKEN: ${{ secrets.ADMIN_TOKEN }}
        run: |
          gh secret set API_KEY --body "${{ steps.generate.outputs.secret }}"
```

## Accessing Secrets in Different Contexts

### In Steps

```yaml
steps:
  - run: echo "Using secret"
    env:
      SECRET: ${{ secrets.MY_SECRET }}
```

### In Job Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:20
      env:
        API_KEY: ${{ secrets.API_KEY }}
```

### In Service Containers

```yaml
services:
  db:
    image: postgres
    env:
      POSTGRES_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

### In Matrix

```yaml
strategy:
  matrix:
    env: [staging, production]
steps:
  - run: ./deploy.sh
    env:
      # Same secret name, value depends on environment
      API_KEY: ${{ secrets[format('{0}_API_KEY', matrix.env)] }}
```

## Passing Secrets to Reusable Workflows

### Explicit

```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/deploy.yml
    secrets:
      api-key: ${{ secrets.API_KEY }}
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

### Inherit All

```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/deploy.yml
    secrets: inherit
```

## Secret Limitations

- Maximum 1000 secrets per repository
- Maximum 100 organization secrets
- Maximum 100 environment secrets
- Secret name max 100 characters
- Secret value max 64 KB

## Files as Secrets

For certificates, keys, etc.:

```yaml
- name: Setup certificate
  run: |
    echo "${{ secrets.CERTIFICATE }}" | base64 -d > cert.pem
    chmod 600 cert.pem
```

Store base64-encoded: `cat cert.pem | base64 | pbcopy`

## Multi-Line Secrets

```yaml
- name: Setup SSH key
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa
```

## Don't Commit These

Files that should NEVER be committed (use secrets instead):

- `.env` files
- `credentials.json`
- Private keys (`*.pem`, `*.key`)
- API tokens
- Database passwords
- Cloud credentials

## Security Checklist

- [ ] Use OIDC instead of long-lived credentials
- [ ] Scope secrets to environments
- [ ] Never echo or log secrets
- [ ] Mask dynamic secrets with `::add-mask::`
- [ ] Rotate secrets regularly
- [ ] Use environment secrets for production
- [ ] Audit secret access via audit log
- [ ] Review PR changes to workflows
