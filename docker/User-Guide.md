# Media Management Service: User Guide

## Table of Contents

1. [Getting Started](#getting-started)
   - [Prerequisites](#prerequisites)
   - [Setup and Installation](#setup-and-installation)
   - [Starting the Service](#starting-the-service)
   - [Stopping the Service](#stopping-the-service)

2. [Available Endpoints](#available-endpoints)
   - [API Gateway Endpoints](#api-gateway-endpoints)
   - [Upload Manager Endpoints](#upload-manager-endpoints)
   - [Streaming Engine Endpoints](#streaming-engine-endpoints)
   - [Content Management Endpoints](#content-management-endpoints)
   - [Status Checker Endpoints](#status-checker-endpoints)

3. [API Implementation Guide](#api-implementation-guide)
   - [Authentication](#authentication)
   - [File Upload API](#file-upload-api)
   - [Media Streaming API](#media-streaming-api)
   - [Content Management API](#content-management-api)
   - [Analytics API](#analytics-api)

4. [Common Use Cases](#common-use-cases)
   - [Uploading and Streaming Media](#uploading-and-streaming-media)
   - [Managing Media Assets](#managing-media-assets)
   - [Integration with Mobile Apps](#integration-with-mobile-apps)
   - [Integration with Web Applications](#integration-with-web-applications)

5. [Troubleshooting](#troubleshooting)
   - [Common Issues](#common-issues)
   - [Logs and Debugging](#logs-and-debugging)
   - [Support Resources](#support-resources)

---

## Getting Started

### Prerequisites

Before running the Media Management Service, ensure you have:

- Docker (version 20.10.0 or higher)
- Docker Compose (version 2.0.0 or higher)
- Git (for cloning the repository)
- Minimum 8GB RAM and 20GB free disk space
- Open ports: 3000-3006, 9000-9001, 27017, 6379, 5672, 15672, 9090, 3100

### Setup and Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/media-management-service.git
   cd media-management-service
   ```

2. Create a `.env` file at the project root with the following environment variables:
   ```dotenv
   # MongoDB
   MONGO_ROOT_USERNAME=admin
   MONGO_ROOT_PASSWORD=securepassword

   # RabbitMQ
   RABBITMQ_USER=mediaservice
   RABBITMQ_PASSWORD=securepassword

   # MinIO
   MINIO_ROOT_USER=minioadmin
   MINIO_ROOT_PASSWORD=minioadmin

   # JWT (change this to a secure random string)
   JWT_SECRET=your-secure-jwt-secret-key-should-be-long-and-random

   # Grafana
   GRAFANA_USER=admin
   GRAFANA_PASSWORD=securepassword
   ```

3. Create necessary directories:
   ```bash
   mkdir -p config/{nginx,profiles,mongodb,redis,rabbitmq,minio,monitoring/{prometheus,grafana}}
   ```

4. Set up transcoding profiles:
   ```bash
   # Create profile configuration files
   cat > config/profiles/1080p.json << 'EOL'
   {
     "width": 1920,
     "height": 1080,
     "videoBitrate": "5000k",
     "audioBitrate": "192k",
     "preset": "medium",
     "frameRate": 30
   }
   EOL

   cat > config/profiles/720p.json << 'EOL'
   {
     "width": 1280,
     "height": 720,
     "videoBitrate": "2500k",
     "audioBitrate": "128k",
     "preset": "medium",
     "frameRate": 30
   }
   EOL

   cat > config/profiles/480p.json << 'EOL'
   {
     "width": 854,
     "height": 480,
     "videoBitrate": "1000k",
     "audioBitrate": "128k",
     "preset": "medium",
     "frameRate": 25
   }
   EOL

   cat > config/profiles/360p.json << 'EOL'
   {
     "width": 640,
     "height": 360,
     "videoBitrate": "600k",
     "audioBitrate": "96k",
     "preset": "medium",
     "frameRate": 25
   }
   EOL
   ```

### Starting the Service

1. Build all services:
   ```bash
   docker-compose build
   ```

2. Start all services:
   ```bash
   docker-compose up -d
   ```

3. Check service status:
   ```bash
   docker-compose ps
   ```

4. View logs (optional):
   ```bash
   docker-compose logs -f
   ```

5. Monitor startup process:
   ```bash
   docker-compose logs -f api-gateway
   ```

The Media Management Service can take a minute or two to fully initialize. All services are running properly when the API Gateway's health check endpoint returns a successful response.

### Stopping the Service

1. Stop all services but preserve data:
   ```bash
   docker-compose down
   ```

2. Stop all services and remove volumes (data will be lost):
   ```bash
   docker-compose down -v
   ```

3. Stop a specific service:
   ```bash
   docker-compose stop upload-manager
   ```

4. Restart a specific service:
   ```bash
   docker-compose restart streaming-engine
   ```

---

## Available Endpoints

### API Gateway Endpoints

The API Gateway serves as the entry point for all client applications.

| Endpoint | Method | Description | Requires Auth |
|----------|--------|-------------|--------------|
| `/health` | GET | Health check status | No |
| `/api/auth/login` | POST | User login | No |
| `/api/auth/register` | POST | User registration | No |
| `/api/auth/token` | POST | Refresh token | No |
| `/api/uploads/*` | * | Forward to Upload Manager | Yes* |
| `/api/media/*` | * | Forward to Streaming Engine | Yes* |
| `/api/assets/*` | * | Forward to Content Management | Yes* |

*Some endpoints may allow anonymous access depending on configuration.

Base URL: `http://localhost:3000`

### Upload Manager Endpoints

The Upload Manager handles chunked file uploads with resume capability.

| Endpoint | Method | Description | Requires Auth |
|----------|--------|-------------|--------------|
| `/health` | GET | Health check status | No |
| `/metrics` | GET | Prometheus metrics | No |
| `/api/uploads/init` | POST | Initialize upload | Yes |
| `/api/uploads/:uploadId/chunks` | POST | Upload file chunk | Yes |
| `/api/uploads/:uploadId` | GET | Get upload status | Yes |

Base URL: `http://localhost:3001`

### Streaming Engine Endpoints

The Streaming Engine provides adaptive streaming of media files.

| Endpoint | Method | Description | Requires Auth |
|----------|--------|-------------|--------------|
| `/health` | GET | Health check status | No |
| `/metrics` | GET | Prometheus metrics | No |
| `/media/:id` | GET | Get media asset details | Yes |
| `/media/:id/stream` | GET | Get streaming URLs | Yes |
| `/public/stream/:token` | GET | Stream media with token | No |
| `/media/:id/events` | POST | Record playback events | Yes |
| `/media/:id/share` | POST | Generate sharing token | Yes |
| `/stream/*` | GET | Stream manifest/segments | Token-based |

Base URL: `http://localhost:3002`

### Content Management Endpoints

The Content Management service provides CRUD operations for media assets.

| Endpoint | Method | Description | Requires Auth |
|----------|--------|-------------|--------------|
| `/health` | GET | Health check status | No |
| `/metrics` | GET | Prometheus metrics | No |
| `/api/assets` | GET | List user's assets | Yes |
| `/api/assets/:id` | GET | Get asset details | Yes |
| `/api/assets/:id` | PATCH | Update asset metadata | Yes |
| `/api/assets/:id` | DELETE | Delete an asset | Yes |
| `/api/assets/:id/versions` | GET | List asset versions | Yes |
| `/api/assets/:id/versions` | POST | Create new asset version | Yes |
| `/api/assets/:id/share/youtube` | POST | Share to YouTube | Yes |

Base URL: `http://localhost:3003`

### Status Checker Endpoints

The Status Checker provides a dashboard for monitoring system health.

| Endpoint | Method | Description | Requires Auth |
|----------|--------|-------------|--------------|
| `/` | GET | Health dashboard (HTML) | No |
| `/api/health` | GET | System health status (JSON) | No |

Base URL: `http://localhost:3006`

---

## API Implementation Guide

### Authentication

The Media Management Service uses JWT (JSON Web Tokens) for authentication.

#### Getting a Token

**Request:**
```http
POST /api/auth/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600
}
```

#### Using the Token

Include the JWT token in the `Authorization` header for all authenticated requests:

```http
GET /api/assets HTTP/1.1
Host: localhost:3000
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### File Upload API

The Upload Manager uses a chunked upload protocol for reliability with large files.

#### 1. Initialize Upload

**Request:**
```http
POST /api/uploads/init HTTP/1.1
Host: localhost:3001
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json

{
  "filename": "my-video.mp4",
  "fileSize": 15728640,
  "fileType": "video/mp4",
  "metadata": {
    "title": "My Awesome Video",
    "description": "This is a description of my video"
  },
  "fileHash": "a1b2c3d4e5f6g7h8i9j0..." // Optional SHA-256 hash
}
```

**Response:**
```json
{
  "uploadId": "12345678-abcd-1234-efgh-123456789abc",
  "chunkSize": 5242880,
  "expiresAt": "2023-08-15T12:00:00.000Z"
}
```

#### 2. Upload File Chunks

**Request:**
```http
POST /api/uploads/12345678-abcd-1234-efgh-123456789abc/chunks HTTP/1.1
Host: localhost:3001
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="chunkNumber"

0
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="totalChunks"

3
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="chunk"; filename="chunk-0"
Content-Type: application/octet-stream

[BINARY DATA]
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

**Response:**
```json
{
  "uploadId": "12345678-abcd-1234-efgh-123456789abc",
  "chunkNumber": 0,
  "received": 1,
  "total": 3
}
```

#### 3. Check Upload Status

**Request:**
```http
GET /api/uploads/12345678-abcd-1234-efgh-123456789abc HTTP/1.1
Host: localhost:3001
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```json
{
  "uploadId": "12345678-abcd-1234-efgh-123456789abc",
  "filename": "my-video.mp4",
  "fileSize": 15728640,
  "fileType": "video/mp4",
  "status": "processing",
  "progress": 45,
  "receivedChunks": 3,
  "totalChunks": 3,
  "metadata": {
    "title": "My Awesome Video",
    "description": "This is a description of my video"
  },
  "createdAt": "2023-08-14T10:00:00.000Z",
  "lastChunkAt": "2023-08-14T10:05:00.000Z"
}
```

Possible status values:
- `initialized`: Upload started but no chunks received
- `uploading`: Chunks being received
- `assembling`: Chunks being combined
- `validating`: Verifying file integrity
- `hashing`: Calculating file hash
- `uploading`: Uploading to storage
- `complete`: Upload completed, processing next
- `processed`: Asset processing completed
- `failed`: Upload failed
- `processing_failed`: Processing failed

### Media Streaming API

The Streaming Engine provides adaptive bitrate streaming.

#### 1. Get Media Asset Details

**Request:**
```http
GET /media/asset_12345 HTTP/1.1
Host: localhost:3002
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```json
{
  "id": "asset_12345",
  "title": "My Awesome Video",
  "description": "This is a description of my video",
  "duration": 120.5,
  "thumbnailUrl": "http://localhost:9000/media-assets/assets/asset_12345/thumbnail.jpg",
  "streamingUrl": "http://localhost:9000/media-assets/assets/asset_12345/manifest.m3u8",
  "status": "ready",
  "variants": [
    {
      "profile": "1080p",
      "width": 1920,
      "height": 1080,
      "bitrate": "5000k"
    },
    {
      "profile": "720p",
      "width": 1280,
      "height": 720,
      "bitrate": "2500k"
    },
    {
      "profile": "480p",
      "width": 854,
      "height": 480,
      "bitrate": "1000k"
    },
    {
      "profile": "360p",
      "width": 640,
      "height": 360,
      "bitrate": "600k"
    }
  ],
  "metadata": {
    "title": "My Awesome Video",
    "description": "This is a description of my video"
  },
  "createdAt": "2023-08-14T10:15:00.000Z"
}
```

#### 2. Get Streaming URLs

**Request:**
```http
GET /media/asset_12345/stream HTTP/1.1
Host: localhost:3002
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```json
{
  "masterUrl": "https://signed-url.example.com/assets/asset_12345/manifest.m3u8?signature=abc123...",
  "variants": {
    "1080p": "https://signed-url.example.com/assets/asset_12345/1080p.mp4?signature=def456...",
    "720p": "https://signed-url.example.com/assets/asset_12345/720p.mp4?signature=ghi789...",
    "480p": "https://signed-url.example.com/assets/asset_12345/480p.mp4?signature=jkl012...",
    "360p": "https://signed-url.example.com/assets/asset_12345/360p.mp4?signature=mno345..."
  },
  "expiresIn": 3600
}
```

#### 3. Generate Sharing Token

**Request:**
```http
POST /media/asset_12345/share HTTP/1.1
Host: localhost:3002
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json

{
  "expiresIn": 86400,
  "requireEmail": false,
  "maxViews": 10
}
```

**Response:**
```json
{
  "token": "abcdef123456",
  "sharingUrl": "http://localhost:3002/public/stream/abcdef123456",
  "expiresIn": 86400,
  "expiresAt": "2023-08-15T10:15:00.000Z"
}
```

#### 4. Stream with Sharing Token

**Request:**
```http
GET /public/stream/abcdef123456 HTTP/1.1
Host: localhost:3002
```

This will redirect to a signed streaming URL.

### Content Management API

The Content Management service provides CRUD operations for media assets.

#### 1. List User's Assets

**Request:**
```http
GET /api/assets?page=1&limit=20 HTTP/1.1
Host: localhost:3003
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```json
{
  "assets": [
    {
      "id": "asset_12345",
      "title": "My Awesome Video",
      "description": "This is a description of my video",
      "thumbnailUrl": "http://localhost:9000/media-assets/assets/asset_12345/thumbnail.jpg",
      "status": "ready",
      "createdAt": "2023-08-14T10:15:00.000Z",
      "duration": 120.5,
      "fileType": "video/mp4"
    },
    {
      "id": "asset_67890",
      "title": "Another Great Video",
      "description": "Another video description",
      "thumbnailUrl": "http://localhost:9000/media-assets/assets/asset_67890/thumbnail.jpg",
      "status": "ready",
      "createdAt": "2023-08-13T15:30:00.000Z",
      "duration": 180.2,
      "fileType": "video/mp4"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 2,
    "pages": 1
  }
}
```

#### 2. Update Asset Metadata

**Request:**
```http
PATCH /api/assets/asset_12345 HTTP/1.1
Host: localhost:3003
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json

{
  "title": "Updated Video Title",
  "description": "Updated video description",
  "tags": ["demo", "tutorial", "testing"]
}
```

**Response:**
```json
{
  "id": "asset_12345",
  "metadata": {
    "title": "Updated Video Title",
    "description": "Updated video description",
    "tags": ["demo", "tutorial", "testing"]
  }
}
```

#### 3. Delete an Asset

**Request:**
```http
DELETE /api/assets/asset_12345 HTTP/1.1
Host: localhost:3003
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```
HTTP/1.1 204 No Content
```

#### 4. Share to YouTube

**Request:**
```http
POST /api/assets/asset_12345/share/youtube HTTP/1.1
Host: localhost:3003
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json

{
  "title": "My Video on YouTube",
  "description": "Check out my awesome video!",
  "tags": ["demo", "tutorial"],
  "private": false
}
```

**Response:**
```json
{
  "success": true,
  "youtubeId": "abc12345xyz",
  "youtubeUrl": "https://www.youtube.com/watch?v=abc12345xyz",
  "status": "published"
}
```

### Analytics API

The Media Management Service includes analytics tracking.

#### 1. Record Playback Events

**Request:**
```http
POST /media/asset_12345/events HTTP/1.1
Host: localhost:3002
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json

{
  "event": "play",
  "position": 30.5,
  "quality": "720p",
  "buffering": false
}
```

**Response:**
```
HTTP/1.1 204 No Content
```

Common event types:
- `play`: Playback started
- `pause`: Playback paused
- `seek`: User seeked to position
- `buffer`: Buffering occurred
- `quality_change`: Playback quality changed
- `error`: Error occurred during playback
- `ended`: Playback completed

---

## Common Use Cases

### Uploading and Streaming Media

#### Complete Upload and Streaming Flow

1. **Initialize Upload:**
   ```javascript
   const response = await fetch('http://localhost:3001/api/uploads/init', {
     method: 'POST',
     headers: {
       'Authorization': 'Bearer YOUR_JWT_TOKEN',
       'Content-Type': 'application/json'
     },
     body: JSON.stringify({
       filename: 'video.mp4',
       fileSize: file.size,
       fileType: file.type,
       metadata: {
         title: 'My Video',
         description: 'Video description'
       }
     })
   });
   
   const { uploadId, chunkSize } = await response.json();
   ```

2. **Upload Chunks:**
   ```javascript
   const totalChunks = Math.ceil(file.size / chunkSize);
   
   for (let i = 0; i < totalChunks; i++) {
     const start = i * chunkSize;
     const end = Math.min(file.size, start + chunkSize);
     const chunk = file.slice(start, end);
     
     const formData = new FormData();
     formData.append('chunk', chunk);
     formData.append('chunkNumber', i);
     formData.append('totalChunks', totalChunks);
     
     await fetch(`http://localhost:3001/api/uploads/${uploadId}/chunks`, {
       method: 'POST',
       headers: {
         'Authorization': 'Bearer YOUR_JWT_TOKEN'
       },
       body: formData
     });
   }
   ```

3. **Check Status Until Processed:**
   ```javascript
   let status = 'uploading';
   let assetId = null;
   
   while (!['processed', 'failed', 'processing_failed'].includes(status)) {
     await new Promise(resolve => setTimeout(resolve, 2000));
     
     const statusResponse = await fetch(`http://localhost:3001/api/uploads/${uploadId}`, {
       headers: {
         'Authorization': 'Bearer YOUR_JWT_TOKEN'
       }
     });
     
     const statusData = await statusResponse.json();
     status = statusData.status;
     
     if (statusData.assetId) {
       assetId = statusData.assetId;
     }
   }
   ```

4. **Get Streaming URL:**
   ```javascript
   const streamResponse = await fetch(`http://localhost:3002/media/${assetId}/stream`, {
     headers: {
       'Authorization': 'Bearer YOUR_JWT_TOKEN'
     }
   });
   
   const streamData = await streamResponse.json();
   
   // Use with a player like Video.js
   player.src({
     src: streamData.masterUrl,
     type: 'application/x-mpegURL'
   });
   ```

### Managing Media Assets

#### Listing and Filtering Assets

```javascript
// List user's assets with pagination
async function getAssets(page = 1, limit = 20) {
  const response = await fetch(`http://localhost:3003/api/assets?page=${page}&limit=${limit}`, {
    headers: {
      'Authorization': 'Bearer YOUR_JWT_TOKEN'
    }
  });
  
  return await response.json();
}

// Search assets by title or description
async function searchAssets(query) {
  const response = await fetch(`http://localhost:3003/api/assets?search=${encodeURIComponent(query)}`, {
    headers: {
      'Authorization': 'Bearer YOUR_JWT_TOKEN'
    }
  });
  
  return await response.json();
}
```

#### Updating Asset Metadata

```javascript
async function updateAsset(assetId, metadata) {
  const response = await fetch(`http://localhost:3003/api/assets/${assetId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': 'Bearer YOUR_JWT_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(metadata)
  });
  
  return await response.json();
}
```

### Integration with Mobile Apps

#### iOS Swift Example

```swift
import Foundation

class MediaService {
    private let baseURL = "http://your-api-domain.com"
    private let token: String
    
    init(token: String) {
        self.token = token
    }
    
    // Upload a video file
    func uploadVideo(fileURL: URL, metadata: [String: Any], completion: @escaping (Result<String, Error>) -> Void) {
        // 1. Initialize upload
        let initURL = URL(string: "\(baseURL)/api/uploads/init")!
        var request = URLRequest(url: initURL)
        request.httpMethod = "POST"
        request.addValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        request.addValue("application/json", forHTTPHeaderField: "Content-Type")
        
        // Get file attributes
        let fileSize = try! FileManager.default.attributesOfItem(atPath: fileURL.path)[.size] as! Int64
        let fileType = "video/mp4" // Determine content type appropriately
        
        // Create request body
        let requestBody: [String: Any] = [
            "filename": fileURL.lastPathComponent,
            "fileSize": fileSize,
            "fileType": fileType,
            "metadata": metadata
        ]
        
        request.httpBody = try! JSONSerialization.data(withJSONObject: requestBody)
        
        // Make request
        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            // Handle initialization response and proceed with chunk upload
            // ...
        }
        task.resume()
    }
    
    // Get streaming URL for an asset
    func getStreamingURL(assetId: String, completion: @escaping (Result<URL, Error>) -> Void) {
        let streamURL = URL(string: "\(baseURL)/media/\(assetId)/stream")!
        var request = URLRequest(url: streamURL)
        request.addValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        
        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            // Parse response and get streaming URL
            // ...
        }
        task.resume()
    }
}
```

#### Android Kotlin Example

```kotlin
import android.content.Context
import okhttp3.*
import org.json.JSONObject
import java.io.File
import java.io.IOException

class MediaService(private val context: Context, private val token: String) {
    private val client = OkHttpClient()
    private val baseUrl = "http://your-api-domain.com"
    
    // Upload a video file
    fun uploadVideo(file: File, metadata: Map<String, Any>, callback: (Result<String>) -> Unit) {
        // 1. Initialize upload
        val initRequestBody = JSONObject().apply {
            put("filename", file.name)
            put("fileSize", file.length())
            put("fileType", "video/mp4") // Set appropriate content type
            put("metadata", JSONObject(metadata))
        }
        
        val initRequest = Request.Builder()
            .url("$baseUrl/api/uploads/init")
            .addHeader("Authorization", "Bearer $token")
            .addHeader("Content-Type", "application/json")
            .post(RequestBody.create(MediaType.parse("application/json"), initRequestBody.toString()))
            .build()
            
        client.newCall(initRequest).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                callback(Result.failure(e))
            }
            
            override fun onResponse(call: Call, response: Response) {
                // Process response and upload chunks
                // ...
            }
        })
    }
    
    // Get streaming URL
    fun getStreamingUrl(assetId: String, callback: (Result<String>) -> Unit) {
        val request = Request.Builder()
            .url("$baseUrl/media/$assetId/stream")
            .addHeader("Authorization", "Bearer $token")
            .build()
            
        client.newCall(request).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                callback(Result.failure(e))
            }
            
            override fun onResponse(call: Call, response: Response) {
                // Parse response and return streaming URL
                // ...
            }
        })
    }
}
```

### Integration with Web Applications

#### React.js Example for Video Upload

```jsx
import React, { useState, useCallback } from 'react';
import axios from 'axios';

const VideoUploader = () => {
  const [file, setFile] = useState(null);
  const [progress, setProgress] = useState(0);
  const [status, setStatus] = useState('');
  const [assetId, setAssetId] = useState(null);
  
  const onFileChange = useCallback((e) => {
    setFile(e.target.files[0]);
  }, []);
  
  const uploadFile = useCallback(async () => {
    if (!file) return;
    
    try {
      // 1. Initialize upload
      const initResponse = await axios.post('/api/uploads/init', {
        filename: file.name,
        fileSize: file.size,
        fileType: file.type,
        metadata: {
          title: file.name,
          description: 'Uploaded from web app'
        }
      });
      
      const { uploadId, chunkSize } = initResponse.data;
      setStatus('Uploading...');
      
      // 2. Upload chunks
      const totalChunks = Math.ceil(file.size / chunkSize);
      
      for (let i = 0; i < totalChunks; i++) {
        const start = i * chunkSize;
        const end = Math.min(file.size, start + chunkSize);
        const chunk = file.slice(start, end);
        
        const formData = new FormData();
        formData.append('chunk', chunk);
        formData.append('chunkNumber', i);
        formData.append('totalChunks', totalChunks);
        
        await axios.post(`/api/uploads/${uploadId}/chunks`, formData);
        
        setProgress(Math.round(((i + 1) / totalChunks) * 100));
      }
      
      // 3. Check processing status
      setStatus('Processing...');
      let processed = false;
      
      while (!processed) {
        await new Promise(resolve => setTimeout(resolve, 2000));
        
        const statusResponse = await axios.get(`/api/uploads/${uploadId}`);
        const { status, assetId: newAssetId } = statusResponse.data;
        
        setStatus(`Processing: ${status}`);
        
        if (status === 'processed' && newAssetId) {
          setAssetId(newAssetId);
          setStatus('Complete');
          processed = true;
        } else if (status === 'failed' || status === 'processing_failed') {
          setStatus(`Failed: ${statusResponse.data.error || 'Unknown error'}`);
          processed = true;
        }
      }
    } catch (error) {
      setStatus(`Error: ${error.message}`);
    }
  }, [file]);
  
  return (
    <div>
      <input type="file" onChange={onFileChange} accept="video/*" />
      <button onClick={uploadFile} disabled={!file}>Upload</button>
      
      {progress > 0 && <progress value={progress} max="100" />}
      <p>Status: {status}</p>
      
      {assetId && (
        <div>
          <p>Video uploaded! Asset ID: {assetId}</p>
          <button onClick={() => window.open(`/watch/${assetId}`, '_blank')}>
            Watch Video
          </button>
        </div>
      )}
    </div>
  );
};

export default VideoUploader;
```

#### Next.js Video Player Component

```jsx
import React, { useEffect, useRef } from 'react';
import videojs from 'video.js';
import 'video.js/dist/video-js.css';
import axios from 'axios';

const VideoPlayer = ({ assetId }) => {
  const videoRef = useRef(null);
  const playerRef = useRef(null);
  
  useEffect(() => {
    if (!assetId) return;
    
    // Get streaming URL
    const getStreamingUrl = async () => {
      try {
        const response = await axios.get(`/api/media/${assetId}/stream`);
        const { masterUrl } = response.data;
        
        if (playerRef.current) {
          playerRef.current.dispose();
        }
        
        // Initialize player
        const videoElement = videoRef.current;
        if (!videoElement) return;
        
        const player = videojs(videoElement, {
          autoplay: false,
          controls: true,
          responsive: true,
          fluid: true,
          sources: [{
            src: masterUrl,
            type: 'application/x-mpegURL'
          }]
        });
        
        player.on('error', (err) => {
          console.error('Video Player Error:', err);
        });
        
        playerRef.current = player;
        
        // Track events
        const events = ['play', 'pause', 'seeking', 'seeked', 'ended', 'error'];
        events.forEach(event => {
          player.on(event, () => {
            // Send analytics event
            axios.post(`/api/media/${assetId}/events`, {
              event,
              position: player.currentTime(),
              quality: player.videoHeight() === 1080 ? '1080p' : 
                      player.videoHeight() === 720 ? '720p' : 
                      player.videoHeight() === 480 ? '480p' : '360p'
            }).catch(err => console.error('Error sending event:', err));
          });
        });
      } catch (error) {
        console.error('Error loading video:', error);
      }
    };
    
    getStreamingUrl();
    
    return () => {
      if (playerRef.current) {
        playerRef.current.dispose();
      }
    };
  }, [assetId]);
  
  return (
    <div>
      <div data-vjs-player>
        <video ref={videoRef} className="video-js vjs-big-play-centered" />
      </div>
    </div>
  );
};

export default VideoPlayer;
```

---

## Troubleshooting

### Common Issues

#### Upload Issues

| Issue | Solution |
|-------|----------|
| Chunks not uploading | Check network connectivity, verify the uploadId is valid |
| Upload timeout | Increase timeout settings, try smaller chunk sizes |
| "Upload not found" error | The upload session might have expired, reinitialize |

#### Streaming Issues

| Issue | Solution |
|-------|----------|
| Video not playing | Check browser console for errors, verify token is valid |
| Buffering issues | Check network speed, try a lower quality stream |
| CORS errors | Ensure CORS is properly configured on the server |

#### Authentication Issues

| Issue | Solution |
|-------|----------|
| "Token expired" error | Generate a new token |
| "Invalid token" error | Verify you're using the correct JWT secret |
| "Access denied" error | Check user permissions for the requested resource |

### Logs and Debugging

To view logs for a specific service:

```bash
# View logs for a specific service
docker-compose logs -f upload-manager

# View logs for multiple services
docker-compose logs -f api-gateway upload-manager

# Tail the last 100 lines of logs
docker-compose logs --tail=100 -f streaming-engine
```

Access monitoring dashboards:
- Status Dashboard: http://localhost:3006
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3100

### Support Resources

- GitHub Issues: [github.com/yourusername/media-management-service/issues](https://github.com/yourusername/media-management-service/issues)
- Documentation: [github.com/yourusername/media-management-service/wiki](https://github.com/yourusername/media-management-service/wiki)
