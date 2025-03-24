# Docker Compose Setup for Media Management Service

This guide provides a complete Docker Compose setup for deploying the Media Management Service as a containerized microservices architecture.

## Folder Structure

Create the following folder structure for your Dockerized application:

```
media-management-service/
├── .env                                # Global environment variables
├── docker-compose.yml                  # Main docker-compose file
├── docker-compose.dev.yml              # Development overrides
├── docker-compose.prod.yml             # Production overrides
│
├── api-gateway/                        # API Gateway service
│   ├── Dockerfile
│   ├── package.json
│   ├── .dockerignore
│   └── src/
│
├── upload-manager/                     # Upload Manager service
│   ├── Dockerfile
│   ├── package.json
│   ├── .dockerignore
│   └── src/
│
├── processing-pipeline/                # Processing Pipeline service
│   ├── Dockerfile
│   ├── package.json
│   ├── .dockerignore
│   └── src/
│
├── streaming-engine/                   # Streaming Engine service
│   ├── Dockerfile
│   ├── package.json
│   ├── .dockerignore
│   └── src/
│
├── content-management/                 # Content Management service
│   ├── Dockerfile
│   ├── package.json
│   ├── .dockerignore
│   └── src/
│
├── status-checker/                     # Status Checker service (health monitoring)
│   ├── Dockerfile
│   ├── package.json
│   ├── .dockerignore
│   └── src/
│
├── shared/                             # Shared code (mounted as volume to all services)
│   ├── logger.js
│   ├── s3-client.js
│   ├── db-client.js
│   ├── redis-client.js
│   ├── rabbitmq-client.js
│   ├── metrics.js
│   └── utils.js
│
├── config/                             # Configuration files
│   ├── nginx/                          # Nginx configuration for API Gateway
│   │   └── nginx.conf
│   ├── profiles/                       # Transcoding profiles
│   │   ├── 1080p.json
│   │   ├── 720p.json
│   │   ├── 480p.json
│   │   └── 360p.json
│   ├── minio/                          # MinIO configuration
│   └── monitoring/                     # Monitoring configuration
│
├── env/                                # Environment files for each service
│   ├── api-gateway.env
│   ├── upload-manager.env
│   ├── processing-pipeline.env
│   ├── streaming-engine.env
│   ├── content-management.env
│   └── status-checker.env
│
├── scripts/                            # Utility scripts
│   ├── setup.sh                        # Setup script
│   ├── generate-token.js               # JWT token generator
│   └── cleanup-temp.js                 # Temporary file cleanup
│
└── docs/                               # Documentation
    ├── api.md                          # API documentation
    ├── deployment.md                   # Deployment guide
    └── architecture.md                 # Architecture overview
```

## Docker Compose Configuration

Create a `docker-compose.yml` file at the root of your project:

```yaml
version: '3.8'

services:
  # API Gateway Service
  api-gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    env_file:
      - ./env/api-gateway.env
    depends_on:
      - mongodb
      - redis
      - upload-manager
      - streaming-engine
      - content-management
    volumes:
      - ./shared:/app/shared
      - api-gateway-logs:/app/logs
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped

  # Upload Manager Service
  upload-manager:
    build:
      context: ./upload-manager
      dockerfile: Dockerfile
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
    env_file:
      - ./env/upload-manager.env
    depends_on:
      - mongodb
      - redis
      - rabbitmq
      - minio
    volumes:
      - ./shared:/app/shared
      - upload-manager-logs:/app/logs
      - upload-temp:/app/temp/uploads
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped

  # Processing Pipeline Service
  processing-pipeline:
    build:
      context: ./processing-pipeline
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=production
    env_file:
      - ./env/processing-pipeline.env
    depends_on:
      - mongodb
      - redis
      - rabbitmq
      - minio
    volumes:
      - ./shared:/app/shared
      - processing-pipeline-logs:/app/logs
      - processing-temp:/app/temp/processing
      - ./config/profiles:/app/config/profiles
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3010/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped
    deploy:
      replicas: 2

  # Streaming Engine Service
  streaming-engine:
    build:
      context: ./streaming-engine
      dockerfile: Dockerfile
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=production
    env_file:
      - ./env/streaming-engine.env
    depends_on:
      - mongodb
      - redis
      - minio
    volumes:
      - ./shared:/app/shared
      - streaming-engine-logs:/app/logs
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped

  # Content Management Service
  content-management:
    build:
      context: ./content-management
      dockerfile: Dockerfile
    ports:
      - "3003:3003"
    environment:
      - NODE_ENV=production
    env_file:
      - ./env/content-management.env
    depends_on:
      - mongodb
      - redis
      - minio
    volumes:
      - ./shared:/app/shared
      - content-management-logs:/app/logs
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3003/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped

  # Status Checker Service
  status-checker:
    build:
      context: ./status-checker
      dockerfile: Dockerfile
    ports:
      - "3006:3006"
    environment:
      - NODE_ENV=production
    env_file:
      - ./env/status-checker.env
    depends_on:
      - api-gateway
      - upload-manager
      - streaming-engine
      - content-management
    volumes:
      - ./shared:/app/shared
      - status-checker-logs:/app/logs
    networks:
      - media-network
    restart: unless-stopped

  # MongoDB - Database
  mongodb:
    image: mongo:5
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_ROOT_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_ROOT_PASSWORD}
      - MONGO_INITDB_DATABASE=media-service
    volumes:
      - mongodb-data:/data/db
      - ./config/mongodb/init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - media-network
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/media-service --quiet
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped
    command: --wiredTigerCacheSizeGB 1

  # Redis - Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
      - ./config/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    command: redis-server /usr/local/etc/redis/redis.conf

  # RabbitMQ - Message Broker
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_USER}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
      - ./config/rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./config/rabbitmq/definitions.json:/etc/rabbitmq/definitions.json:ro
    networks:
      - media-network
    healthcheck:
      test: ["CMD", "rabbitmqctl", "status"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    restart: unless-stopped

  # MinIO - S3-compatible Storage
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
    volumes:
      - minio-data:/data
    networks:
      - media-network
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # MinIO Setup - create buckets and users on startup
  minio-setup:
    image: minio/mc
    depends_on:
      - minio
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
    entrypoint: >
      /bin/sh -c "
      sleep 10;
      /usr/bin/mc config host add myminio http://minio:9000 ${MINIO_ROOT_USER} ${MINIO_ROOT_PASSWORD};
      /usr/bin/mc mb --ignore-existing myminio/media-assets;
      /usr/bin/mc policy set download myminio/media-assets;
      /usr/bin/mc admin policy set myminio readwrite user=uploaduser;
      exit 0;
      "
    networks:
      - media-network

  # Prometheus - Metrics
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - media-network
    restart: unless-stopped

  # Grafana - Dashboard
  grafana:
    image: grafana/grafana
    ports:
      - "3100:3000"
    volumes:
      - ./config/grafana/provisioning:/etc/grafana/provisioning:ro
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    networks:
      - media-network
    restart: unless-stopped

networks:
  media-network:
    driver: bridge

volumes:
  api-gateway-logs:
  upload-manager-logs:
  processing-pipeline-logs:
  streaming-engine-logs:
  content-management-logs:
  status-checker-logs:
  upload-temp:
  processing-temp:
  mongodb-data:
  redis-data:
  rabbitmq-data:
  minio-data:
  prometheus-data:
  grafana-data:
```

## Dockerfiles

Let's create a Dockerfile for each service:

### API Gateway Dockerfile

```dockerfile
FROM node:16-alpine

# Install dependencies
RUN apk add --no-cache curl ca-certificates

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm ci --only=production

# Copy app source
COPY . .

# Create logs directory
RUN mkdir -p logs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start the app
CMD ["node", "src/index.js"]
```

### Upload Manager Dockerfile

```dockerfile
FROM node:16-alpine

# Install dependencies
RUN apk add --no-cache curl ca-certificates

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm ci --only=production

# Copy app source
COPY . .

# Create logs and temp directories
RUN mkdir -p logs temp/uploads

# Expose port
EXPOSE 3001

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3001/health || exit 1

# Start the app
CMD ["node", "src/index.js"]
```

### Processing Pipeline Dockerfile

```dockerfile
FROM node:16-alpine

# Install dependencies including FFmpeg
RUN apk add --no-cache curl ca-certificates ffmpeg

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm ci --only=production

# Copy app source
COPY . .

# Create logs and temp directories
RUN mkdir -p logs temp/processing config/profiles

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3010/health || exit 1

# Start the app
CMD ["node", "src/index.js"]
```

### Streaming Engine Dockerfile

```dockerfile
FROM node:16-alpine

# Install dependencies
RUN apk add --no-cache curl ca-certificates

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm ci --only=production

# Copy app source
COPY . .

# Create logs directory
RUN mkdir -p logs

# Expose port
EXPOSE 3002

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3002/health || exit 1

# Start the app
CMD ["node", "src/index.js"]
```

### Content Management Dockerfile

```dockerfile
FROM node:16-alpine

# Install dependencies
RUN apk add --no-cache curl ca-certificates

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm ci --only=production

# Copy app source
COPY . .

# Create logs directory
RUN mkdir -p logs

# Expose port
EXPOSE 3003

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3003/health || exit 1

# Start the app
CMD ["node", "src/index.js"]
```

### Status Checker Dockerfile

```dockerfile
FROM node:16-alpine

# Install dependencies
RUN apk add --no-cache curl ca-certificates

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm ci --only=production

# Copy app source
COPY . .

# Create logs directory
RUN mkdir -p logs

# Expose port
EXPOSE 3006

# Start the app
CMD ["node", "src/index.js"]
```

## Environment Configuration

Create a `.env` file at the project root:

```dotenv
# MongoDB
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=securepassword
MONGODB_URI=mongodb://admin:securepassword@mongodb:27017/media-service?authSource=admin

# Redis
REDIS_URL=redis://redis:6379

# RabbitMQ
RABBITMQ_USER=mediaservice
RABBITMQ_PASSWORD=securepassword
RABBITMQ_URL=amqp://mediaservice:securepassword@rabbitmq:5672

# MinIO
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
S3_ENDPOINT=http://minio:9000
S3_BUCKET=media-assets
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

# JWT
JWT_SECRET=your-secure-jwt-secret-key-should-be-long-and-random

# Grafana
GRAFANA_USER=admin
GRAFANA_PASSWORD=securepassword
```

Create service-specific environment files:

### env/api-gateway.env

```dotenv
NODE_ENV=production
SERVICE_NAME=api-gateway
PORT=3000
LOG_LEVEL=info

# Service URLs
UPLOAD_SERVICE_URL=http://upload-manager:3001
STREAMING_SERVICE_URL=http://streaming-engine:3002
CONTENT_SERVICE_URL=http://content-management:3003

# Database
MONGODB_URI=mongodb://admin:securepassword@mongodb:27017/media-service?authSource=admin
MONGODB_DB_NAME=media-service

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET=your-secure-jwt-secret-key-should-be-long-and-random

# CORS
CORS_ORIGIN=*

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
```

### env/upload-manager.env

```dotenv
NODE_ENV=production
SERVICE_NAME=upload-manager
PORT=3001
LOG_LEVEL=info

# Upload settings
TEMP_UPLOAD_DIR=/app/temp/uploads
CHUNK_SIZE=5242880
MAX_FILE_SIZE=5368709120
UPLOAD_EXPIRE_HOURS=24

# S3 Storage
S3_ENDPOINT=http://minio:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

# Database
MONGODB_URI=mongodb://admin:securepassword@mongodb:27017/media-service?authSource=admin
MONGODB_DB_NAME=media-service

# Redis
REDIS_URL=redis://redis:6379

# RabbitMQ
RABBITMQ_URL=amqp://mediaservice:securepassword@rabbitmq:5672
RABBITMQ_QUEUE=media-processing

# JWT
JWT_SECRET=your-secure-jwt-secret-key-should-be-long-and-random

# CORS
CORS_ORIGIN=*

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
```

### env/processing-pipeline.env

```dotenv
NODE_ENV=production
SERVICE_NAME=processing-pipeline
LOG_LEVEL=info
PORT=3010

# Processing settings
WORK_DIR=/app/temp/processing
CONCURRENT_PROCESSING=2

# S3 Storage
S3_ENDPOINT=http://minio:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
MEDIA_PUBLIC_URL=http://minio:9000/media-assets

# Database
MONGODB_URI=mongodb://admin:securepassword@mongodb:27017/media-service?authSource=admin
MONGODB_DB_NAME=media-service

# Redis
REDIS_URL=redis://redis:6379

# RabbitMQ
RABBITMQ_URL=amqp://mediaservice:securepassword@rabbitmq:5672
RABBITMQ_QUEUE=media-processing

# Profiles
PROFILES_DIR=/app/config/profiles
```

### env/streaming-engine.env

```dotenv
NODE_ENV=production
SERVICE_NAME=streaming-engine
PORT=3002
LOG_LEVEL=info

# S3 Storage
S3_ENDPOINT=http://minio:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

# Database
MONGODB_URI=mongodb://admin:securepassword@mongodb:27017/media-service?authSource=admin
MONGODB_DB_NAME=media-service

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET=your-secure-jwt-secret-key-should-be-long-and-random

# CORS
CORS_ORIGIN=*

# Analytics
ANALYTICS_ENABLED=true
ANALYTICS_SAMPLE_RATE=10

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=200
```

### env/content-management.env

```dotenv
NODE_ENV=production
SERVICE_NAME=content-management
PORT=3003
LOG_LEVEL=info

# S3 Storage
S3_ENDPOINT=http://minio:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

# Database
MONGODB_URI=mongodb://admin:securepassword@mongodb:27017/media-service?authSource=admin
MONGODB_DB_NAME=media-service

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET=your-secure-jwt-secret-key-should-be-long-and-random

# CORS
CORS_ORIGIN=*

# Cache
ENABLE_REDIS_CACHE=true
ASSET_CACHE_TTL=3600

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
```

### env/status-checker.env

```dotenv
NODE_ENV=production
SERVICE_NAME=status-checker
PORT=3006
LOG_LEVEL=info

# Services to check
CHECK_INTERVAL=30000
SERVICES_TO_CHECK=http://api-gateway:3000/health,http://upload-manager:3001/health,http://streaming-engine:3002/health,http://content-management:3003/health

# Notification channels
NOTIFICATION_EMAIL=admin@example.com
SLACK_WEBHOOK_URL=
```

## Running the Dockerized Application

### Prerequisites

- Docker and Docker Compose installed
- Git (for cloning the repository)

### Setup Steps

1. Clone the repository:
```bash
git clone https://your-repository-url/media-management-service.git
cd media-management-service
```

2. Create all the necessary directories and files as outlined in the folder structure.

3. Build and start the services:
```bash
# Build all services
docker-compose build

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f
```

4. Check the status of the services:
```bash
docker-compose ps
```

5. Access the service endpoints:
   - API Gateway: http://localhost:3000
   - Upload Manager: http://localhost:3001
   - Streaming Engine: http://localhost:3002
   - Content Management: http://localhost:3003
   - Status Checker Dashboard: http://localhost:3006
   - MinIO Console: http://localhost:9001
   - Prometheus: http://localhost:9090
   - Grafana: http://localhost:3100

### Useful Commands

- Restart all services:
```bash
docker-compose restart
```

- Restart a specific service:
```bash
docker-compose restart upload-manager
```

- View logs for a specific service:
```bash
docker-compose logs -f upload-manager
```

- Stop all services:
```bash
docker-compose down
```

- Stop all services and remove volumes:
```bash
docker-compose down -v
```

## Testing the Dockerized Services

### Generate a JWT Token

```bash
docker-compose exec api-gateway node /app/scripts/generate-token.js test-user
```

### Upload a Test File

```bash
# Download a test file
curl -o test-video.mp4 https://test-videos.co.uk/vids/bigbuckbunny/mp4/h264/720/Big_Buck_Bunny_720_10s_1MB.mp4

# Run the upload test
docker-compose exec upload-manager node /app/scripts/test-upload.js /app/test-video.mp4
```

### Monitor Processing

```bash
# Check MongoDB for processed assets
docker-compose exec mongodb mongosh --username admin --password securepassword media-service --eval "db['media-assets'].find().pretty()"
```

### Test Streaming

1. Get an asset ID from the MongoDB query above
2. Use the JWT token to request streaming URL:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:3002/media/ASSET_ID
```

## Developing with Docker

For development, you can use the `docker-compose.dev.yml` file to override certain settings:

```yaml
version: '3.8'

services:
  api-gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile.dev
    volumes:
      - ./api-gateway:/app
      - /app/node_modules
    command: npm run dev

  upload-manager:
    build:
      context: ./upload-manager
      dockerfile: Dockerfile.dev
    volumes:
      - ./upload-manager:/app
      - /app/node_modules
    command: npm run dev

  processing-pipeline:
    build:
      context: ./processing-pipeline
      dockerfile: Dockerfile.dev
    volumes:
      - ./processing-pipeline:/app
      - /app/node_modules
    command: npm run dev

  streaming-engine:
    build:
      context: ./streaming-engine
      dockerfile: Dockerfile.dev
    volumes:
      - ./streaming-engine:/app
      - /app/node_modules
    command: npm run dev

  content-management:
    build:
      context: ./content-management
      dockerfile: Dockerfile.dev
    volumes:
      - ./content-management:/app
      - /app/node_modules
    command: npm run dev

  status-checker:
    build:
      context: ./status-checker
      dockerfile: Dockerfile.dev
    volumes:
      - ./status-checker:/app
      - /app/node_modules
    command: npm run dev
```

Run with:

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

## Creating a Development Dockerfile

For each service, create a `Dockerfile.dev`:

```dockerfile
FROM node:16-alpine

# Install dependencies
RUN apk add --no-cache curl ca-certificates

# For processing-pipeline, also install ffmpeg
RUN apk add --no-cache ffmpeg

# Create app directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install all dependencies including devDependencies
RUN npm install

# Create necessary directories
RUN mkdir -p logs temp

# Start the app in development mode
CMD ["npm", "run", "dev"]
```
