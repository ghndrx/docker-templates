# syntax=docker/dockerfile:1.7
#
# Node.js Multi-Stage Dockerfile with PNPM
# Features: Fast builds, minimal image, non-root, security hardened
#
# Build args:
#   NODE_VERSION - Node.js version (default: 22)
#
# Usage:
#   docker build -t myapp:latest .
#   docker run --rm -p 3000:3000 myapp:latest

# =============================================================================
# Stage 1: Install dependencies
# =============================================================================
ARG NODE_VERSION=22

FROM node:${NODE_VERSION}-slim AS deps

# Enable corepack for pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /app

# Copy dependency files only (layer caching)
COPY package.json pnpm-lock.yaml* ./

# Fetch dependencies to pnpm store
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm fetch --frozen-lockfile

# Install production dependencies only
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --prod

# =============================================================================
# Stage 2: Build application (if needed)
# =============================================================================
FROM node:${NODE_VERSION}-slim AS builder

RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /app

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY package.json pnpm-lock.yaml* ./

# Install all dependencies (including devDependencies for build)
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build application (TypeScript, bundler, etc.)
RUN pnpm build

# Prune dev dependencies after build
RUN pnpm prune --prod

# =============================================================================
# Stage 3: Production runtime
# =============================================================================
FROM node:${NODE_VERSION}-slim AS runtime

# Security: Create non-root user
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app

# Copy production dependencies
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules

# Copy built application
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

# Node.js production optimizations
ENV NODE_ENV=production \
    NODE_OPTIONS="--max-old-space-size=512"

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node -e "fetch('http://localhost:3000/health').then(r => r.ok ? process.exit(0) : process.exit(1)).catch(() => process.exit(1))"

# Start application
CMD ["node", "dist/index.js"]
