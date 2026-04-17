# syntax=docker/dockerfile:1

# === Build stage ===
FROM node:22-alpine AS builder

WORKDIR /app

# Cache dependency installation
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund --omit=dev

# === Final stage ===
FROM node:22-alpine

RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -D appuser

WORKDIR /app

# Copy only production dependencies and source
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 4000

CMD ["node", "server.js"]
