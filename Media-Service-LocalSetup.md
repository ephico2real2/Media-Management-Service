# Media Management Service: Enhanced Local Setup Guide

This comprehensive guide will help you set up a robust Media Management Service locally with improved structure, better coding practices, and enhanced utilities.

## Project Folder Structure

Start by creating this enhanced folder structure:

```
media-management-service/
├── api-gateway/              # API Gateway service
│   ├── src/
│   ├── package.json
│   └── .env -> ../env/api-gateway.env
├── upload-manager/           # Upload Manager service
│   ├── src/
│   │   ├── routes/
│   │   ├── middlewares/
│   │   ├── services/
│   │   └── index.js
│   ├── package.json
│   └── .env -> ../env/upload-manager.env
├── processing-pipeline/      # Processing Pipeline service
│   ├── src/
│   │   ├── utils/
│   │   ├── services/
│   │   └── index.js
│   ├── package.json
│   └── .env -> ../env/processing-pipeline.env
├── streaming-engine/         # Streaming Engine service
│   ├── src/
│   │   ├── routes/
│   │   ├── middlewares/
│   │   ├── services/
│   │   └── index.js
│   ├── package.json
│   └── .env -> ../env/streaming-engine.env
├── content-management/       # Content Management service
│   ├── src/
│   │   ├── routes/
│   │   ├── models/
│   │   ├── controllers/
│   │   └── index.js
│   ├── package.json
│   └── .env -> ../env/content-management.env
├── analytics-engine/         # Analytics Engine service
│   ├── src/
│   ├── package.json
│   └── .env -> ../env/analytics-engine.env
├── distribution/             # Distribution service
│   ├── src/
│   ├── package.json
│   └── .env -> ../env/distribution.env
├── status-checker/           # Health monitoring service
│   ├── src/
│   ├── package.json
│   └── .env -> ../env/status-checker.env
├── shared/                   # Shared code between services
│   ├── s3-client.js
│   ├── utils.js
│   ├── logger.js
│   ├── db-client.js
│   ├── redis-client.js
│   └── rabbitmq-client.js
├── scripts/                  # Utility scripts
│   ├── setup.sh
│   ├── generate-token.js
│   └── cleanup-temp.js
├── env/                      # Environment files
│   ├── api-gateway.env
│   ├── upload-manager.env
│   ├── processing-pipeline.env
│   ├── streaming-engine.env
│   ├── content-management.env
│   ├── analytics-engine.env
│   ├── distribution.env
│   └── status-checker.env
├── temp/                     # Temporary storage directory
│   ├── uploads/
│   └── processing/
├── tests/                    # Test files
│   ├── integration/
│   └── unit/
├── .eslintrc.json            # ESLint configuration
├── .prettierrc               # Prettier configuration
├── .gitignore                # Git ignore file
├── package.json              # Root package.json for dev dependencies
└── README.md
```

## Prerequisites

Install the following dependencies on your local machine:

1. Node.js (v16+) and npm
2. MongoDB Community Edition
3. Redis
4. RabbitMQ
5. FFmpeg and FFprobe
6. MinIO Server

## Step 1: Set Up Core Dependencies

### Install Node.js and npm
```bash
# For Ubuntu/Debian
sudo apt update
sudo apt install nodejs npm curl ca-certificates

# For macOS
brew install node

# Verify installation
node --version  # Should be v16+
npm --version
```

### Install FFmpeg
```bash
# For Ubuntu/Debian
sudo apt install ffmpeg

# For macOS
brew install ffmpeg

# Verify installation
ffmpeg -version
ffprobe -version
```

### Install MongoDB
```bash
# For Ubuntu/Debian
sudo apt install gnupg
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
sudo apt update
sudo apt install mongodb-org mongodb-mongosh

# For macOS
brew tap mongodb/brew
brew install mongodb-community mongosh

# Start MongoDB service
sudo systemctl start mongod    # Linux
brew services start mongodb-community  # macOS

# Verify MongoDB is running
mongosh --eval "db.version()"
```

### Install Redis
```bash
# For Ubuntu/Debian
sudo apt install redis-server

# For macOS
brew install redis

# Start Redis service
sudo systemctl start redis-server  # Linux
brew services start redis          # macOS

# Verify Redis is running
redis-cli ping  # Should return PONG
```

### Install RabbitMQ
```bash
# For Ubuntu/Debian
sudo apt install rabbitmq-server

# For macOS
brew install rabbitmq

# Start RabbitMQ service
sudo systemctl start rabbitmq-server  # Linux
brew services start rabbitmq          # macOS

# Verify RabbitMQ is running
rabbitmqctl status
```

### Install MinIO Server
```bash
# For Ubuntu/Debian
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# For macOS
brew install minio/stable/minio

# Create storage directory
mkdir -p ~/minio/data

# Start MinIO server
minio server ~/minio/data --console-address ":9001" &

# Install MinIO client
wget https://dl.min.io/client/mc/release/linux-amd64/mc  # Linux
chmod +x mc
sudo mv mc /usr/local/bin/

brew install minio/stable/mc  # macOS

# Configure MinIO client
mc config host add local http://localhost:9000 minioadmin minioadmin

# Create bucket for media assets
mc mb --ignore-existing local/media-assets

# Set download policy for the bucket
mc policy set download local/media-assets
```

## Step 2: Initialize Project Structure

### Create Project Directory
```bash
mkdir -p media-management-service
cd media-management-service

# Create necessary directories
mkdir -p \
  api-gateway/src \
  upload-manager/src/{routes,middlewares,services} \
  processing-pipeline/src/{utils,services} \
  streaming-engine/src/{routes,middlewares,services} \
  content-management/src/{routes,models,controllers} \
  analytics-engine/src \
  distribution/src \
  status-checker/src \
  shared \
  scripts \
  env \
  temp/{uploads,processing} \
  tests/{integration,unit}

# Initialize the root package.json
npm init -y

# Install development dependencies in the root
npm install --save-dev eslint prettier nodemon jest
```

### Create Configuration Files

#### .eslintrc.json
```bash
cat > .eslintrc.json << 'EOL'
{
  "env": {
    "node": true,
    "es2021": true,
    "jest": true
  },
  "extends": ["eslint:recommended"],
  "parserOptions": {
    "ecmaVersion": 2022,
    "sourceType": "module"
  },
  "rules": {
    "no-unused-vars": ["warn"],
    "quotes": ["error", "single"],
    "semi": ["error", "always"],
    "no-console": ["warn", { "allow": ["warn", "error", "info"] }]
  }
}
EOL
```

#### .prettierrc
```bash
cat > .prettierrc << 'EOL'
{
  "singleQuote": true,
  "semi": true,
  "printWidth": 100,
  "tabWidth": 2,
  "trailingComma": "es5"
}
EOL
```

#### .gitignore
```bash
cat > .gitignore << 'EOL'
# Dependencies
node_modules/

# Environment files
.env
.env.local
.env.*.local

# Logs
logs
*.log
npm-debug.log*

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Temporary files
temp/
tmp/

# Testing
coverage/

# IDE specific files
.idea/
.vscode/
*.swp
*.swo

# OS specific files
.DS_Store
Thumbs.db
EOL
```

## Step 3: Set Up Shared Utilities

### Create Shared S3 Client
```bash
cat > shared/s3-client.js << 'EOL'
const { S3Client } = require('@aws-sdk/client-s3');
const path = require('path');
const logger = require('./logger');

// Configuration for S3-compatible storage
const config = {
  s3: {
    endpoint: process.env.S3_ENDPOINT || 'http://localhost:9000',
    bucket: process.env.S3_BUCKET || 'media-assets',
    region: process.env.S3_REGION || 'us-east-1',
    forcePathStyle: process.env.S3_FORCE_PATH_STYLE === 'true' || true,
    credentials: {
      accessKeyId: process.env.S3_ACCESS_KEY || 'minioadmin',
      secretAccessKey: process.env.S3_SECRET_KEY || 'minioadmin'
    }
  }
};

// Initialize S3 client with provider-agnostic configuration
let s3Client;

try {
  s3Client = new S3Client({
    endpoint: config.s3.endpoint,
    region: config.s3.region,
    forcePathStyle: config.s3.forcePathStyle,
    credentials: config.s3.credentials
  });
  logger.info('S3 client initialized successfully');
} catch (error) {
  logger.error(`Failed to initialize S3 client: ${error.message}`);
  throw error;
}

// Function to generate public URL based on the storage provider
function getPublicUrl(key) {
  // This can be customized based on your storage provider
  if (process.env.MEDIA_PUBLIC_URL) {
    return `${process.env.MEDIA_PUBLIC_URL}/${key}`;
  }
  
  // Default for MinIO
  return `${config.s3.endpoint}/${config.s3.bucket}/${key}`;
}

module.exports = {
  s3Client,
  s3Config: config.s3,
  getPublicUrl
};
EOL
```

### Create Logger Utility
```bash
cat > shared/logger.js << 'EOL'
const winston = require('winston');
const path = require('path');

// Define log levels
const levels = {
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  debug: 4,
};

// Define log colors
const colors = {
  error: 'red',
  warn: 'yellow',
  info: 'green',
  http: 'magenta',
  debug: 'blue',
};

// Add colors to Winston
winston.addColors(colors);

// Define the format
const format = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss:ms' }),
  winston.format.colorize({ all: true }),
  winston.format.printf(
    (info) => `[${info.timestamp}] ${info.level}: ${info.message}`
  )
);

// Get service name from environment or derive from directory
function getServiceName() {
  if (process.env.SERVICE_NAME) {
    return process.env.SERVICE_NAME;
  }
  
  try {
    // Try to determine service name from directory structure
    const cwd = process.cwd();
    const dirName = path.basename(cwd);
    
    // Check if it's one of our services
    const serviceNames = [
      'api-gateway', 'upload-manager', 'processing-pipeline', 
      'streaming-engine', 'content-management', 'analytics-engine', 
      'distribution', 'status-checker'
    ];
    
    if (serviceNames.includes(dirName)) {
      return dirName;
    }
  } catch (error) {
    // Fallback
  }
  
  return 'media-service';
}

// Determine log level from environment
const level = process.env.LOG_LEVEL || 'info';

// Create transports
const transports = [
  new winston.transports.Console(),
  new winston.transports.File({
    filename: path.join(process.cwd(), 'logs', 'error.log'),
    level: 'error',
  }),
  new winston.transports.File({
    filename: path.join(process.cwd(), 'logs', `${getServiceName()}.log`),
  }),
];

// Create the logger
const logger = winston.createLogger({
  level,
  levels,
  format,
  transports,
});

module.exports = logger;
EOL
```

### Create Utilities
```bash
cat > shared/utils.js << 'EOL'
const crypto = require('crypto');
const path = require('path');
const fs = require('fs-extra');

/**
 * Generate a unique upload ID
 * @returns {string} Unique ID
 */
function generateUploadId() {
  return crypto.randomUUID();
}

/**
 * Sanitize a filename to make it safe for filesystem
 * @param {string} filename - Original filename
 * @returns {string} Sanitized filename
 */
function sanitizeFilename(filename) {
  return filename.replace(/[^a-zA-Z0-9.\-_]/g, '_');
}

/**
 * Calculate file hash for deduplication
 * @param {Buffer|string} fileData - File data or path
 * @param {string} algorithm - Hash algorithm
 * @returns {Promise<string>} Hash of the file
 */
async function calculateFileHash(fileData, algorithm = 'sha256') {
  const hashSum = crypto.createHash(algorithm);
  
  if (typeof fileData === 'string' && fs.existsSync(fileData)) {
    // If fileData is a path
    return new Promise((resolve, reject) => {
      const stream = fs.createReadStream(fileData);
      stream.on('error', reject);
      stream.on('data', chunk => hashSum.update(chunk));
      stream.on('end', () => resolve(hashSum.digest('hex')));
    });
  } else {
    // If fileData is a buffer
    hashSum.update(fileData);
    return hashSum.digest('hex');
  }
}

/**
 * Get absolute path by resolving relative to project root
 * @param {string} relativePath - Path relative to project root
 * @returns {string} Absolute path
 */
function getAbsolutePath(relativePath) {
  // Assuming this file is in /shared directory at project root
  const projectRoot = path.resolve(__dirname, '..');
  return path.resolve(projectRoot, relativePath);
}

/**
 * Retry a function with exponential backoff
 * @param {Function} fn - Function to retry
 * @param {Object} options - Retry options
 * @returns {Promise<any>} Function result
 */
async function withRetry(fn, options = {}) {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    factor = 2,
    jitter = 0.1,
    onRetry = null,
  } = options;
  
  let attempt = 0;
  
  while (true) {
    try {
      return await fn();
    } catch (error) {
      attempt++;
      
      if (attempt >= maxRetries) {
        throw error;
      }
      
      const delay = calculateBackoff(baseDelay, factor, attempt, jitter);
      
      if (onRetry) {
        onRetry({ attempt, error, delay });
      }
      
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

/**
 * Calculate backoff time with exponential growth and jitter
 */
function calculateBackoff(baseDelay, factor, attempt, jitter = 0.1) {
  const exponentialDelay = baseDelay * Math.pow(factor, attempt - 1);
  const jitterAmount = exponentialDelay * jitter * (Math.random() * 2 - 1);
  return exponentialDelay + jitterAmount;
}

module.exports = {
  generateUploadId,
  sanitizeFilename,
  calculateFileHash,
  getAbsolutePath,
  withRetry
};
EOL
```

### Create Database Client
```bash
cat > shared/db-client.js << 'EOL'
const { MongoClient } = require('mongodb');
const logger = require('./logger');
const { withRetry } = require('./utils');

// Configuration
const config = {
  uri: process.env.MONGODB_URI || 'mongodb://localhost:27017',
  dbName: process.env.MONGODB_DB_NAME || 'media-service',
  options: {
    useUnifiedTopology: true,
    connectTimeoutMS: 10000,
    socketTimeoutMS: 45000,
  }
};

let client = null;
let db = null;

/**
 * Initialize MongoDB connection
 * @returns {Promise<Object>} MongoDB database instance
 */
async function connect() {
  if (db) return db;
  
  try {
    logger.info('Connecting to MongoDB...');
    
    client = new MongoClient(config.uri, config.options);
    
    // Connect with retry capability
    await withRetry(
      () => client.connect(),
      {
        maxRetries: 5,
        baseDelay: 1000,
        factor: 1.5,
        onRetry: ({ attempt, error }) => {
          logger.warn(`MongoDB connection attempt ${attempt} failed: ${error.message}`);
        }
      }
    );
    
    db = client.db(config.dbName);
    logger.info('Connected to MongoDB successfully');
    
    return db;
  } catch (error) {
    logger.error(`Failed to connect to MongoDB: ${error.message}`);
    throw error;
  }
}

/**
 * Get MongoDB database instance
 * @returns {Object} MongoDB database instance
 */
function getDb() {
  if (!db) {
    throw new Error('Database not initialized. Call connect() first.');
  }
  return db;
}

/**
 * Close MongoDB connection
 */
async function close() {
  if (client) {
    await client.close();
    client = null;
    db = null;
    logger.info('MongoDB connection closed');
  }
}

// Setup graceful shutdown
process.on('SIGINT', async () => {
  await close();
  process.exit(0);
});

process.on('SIGTERM', async () => {
  await close();
  process.exit(0);
});

module.exports = {
  connect,
  getDb,
  close
};
EOL
```

### Create Redis Client
```bash
cat > shared/redis-client.js << 'EOL'
const { createClient } = require('redis');
const logger = require('./logger');
const { withRetry } = require('./utils');

// Configuration
const config = {
  url: process.env.REDIS_URL || 'redis://localhost:6379',
  retryStrategy: {
    maxRetries: 10,
    baseDelay: 100,
    factor: 1.5
  }
};

let client = null;

/**
 * Initialize Redis connection
 * @returns {Promise<Object>} Redis client
 */
async function connect() {
  if (client && client.isReady) return client;
  
  try {
    logger.info('Connecting to Redis...');
    
    client = createClient({ url: config.url });
    
    // Setup event handlers
    client.on('error', (err) => {
      logger.error(`Redis Error: ${err.message}`);
    });
    
    client.on('reconnecting', () => {
      logger.warn('Redis reconnecting...');
    });
    
    // Connect with retry capability
    await withRetry(
      () => client.connect(),
      {
        maxRetries: config.retryStrategy.maxRetries,
        baseDelay: config.retryStrategy.baseDelay,
        factor: config.retryStrategy.factor,
        onRetry: ({ attempt, error }) => {
          logger.warn(`Redis connection attempt ${attempt} failed: ${error.message}`);
        }
      }
    );
    
    logger.info('Connected to Redis successfully');
    
    return client;
  } catch (error) {
    logger.error(`Failed to connect to Redis: ${error.message}`);
    throw error;
  }
}

/**
 * Get Redis client
 * @returns {Object} Redis client
 */
function getClient() {
  if (!client || !client.isReady) {
    throw new Error('Redis not initialized. Call connect() first.');
  }
  return client;
}

/**
 * Close Redis connection
 */
async function close() {
  if (client) {
    await client.quit();
    client = null;
    logger.info('Redis connection closed');
  }
}

// Setup graceful shutdown
process.on('SIGINT', async () => {
  await close();
});

process.on('SIGTERM', async () => {
  await close();
});

module.exports = {
  connect,
  getClient,
  close
};
EOL
```

### Create RabbitMQ Client
```bash
cat > shared/rabbitmq-client.js << 'EOL'
const amqp = require('amqplib');
const logger = require('./logger');
const { withRetry } = require('./utils');

// Configuration
const config = {
  url: process.env.RABBITMQ_URL || 'amqp://localhost',
  retryStrategy: {
    maxRetries: 10,
    baseDelay: 1000,
    factor: 1.5
  }
};

let connection = null;
let channel = null;

/**
 * Initialize RabbitMQ connection
 * @returns {Promise<Object>} RabbitMQ channel
 */
async function connect() {
  if (channel) return channel;
  
  try {
    logger.info('Connecting to RabbitMQ...');
    
    // Connect with retry capability
    connection = await withRetry(
      () => amqp.connect(config.url),
      {
        maxRetries: config.retryStrategy.maxRetries,
        baseDelay: config.retryStrategy.baseDelay,
        factor: config.retryStrategy.factor,
        onRetry: ({ attempt, error }) => {
          logger.warn(`RabbitMQ connection attempt ${attempt} failed: ${error.message}`);
        }
      }
    );
    
    // Create channel
    channel = await connection.createChannel();
    
    logger.info('Connected to RabbitMQ successfully');
    
    // Setup connection error handler
    connection.on('error', (err) => {
      logger.error(`RabbitMQ connection error: ${err.message}`);
      channel = null;
      connection = null;
      // Auto-reconnect after delay
      setTimeout(() => connect(), 5000);
    });
    
    return channel;
  } catch (error) {
    logger.error(`Failed to connect to RabbitMQ: ${error.message}`);
    throw error;
  }
}

/**
 * Get RabbitMQ channel
 * @returns {Object} RabbitMQ channel
 */
function getChannel() {
  if (!channel) {
    throw new Error('RabbitMQ not initialized. Call connect() first.');
  }
  return channel;
}

/**
 * Create or verify queue
 * @param {string} queueName - Queue name
 * @param {Object} options - Queue options
 * @returns {Promise<Object>} Queue details
 */
async function assertQueue(queueName, options = { durable: true }) {
  const ch = await connect();
  return ch.assertQueue(queueName, options);
}

/**
 * Send message to queue
 * @param {string} queueName - Queue name
 * @param {Object} message - Message to send
 * @param {Object} options - Message options
 * @returns {Promise<boolean>} Success status
 */
async function sendToQueue(queueName, message, options = { persistent: true }) {
  const ch = await connect();
  await assertQueue(queueName);
  return ch.sendToQueue(queueName, Buffer.from(JSON.stringify(message)), options);
}

/**
 * Consume messages from queue
 * @param {string} queueName - Queue name
 * @param {Function} callback - Message handler
 * @param {Object} options - Consumer options
 */
async function consume(queueName, callback, options = { noAck: false }) {
  const ch = await connect();
  await assertQueue(queueName);
  return ch.consume(queueName, async (msg) => {
    if (msg !== null) {
      try {
        const content = JSON.parse(msg.content.toString());
        await callback(content, msg);
        ch.ack(msg);
      } catch (error) {
        logger.error(`Error processing message: ${error.message}`);
        ch.nack(msg, false, true); // Requeue
      }
    }
  }, options);
}

/**
 * Close RabbitMQ connection
 */
async function close() {
  if (channel) {
    await channel.close();
    channel = null;
  }
  
  if (connection) {
    await connection.close();
    connection = null;
    logger.info('RabbitMQ connection closed');
  }
}

// Setup graceful shutdown
process.on('SIGINT', async () => {
  await close();
});

process.on('SIGTERM', async () => {
  await close();
});

module.exports = {
  connect,
  getChannel,
  assertQueue,
  sendToQueue,
  consume,
  close
};
EOL
```

## Step 4: Set Up Environment Files

### Create Environment Files
```bash
# Create environment files for each service

# API Gateway
cat > env/api-gateway.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=api-gateway
PORT=3000
LOG_LEVEL=info
UPLOAD_SERVICE_URL=http://localhost:3001
STREAMING_SERVICE_URL=http://localhost:3002
CONTENT_SERVICE_URL=http://localhost:3003
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
REDIS_URL=redis://localhost:6379
JWT_SECRET=api-gateway-dev-secret-change-me
CORS_ORIGIN=*
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
EOL

# Upload Manager
cat > env/upload-manager.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=upload-manager
PORT=3001
LOG_LEVEL=info
TEMP_UPLOAD_DIR=../temp/uploads
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
REDIS_URL=redis://localhost:6379
RABBITMQ_URL=amqp://localhost
RABBITMQ_QUEUE=media-processing
CHUNK_SIZE=5242880
MAX_FILE_SIZE=5368709120
UPLOAD_EXPIRE_HOURS=24
JWT_SECRET=upload-manager-dev-secret-change-me
CORS_ORIGIN=*
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
EOL

# Processing Pipeline
cat > env/processing-pipeline.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=processing-pipeline
LOG_LEVEL=info
WORK_DIR=../temp/processing
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
MONGODB_COLLECTION=media-assets
REDIS_URL=redis://localhost:6379
RABBITMQ_URL=amqp://localhost
RABBITMQ_QUEUE=media-processing
MEDIA_PUBLIC_URL=http://localhost:9000/media-assets
CONCURRENT_PROCESSING=2
EOL

# Streaming Engine
cat > env/streaming-engine.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=streaming-engine
PORT=3002
LOG_LEVEL=info
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
REDIS_URL=redis://localhost:6379
JWT_SECRET=streaming-engine-dev-secret-change-me
CORS_ORIGIN=*
ANALYTICS_ENABLED=true
ANALYTICS_SAMPLE_RATE=10
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=200
EOL

# Content Management
cat > env/content-management.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=content-management
PORT=3003
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
JWT_SECRET=content-management-dev-secret-change-me
CORS_ORIGIN=*
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100
EOL

# Analytics Engine
cat > env/analytics-engine.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=analytics-engine
PORT=3004
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
REDIS_URL=redis://localhost:6379
ELASTICSEARCH_URL=http://localhost:9200
JWT_SECRET=analytics-engine-dev-secret-change-me
EOL

# Distribution
cat > env/distribution.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=distribution
PORT=3005
LOG_LEVEL=info
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB_NAME=media-service
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=media-assets
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
CDN_DOMAIN=localhost:9000
JWT_SECRET=distribution-dev-secret-change-me
EOL

# Status Checker
cat > env/status-checker.env << 'EOL'
NODE_ENV=development
SERVICE_NAME=status-checker
PORT=3006
LOG_LEVEL=info
CHECK_INTERVAL=30000
SERVICES_TO_CHECK=http://localhost:3000/health,http://localhost:3001/health,http://localhost:3002/health,http://localhost:3003/health
NOTIFICATION_EMAIL=admin@example.com
SLACK_WEBHOOK_URL=
EOL

# Create symbolic links for .env files
cd api-gateway && ln -sf ../env/api-gateway.env .env && cd ..
cd upload-manager && ln -sf ../env/upload-manager.env .env && cd ..
cd processing-pipeline && ln -sf ../env/processing-pipeline.env .env && cd ..
cd streaming-engine && ln -sf ../env/streaming-engine.env .env && cd ..
cd content-management && ln -sf ../env/content-management.env .env && cd ..
cd analytics-engine && ln -sf ../env/analytics-engine.env .env && cd ..
cd distribution && ln -sf ../env/distribution.env .env && cd ..
cd status-checker && ln -sf ../env/status-checker.env .env && cd ..
```

## Step 5: Create Utility Scripts

### Token Generator Script
```bash
cat > scripts/generate-token.js << 'EOL'
require('dotenv').config({ path: '../env/streaming-engine.env' });
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

// Generate a secure secret if one isn't provided
const JWT_SECRET = process.env.JWT_SECRET || crypto.randomBytes(32).toString('hex');
const userId = process.argv[2] || 'test-user';
const expiresIn = process.argv[3] || '1h';
const role = process.argv[4] || 'user';

const payload = {
  id: userId,
  role: role,
  iat: Math.floor(Date.now() / 1000)
};

const token = jwt.sign(payload, JWT_SECRET, { expiresIn });

console.log('\nGenerated JWT Token:');
console.log('--------------------');
console.log(token);
console.log('\nPayload:');
console.log('--------');
console.log(JSON.stringify(JSON.parse(Buffer.from(token.split('.')[1], 'base64').toString()), null, 2));
console.log('\nExpires:');
console.log('--------');
const exp = new Date(JSON.parse(Buffer.from(token.split('.')[1], 'base64').toString()).exp * 1000);
console.log(exp.toISOString());
EOL
```

### Cleanup Script
```bash
cat > scripts/cleanup-temp.js << 'EOL'
require('dotenv').config();
const fs = require('fs-extra');
const path = require('path');
const { createClient } = require('redis');
const logger = require('../shared/logger');

// Configuration
const config = {
  uploadsDir: process.env.TEMP_UPLOAD_DIR || '../temp/uploads',
  processingDir: process.env.WORK_DIR || '../temp/processing',
  maxAgeHours: parseInt(process.env.TEMP_FILE_MAX_AGE_HOURS || '24'),
  redisUrl: process.env.REDIS_URL || 'redis://localhost:6379'
};

// Convert hours to milliseconds
const maxAgeMs = config.maxAgeHours * 60 * 60 * 1000;

async function cleanup() {
  logger.info('Starting cleanup of temporary files...');
  
  const now = Date.now();
  let deletedCount = 0;
  let errorCount = 0;
  
  // Connect to Redis to check active uploads
  const redisClient = createClient({ url: config.redisUrl });
  await redisClient.connect();
  
  try {
    // Process uploads directory
    if (fs.existsSync(config.uploadsDir)) {
      const uploadDirs = fs.readdirSync(config.uploadsDir);
      logger.info(`Found ${uploadDirs.length} upload directories`);
      
      for (const dir of uploadDirs) {
        const dirPath = path.join(config.uploadsDir, dir);
        const stats = fs.statSync(dirPath);
        const age = now - stats.mtimeMs;
        
        // Check if this is an active upload in Redis
        const isActive = await redisClient.exists(`upload:${dir}`);
        
        if (age > maxAgeMs || !isActive) {
          try {
            // Remove the directory
            fs.removeSync(dirPath);
            deletedCount++;
            logger.info(`Deleted old upload directory: ${dir} (${age / 3600000} hours old)`);
          } catch (error) {
            errorCount++;
            logger.error(`Failed to delete directory ${dir}: ${error.message}`);
          }
        }
      }
    }
    
    // Process processing directory
    if (fs.existsSync(config.processingDir)) {
      const processingDirs = fs.readdirSync(config.processingDir);
      logger.info(`Found ${processingDirs.length} processing directories`);
      
      for (const dir of processingDirs) {
        const dirPath = path.join(config.processingDir, dir);
        const stats = fs.statSync(dirPath);
        const age = now - stats.mtimeMs;
        
        if (age > maxAgeMs) {
          try {
            // Remove the directory
            fs.removeSync(dirPath);
            deletedCount++;
            logger.info(`Deleted old processing directory: ${dir} (${age / 3600000} hours old)`);
          } catch (error) {
            errorCount++;
            logger.error(`Failed to delete directory ${dir}: ${error.message}`);
          }
        }
      }
    }
    
    logger.info(`Cleanup complete. Deleted ${deletedCount} directories. Errors: ${errorCount}`);
  } catch (error) {
    logger.error(`Cleanup failed: ${error.message}`);
  } finally {
    await redisClient.quit();
  }
}

// Run cleanup
cleanup().catch(error => {
  logger.error(`Unhandled error during cleanup: ${error.message}`);
  process.exit(1);
});
EOL
```

### Setup Script
```bash
cat > scripts/setup.sh << 'EOL'
#!/bin/bash

# Media Management Service Setup Script
echo "Setting up Media Management Service..."

# Create log directories
mkdir -p logs
for service in api-gateway upload-manager processing-pipeline streaming-engine content-management analytics-engine distribution status-checker; do
  mkdir -p $service/logs
done

# Install dependencies for shared modules
echo "Installing shared dependencies..."
npm install --save winston express-winston @aws-sdk/client-s3 @aws-sdk/s3-request-presigner mongodb redis amqplib jsonwebtoken fs-extra

# Install service-specific dependencies
echo "Installing service-specific dependencies..."

# API Gateway
cd api-gateway
npm init -y
npm install express cors helmet express-rate-limit express-winston jsonwebtoken http-proxy-middleware
cd ..

# Upload Manager
cd upload-manager
npm init -y
npm install express cors helmet express-rate-limit multer express-fileupload crypto uuid fs-extra path dotenv
cd ..

# Processing Pipeline
cd processing-pipeline
npm init -y
npm install dotenv child_process os path fs-extra
cd ..

# Streaming Engine
cd streaming-engine
npm init -y
npm install express cors helmet express-rate-limit jsonwebtoken ioredis dotenv
cd ..

# Content Management
cd content-management
npm init -y
npm install express cors helmet express-rate-limit jsonwebtoken dotenv
cd ..

# Update package.json in each service
for service in api-gateway upload-manager processing-pipeline streaming-engine content-management; do
  cat > $service/package.json << EOF
{
  "name": "$service",
  "version": "1.0.0",
  "description": "$service for Media Management Service",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "lint": "eslint src/",
    "format": "prettier --write \"src/**/*.js\"",
    "test": "jest"
  },
  "keywords": [
    "media",
    "streaming",
    "upload"
  ],
  "author": "",
  "license": "ISC"
}
EOF
done

echo "Setup complete! Next steps:"
echo "1. Update environment variables in env/ directory"
echo "2. Create implementation files in each service's src/ directory"
echo "3. Start services using 'npm run dev' in each service directory"
EOL

chmod +x scripts/setup.sh
```

## Step 6: Implement Upload Manager

### Create Index File
```bash
cat > upload-manager/src/index.js << 'EOL'
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const fs = require('fs-extra');
const path = require('path');
const logger = require('../../shared/logger');
const uploadRoutes = require('./routes/upload-routes');
const { connect: connectRedis } = require('../../shared/redis-client');
const { connect: connectMongo } = require('../../shared/db-client');
const { getAbsolutePath } = require('../../shared/utils');

// Initialize Express app
const app = express();
const port = process.env.PORT || 3001;

// Ensure temp directory exists
const tempDir = getAbsolutePath(process.env.TEMP_UPLOAD_DIR || '../temp/uploads');
fs.ensureDirSync(tempDir);
logger.info(`Temporary uploads directory: ${tempDir}`);

// Middleware
app.use(helmet()); // Security headers
app.use(cors({
  origin: process.env.CORS_ORIGIN || '*',
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS || '60000'),
  max: parseInt(process.env.RATE_LIMIT_MAX || '100'),
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many requests, please try again later' }
});
app.use(limiter);

app.use(express.json());

// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'UP',
    service: 'upload-manager',
    timestamp: new Date().toISOString()
  });
});

// Routes
app.use('/api/uploads', uploadRoutes);

// Error handling middleware
app.use((err, req, res, next) => {
  logger.error(`Error: ${err.message}`);
  res.status(err.status || 500).json({
    error: err.message || 'Internal Server Error'
  });
});

// Initialize connections and start server
async function start() {
  try {
    // Connect to Redis
    await connectRedis();
    
    // Connect to MongoDB
    await connectMongo();
    
    // Start the server
    app.listen(port, () => {
      logger.info(`Upload Manager service running on port ${port}`);
    });
  } catch (error) {
    logger.error(`Failed to start Upload Manager service: ${error.message}`);
    process.exit(1);
  }
}

// Start the service
start();
EOL
```

### Create Upload Routes
```bash
cat > upload-manager/src/routes/upload-routes.js << 'EOL'
const express = require('express');
const multer = require('multer');
const fs = require('fs-extra');
const path = require('path');
const logger = require('../../../shared/logger');
const { getClient: getRedisClient } = require('../../../shared/redis-client');
const { getDb } = require('../../../shared/db-client');
const { s3Client, s3Config } = require('../../../shared/s3-client');
const { PutObjectCommand } = require('@aws-sdk/client-s3');
const { sendToQueue } = require('../../../shared/rabbitmq-client');
const { 
  generateUploadId, 
  sanitizeFilename, 
  calculateFileHash,
  getAbsolutePath
} = require('../../../shared/utils');

const router = express.Router();

// Configuration
const config = {
  tempDir: getAbsolutePath(process.env.TEMP_UPLOAD_DIR || '../temp/uploads'),
  chunkSize: parseInt(process.env.CHUNK_SIZE || '5242880'), // 5MB default
  maxFileSize: parseInt(process.env.MAX_FILE_SIZE || '5368709120'), // 5GB default
  uploadExpireHours: parseInt(process.env.UPLOAD_EXPIRE_HOURS || '24'),
  rabbitmq: {
    queue: process.env.RABBITMQ_QUEUE || 'media-processing'
  }
};

// Configure multer for handling file uploads
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    const uploadId = req.params.uploadId;
    const uploadDir = path.join(config.tempDir, uploadId);
    
    fs.ensureDirSync(uploadDir);
    
    cb(null, uploadDir);
  },
  filename: function (req, file, cb) {
    const chunkNumber = req.body.chunkNumber;
    cb(null, `chunk-${chunkNumber}`);
  }
});

const upload = multer({ 
  storage,
  limits: { fileSize: config.chunkSize + 1024 } // Allow a bit extra for metadata
});

// Initialize upload session
router.post('/init', async (req, res) => {
  try {
    const { filename, fileSize, fileType, metadata } = req.body;
    
    if (!filename || !fileSize || !fileType) {
      return res.status(400).json({ error: 'Missing required parameters' });
    }
    
    // Validate file size
    if (fileSize > config.maxFileSize) {
      return res.status(400).json({ 
        error: `File too large. Maximum size is ${config.maxFileSize / 1024 / 1024}MB` 
      });
    }
    
    // Sanitize filename
    const sanitizedFilename = sanitizeFilename(filename);
    
    // Create a unique upload ID
    const uploadId = generateUploadId();
    
    // Get Redis client
    const redisClient = getRedisClient();
    
    // Store upload metadata in Redis
    await redisClient.hSet(`upload:${uploadId}`, {
      filename: sanitizedFilename,
      fileSize: String(fileSize),
      fileType,
      metadata: JSON.stringify(metadata || {}),
      status: 'initialized',
      createdAt: new Date().toISOString(),
      userId: req.user?.id || 'anonymous',
      totalChunks: Math.ceil(fileSize / config.chunkSize),
      receivedChunks: '0',
    });
    
    // Set expiration for upload session
    const expirySeconds = config.uploadExpireHours * 60 * 60;
    await redisClient.expire(`upload:${uploadId}`, expirySeconds);
    
    logger.info(`Upload initialized: ${uploadId} for file ${sanitizedFilename} (${fileSize} bytes)`);
    
    res.status(201).json({
      uploadId,
      chunkSize: config.chunkSize,
      expiresAt: new Date(Date.now() + expirySeconds * 1000).toISOString()
    });
  } catch (error) {
    logger.error(`Upload initialization error: ${error.message}`);
    res.status(500).json({ error: 'Failed to initialize upload' });
  }
});

// Upload chunk
router.post('/:uploadId/chunks', upload.single('chunk'), async (req, res) => {
  try {
    const { uploadId } = req.params;
    const { chunkNumber, totalChunks } = req.body;
    
    if (!chunkNumber) {
      return res.status(400).json({ error: 'Missing chunk number' });
    }
    
    // Get Redis client
    const redisClient = getRedisClient();
    
    // Validate upload exists
    const uploadExists = await redisClient.exists(`upload:${uploadId}`);
    if (!uploadExists) {
      return res.status(404).json({ error: 'Upload session not found or expired' });
    }
    
    // Update received chunks
    const uploadInfo = await redisClient.hGetAll(`upload:${uploadId}`);
    let receivedChunks = parseInt(uploadInfo.receivedChunks) + 1;
    
    await redisClient.hSet(`upload:${uploadId}`, {
      receivedChunks: String(receivedChunks),
      lastChunkAt: new Date().toISOString()
    });
    
    logger.info(`Chunk ${chunkNumber} received for upload ${uploadId} (${receivedChunks}/${uploadInfo.totalChunks})`);
    
    // Check if upload is complete
    if (receivedChunks >= parseInt(uploadInfo.totalChunks)) {
      // Update status to 'assembling'
      await redisClient.hSet(`upload:${uploadId}`, {
        status: 'assembling'
      });
      
      logger.info(`All chunks received for upload ${uploadId}, initiating assembly`);
      
      // Trigger async assembly process
      processUpload(uploadId).catch(error => {
        logger.error(`Error in processUpload for ${uploadId}: ${error.message}`);
      });
    }
    
    res.status(200).json({
      uploadId,
      chunkNumber,
      received: receivedChunks,
      total: parseInt(uploadInfo.totalChunks)
    });
  } catch (error) {
    logger.error(`Chunk upload error: ${error.message}`);
    res.status(500).json({ error: 'Failed to process chunk' });
  }
});

// Get upload status
router.get('/:uploadId', async (req, res) => {
  try {
    const { uploadId } = req.params;
    
    // Get Redis client
    const redisClient = getRedisClient();
    
    const uploadInfo = await redisClient.hGetAll(`upload:${uploadId}`);
    
    if (!uploadInfo || Object.keys(uploadInfo).length === 0) {
      return res.status(404).json({ error: 'Upload not found' });
    }
    
    // Convert string values to appropriate types
    const response = {
      ...uploadInfo,
      fileSize: parseInt(uploadInfo.fileSize),
      totalChunks: parseInt(uploadInfo.totalChunks),
      receivedChunks: parseInt(uploadInfo.receivedChunks),
      metadata: JSON.parse(uploadInfo.metadata || '{}'),
      progress: Math.floor((parseInt(uploadInfo.receivedChunks) / parseInt(uploadInfo.totalChunks)) * 100)
    };
    
    res.status(200).json(response);
  } catch (error) {
    logger.error(`Error fetching upload status: ${error.message}`);
    res.status(500).json({ error: 'Failed to fetch upload status' });
  }
});

// Async function to process completed upload
async function processUpload(uploadId) {
  // Get Redis client
  const redisClient = getRedisClient();
  
  try {
    // Get upload info
    const uploadInfo = await redisClient.hGetAll(`upload:${uploadId}`);
    const uploadDir = path.join(config.tempDir, uploadId);
    const outputFilePath = path.join(uploadDir, uploadInfo.filename);
    
    logger.info(`Processing upload ${uploadId}: Assembling chunks into ${outputFilePath}`);
    
    // Create write stream for final file
    const writeStream = fs.createWriteStream(outputFilePath);
    
    // Combine all chunks
    const totalChunks = parseInt(uploadInfo.totalChunks);
    for (let i = 0; i < totalChunks; i++) {
      const chunkPath = path.join(uploadDir, `chunk-${i}`);
      if (fs.existsSync(chunkPath)) {
        // Append chunk to output file
        await new Promise((resolve, reject) => {
          const readStream = fs.createReadStream(chunkPath);
          readStream.pipe(writeStream, { end: false });
          readStream.on('end', resolve);
          readStream.on('error', reject);
        });
        
        // Remove chunk file
        fs.unlinkSync(chunkPath);
      } else {
        throw new Error(`Missing chunk file: ${chunkPath}`);
      }
    }
    
    // Close write stream
    writeStream.end();
    
    logger.info(`All chunks assembled for upload ${uploadId}`);
    
    // Update status
    await redisClient.hSet(`upload:${uploadId}`, {
      status: 'validating'
    });
    
    // Validate file size
    const stats = fs.statSync(outputFilePath);
    if (stats.size !== parseInt(uploadInfo.fileSize)) {
      throw new Error(`File size mismatch. Expected: ${uploadInfo.fileSize}, Got: ${stats.size}`);
    }
    
    // Calculate file hash for deduplication
    await redisClient.hSet(`upload:${uploadId}`, { status: 'hashing' });
    
    const fileHash = await calculateFileHash(outputFilePath);
    
    logger.info(`File hash calculated for upload ${uploadId}: ${fileHash}`);
    
    // Check for duplicates in MongoDB
    const db = getDb();
    const existingAsset = await db.collection('media-assets').findOne({ fileHash });
    
    // If duplicate found and deduplication is enabled, skip upload and link to existing asset
    if (existingAsset && process.env.ENABLE_DEDUPLICATION === 'true') {
      logger.info(`Duplicate file detected for upload ${uploadId}, linking to existing asset ${existingAsset._id}`);
      
      await redisClient.hSet(`upload:${uploadId}`, {
        status: 'duplicate',
        assetId: existingAsset._id,
        duplicateOf: existingAsset.originalKey,
        completedAt: new Date().toISOString()
      });
      
      // Clean up
      fs.rmSync(uploadDir, { recursive: true, force: true });
      
      return;
    }
    
    // Upload to S3
    await redisClient.hSet(`upload:${uploadId}`, {
      status: 'uploading',
      fileHash
    });
    
    // Unique S3 key based on user and hash
    const s3Key = `uploads/${uploadInfo.userId}/${fileHash}/${uploadInfo.filename}`;
    
    logger.info(`Uploading file to S3: ${s3Key}`);
    
    await s3Client.send(new PutObjectCommand({
      Bucket: s3Config.bucket,
      Key: s3Key,
      Body: fs.createReadStream(outputFilePath),
      ContentType: uploadInfo.fileType,
      Metadata: {
        uploadId,
        userId: uploadInfo.userId,
        originalFilename: uploadInfo.filename,
        fileHash
      }
    }));
    
    logger.info(`File uploaded to S3 for upload ${uploadId}`);
    
    // Update Redis with S3 info
    await redisClient.hSet(`upload:${uploadId}`, {
      status: 'complete',
      s3Key,
      s3Bucket: s3Config.bucket,
      completedAt: new Date().toISOString()
    });
    
    // Send message to processing queue
    await sendToQueue(config.rabbitmq.queue, {
      type: 'new_upload',
      uploadId,
      s3Key,
      s3Bucket: s3Config.bucket,
      fileType: uploadInfo.fileType,
      metadata: JSON.parse(uploadInfo.metadata || '{}'),
      fileHash,
      timestamp: new Date().toISOString()
    });
    
    logger.info(`Processing message sent to queue for upload ${uploadId}`);
    
    // Clean up temporary directory
    fs.rmSync(uploadDir, { recursive: true, force: true });
    
  } catch (error) {
    logger.error(`Processing error for upload ${uploadId}: ${error.message}`);
    
    // Update status to failed
    await redisClient.hSet(`upload:${uploadId}`, {
      status: 'failed',
      error: error.message
    });
    
    // Try to clean up
    try {
      const uploadDir = path.join(config.tempDir, uploadId);
      if (fs.existsSync(uploadDir)) {
        fs.rmSync(uploadDir, { recursive: true, force: true });
      }
    } catch (cleanupError) {
      logger.error(`Failed to clean up after error: ${cleanupError.message}`);
    }
  }
}

module.exports = router;
EOL
```

## Step 7: Create Authentication Middleware

```bash
cat > upload-manager/src/middlewares/auth.js << 'EOL'
const jwt = require('jsonwebtoken');
const logger = require('../../../shared/logger');

/**
 * Authentication middleware
 * @param {Object} options - Options for the middleware
 * @returns {Function} Express middleware
 */
function authMiddleware(options = {}) {
  const {
    required = true,
    roles = null
  } = options;
  
  return async (req, res, next) => {
    try {
      const authHeader = req.headers.authorization;
      
      if (!authHeader || !authHeader.startsWith('Bearer ')) {
        if (required) {
          return res.status(401).json({ error: 'Authentication required' });
        } else {
          // Allow anonymous access for optional auth
          return next();
        }
      }
      
      const token = authHeader.split(' ')[1];
      const secret = process.env.JWT_SECRET;
      
      if (!secret) {
        logger.error('JWT_SECRET is not defined in environment');
        return res.status(500).json({ error: 'Server configuration error' });
      }
      
      // Verify token
      try {
        const decoded = jwt.verify(token, secret);
        req.user = decoded;
        
        // Check role if required
        if (roles && !roles.includes(decoded.role)) {
          return res.status(403).json({ 
            error: 'Access denied. Insufficient permissions' 
          });
        }
        
        next();
      } catch (error) {
        if (error.name === 'TokenExpiredError') {
          return res.status(401).json({ error: 'Token expired' });
        } else {
          return res.status(401).json({ error: 'Invalid token' });
        }
      }
    } catch (error) {
      logger.error(`Auth middleware error: ${error.message}`);
      res.status(500).json({ error: 'Authentication error' });
    }
  };
}

module.exports = authMiddleware;
EOL
```

## Step 8: Add Helper Scripts for Running Services

```bash
# Add script to package.json for running all services
cat > package.json << 'EOL'
{
  "name": "media-management-service",
  "version": "1.0.0",
  "description": "Media Management Service",
  "scripts": {
    "setup": "sh scripts/setup.sh",
    "lint": "eslint --ext .js .",
    "format": "prettier --write \"**/*.js\"",
    "start:api": "cd api-gateway && npm start",
    "start:upload": "cd upload-manager && npm start",
    "start:processing": "cd processing-pipeline && npm start",
    "start:streaming": "cd streaming-engine && npm start",
    "start:content": "cd content-management && npm start",
    "start:all": "concurrently \"npm:start:upload\" \"npm:start:processing\" \"npm:start:streaming\" \"npm:start:content\"",
    "dev:api": "cd api-gateway && npm run dev",
    "dev:upload": "cd upload-manager && npm run dev",
    "dev:processing": "cd processing-pipeline && npm run dev",
    "dev:streaming": "cd streaming-engine && npm run dev",
    "dev:content": "cd content-management && npm run dev",
    "dev": "concurrently \"npm:dev:upload\" \"npm:dev:processing\" \"npm:dev:streaming\" \"npm:dev:content\"",
    "cleanup": "node scripts/cleanup-temp.js",
    "token": "node scripts/generate-token.js"
  },
  "keywords": [
    "media",
    "streaming",
    "upload"
  ],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "concurrently": "^7.6.0",
    "eslint": "^8.33.0",
    "prettier": "^2.8.3",
    "nodemon": "^2.0.20",
    "jest": "^29.4.1"
  }
}
EOL

# Install concurrently for running multiple services
npm install --save-dev concurrently
```

## Step 9: Run the Setup and Start Development

```bash
# Run the setup script
./scripts/setup.sh

# Install nodemon globally for development
npm install -g nodemon

# Start the Upload Manager service in development mode
cd upload-manager
npm run dev

# In a new terminal, start the Processing Pipeline
cd processing-pipeline
npm run dev

# In a new terminal, start the Streaming Engine
cd streaming-engine
npm run dev

# In a new terminal, start the Content Management service
cd content-management
npm run dev
```

## Testing the System

Follow the testing guides provided in the previous artifacts. With the enhanced structure and utilities, you can use the same testing approaches but with more robust error handling and logging.

## Next Steps

- Implement the remaining services (Processing Pipeline, Streaming Engine, Content Management)
- Add comprehensive unit and integration tests
- Integrate with a CI/CD pipeline
- Set up monitoring and alerting for production use

This improved guide provides a more robust foundation for your Media Management Service with better error handling, logging, configuration, and development practices.
