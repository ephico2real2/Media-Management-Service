# Enhanced Testing and Validation Guide

This comprehensive guide provides detailed instructions for testing, validating, and monitoring your Media Management Service. It covers manual testing, automated testing, performance evaluation, and monitoring strategies to ensure your service is reliable and production-ready.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Manual Testing](#manual-testing)
3. [Automated Testing with Jest](#automated-testing-with-jest)
4. [API Testing with Postman/Newman](#api-testing-with-postmannewman)
5. [CI/CD Integration with GitHub Actions](#cicd-integration-with-github-actions)
6. [Load Testing with Artillery](#load-testing-with-artillery)
7. [Security Testing](#security-testing)
8. [Streaming and Playback Testing](#streaming-and-playback-testing)
9. [Monitoring and Observability](#monitoring-and-observability)
10. [Troubleshooting Common Issues](#troubleshooting-common-issues)

## Prerequisites

Before beginning the testing process, ensure all dependencies are installed and the services are running:

```bash
# Install testing dependencies
npm install --save-dev jest supertest newman artillery

# Install global tools
npm install -g newman artillery

# Install monitoring tools
npm install winston prom-client
```

## Manual Testing

### 1. Testing Basic Upload Flow

#### 1.1 Upload Initialization

```bash
# Initialize an upload
curl -X POST http://localhost:3001/api/uploads/init \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "test-video.mp4",
    "fileSize": 1000000,
    "fileType": "video/mp4",
    "metadata": {
      "title": "Test Video",
      "description": "Test upload"
    }
  }'
```

Expected response:
```json
{
  "uploadId": "abc123...",
  "chunkSize": 5242880,
  "expiresAt": "..."
}
```

#### 1.2 Testing with HTML Upload Interface

Create a file `test-upload.html` with the code provided in the previous guide section and open it in your browser to test the full upload flow.

### 2. Testing Processing Pipeline

After uploading a file, check the MongoDB for processing status:

```bash
mongosh
> use media-service
> db["media-assets"].findOne({})
```

Look for:
- Variants with different resolutions
- Thumbnail URLs
- Streaming URLs
- Processing completion status

### 3. Testing Streaming

Generate a JWT token for testing:

```bash
node scripts/generate-token.js test-user
```

Test with curl:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:3002/media/ASSET_ID
```

Use the `video-player.html` file from the previous guide to test playback in the browser.

## Automated Testing with Jest

### 1. Setting Up Jest for Integration Testing

Create a `jest.config.js` file at the project root:

```javascript
module.exports = {
  testEnvironment: 'node',
  testMatch: [
    '**/tests/**/*.test.js',
    '**/?(*.)+(spec|test).js'
  ],
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov'],
  verbose: true,
  setupFilesAfterEnv: ['./jest.setup.js']
};
```

Create `jest.setup.js`:

```javascript
jest.setTimeout(30000); // 30 seconds timeout for all tests

// Global setup for all tests
beforeAll(async () => {
  // Initialize connections, etc.
  console.log('Setting up tests...');
});

// Global teardown
afterAll(async () => {
  // Close connections, cleanup, etc.
  console.log('Tearing down tests...');
});
```

### 2. Upload Manager Tests

Create `upload-manager/tests/upload-init.test.js`:

```javascript
const request = require('supertest');
const app = require('../src/app'); // Ensure your app exports the Express app
const { connect: connectRedis, close: closeRedis } = require('../../shared/redis-client');

describe('Upload Manager API', () => {
  beforeAll(async () => {
    await connectRedis();
  });

  afterAll(async () => {
    await closeRedis();
  });

  describe('POST /api/uploads/init', () => {
    it('should initialize an upload session with valid data', async () => {
      const res = await request(app)
        .post('/api/uploads/init')
        .send({
          filename: 'test-video.mp4',
          fileSize: 1024 * 1024,
          fileType: 'video/mp4',
          metadata: {
            title: 'Test Video',
            description: 'Test upload'
          }
        });

      expect(res.statusCode).toBe(201);
      expect(res.body).toHaveProperty('uploadId');
      expect(res.body).toHaveProperty('chunkSize');
      expect(res.body).toHaveProperty('expiresAt');
    });

    it('should reject initialization with missing required fields', async () => {
      const res = await request(app)
        .post('/api/uploads/init')
        .send({
          filename: 'test-video.mp4'
          // Missing fileSize and fileType
        });

      expect(res.statusCode).toBe(400);
      expect(res.body).toHaveProperty('error');
    });

    it('should reject uploads with invalid file types', async () => {
      const res = await request(app)
        .post('/api/uploads/init')
        .send({
          filename: 'malicious.exe',
          fileSize: 1024 * 1024,
          fileType: 'application/x-msdownload'
        });

      expect(res.statusCode).toBe(400);
      expect(res.body).toHaveProperty('error');
    });
  });

  describe('GET /api/uploads/:uploadId', () => {
    let uploadId;

    beforeAll(async () => {
      // Create a test upload first
      const res = await request(app)
        .post('/api/uploads/init')
        .send({
          filename: 'test-video.mp4',
          fileSize: 1024 * 1024,
          fileType: 'video/mp4'
        });

      uploadId = res.body.uploadId;
    });

    it('should return upload status for valid upload ID', async () => {
      const res = await request(app)
        .get(`/api/uploads/${uploadId}`);

      expect(res.statusCode).toBe(200);
      expect(res.body).toHaveProperty('status');
      expect(res.body).toHaveProperty('fileSize');
      expect(res.body).toHaveProperty('fileType');
    });

    it('should return 404 for non-existent upload ID', async () => {
      const res = await request(app)
        .get('/api/uploads/non-existent-id');

      expect(res.statusCode).toBe(404);
    });
  });
});
```

### 3. Processing Pipeline Tests

Create `processing-pipeline/tests/ffmpeg-utils.test.js`:

```javascript
const { analyzeMedia, generateThumbnail } = require('../src/utils/ffmpeg-utils');
const fs = require('fs-extra');
const path = require('path');
const os = require('os');

describe('FFmpeg Utils', () => {
  const testVideoPath = path.join(os.tmpdir(), 'test-video.mp4');
  const tempDir = path.join(os.tmpdir(), 'test-processing');
  
  beforeAll(async () => {
    // Ensure test video exists
    if (!fs.existsSync(testVideoPath)) {
      // Download test video if needed
      // For unit tests, we could just use a very small test file
      fs.copyFileSync(
        path.join(__dirname, '../../../test-assets/test-video.mp4'), 
        testVideoPath
      );
    }
    
    fs.ensureDirSync(tempDir);
  });
  
  afterAll(() => {
    // Clean up
    fs.removeSync(tempDir);
  });
  
  it('should analyze media file correctly', async () => {
    const result = await analyzeMedia(testVideoPath);
    
    expect(result).toHaveProperty('format');
    expect(result).toHaveProperty('streams');
    expect(result.format).toHaveProperty('duration');
    
    // Find video stream
    const videoStream = result.streams.find(s => s.codec_type === 'video');
    expect(videoStream).toBeDefined();
    expect(videoStream).toHaveProperty('width');
    expect(videoStream).toHaveProperty('height');
  });
  
  it('should generate thumbnail from video', async () => {
    const thumbnailPath = await generateThumbnail(testVideoPath, tempDir);
    
    expect(fs.existsSync(thumbnailPath)).toBe(true);
    
    // Verify it's an image file
    const stats = fs.statSync(thumbnailPath);
    expect(stats.size).toBeGreaterThan(0);
  });
});
```

### 4. Content Management Tests

Create `content-management/tests/asset-routes.test.js`:

```javascript
const request = require('supertest');
const app = require('../src/app');
const { connect: connectMongo, close: closeMongo } = require('../../shared/db-client');
const jwt = require('jsonwebtoken');

describe('Content Management API', () => {
  let authToken;
  let testAssetId;
  
  beforeAll(async () => {
    await connectMongo();
    
    // Generate test auth token
    authToken = jwt.sign({ id: 'test-user' }, process.env.JWT_SECRET || 'test-secret', { expiresIn: '1h' });
    
    // Create a test asset in the database
    const db = require('../../shared/db-client').getDb();
    const result = await db.collection('media-assets').insertOne({
      userId: 'test-user',
      filename: 'test-video.mp4',
      fileType: 'video/mp4',
      status: 'ready',
      metadata: {
        title: 'Test Video',
        description: 'Test asset'
      },
      thumbnailUrl: 'http://example.com/thumbnail.jpg',
      streamingUrl: 'http://example.com/manifest.m3u8',
      createdAt: new Date()
    });
    
    testAssetId = result.insertedId.toString();
  });
  
  afterAll(async () => {
    // Clean up test data
    const db = require('../../shared/db-client').getDb();
    await db.collection('media-assets').deleteOne({ _id: testAssetId });
    
    await closeMongo();
  });
  
  describe('GET /api/assets', () => {
    it('should return assets for authenticated user', async () => {
      const res = await request(app)
        .get('/api/assets')
        .set('Authorization', `Bearer ${authToken}`);
      
      expect(res.statusCode).toBe(200);
      expect(res.body).toHaveProperty('assets');
      expect(Array.isArray(res.body.assets)).toBe(true);
      expect(res.body.assets.length).toBeGreaterThan(0);
    });
    
    it('should reject unauthenticated requests', async () => {
      const res = await request(app)
        .get('/api/assets');
      
      expect(res.statusCode).toBe(401);
    });
  });
  
  describe('GET /api/assets/:id', () => {
    it('should return a specific asset', async () => {
      const res = await request(app)
        .get(`/api/assets/${testAssetId}`)
        .set('Authorization', `Bearer ${authToken}`);
      
      expect(res.statusCode).toBe(200);
      expect(res.body).toHaveProperty('id', testAssetId);
      expect(res.body).toHaveProperty('title');
      expect(res.body).toHaveProperty('status');
    });
    
    it('should return 404 for non-existent asset', async () => {
      const res = await request(app)
        .get('/api/assets/non-existent-id')
        .set('Authorization', `Bearer ${authToken}`);
      
      expect(res.statusCode).toBe(404);
    });
  });
});
```

Add the tests to your package.json:

```json
"scripts": {
  "test": "jest",
  "test:watch": "jest --watch",
  "test:coverage": "jest --coverage"
}
```

## API Testing with Postman/Newman

### 1. Create a Postman Collection

1. Download and install Postman from [https://www.postman.com/downloads/](https://www.postman.com/downloads/)
2. Create a new Collection named "Media Management Service"
3. Add the following request folders:
   - Upload Manager
   - Processing Pipeline
   - Streaming Engine
   - Content Management

### 2. Create Environment Variables

Create an environment with variables:
- `baseUrl`: http://localhost:3001
- `streamingUrl`: http://localhost:3002
- `contentUrl`: http://localhost:3003
- `authToken`: (leave empty initially)
- `uploadId`: (leave empty initially)
- `assetId`: (leave empty initially)

### 3. Add Test Requests

Add the following requests:

#### Upload Manager Requests
1. **Initialize Upload**
   - Method: POST
   - URL: {{baseUrl}}/api/uploads/init
   - Body (JSON):
     ```json
     {
       "filename": "test-video.mp4",
       "fileSize": 1048576,
       "fileType": "video/mp4",
       "metadata": {
         "title": "Test Video",
         "description": "API test upload"
       }
     }
     ```
   - Tests:
     ```javascript
     pm.test("Status code is 201", function () {
       pm.response.to.have.status(201);
     });
     
     pm.test("Response contains uploadId", function () {
       var jsonData = pm.response.json();
       pm.expect(jsonData).to.have.property('uploadId');
       pm.environment.set("uploadId", jsonData.uploadId);
     });
     ```

2. **Get Upload Status**
   - Method: GET
   - URL: {{baseUrl}}/api/uploads/{{uploadId}}
   - Tests:
     ```javascript
     pm.test("Status code is 200", function () {
       pm.response.to.have.status(200);
     });
     
     pm.test("Response contains status", function () {
       var jsonData = pm.response.json();
       pm.expect(jsonData).to.have.property('status');
     });
     ```

#### Streaming Engine Requests
1. **Generate Auth Token** (Pre-request Script):
   - Add a request with this pre-request script:
     ```javascript
     const jwt = require('jsonwebtoken');
     const token = jwt.sign({ id: 'test-user' }, 'your-secret-key-change-me', { expiresIn: '1h' });
     pm.environment.set("authToken", token);
     ```

2. **Get Asset**
   - Method: GET
   - URL: {{streamingUrl}}/media/{{assetId}}
   - Headers:
     - Authorization: Bearer {{authToken}}
   - Tests:
     ```javascript
     pm.test("Status code is 200", function () {
       pm.response.to.have.status(200);
     });
     ```

3. **Get Streaming URL**
   - Method: GET
   - URL: {{streamingUrl}}/media/{{assetId}}/stream
   - Headers:
     - Authorization: Bearer {{authToken}}
   - Tests:
     ```javascript
     pm.test("Status code is 200", function () {
       pm.response.to.have.status(200);
     });
     
     pm.test("Response contains streaming URLs", function () {
       var jsonData = pm.response.json();
       pm.expect(jsonData).to.have.property('masterUrl');
     });
     ```

### 4. Export and Run with Newman

Export the collection as `media-service-tests.postman_collection.json` and the environment as `media-service-environment.json`.

Run with Newman:

```bash
newman run media-service-tests.postman_collection.json -e media-service-environment.json
```

For HTML reports:

```bash
npm install -g newman-reporter-htmlextra
newman run media-service-tests.postman_collection.json -e media-service-environment.json -r htmlextra
```

## CI/CD Integration with GitHub Actions

### 1. Setup GitHub Actions Workflow

Create `.github/workflows/test.yml`:

```yaml
name: Media Management Service Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mongodb:
        image: mongo:5
        ports:
          - 27017:27017
      redis:
        image: redis:7
        ports:
          - 6379:6379
      rabbitmq:
        image: rabbitmq:3-management
        ports:
          - 5672:5672
          - 15672:15672

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 16
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install FFmpeg
        run: |
          sudo apt-get update
          sudo apt-get install -y ffmpeg

      - name: Setup MinIO
        run: |
          docker run -d -p 9000:9000 -p 9001:9001 --name minio \
            -e "MINIO_ROOT_USER=minioadmin" \
            -e "MINIO_ROOT_PASSWORD=minioadmin" \
            minio/minio server /data --console-address ":9001"
          sleep 5
          docker run --network host --rm minio/mc config host add local http://localhost:9000 minioadmin minioadmin
          docker run --network host --rm minio/mc mb --ignore-existing local/media-assets
          docker run --network host --rm minio/mc policy set download local/media-assets

      - name: Run Jest tests
        run: npm test

      - name: Download test file
        run: curl -o test-video.mp4 https://test-videos.co.uk/vids/bigbuckbunny/mp4/h264/720/Big_Buck_Bunny_720_10s_1MB.mp4

      - name: Run upload test
        run: node scripts/test-upload.js ./test-video.mp4

      - name: Run API tests with Newman
        run: |
          npm install -g newman newman-reporter-htmlextra
          newman run media-service-tests.postman_collection.json -e media-service-environment.json -r htmlextra,cli

      - name: Upload test reports
        uses: actions/upload-artifact@v2
        if: always()
        with:
          name: test-reports
          path: |
            newman/
            coverage/
```

### 2. Upload Test Script

Create `scripts/test-upload.js`:

```javascript
const fs = require('fs');
const path = require('path');
const axios = require('axios');
const FormData = require('form-data');

const UPLOAD_API_URL = process.env.UPLOAD_API_URL || 'http://localhost:3001/api/uploads';
const FILE_PATH = process.argv[2];

if (!FILE_PATH || !fs.existsSync(FILE_PATH)) {
  console.error(`File not found: ${FILE_PATH}`);
  process.exit(1);
}

const fileName = path.basename(FILE_PATH);
const fileSize = fs.statSync(FILE_PATH).size;
const fileType = fileName.endsWith('.mp4') ? 'video/mp4' : 'audio/mpeg';

async function uploadFile() {
  try {
    console.log(`Uploading ${fileName} (${(fileSize / 1024 / 1024).toFixed(2)} MB)`);
    
    // Step 1: Initialize upload
    console.log('\nInitializing upload...');
    const initResponse = await axios.post(`${UPLOAD_API_URL}/init`, {
      filename: fileName,
      fileSize: fileSize,
      fileType: fileType,
      metadata: {
        title: 'CI Test Upload',
        description: 'Uploaded by CI process'
      }
    });
    
    const { uploadId, chunkSize } = initResponse.data;
    console.log(`Upload initialized. ID: ${uploadId}, Chunk size: ${chunkSize} bytes`);
    
    // Step 2: Upload chunks
    const fileBuffer = fs.readFileSync(FILE_PATH);
    const totalChunks = Math.ceil(fileSize / chunkSize);
    console.log(`\nUploading ${totalChunks} chunks...`);
    
    for (let i = 0; i < totalChunks; i++) {
      const start = i * chunkSize;
      const end = Math.min(fileSize, start + chunkSize);
      const chunk = fileBuffer.slice(start, end);
      
      const formData = new FormData();
      formData.append('chunk', Buffer.from(chunk), {
        filename: `chunk-${i}`,
        contentType: 'application/octet-stream'
      });
      formData.append('chunkNumber', i);
      formData.append('totalChunks', totalChunks);
      
      const uploadResponse = await axios.post(
        `${UPLOAD_API_URL}/${uploadId}/chunks`,
        formData,
        { headers: formData.getHeaders() }
      );
      
      console.log(`Chunk ${i+1}/${totalChunks} uploaded (${Math.round((end - start) / 1024)} KB)`);
    }
    
    // Step 3: Monitor processing
    console.log('\nMonitoring processing status...');
    let status = 'uploading';
    let attempts = 0;
    const maxAttempts = 10;
    
    while (!['complete', 'processed', 'failed', 'processing_failed'].includes(status) && attempts < maxAttempts) {
      await new Promise(resolve => setTimeout(resolve, 2000));
      attempts++;
      
      try {
        const statusResponse = await axios.get(`${UPLOAD_API_URL}/${uploadId}`);
        status = statusResponse.data.status;
        
        console.log(`Current status: ${status} (${statusResponse.data.progress || 0}%)`);
        
        if (statusResponse.data.assetId) {
          console.log(`Asset ID: ${statusResponse.data.assetId}`);
          // Success criteria for CI
          process.exit(0);
        }
        
        if (statusResponse.data.error) {
          console.error(`Error: ${statusResponse.data.error}`);
          process.exit(1);
        }
      } catch (error) {
        console.error(`Error checking status: ${error.message}`);
      }
    }
    
    if (attempts >= maxAttempts) {
      console.error('Max attempts reached while checking status');
      process.exit(1);
    }
    
    if (status === 'failed' || status === 'processing_failed') {
      console.error(`Upload failed with status: ${status}`);
      process.exit(1);
    }
    
    console.log('\nUpload process completed successfully!');
    process.exit(0);
    
  } catch (error) {
    console.error('Error:', error.response?.data || error.message);
    process.exit(1);
  }
}

uploadFile();
```

## Load Testing with Artillery

### 1. Create Artillery Configuration

Create `upload-load-test.yml`:

```yaml
config:
  target: "http://localhost:3001"
  phases:
    - duration: 30
      arrivalRate: 2
      rampTo: 10
      name: "Warm up phase"
    - duration: 60
      arrivalRate: 10
      name: "Sustained load"
  environments:
    production:
      target: "https://your-production-url.com"
      phases:
        - duration: 300
          arrivalRate: 50
          name: "Production test"
  plugins:
    expect: {}
  processor: "./load-test-functions.js"
scenarios:
  - name: "Initialize uploads"
    flow:
      - function: "generateFileName"
      - post:
          url: "/api/uploads/init"
          json:
            filename: "{{ filename }}"
            fileSize: 1048576
            fileType: "video/mp4"
            metadata:
              title: "Load Test Video"
              description: "Created during load testing"
          expect:
            - statusCode: 201
            - contentType: "application/json"
            - hasProperty: "uploadId"
          capture:
            - json: "$.uploadId"
              as: "uploadId"
      - get:
          url: "/api/uploads/{{ uploadId }}"
          expect:
            - statusCode: 200
            - hasProperty: "status"
```

Create `load-test-functions.js`:

```javascript
'use strict';

module.exports = {
  generateFileName
};

function generateFileName(userContext, events, done) {
  const timestamp = new Date().getTime();
  const random = Math.floor(Math.random() * 10000);
  userContext.vars.filename = `loadtest_${timestamp}_${random}.mp4`;
  return done();
}
```

### 2. Run Artillery Tests

```bash
artillery run upload-load-test.yml
```

For HTML report:

```bash
artillery run --output report.json upload-load-test.yml
artillery report report.json
```

## Security Testing

### 1. JWT Token Testing

Create `scripts/jwt-test.js`:

```javascript
require('dotenv').config();
const jwt = require('jsonwebtoken');
const axios = require('axios');

const STREAMING_URL = process.env.STREAMING_URL || 'http://localhost:3002';
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key-change-me';
const ASSET_ID = process.argv[2];

if (!ASSET_ID) {
  console.error('Please provide an asset ID as an argument');
  process.exit(1);
}

// Generate valid token
const validToken = jwt.sign({ id: 'test-user' }, JWT_SECRET, { expiresIn: '1h' });

// Generate expired token
const expiredToken = jwt.sign({ id: 'test-user' }, JWT_SECRET, { expiresIn: '-10s' });

// Generate token with wrong signature
const wrongSecretToken = jwt.sign({ id: 'test-user' }, 'wrong-secret', { expiresIn: '1h' });

// Generate token with tampered payload
const [header, payload, signature] = validToken.split('.');
const tamperedPayload = Buffer.from(JSON.stringify({
  id: 'admin-user',
  role: 'admin',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + 3600
})).toString('base64').replace(/=/g, '');
const tamperedToken = `${header}.${tamperedPayload}.${signature}`;

// Test cases
const testCases = [
  { name: 'Valid token', token: validToken, expectedStatus: 200 },
  { name: 'Expired token', token: expiredToken, expectedStatus: 401 },
  { name: 'Wrong signature', token: wrongSecretToken, expectedStatus: 401 },
  { name: 'Tampered payload', token: tamperedToken, expectedStatus: 401 },
  { name: 'No token', token: null, expectedStatus: 401 }
];

async function runTests() {
  let passed = 0;
  let failed = 0;
  
  for (const test of testCases) {
    try {
      const headers = test.token 
        ? { Authorization: `Bearer ${test.token}` }
        : {};
      
      const response = await axios.get(`${STREAMING_URL}/media/${ASSET_ID}`, { 
        headers,
        validateStatus: () => true // Don't throw on non-2xx
      });
      
      if (response.status === test.expectedStatus) {
        console.log(`✅ ${test.name}: PASSED (Got ${response.status} as expected)`);
        passed++;
      } else {
        console.log(`❌ ${test.name}: FAILED (Expected ${test.expectedStatus}, got ${response.status})`);
        console.log(`   Response: ${JSON.stringify(response.data)}`);
        failed++;
      }
    } catch (error) {
      console.log(`❌ ${test.name}: ERROR (${error.message})`);
      failed++;
    }
  }
  
  console.log(`\nResults: ${passed} passed, ${failed} failed`);
  process.exit(failed > 0 ? 1 : 0);
}

runTests();
```

Run the tests:

```bash
node scripts/jwt-test.js YOUR_ASSET_ID
```

### 2. Rate Limiting Test

Create `scripts/rate-limit-test.js`:

```javascript
const axios = require('axios');

const API_URL = process.env.API_URL || 'http://localhost:3001';
const NUM_REQUESTS = 150; // Should exceed your rate limit

async function testRateLimit() {
  console.log(`Sending ${NUM_REQUESTS} requests to test rate limiting...`);
  
  const results = {
    success: 0,
    rateLimited: 0,
    errors: 0
  };
  
  const requests = [];
  
  for (let i = 0; i < NUM_REQUESTS; i++) {
    requests.push(axios.post(`${API_URL}/api/uploads/init`, {
      filename: `rate-limit-test-${i}.mp4`,
      fileSize: 1048576,
      fileType: 'video/mp4'
    }, {
      validateStatus: () => true // Don't throw on non-2xx
    }));
  }
  
  const responses = await Promise.all(requests);
  
  for (const response of responses) {
    if (response.status === 200 || response.status === 201) {
      results.success++;
    } else if (response.status === 429) {
      results.rateLimited++;
    } else {
      results.errors++;
    }
  }
  
  console.log('\nRate Limit Test Results:');
  console.log(`✅ Successful requests: ${results.success}`);
  console.log(`🛑 Rate limited requests: ${results.rateLimited}`);
  console.log(`❌ Other errors: ${results.errors}`);
  
  if (results.rateLimited > 0) {
    console.log('\nRate limiting is working as expected! 👍');
  } else {
    console.log('\nWarning: No requests were rate limited. Check your rate limit configuration.');
  }
}

testRateLimit();
```

Run the test:

```bash
node scripts/rate-limit-test.js
```

## Streaming and Playback Testing

### 1. Enhanced Video Player Test

Create `enhanced-video-player.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Enhanced Media Player Test</title>
  <link href="https://unpkg.com/video.js/dist/video-js.css" rel="stylesheet">
  <script src="https://unpkg.com/video.js/dist/video.js"></script>
  <script src="https://unpkg.com/videojs-contrib-hls/dist/videojs-contrib-hls.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 0; padding: 20px; background-color: #f5f5f5; }
    .container { max-width: 800px; margin: 0 auto; background-color: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    .player-container { margin-top: 20px; }
    .video-js { width: 100%; height: 450px; }
    .controls { margin-top: 20px; }
    input, button, select { padding: 8px; margin-right: 10px; margin-bottom: 10px; }
    button { background-color: #4CAF50; color: white; border: none; border-radius: 4px; cursor: pointer; }
    button:hover { background-color: #45a049; }
    .status { margin-top: 20px; padding: 10px; border: 1px solid #ddd; border-radius: 4px; background-color: #f9f9f9; }
    .error { color: #d32f2f; background-color: #ffebee; padding: 10px; border-radius: 4px; margin-top: 10px; display: none; }
    .profiles { margin-top: 15px; }
    .playback-stats { margin-top: 15px; font-size: 0.9em; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Enhanced Media Player Test</h1>
    
    <div class="controls">
      <input type="text" id="assetId" placeholder="Asset ID">
      <input type="text" id="tokenInput" placeholder="JWT Token">
      <button onclick="loadVideo()">Load Video</button>
      <div class="error" id="errorMessage"></div>
      
      <div class="profiles">
        <label for="qualitySelect">Quality:</label>
        <select id="qualitySelect" disabled>
          <option value="auto">Auto</option>
          <option value="1080p">1080p</option>
          <option value="720p">720p</option>
          <option value="480p">480p</option>
          <option value="360p">360p</option>
        </select>
        
        <button id="togglePlaybackStats" onclick="toggleStats()">Show Playback Stats</button>
        <button onclick="simulateNetworkChange()">Simulate Network Change</button>
      </div>
    </div>
    
    <div class="player-container">
      <video id="my-video" class="video-js vjs-default-skin" controls></video>
    </div>
    
    <div class="playback-stats" id="playbackStats" style="display: none">
      <div>Resolution: <span id="currentResolution">-</span></div>
      <div>Bitrate: <span id="currentBitrate">-</span> kbps</div>
      <div>Buffer: <span id="bufferLength">-</span> seconds</div>
      <div>Dropped Frames: <span id="droppedFrames">-</span></div>
    </div>
    
    <div class="status">
      <h3>Status:</h3>
      <pre id="status">No video loaded</pre>
    </div>
  </div>
  
  <script>
    let player;
    let statsInterval;
    let lastPosition = 0;
    let variants = {};
    
    document.addEventListener('DOMContentLoaded', function() {
      player = videojs('my-video', {
        html5: {
          hls: {
            overrideNative: true
          }
        }
      });
      
      // Add error handling
      player.on('error', function() {
        const error = player.error();
        showError(`Playback error: ${error.code} - ${error.message}`);
      });
      
      // Save position for resume capability
      player.on('timeupdate', function() {
        const currentTime = player.currentTime();
        if (currentTime > 5) {
          lastPosition = currentTime;
          localStorage.setItem('lastVideoPosition', currentTime);
          localStorage.setItem('lastVideoAsset', document.getElementById('assetId').value);
        }
      });
      
      // Add quality selection handling
      document.getElementById('qualitySelect').addEventListener('change', function() {
        const quality = this.value;
        if (quality === 'auto') {
          player.src({
            src: variants.masterUrl,
            type: 'application/x-mpegURL'
          });
        } else if (variants[quality]) {
          player.src({
            src: variants[quality],
            type: 'video/mp4'
          });
        }
        
        // Resume from last position
        const currentPosition = player.currentTime();
        player.one('loadedmetadata', function() {
          player.currentTime(currentPosition);
          player.play();
        });
      });
    });
    
    async function loadVideo() {
      const assetId = document.getElementById('assetId').value;
      const token = document.getElementById('tokenInput').value;
      
      if (!assetId) {
        showError('Please enter an Asset ID');
        return;
      }
      
      if (!token) {
        showError('Please enter a JWT token');
        return;
      }
      
      try {
        hideError();
        document.getElementById('status').textContent = 'Fetching stream information...';
        
        // Check token validity
        try {
          const tokenParts = token.split('.');
          if (tokenParts.length !== 3) {
            throw new Error('Invalid token format');
          }
          
          const payload = JSON.parse(atob(tokenParts[1]));
          const currentTime = Math.floor(Date.now() / 1000);
          
          if (payload.exp && payload.exp < currentTime) {
            throw new Error('Token has expired');
          }
        } catch (e) {
          showError(`Token validation error: ${e.message}`);
          return;
        }
        
        // Fetch streaming URL
        const response = await fetch(`http://localhost:3002/media/${assetId}/stream`, {
          headers: {
            'Authorization': `Bearer ${token}`
          }
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(`Error ${response.status}: ${errorData.error || response.statusText}`);
        }
        
        const data = await response.json();
        document.getElementById('status').textContent = JSON.stringify(data, null, 2);
        
        // Store variants for quality selection
        variants = data.variants;
        variants.masterUrl = data.masterUrl;
        
        // Enable quality selection
        const qualitySelect = document.getElementById('qualitySelect');
        qualitySelect.disabled = false;
        
        // Clear existing options
        while (qualitySelect.options.length > 1) {
          qualitySelect.remove(1);
        }
        
        // Add available qualities
        for (const [profile, url] of Object.entries(variants)) {
          if (profile !== 'masterUrl') {
            const option = document.createElement('option');
            option.value = profile;
            option.textContent = profile;
            qualitySelect.appendChild(option);
          }
        }
        
        // Check for resume capability
        const lastAsset = localStorage.getItem('lastVideoAsset');
        const lastPos = parseFloat(localStorage.getItem('lastVideoPosition') || 0);
        
        let startPosition = 0;
        if (lastAsset === assetId && lastPos > 5) {
          if (confirm(`Resume playback from ${Math.floor(lastPos / 60)}:${Math.floor(lastPos % 60).toString().padStart(2, '0')}?`)) {
            startPosition = lastPos;
          }
        }
        
        // Load the video
        player.src({
          src: data.masterUrl,
          type: 'application/x-mpegURL'
        });
        
        // Set initial position if resuming
        if (startPosition > 0) {
          player.one('loadedmetadata', function() {
            player.currentTime(startPosition);
          });
        }
        
        player.play();
        
        // Start stats monitoring
        startStatsMonitoring();
        
      } catch (error) {
        console.error('Error loading video:', error);
        document.getElementById('status').textContent = error.message;
        showError(error.message);
      }
    }
    
    function simulateNetworkChange() {
      // Simulate network conditions changing to test ABR
      if (player && player.tech && player.tech_.hls) {
        // Force a quality level change
        const levels = player.tech_.hls.representations();
        if (levels && levels.length > 1) {
          // Switch to lowest quality
          player.tech_.hls.selectRepresentation(levels[0]);
          console.log('Switched to lowest quality to simulate network degradation');
          
          // After 5 seconds, return to auto
          setTimeout(() => {
            player.tech_.hls.selectRepresentation('auto');
            console.log('Returned to auto-quality selection');
          }, 5000);
        }
      }
    }
    
    function startStatsMonitoring() {
      if (statsInterval) {
        clearInterval(statsInterval);
      }
      
      statsInterval = setInterval(() => {
        if (!player || !player.readyState()) return;
        
        try {
          // Get current resolution
          const width = player.videoWidth();
          const height = player.videoHeight();
          document.getElementById('currentResolution').textContent = `${width}x${height}`;
          
          // Estimate bitrate (rough approximation)
          const tech = player.tech({ IWillNotUseThisInPlugins: true });
          if (tech.hls && tech.hls.bandwidth) {
            const bitrate = Math.round(tech.hls.bandwidth / 1000);
            document.getElementById('currentBitrate').textContent = bitrate;
          }
          
          // Get buffer length
          const bufferEnd = player.buffered().end(0) || 0;
          const currentTime = player.currentTime();
          const bufferLength = Math.round((bufferEnd - currentTime) * 10) / 10;
          document.getElementById('bufferLength').textContent = bufferLength;
          
          // Attempt to get dropped frames (only works in some browsers)
          if (player.getVideoPlaybackQuality) {
            const quality = player.getVideoPlaybackQuality();
            document.getElementById('droppedFrames').textContent = quality.droppedVideoFrames;
          }
        } catch (e) {
          console.warn('Error updating playback stats:', e);
        }
      }, 1000);
    }
    
    function toggleStats() {
      const statsElem = document.getElementById('playbackStats');
      const isVisible = statsElem.style.display !== 'none';
      
      statsElem.style.display = isVisible ? 'none' : 'block';
      document.getElementById('togglePlaybackStats').textContent = 
        isVisible ? 'Show Playback Stats' : 'Hide Playback Stats';
    }
    
    function showError(message) {
      const errorElem = document.getElementById('errorMessage');
      errorElem.textContent = message;
      errorElem.style.display = 'block';
    }
    
    function hideError() {
      document.getElementById('errorMessage').style.display = 'none';
    }
  </script>
</body>
</html>
```

### 2. ABR Testing

To test Adaptive Bitrate (ABR) streaming:

1. Open the enhanced video player in Chrome
2. Load a video
3. Open Chrome DevTools (F12)
4. Go to Network tab
5. Enable the "Disable cache" option
6. Use the Network Throttling dropdown to select "Slow 3G"
7. Observe the video quality automatically adjusting

You can also use the "Simulate Network Change" button in the enhanced player to force quality changes and test adaptation.

## Monitoring and Observability

### 1. Structured Logging with Winston

Install Winston:

```bash
npm install winston winston-daily-rotate-file
```

Create `shared/logger.js`:

```javascript
const winston = require('winston');
require('winston-daily-rotate-file');
const { format } = winston;

// Define log levels
const levels = {
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  debug: 4,
};

// Create the logger configuration
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  levels,
  format: format.combine(
    format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss:ms' }),
    format.errors({ stack: true }),
    format.json()
  ),
  defaultMeta: { 
    service: process.env.SERVICE_NAME || 'media-service',
    environment: process.env.NODE_ENV || 'development'
  },
  transports: [
    // Console transport for all environments
    new winston.transports.Console({
      format: format.combine(
        format.colorize({ all: true }),
        format.printf(
          (info) => `[${info.timestamp}] ${info.level}: ${info.message} ${info.stack || ''}`
        )
      )
    }),
    
    // Rotate file transports for production
    ...(process.env.NODE_ENV === 'production' ? [
      new winston.transports.DailyRotateFile({
        filename: 'logs/%DATE%-app.log',
        datePattern: 'YYYY-MM-DD',
        maxSize: '20m',
        maxFiles: '14d'
      }),
      new winston.transports.DailyRotateFile({
        filename: 'logs/%DATE%-error.log',
        datePattern: 'YYYY-MM-DD',
        level: 'error',
        maxSize: '20m',
        maxFiles: '30d'
      })
    ] : [])
  ]
});

// Create a stream object with a 'write' function that will be used by morgan
logger.stream = {
  write: (message) => logger.http(message.trim()),
};

module.exports = logger;
```

### 2. Metrics with Prometheus

Install Prometheus client:

```bash
npm install prom-client
```

Create `shared/metrics.js`:

```javascript
const promClient = require('prom-client');
const logger = require('./logger');

// Create a Registry to register metrics
const register = new promClient.Registry();

// Add default metrics
promClient.collectDefaultMetrics({ register });

// Create custom metrics
const httpRequestDurationMicroseconds = new promClient.Histogram({
  name: 'http_request_duration_ms',
  help: 'Duration of HTTP requests in ms',
  labelNames: ['method', 'route', 'status'],
  buckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const uploadSizeBytes = new promClient.Histogram({
  name: 'upload_size_bytes',
  help: 'Size of uploaded files in bytes',
  buckets: [1024, 1024 * 1024, 10 * 1024 * 1024, 100 * 1024 * 1024, 1024 * 1024 * 1024]
});

const uploadDurationSeconds = new promClient.Histogram({
  name: 'upload_duration_seconds',
  help: 'Duration of uploads in seconds',
  buckets: [1, 5, 10, 30, 60, 300, 600]
});

const processingDurationSeconds = new promClient.Histogram({
  name: 'processing_duration_seconds',
  help: 'Duration of media processing in seconds',
  buckets: [1, 5, 10, 30, 60, 300, 600, 1800]
});

// Register the metrics
register.registerMetric(httpRequestDurationMicroseconds);
register.registerMetric(httpRequestTotal);
register.registerMetric(uploadSizeBytes);
register.registerMetric(uploadDurationSeconds);
register.registerMetric(processingDurationSeconds);

// Express middleware to measure request duration
const requestDurationMiddleware = (req, res, next) => {
  const start = Date.now();
  
  // The 'finish' event will fire when the response is done sending
  res.on('finish', () => {
    const duration = Date.now() - start;
    const route = req.route ? req.route.path : req.path;
    
    // Update metrics
    try {
      httpRequestDurationMicroseconds
        .labels(req.method, route, res.statusCode)
        .observe(duration);
      
      httpRequestTotal
        .labels(req.method, route, res.statusCode)
        .inc();
        
      // Record upload size if applicable
      if (req.path.includes('/uploads') && req.method === 'POST' && req.body?.fileSize) {
        uploadSizeBytes.observe(parseInt(req.body.fileSize));
      }
    } catch (err) {
      logger.warn(`Error updating metrics: ${err.message}`);
    }
  });
  
  next();
};

// Export the middleware and registry
module.exports = {
  register,
  requestDurationMiddleware,
  metrics: {
    httpRequestDurationMicroseconds,
    httpRequestTotal,
    uploadSizeBytes,
    uploadDurationSeconds,
    processingDurationSeconds
  }
};
```

Add metrics endpoint to your Express app:

```javascript
const { register, requestDurationMiddleware } = require('../../shared/metrics');

// Add middleware
app.use(requestDurationMiddleware);

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  try {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
  } catch (err) {
    res.status(500).end(err.message);
  }
});
```

## Troubleshooting Common Issues

### 1. Upload Issues

- **Problem**: Chunks not being processed
  **Solution**: Check Redis for upload status, verify temp directory permissions

  ```bash
  # Check Redis keys for upload
  redis-cli keys "upload:*"
  redis-cli hgetall "upload:YOUR_UPLOAD_ID"
  
  # Check temp directory permissions
  ls -la temp/uploads
  sudo chmod -R 755 temp/uploads
  ```

- **Problem**: Upload hanging at the end
  **Solution**: Check RabbitMQ connectivity, ensure messages are being consumed

  ```bash
  # Check RabbitMQ queues
  rabbitmqctl list_queues
  
  # Check RabbitMQ connections
  rabbitmqctl list_connections
  
  # Restart RabbitMQ if needed
  sudo systemctl restart rabbitmq-server
  ```

### 2. Processing Issues

- **Problem**: FFmpeg errors
  **Solution**: Check FFmpeg installation, verify input file validity

  ```bash
  # Verify FFmpeg installation
  ffmpeg -version
  
  # Test FFmpeg with the input file directly
  ffmpeg -i your-file.mp4 -t 5 test-output.mp4
  ```

- **Problem**: S3 upload failures
  **Solution**: Verify MinIO is running, check credentials and bucket permissions

  ```bash
  # Check MinIO status
  curl -s http://localhost:9000/minio/health/live
  
  # List buckets with MinIO client
  mc ls local
  
  # Check bucket permissions
  mc policy info local/media-assets
  ```

### 3. Streaming Issues

- **Problem**: HLS manifest not loading
  **Solution**: Check CORS settings, verify manifest exists in MinIO

  ```bash
  # Set correct CORS policy for MinIO
  mc admin policy set local cors='{"allow_origins": ["*"]}'
  
  # Check if manifest file exists
  mc ls local/media-assets/assets/YOUR_ASSET_ID/
  ```

- **Problem**: Video stuttering
  **Solution**: Check network bandwidth, verify transcoding settings

  ```bash
  # Test direct download speed from MinIO
  time curl -o /dev/null http://localhost:9000/media-assets/assets/YOUR_ASSET_ID/720p.mp4
  
  # Check transcoding profile settings
  cat config/profiles/720p.json
  ```

### 4. Authentication Issues

- **Problem**: JWT token rejected
  **Solution**: Verify secret keys match across services

  ```bash
  # Check environment variables
  grep JWT_SECRET .env
  grep JWT_SECRET */src/*.js
  
  # Test token verification
  node scripts/jwt-test.js YOUR_ASSET_ID
  ```

### 5. Database Connectivity

- **Problem**: MongoDB connection failures
  **Solution**: Check MongoDB status and connection string

  ```bash
  # Check MongoDB status
  mongosh --eval "db.runCommand({ping:1})"
  
  # Check connection string format
  grep MONGODB_URI .env
  
  # Test connection directly
  mongosh "mongodb://localhost:27017/media-service"
  ```
