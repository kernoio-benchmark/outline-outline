# Outline Dockerfile - Production Build
ARG APP_PATH=/opt/outline

# Build stage - dependencies and build
FROM node:22.21.0 AS deps

ARG APP_PATH
WORKDIR $APP_PATH

# Install system dependencies
RUN apt-get update && apt-get install -y cmake && rm -rf /var/lib/apt/lists/*

# Copy dependency manifests first for better caching
COPY package.json yarn.lock .yarnrc.yml ./
COPY patches ./patches

# Configure Node.js memory and enable Corepack
ENV NODE_OPTIONS="--max-old-space-size=24000"
RUN corepack enable

# Install dependencies with frozen lockfile
RUN yarn install --immutable --network-timeout 1000000 && \
    yarn cache clean

# Copy source code
COPY . .

# Build the application
ARG CDN_URL
RUN yarn build

# Remove dev dependencies for production
RUN yarn workspaces focus --production && \
    yarn cache clean

# Runtime stage
FROM node:22.21.0-slim AS runner

LABEL org.opencontainers.image.source="https://github.com/outline/outline"

ARG APP_PATH
WORKDIR $APP_PATH

ENV NODE_ENV=production
ENV PORT=3000

# Create non-root user compatible with Debian and BusyBox
RUN addgroup --gid 1001 nodejs && \
    adduser --uid 1001 --ingroup nodejs nodejs && \
    mkdir -p /var/lib/outline && \
    chown -R nodejs:nodejs /var/lib/outline && \
    chown -R nodejs:nodejs $APP_PATH

# Install wget and curl for healthchecks
RUN apt-get update && \
    apt-get install -y --no-install-recommends wget curl && \
    rm -rf /var/lib/apt/lists/*

# Copy built application from deps stage with proper ownership
COPY --from=deps --chown=nodejs:nodejs $APP_PATH/build ./build
COPY --from=deps --chown=nodejs:nodejs $APP_PATH/server ./server
COPY --from=deps --chown=nodejs:nodejs $APP_PATH/public ./public
COPY --from=deps --chown=nodejs:nodejs $APP_PATH/.sequelizerc ./.sequelizerc
COPY --from=deps --chown=nodejs:nodejs $APP_PATH/node_modules ./node_modules
COPY --from=deps --chown=nodejs:nodejs $APP_PATH/package.json ./package.json

# Set up local file storage directory
ENV FILE_STORAGE_LOCAL_ROOT_DIR=/var/lib/outline/data
RUN mkdir -p "$FILE_STORAGE_LOCAL_ROOT_DIR" && \
    chown -R nodejs:nodejs "$FILE_STORAGE_LOCAL_ROOT_DIR" && \
    chmod 1777 "$FILE_STORAGE_LOCAL_ROOT_DIR"

VOLUME /var/lib/outline/data

# Switch to non-root user
USER nodejs

# Healthcheck
HEALTHCHECK --interval=1m CMD wget -qO- "http://localhost:${PORT:-3000}/_health" | grep -q "OK" || exit 1

EXPOSE 3000

CMD ["node", "build/server/index.js"]
