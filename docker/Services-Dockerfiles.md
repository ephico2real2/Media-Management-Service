# Service Dockerfiles

## Base Dockerfile

First, let's create a base Dockerfile that contains common dependencies. This will be used as the foundation for our service-specific Dockerfiles.

```dockerfile
# docker/base/Dockerfile
FROM node:16-alpine

# Install dependencies required by all services
RUN apk add --no-cache \
    curl \
    ca-certificates \
    bash \
    tzdata

# Set working directory
WORKDIR /app

# Set timezone
ENV TZ=UTC

# Create node user for security
RUN addgroup -g 1000 node && \
    adduser -u 1000 -G node -s /bin/sh -D node && \
    mkdir -p /app/node_modules && \
    chown -R node:node /app

# Switch to non-root user
USER node

# Set default command
CMD ["node", "src/index.js"]
```

## Upload Manager Dockerfile

```dockerfile
# upload-manager/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=upload-manager

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Create temp directories
RUN mkdir -p /app/temp/uploads && \
    chown -R node:node /app/temp

# Expose service port
EXPOSE 3001

# Start the service
CMD ["node", "src/index.js"]
```

## Processing Pipeline Dockerfile

```dockerfile
# processing-pipeline/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=processing-pipeline

# Install FFmpeg and additional dependencies
USER root
RUN apk add --no-cache \
    ffmpeg \
    python3

# Switch back to node user
USER node

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Create temp processing directory
RUN mkdir -p /app/temp/processing && \
    chown -R node:node /app/temp

# Expose service port (if any)
# EXPOSE 3000

# Start the service
CMD ["node", "src/index.js"]
```

## Streaming Engine Dockerfile

```dockerfile
# streaming-engine/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=streaming-engine

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Expose service port
EXPOSE 3002

# Start the service
CMD ["node", "src/index.js"]
```

## Content Management Dockerfile

```dockerfile
# content-management/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=content-management

# Install MongoDB Shell
USER root
RUN apk add --no-cache mongodb-tools

# Switch back to node user
USER node

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Expose service port
EXPOSE 3003

# Start the service
CMD ["node", "src/index.js"]
```

## API Gateway Dockerfile

```dockerfile
# api-gateway/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=api-gateway

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Expose service port
EXPOSE 3000

# Start the service
CMD ["node", "src/index.js"]
```

## Analytics Engine Dockerfile

```dockerfile
# analytics-engine/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=analytics-engine

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Expose service port
EXPOSE 3004

# Start the service
CMD ["node", "src/index.js"]
```

## Status Checker Dockerfile

```dockerfile
# status-checker/Dockerfile
FROM media-management-service/base:latest

# Set service name for logs and metrics
ENV SERVICE_NAME=status-checker

# Copy package.json and package-lock.json
COPY --chown=node:node package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY --chown=node:node src ./src
COPY --chown=node:node .env* ./

# Expose service port
EXPOSE 3006

# Start the service
CMD ["node", "src/index.js"]
```

## NGINX Dockerfile (for API Gateway alternative)

```dockerfile
# docker/nginx/Dockerfile
FROM nginx:1.23-alpine

# Install curl for healthcheck
RUN apk add --no-cache curl

# Remove default configuration
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom configuration
COPY config/nginx/nginx.conf /etc/nginx/conf.d/

# Expose ports
EXPOSE 80

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1
```
