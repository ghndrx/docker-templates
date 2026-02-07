# Node.js Docker Templates

Production-ready Dockerfile templates using [pnpm](https://pnpm.io/) - the fast, disk-efficient package manager.

## Templates

| File | Base Image | Size | Use Case |
|------|-----------|------|----------|
| `Dockerfile` | node:22-slim | ~200MB | Standard production |
| `Dockerfile.alpine` | node:22-alpine | ~130MB | Size optimized |
| `Dockerfile.distroless` | distroless/nodejs22 | ~110MB | Maximum security |

## Quick Start

```bash
# Copy template to your project
cp Dockerfile .dockerignore /path/to/your/project/

# Build
docker build -t myapp .

# Run
docker run -p 3000:3000 myapp
```

## Features

### 🚀 PNPM Package Manager
- **2x faster** than npm
- **Disk efficient** via content-addressable storage
- Strict dependency resolution
- Native workspace support

### 📦 Multi-Stage Build
```dockerfile
# Stage 1: Install dependencies
FROM node:22-slim AS deps
RUN pnpm install --frozen-lockfile --prod

# Stage 2: Build application
FROM node:22-slim AS builder
RUN pnpm build

# Stage 3: Minimal runtime
FROM node:22-slim AS runtime
COPY --from=builder /app/dist ./dist
```

### 🔒 Security Hardened
- Non-root user (`appuser:1000`)
- No devDependencies in production
- Health checks included
- Distroless option for zero CVE base

### ⚡ Build Cache Optimization
```dockerfile
# Cache pnpm store across builds
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile
```

## Project Setup

### Required: package.json

```json
{
  "name": "myapp",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "express": "^4.21.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/node": "^22.0.0"
  }
}
```

### Generate Lock File

```bash
# Install pnpm if needed
npm install -g pnpm
# Or via corepack
corepack enable && corepack prepare pnpm@latest --activate

# Create lock file
pnpm install
```

## Customization

### Change Node Version

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-slim AS deps
```

### Change Port

```dockerfile
EXPOSE 8080
HEALTHCHECK ... 'http://localhost:8080/health' ...
```

### Add Native Dependencies

For packages requiring native compilation (node-gyp):

```dockerfile
FROM node:22-slim AS deps

# Install build tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*
```

### Custom Entrypoint

```dockerfile
# With npm script
CMD ["pnpm", "start"]

# Direct node execution
CMD ["node", "--enable-source-maps", "dist/index.js"]

# With environment validation
CMD ["sh", "-c", "node dist/validate-env.js && node dist/index.js"]
```

### For Next.js / Nuxt / Other Frameworks

```dockerfile
# Copy standalone output (Next.js)
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
CMD ["node", "server.js"]
```

## Choosing the Right Template

| Template | Size | Security | Debug | Native Modules |
|----------|------|----------|-------|----------------|
| `Dockerfile` (slim) | ⭐⭐ | ⭐⭐ | ✅ Easy | ✅ Works |
| `Dockerfile.alpine` | ⭐⭐⭐ | ⭐⭐ | ✅ Easy | ⚠️ May need rebuild |
| `Dockerfile.distroless` | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ❌ Hard | ✅ Works |

### When to Use Each

- **slim**: Default choice, best compatibility
- **alpine**: Size-constrained environments, simple apps
- **distroless**: Production Kubernetes, compliance requirements

## Troubleshooting

### "Cannot find module" errors

Ensure your `build` script outputs to `dist/`:
```json
{
  "compilerOptions": {
    "outDir": "./dist"
  }
}
```

### pnpm-lock.yaml not found

Generate it locally first:
```bash
pnpm install
git add pnpm-lock.yaml
```

### Native module compilation fails on Alpine

Install build dependencies:
```dockerfile
RUN apk add --no-cache python3 make g++
```

### Health check failing

Ensure your app has a `/health` endpoint or adjust:
```dockerfile
HEALTHCHECK CMD node -e "process.exit(0)"
```

## Resources

- [pnpm Documentation](https://pnpm.io/)
- [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md)
- [Distroless Node.js](https://github.com/GoogleContainerTools/distroless/tree/main/nodejs)
- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
