# CI/CD Templates

Modern CI/CD workflows for Docker image builds with security scanning, SBOM generation, signing, and multi-platform support.

## Features

- ✅ **Multi-platform builds** (linux/amd64, linux/arm64)
- ✅ **SBOM generation** (CycloneDX format)
- ✅ **Provenance attestation** (SLSA compliance)
- ✅ **Image signing** (Cosign keyless signing via OIDC)
- ✅ **Vulnerability scanning** (Docker Scout integration)
- ✅ **Cache optimization** (GitHub Actions cache, registry cache)
- ✅ **Digest pinning** (immutable image references)
- ✅ **Multi-registry push** (GHCR, Docker Hub, ECR)

## Quick Start

### 1. Copy Workflow Template

```bash
# Copy to your repository
cp ci-cd/docker-build-push.yml .github/workflows/

# Customize for your project:
# - IMAGE_NAME
# - REGISTRY
# - DOCKERFILE path
```

### 2. Configure Secrets

Required GitHub repository secrets:

```bash
# For Docker Hub
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN

# For GHCR (auto-configured via ${{ secrets.GITHUB_TOKEN }})
# No additional setup required

# For AWS ECR
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
```

### 3. Enable Permissions

In `.github/workflows/docker-build-push.yml`, ensure:

```yaml
permissions:
  contents: read
  packages: write  # For GHCR push
  id-token: write  # For Cosign keyless signing
  attestations: write  # For provenance
  security-events: write  # For Scout results
```

## Workflow Overview

### docker-build-push.yml

Comprehensive production-ready workflow with:

1. **Metadata extraction** - tags, labels, annotations
2. **Multi-platform build** - QEMU emulation for cross-arch
3. **Layer caching** - GitHub Actions cache + registry cache
4. **SBOM generation** - Attached as attestation
5. **Provenance** - Build metadata attestation
6. **Image signing** - Cosign with GitHub OIDC
7. **Security scanning** - Docker Scout CVE analysis
8. **Multi-registry push** - Parallel push to registries

### Triggers

```yaml
on:
  push:
    branches: [main, develop]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger
```

### Image Tagging Strategy

| Event | Tags Generated |
|-------|---------------|
| `main` branch push | `latest`, `edge`, `sha-<short-sha>` |
| Tag `v1.2.3` push | `1.2.3`, `1.2`, `1`, `latest` |
| PR | `pr-<number>` (cached build, no push) |
| Branch push | `branch-<name>`, `sha-<short-sha>` |

## Advanced Usage

### Multi-Registry Push

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    tags: |
      ghcr.io/${{ github.repository }}:latest
      docker.io/myorg/myapp:latest
      123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### Custom Build Args

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    build-args: |
      VERSION=${{ github.ref_name }}
      BUILD_DATE=${{ steps.prep.outputs.date }}
      GIT_COMMIT=${{ github.sha }}
```

### Conditional Signing (production only)

```yaml
- name: Sign image
  if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
  run: cosign sign --yes ${{ env.IMAGE }}@${{ steps.build.outputs.digest }}
```

## Verification

### Verify Signature

```bash
# Using Cosign
cosign verify ghcr.io/myorg/myapp:latest \
  --certificate-identity-regexp="https://github.com/myorg/myapp" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com"
```

### Verify Attestations

```bash
# View all attestations
cosign verify-attestation ghcr.io/myorg/myapp:latest \
  --certificate-identity-regexp="https://github.com/myorg/myapp" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  --type=https://cyclonedx.org/schema | jq

# Verify SBOM attestation
cosign verify-attestation ghcr.io/myorg/myapp:latest \
  --type cyclonedx \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com"

# Verify provenance attestation
cosign verify-attestation ghcr.io/myorg/myapp:latest \
  --type slsaprovenance \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com"
```

### Check Vulnerability Scan

```bash
# Using Docker Scout CLI
docker scout cves ghcr.io/myorg/myapp:latest

# View recommendations
docker scout recommendations ghcr.io/myorg/myapp:latest

# Compare with base image
docker scout compare --to alpine:latest ghcr.io/myorg/myapp:latest
```

## Security Best Practices

### 1. Pin Actions by SHA

```yaml
# ❌ Avoid
uses: docker/build-push-action@v6

# ✅ Prefer
uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbbc5dbf3a673a178caa  # v6.0.0
```

### 2. Use Digest References in Dockerfile

```dockerfile
# ❌ Avoid
FROM node:20-alpine

# ✅ Prefer
FROM node:20-alpine@sha256:2d5e8a8a51bc341fd5f2eed6d91455c3a3d147e91a14298fc564b5dc519c1666
```

### 3. Scan in Pull Requests

```yaml
- name: Build for scanning
  uses: docker/build-push-action@v6
  with:
    load: true  # Load into local Docker
    push: false

- name: Scout scan
  uses: docker/scout-action@v1
  with:
    command: cves
    only-severities: critical,high
    exit-code: true  # Fail on critical/high CVEs
```

### 4. Minimal Permissions

Use `permissions:` to follow least-privilege:

```yaml
permissions:
  contents: read      # Read code
  packages: write     # Push to GHCR
  id-token: write     # OIDC signing
  attestations: write # Attach attestations
```

## Troubleshooting

### Build fails on ARM64

```yaml
# Add explicit platform specification
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3
  with:
    platforms: linux/arm64

- name: Build
  uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64,linux/arm64
```

### Cosign signing fails

```yaml
# Ensure id-token permission is set
permissions:
  id-token: write

# Use --yes flag to avoid interactive prompt
- run: cosign sign --yes $IMAGE@$DIGEST
```

### Scout scan not appearing

```yaml
# Enable security-events write permission
permissions:
  security-events: write

# Ensure Scout is enabled on your registry
# For GHCR: Settings → Code security and analysis → Docker Scout
```

## References

- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Cosign Keyless Signing](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)
- [Docker Scout](https://docs.docker.com/scout/)
- [SLSA Provenance](https://slsa.dev/spec/v1.0/provenance)
- [CycloneDX SBOM](https://cyclonedx.org/)
- [GitHub Actions Security](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

## License

MIT
