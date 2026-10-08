# =============================================================================
# 0xSCADA Edge Dockerfile
# Issues #26, #30 — Docker container deployment for edge environments
# =============================================================================

# ---------------------------------------------------------------------------
# Stage 1: Build
# ---------------------------------------------------------------------------
# Node 22: its bundled npm 10.9+ reads the lockfile format current npm and
# Dependabot emit. Node 20 ships npm 10.8.2, which fails `npm ci` with EUSAGE
# (#598, #604). Guarded by test/ci/workflow-node-version.test.ts.
FROM node:22-alpine AS builder

WORKDIR /app

# Install build dependencies for native modules (node-opcua, etc.)
RUN apk add --no-cache python3 make g++

# Install dependencies first (Docker layer caching)
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts=false

# Copy source and build
# Same inputs as the root Dockerfile (tsc, rootDir ".", outDir "dist"); the old
# explicit list copied a drizzle.config.ts that no longer exists and broke the build.
COPY . .

# tsconfig is noEmit; the runnable server is the esbuild bundle (package.json `start`).
RUN npm run build:server

# Prune dev dependencies
RUN npm prune --production

# ---------------------------------------------------------------------------
# Stage 2: Production
# ---------------------------------------------------------------------------
FROM node:22-alpine

# Security: run as non-root
RUN addgroup -S scada && adduser -S scada -G scada

WORKDIR /app

# Copy built artifacts
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/client ./client
COPY --from=builder /app/contracts ./contracts

# Resource hint labels
LABEL org.opencontainers.image.title="0xSCADA Edge"
LABEL org.opencontainers.image.description="Industrial SCADA edge runtime"
LABEL org.opencontainers.image.source="https://github.com/NickFlach/0xSCADA"
LABEL com.0xscada.component="edge"
LABEL com.0xscada.min-memory="128Mi"
LABEL com.0xscada.min-cpu="250m"

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:5000/api/health || exit 1

# Default environment
ENV NODE_ENV=production \
    ENABLE_API_KEYS=true \
    PORT=5000 \
    LOG_LEVEL=warn \
    GATEWAY_MODE=edge

EXPOSE 5000

USER scada

CMD ["node", "dist/server/index.cjs"]
