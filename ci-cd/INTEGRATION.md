# Integration Guide

Step-by-step guide to integrate these CI/CD templates into your Docker project.

## Prerequisites

- GitHub repository with a Dockerfile
- GitHub Actions enabled
- Permissions to create secrets and workflows

## Step 1: Copy Workflow

```bash
# In your project repository
mkdir -p .github/workflows
cp path/to/docker-templates/ci-cd/docker-build-push.yml .github/workflows/
```

## Step 2: Customize Workflow

Edit `.github/workflows/docker-build-push.yml`:

### Basic Customization

```yaml
env:
  REGISTRY: ghcr.io  # or docker.io for Docker Hub
  IMAGE_NAME: ${{ github.repository }}  # e.g., myorg/myapp

# If your Dockerfile is not at the root:
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: ./app
    file: ./app/Dockerfile.prod
```

### Multi-Registry Push

Uncomment and configure additional registry logins:

```yaml
- name: Log in to Docker Hub
  if: github.event_name != 'pull_request'
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

# Update tags in metadata step:
- name: Extract metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: |
      ghcr.io/${{ github.repository }}
      docker.io/myorg/myapp
```

### Fail on CVEs

To block deployment if critical vulnerabilities are found:

```yaml
- name: Docker Scout CVE scan
  uses: docker/scout-action@v1
  with:
    exit-code: true  # Fail the build
    only-severities: critical,high
```

## Step 3: Configure Repository Settings

### Enable Permissions

1. Go to **Settings** → **Actions** → **General**
2. Under "Workflow permissions", select:
   - ☑️ Read and write permissions
   - ☑️ Allow GitHub Actions to create and approve pull requests

### Add Secrets (if using Docker Hub)

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add:
   - `DOCKERHUB_USERNAME`: Your Docker Hub username
   - `DOCKERHUB_TOKEN`: Docker Hub access token (not password)

To create a Docker Hub token:
```bash
# Visit: https://hub.docker.com/settings/security
# Click "New Access Token"
# Copy the token (you won't see it again)
```

### Enable Docker Scout (optional)

1. Go to repository **Settings** → **Code security and analysis**
2. Enable **Docker Scout**
3. Scout will now analyze all pushed images

## Step 4: Test the Workflow

### Test with a PR (build only, no push)

```bash
git checkout -b test-ci
git add .github/workflows/docker-build-push.yml
git commit -m "ci: add Docker build workflow"
git push origin test-ci

# Create PR on GitHub
# Workflow will run but not push images
```

### Test with a Push (build and push)

```bash
git checkout main
git merge test-ci
git push origin main

# Workflow will:
# 1. Build multi-platform image
# 2. Push to GHCR
# 3. Generate and attach SBOM
# 4. Sign with Cosign
# 5. Scan for vulnerabilities
```

### Test with a Tag (versioned release)

```bash
git tag v1.0.0
git push origin v1.0.0

# Creates tags: 1.0.0, 1.0, 1, latest
```

## Step 5: Verify Results

### Check Image in GHCR

```bash
# View in GitHub
# Go to: https://github.com/myorg/myapp/pkgs/container/myapp

# Pull locally
docker pull ghcr.io/myorg/myapp:latest
```

### Verify Signature

```bash
# Install Cosign
brew install cosign  # macOS
# or: go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Verify signature
cosign verify ghcr.io/myorg/myapp:latest \
  --certificate-identity-regexp="https://github.com/myorg/myapp" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com"
```

### View SBOM

```bash
# View SBOM attestation
cosign verify-attestation ghcr.io/myorg/myapp:latest \
  --type cyclonedx \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  | jq -r '.payload' | base64 -d | jq '.predicate'
```

### Check Vulnerability Report

1. Go to **Security** → **Code scanning**
2. View Docker Scout results
3. Click on findings for remediation advice

## Step 6: CI/CD Badge (optional)

Add a badge to your README.md:

```markdown
[![Docker CI](https://github.com/myorg/myapp/actions/workflows/docker-build-push.yml/badge.svg)](https://github.com/myorg/myapp/actions/workflows/docker-build-push.yml)
```

## Common Customizations

### Build Args

Pass build-time variables:

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    build-args: |
      NODE_ENV=production
      API_URL=${{ secrets.API_URL }}
      VERSION=${{ github.ref_name }}
```

### Platform-Specific Builds

Build only for specific platforms:

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64  # Remove arm64 for faster builds
```

### Conditional Steps

Run steps only on main branch:

```yaml
- name: Production-only step
  if: github.ref == 'refs/heads/main'
  run: echo "Deploying to production..."
```

### Matrix Builds

Build multiple variants:

```yaml
jobs:
  build:
    strategy:
      matrix:
        variant: [alpine, debian, distroless]
    steps:
      - name: Build
        uses: docker/build-push-action@v6
        with:
          file: Dockerfile.${{ matrix.variant }}
          tags: myapp:${{ matrix.variant }}
```

## Troubleshooting

### "permission denied" when pushing

Ensure workflow permissions are set correctly:
- Settings → Actions → General → Workflow permissions → Read and write

### "rate limit exceeded" on Docker Hub

Add authentication even for public images:
- Add DOCKERHUB_USERNAME and DOCKERHUB_TOKEN secrets

### Build fails on ARM64

Add QEMU setup (already included in template):
```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3
```

### Cosign fails with "no matching signatures"

Verify OIDC token permissions:
```yaml
permissions:
  id-token: write  # Required for keyless signing
```

## Next Steps

- [ ] Enable Dependabot for automatic dependency updates
- [ ] Add deployment workflow (e.g., to Kubernetes)
- [ ] Configure branch protection rules requiring CI pass
- [ ] Set up notifications for failed builds
- [ ] Integrate with monitoring/observability platform

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Cosign Keyless Signing](https://github.com/sigstore/cosign)
- [Docker Scout CLI](https://docs.docker.com/scout/)
