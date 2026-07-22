# ========================================
# Base Stage: Alpine Linux with Bun
# ========================================
FROM node:22-alpine AS base

# ========================================
# Dependencies Stage: Install Dependencies
# ========================================
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

ENV PNPM_HOME="/root/.local/share/pnpm"
ENV PATH="$PNPM_HOME:$PATH"

# Install turbo globally
RUN npm install -g pnpm@10.15.0
RUN pnpm install -g turbo

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml turbo.json ./
RUN mkdir -p apps/mail apps/server packages
COPY apps/mail/package.json ./apps/mail/
COPY apps/server/package.json ./apps/server/
COPY packages/ ./packages/
COPY patches/ ./patches/

RUN pnpm install --prod --ignore-scripts


# ========================================
# Builder Stage: Build the Application
# ========================================
FROM base AS builder
WORKDIR /app

RUN npm install -g pnpm@10.15.0 wrangler

# Copy workspace configuration and source code
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml turbo.json ./
COPY . .

ENV CI=true

# Install all dependencies (including devDependencies for build)
RUN pnpm install --frozen-lockfile

# Set environment variables for build
ENV NEXT_TELEMETRY_DISABLED=1 \
    NODE_ENV=production \
    DOCKER_BUILD=true

# Build the application
RUN cd apps/mail && pnpm run build

# ========================================
# Runner Stage: Production Environment
# ========================================
# apps/mail builds with ssr:false (react-router.config.ts) — it's a static
# SPA meant for Cloudflare Pages/Workers, not a Node/Bun server. There is no
# server.js to run, so we serve the prerendered build/client/ output directly.
FROM base AS runner
WORKDIR /app

RUN npm install -g serve

RUN addgroup -S -g 1001 pnpm && \
    adduser -S -u 1001 nextjs -G pnpm

ENV NODE_ENV=production \
    PORT=3000 \
    HOSTNAME="0.0.0.0"

# Copy entrypoint and run it
COPY scripts/docker/ /app/scripts
RUN chmod -R +x /app/scripts/*

# Copy only the static build output, not the whole monorepo/node_modules
COPY --from=builder --chown=nextjs:pnpm /app/apps/mail/build/client ./apps/mail/build/client

# Switch to non-root user
USER nextjs

# Set server port and host
EXPOSE 3000

# Start the server
CMD ["/app/scripts/entrypoint.sh"]