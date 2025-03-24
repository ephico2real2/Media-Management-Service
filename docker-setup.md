# Docker Folder Structure

Create this folder structure for the dockerized Media Management Service:

```
media-management-service/
├── api-gateway/                    # API Gateway service
│   ├── Dockerfile
│   ├── src/
│   ├── package.json
│   └── .env.example
├── upload-manager/                 # Upload Manager service
│   ├── Dockerfile
│   ├── src/
│   │   ├── routes/
│   │   ├── middlewares/
│   │   ├── services/
│   │   └── index.js
│   ├── package.json
│   └── .env.example
├── processing-pipeline/            # Processing Pipeline service
│   ├── Dockerfile
│   ├── src/
│   │   ├── utils/
│   │   ├── services/
│   │   └── index.js
│   ├── package.json
│   └── .env.example
├── streaming-engine/               # Streaming Engine service
│   ├── Dockerfile
│   ├── src/
│   │   ├── routes/
│   │   ├── middlewares/
│   │   ├── services/
│   │   └── index.js
│   ├── package.json
│   └── .env.example
├── content-management/             # Content Management service
│   ├── Dockerfile
│   ├── src/
│   │   ├── routes/
│   │   ├── models/
│   │   ├── controllers/
│   │   └── index.js
│   ├── package.json
│   └── .env.example
├── analytics-engine/               # Analytics Engine service
│   ├── Dockerfile
│   ├── src/
│   ├── package.json
│   └── .env.example
├── distribution/                   # Distribution service
│   ├── Dockerfile
│   ├── src/
│   ├── package.json
│   └── .env.example
├── status-checker/                 # Health monitoring service
│   ├── Dockerfile
│   ├── src/
│   ├── package.json
│   └── .env.example
├── shared/                         # Shared code between services
│   ├── s3-client.js
│   ├── db-client.js
│   ├── redis-client.js
│   ├── rabbitmq-client.js
│   ├── logger.js
│   └── utils.js
├── config/                         # Configuration files
│   ├── nginx/
│   │   └── nginx.conf
│   ├── profiles/                   # Transcode profiles
│   │   ├── 1080p.json
│   │   ├── 720p.json
│   │   ├── 480p.json
│   │   └── 360p.json
│   └── minio/
│       └── create-buckets.sh
├── tests/                          # Test scripts
│   ├── postman/
│   └── load-tests/
├── scripts/                        # Utility scripts
│   ├── generate-token.js
│   └── test-upload.js
├── docker/                         # Docker-specific files
│   ├── base/                       # Base image for services
│   │   └── Dockerfile
│   └── nginx/
│       └── Dockerfile
├── .env.example                    # Example .env file
├── docker-compose.yml              # Main Docker Compose file
├── docker-compose.dev.yml          # Development Compose override
├── docker-compose.prod.yml         # Production Compose override
└── README.md
```

Key aspects of this folder structure:

1. **Service-Specific Folders**: Each microservice has its own directory with a Dockerfile
2. **Shared Code**: Common code is centralized in the `shared` directory
3. **Configuration**: External configuration files in the `config` directory
4. **Docker Base Image**: Common Dockerfile in `docker/base` to reduce duplication
5. **Environment Files**: `.env.example` files to document required environment variables

This structure promotes maintainability, reusability, and follows Docker best practices.
