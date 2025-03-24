# Microservice Integration Guide

This guide explains how to integrate the Media Management Service into the larger microservice architecture shown in the original system diagram.

## Updated Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENT LAYER                                    │
│                                                                                 │
│   ┌───────────────────────────┐           ┌───────────────────────────┐         │
│   │                           │           │                           │         │
│   │      Swift iOS App        │           │      Android App          │         │
│   │   - Native UI             │           │   - Native UI             │         │
│   │   - WKWebView for YouTube │           │   - WebView for YouTube   │         │
│   │   - Auto-play previews    │           │   - Auto-play previews    │         │
│   │   - Push notifications    │           │   - Push notifications    │         │
│   │   - Local storage         │           │   - Local storage         │         │
│   │                           │           │                           │         │
│   └───────────────┬───────────┘           └───────────────┬───────────┘         │
│                   │                                       │                     │
└───────────────────┼───────────────────────────────────────┼─────────────────────┘
                    │                                       │                      
                    │            REST/GraphQL APIs          │                      
                    └───────────────────┬───────────────────┘                      
                                        │                                          
┌───────────────────────────────────────┼───────────────────────────────────────┐
│                                       ▼                                        │
│                              API GATEWAY LAYER                                 │
│                                                                               │
│     ┌───────────────────────────────────────────────────────────────────┐     │
│     │                                                                   │     │
│     │                    Next.js Backend API                            │     │
│     │   - Authentication & Authorization                                │     │
│     │   - API orchestration                                            │     │
│     │   - Request routing                                              │     │
│     │   - Response formatting                                          │     │
│     │                                                                   │     │
│     └────┬────────────┬─────────────────────┬──────────────────┬───────┘     │
│          │            │                     │                  │              │
└──────────┼────────────┼─────────────────────┼──────────────────┼──────────────┘
           │            │                     │                  │               
           │            │                     │                  │               
┌──────────┼────────────┼─────────────────────┼──────────────────┼──────────────┐
│          │   SERVICE  │   LAYER             │                  │              │
│          ▼            ▼                     ▼                  ▼              │
│  ┌────────────────┐ ┌───────────────┐ ┌────────────────┐ ┌───────────────┐   │
│  │                │ │               │ │                │ │               │   │
│  │ Wowza Streaming│ │ Social Media  │ │ Notification   │ │ Preview      │   │
│  │ Service        │ │ Service       │ │ Service        │ │ Generation   │   │
│  │ - Live streams │ │ - YouTube     │ │ - Push notif.  │ │ Service      │   │
│  │ - Processing   │ │ - OAuth       │ │ - Event mgmt   │ │ - Create     │   │
│  │ - Segments     │ │ - Uploads     │ │ - Preferences  │ │   previews   │   │
│  │                │ │               │ │ - History      │ │ - Optimize   │   │
│  │                │ │               │ │                │ │   for mobile │   │
│  │                │ │               │ │                │ │ - Generate   │   │
│  │                │ │               │ │                │ │   thumbnails │   │
│  └───────┬────────┘ └────────┬──────┘ └────────┬───────┘ └───────┬───────┘   │
│          │                   │                 │                 │           │
└──────────┼───────────────────┼─────────────────┼─────────────────┼───────────┘
           │                   │                 │                 │            
           │                   │                 │                 │            
┌──────────┼───────────────────┼─────────────────┼─────────────────┼───────────┐
│ MESSAGING│& STORAGE LAYER    │                 │                 │           │
│          ▼                   ▼                 ▼                 ▼           │
│     ┌────────────────────────────────────────────────────────────┐           │
│     │                                                            │           │
│     │                   RabbitMQ                                 │           │
│     │   - Exchange-based routing                                 │           │
│     │   - Multiple queue patterns                                │           │
│     │   - Message acknowledgment                                 │           │
│     │                                                            │           │
│     └───────────────────────────────┬────────────────────────────┘           │
│                                     │                                        │
│     ┌─────────────────┐    ┌────────┴──────────┐    ┌────────────────────┐  │
│     │                 │    │                   │    │                    │  │
│     │     Redis       │    │     MongoDB       │    │  Storage Provider  │  │
│     │  - Caching      │    │  - User data      │    │  - S3 Storage      │  │
│     │  - Session mgmt │    │  - Stream metadata│    │    OR             │  │
│     │  - Rate limiting│    │  - Notifications  │    │  - NFS Server      │  │
│     │                 │    │  - Previews data  │    │  - Video content   │  │
│     │                 │    │  - Analytics      │    │  - Previews        │  │
│     │                 │    │                   │    │  - Thumbnails      │  │
│     └─────────────────┘    └───────────────────┘    └────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │                                         
                                     ▼                                         
┌────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SERVICES                                  │
│                                                                            │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │                │  │                │  │                │                │
│  │  Firebase      │  │  Apple         │  │  YouTube       │                │
│  │  Cloud         │  │  Push          │  │  API           │                │
│  │  Messaging     │  │  Notification  │  │                │                │
│  │  (FCM)         │  │  Service       │  │                │                │
│  │                │  │  (APNS)        │  │                │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## Updated Architecture with Media Management Service 

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENT LAYER                                    │
│                                                                                 │
│   ┌───────────────────────────┐           ┌───────────────────────────┐         │
│   │                           │           │                           │         │
│   │      Swift iOS App        │           │      Android App          │         │
│   │   - Native UI             │           │   - Native UI             │         │
│   │   - WKWebView for YouTube │           │   - WebView for YouTube   │         │
│   │   - Auto-play previews    │           │   - Auto-play previews    │         │
│   │   - Push notifications    │           │   - Push notifications    │         │
│   │   - Local storage         │           │   - Local storage         │         │
│   │                           │           │                           │         │
│   └───────────────┬───────────┘           └───────────────┬───────────┘         │
│                   │                                       │                     │
└───────────────────┼───────────────────────────────────────┼─────────────────────┘
                    │                                       │                      
                    │            REST/GraphQL APIs          │                      
                    └───────────────────┬───────────────────┘                      
                                        │                                          
┌───────────────────────────────────────┼───────────────────────────────────────┐
│                                       ▼                                        │
│                              API GATEWAY LAYER                                 │
│                                                                               │
│     ┌───────────────────────────────────────────────────────────────────┐     │
│     │                                                                   │     │
│     │                    Next.js Backend API                            │     │
│     │   - Authentication & Authorization                                │     │
│     │   - API orchestration                                            │     │
│     │   - Request routing                                              │     │
│     │   - Response formatting                                          │     │
│     │                                                                   │     │
│     └────┬────────────┬─────────────────────┬──────────────────┬───────┘     │
│          │            │                     │                  │              │
└──────────┼────────────┼─────────────────────┼──────────────────┼──────────────┘
           │            │                     │                  │               
           │            │                     │                  │               
┌──────────┼────────────┼─────────────────────┼──────────────────┼──────────────┐
│          │   SERVICE  │   LAYER             │                  │              │
│          ▼            ▼                     ▼                  ▼              │
│  ┌────────────────┐ ┌───────────────┐ ┌────────────────┐ ┌───────────────┐   │
│  │                │ │               │ │                │ │               │   │
│  │ Wowza Streaming│ │ Social Media  │ │ Notification   │ │ Preview      │   │
│  │ Service        │ │ Service       │ │ Service        │ │ Generation   │   │
│  │ - Live streams │ │ - YouTube     │ │ - Push notif.  │ │ Service      │   │
│  │ - Processing   │ │ - OAuth       │ │ - Event mgmt   │ │ - Create     │   │
│  │ - Segments     │ │ - Uploads     │ │ - Preferences  │ │   previews   │   │
│  │                │ │               │ │ - History      │ │ - Optimize   │   │
│  │                │ │               │ │                │ │   for mobile │   │
│  │                │ │               │ │                │ │ - Generate   │   │
│  │                │ │               │ │                │ │   thumbnails │   │
│  └───────┬────────┘ └────────┬──────┘ └────────┬───────┘ └───────┬───────┘   │
│          │                   │                 │                 │           │
│          │                   │                 │                 │           │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │                   Media Management Service                            │   │
│  │  ┌─────────────┐  ┌────────────────┐  ┌───────────────┐  ┌─────────┐  │   │
│  │  │             │  │                │  │               │  │         │  │   │
│  │  │   Upload    │  │  Processing    │  │  Streaming    │  │ Content │  │   │
│  │  │   Manager   │  │  Pipeline      │  │  Engine       │  │ Manager │  │   │
│  │  │             │  │                │  │               │  │         │  │   │
│  │  └──────┬──────┘  └───────┬────────┘  └───────┬───────┘  └────┬────┘  │   │
│  │         │                 │                   │                │       │   │
│  │         └──────────┬──────┴─────────┬─────────┴────────┬──────┘       │   │
│  │                    │                │                  │               │   │
│  │                    ▼                ▼                  ▼               │   │
│  │           ┌────────────────────────────────────────────────────┐      │   │
│  │           │                Status Checker                      │      │   │
│  │           └────────────────────────────────────────────────────┘      │   │
│  │                                                                       │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│          │                   │                 │                 │           │
└──────────┼───────────────────┼─────────────────┼─────────────────┼───────────┘
           │                   │                 │                 │            
           │                   │                 │                 │            
┌──────────┼───────────────────┼─────────────────┼─────────────────┼───────────┐
│ MESSAGING│& STORAGE LAYER    │                 │                 │           │
│          ▼                   ▼                 ▼                 ▼           │
│     ┌────────────────────────────────────────────────────────────┐           │
│     │                                                            │           │
│     │                   RabbitMQ                                 │           │
│     │   - Exchange-based routing                                 │           │
│     │   - Multiple queue patterns                                │           │
│     │   - Message acknowledgment                                 │           │
│     │                                                            │           │
│     └───────────────────────────────┬────────────────────────────┘           │
│                                     │                                        │
│     ┌─────────────────┐    ┌────────┴──────────┐    ┌────────────────────┐  │
│     │                 │    │                   │    │                    │  │
│     │     Redis       │    │     MongoDB       │    │  Storage Provider  │  │
│     │  - Caching      │    │  - User data      │    │  - MinIO Storage   │  │
│     │  - Session mgmt │    │  - Stream metadata│    │    (S3-compatible) │  │
│     │  - Rate limiting│    │  - Notifications  │    │  - Video content   │  │
│     │                 │    │  - Assets metadata│    │  - Thumbnails      │  │
│     │                 │    │  - Analytics      │    │  - Streaming files │  │
│     │                 │    │                   │    │                    │  │
│     └─────────────────┘    └───────────────────┘    └────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │                                         
                                     ▼                                         
┌────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SERVICES                                  │
│                                                                            │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │                │  │                │  │                │                │
│  │  Firebase      │  │  Apple         │  │  YouTube       │                │
│  │  Cloud         │  │  Push          │  │  API           │                │
│  │  Messaging     │  │  Notification  │  │                │                │
│  │  (FCM)         │  │  Service       │  │                │                │
│  │                │  │  (APNS)        │  │                │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## Integration Steps

### 1. API Gateway Integration

The Media Management Service needs to be integrated with the existing Next.js Backend API Gateway:

1. **Add routes in API Gateway**:

```javascript
// In the Next.js API routes
// file: /pages/api/media/[...path].js

import { createProxyMiddleware } from 'http-proxy-middleware';

// Create proxy middleware
export const config = {
  api: {
    bodyParser: false,
    externalResolver: true,
  },
};

const mediaServiceProxy = createProxyMiddleware({
  target: 'http://api-gateway:3000', // The Media Management Service API Gateway
  pathRewrite: {
    '^/api/media': '', // Remove /api/media prefix when forwarding
  },
  changeOrigin: true,
});

export default function handler(req, res) {
  // Add authentication middleware if needed
  return mediaServiceProxy(req, res);
}
```

2. **Update authentication flow**:

Ensure the API Gateway passes authentication tokens to the Media Management Service:

```javascript
// Example authentication middleware in Next.js
const authMiddleware = (req, res, next) => {
  // Verify token
  const token = req.headers.authorization;
  
  if (!token) {
    return res.status(401).json({ error: 'Authorization token required' });
  }
  
  // Forward the token to the Media Management Service
  req.headers['x-forwarded-user'] = JSON.stringify(req.user);
  next();
};
```

### 2. Connecting with Existing Services

#### Integration with Notification Service

1. **Create a notifier service in Media Management**:

```javascript
// shared/notification-client.js
const axios = require('axios');

class NotificationClient {
  constructor(notificationServiceUrl) {
    this.baseUrl = notificationServiceUrl || process.env.NOTIFICATION_SERVICE_URL;
  }
  
  async sendNotification(userId, notification) {
    try {
      const response = await axios.post(`${this.baseUrl}/notifications`, {
        userId,
        type: notification.type,
        title: notification.title,
        message: notification.message,
        data: notification.data
      });
      
      return response.data;
    } catch (error) {
      console.error('Error sending notification:', error.message);
      throw error;
    }
  }
  
  async sendProcessingCompleteNotification(userId, assetId, assetTitle) {
    return this.sendNotification(userId, {
      type: 'media_processing_complete',
      title: 'Processing Complete',
      message: `Your upload "${assetTitle}" is now ready to stream`,
      data: { assetId }
    });
  }
}

module.exports = new NotificationClient();
```

2. **Update Processing Pipeline to send notifications**:

```javascript
// In processing-pipeline/src/index.js
const notificationClient = require('../../shared/notification-client');

// After asset processing is complete
async function sendProcessingCompleteNotification(asset) {
  try {
    if (asset.userId && asset.userId !== 'anonymous') {
      await notificationClient.sendProcessingCompleteNotification(
        asset.userId,
        asset._id,
        asset.metadata?.title || 'Untitled'
      );
      logger.info(`Sent processing complete notification for asset ${asset._id}`);
    }
  } catch (error) {
    logger.error(`Failed to send notification: ${error.message}`);
    // Non-critical error, don't throw
  }
}
```

#### Integration with Social Media Service

Create a client for sharing to social media:

```javascript
// shared/social-media-client.js
const axios = require('axios');

class SocialMediaClient {
  constructor(socialMediaServiceUrl) {
    this.baseUrl = socialMediaServiceUrl || process.env.SOCIAL_MEDIA_SERVICE_URL;
  }
  
  async shareToYoutube(userId, assetId, shareData) {
    try {
      const response = await axios.post(`${this.baseUrl}/youtube/share`, {
        userId,
        assetId,
        title: shareData.title,
        description: shareData.description,
        tags: shareData.tags,
        private: shareData.private || false
      });
      
      return response.data;
    } catch (error) {
      console.error('Error sharing to YouTube:', error.message);
      throw error;
    }
  }
}

module.exports = new SocialMediaClient();
```

Add a sharing endpoint to Content Management Service:

```javascript
// In content-management/src/routes/asset-routes.js

// Share to social media
router.post('/api/assets/:id/share/youtube', authMiddleware(), async (req, res) => {
  try {
    const assetId = req.params.id;
    const { title, description, tags, private } = req.body;
    
    // Validate required fields
    if (!title) {
      return res.status(400).json({ error: 'Title is required' });
    }
    
    // Get the asset
    const asset = await db.collection('media-assets').findOne({ 
      _id: assetId,
      deletedAt: { $exists: false }
    });
    
    if (!asset) {
      return res.status(404).json({ error: 'Asset not found' });
    }
    
    // Check ownership
    if (asset.userId !== req.user.id) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    // Check if asset is ready
    if (asset.status !== 'ready') {
      return res.status(400).json({ error: 'Asset is not ready for sharing' });
    }
    
    // Share to YouTube
    const result = await socialMediaClient.shareToYoutube(req.user.id, assetId, {
      title,
      description,
      tags,
      private
    });
    
    // Return the result
    res.json(result);
  } catch (error) {
    logger.error(`Error sharing to YouTube: ${error.message}`);
    res.status(500).json({ error: 'Failed to share to YouTube' });
  }
});
```

### 3. Accessing the Media Management Service

#### Client-Side Integration

Add Media Management Service API client in the mobile apps:

```swift
// Example for Swift iOS App
class MediaManagementClient {
    private let baseURL = "https://api.yourdomain.com/api/media"
    private let session = URLSession.shared
    
    func uploadMedia(fileURL: URL, metadata: [String: Any], progressHandler: @escaping (Float) -> Void, completion: @escaping (Result<String, Error>) -> Void) {
        // Implementation for chunked upload
        // ...
    }
    
    func getMediaAssets(page: Int, limit: Int, completion: @escaping (Result<[MediaAsset], Error>) -> Void) {
        // Implementation to fetch user's media assets
        // ...
    }
    
    func getStreamingURL(assetId: String, completion: @escaping (Result<URL, Error>) -> Void) {
        // Implementation to get streaming URL
        // ...
    }
}
```

```kotlin
// Example for Android App
class MediaManagementClient(private val context: Context) {
    private val baseUrl = "https://api.yourdomain.com/api/media"
    
    fun uploadMedia(fileUri: Uri, metadata: Map<String, Any>, 
                   progressCallback: (Float) -> Unit, 
                   completion: (Result<String>) -> Unit) {
        // Implementation for chunked upload
        // ...
    }
    
    fun getMediaAssets(page: Int, limit: Int, completion: (Result<List<MediaAsset>>) -> Unit) {
        // Implementation to fetch user's media assets
        // ...
    }
    
    fun getStreamingUrl(assetId: String, completion: (Result<String>) -> Unit) {
        // Implementation to get streaming URL
        // ...
    }
}
```

#### Web Backend Integration

Add Media Management Service client in the Next.js backend:

```javascript
// services/mediaManagementService.js
const axios = require('axios');

class MediaManagementService {
  constructor() {
    this.baseUrl = process.env.MEDIA_MANAGEMENT_SERVICE_URL || 'http://api-gateway:3000';
  }
  
  async getAssetById(assetId, token) {
    try {
      const response = await axios.get(`${this.baseUrl}/media/${assetId}`, {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      
      return response.data;
    } catch (error) {
      console.error(`Error fetching asset ${assetId}:`, error.message);
      throw error;
    }
  }
  
  async getStreamingUrl(assetId, token) {
    try {
      const response = await axios.get(`${this.baseUrl}/media/${assetId}/stream`, {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      
      return response.data;
    } catch (error) {
      console.error(`Error getting streaming URL for asset ${assetId}:`, error.message);
      throw error;
    }
  }
  
  async getUserAssets(userId, page = 1, limit = 20, token) {
    try {
      const response = await axios.get(`${this.baseUrl}/assets`, {
        params: { page, limit, userId },
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      
      return response.data;
    } catch (error) {
      console.error(`Error fetching assets for user ${userId}:`, error.message);
      throw error;
    }
  }
}

module.exports = new MediaManagementService();
```

### 4. Using Shared Storage and Messaging

#### RabbitMQ Integration

Configure RabbitMQ exchanges for communicating between services:

1. **Create a media events exchange**:

```javascript
// In shared/rabbitmq-client.js

async function setupExchanges() {
  const channel = await connect();
  
  // Create exchanges
  await channel.assertExchange('media-events', 'topic', { durable: true });
  
  logger.info('RabbitMQ exchanges set up successfully');
}

async function publishMediaEvent(routingKey, message) {
  const channel = await connect();
  
  return channel.publish(
    'media-events', 
    routingKey,
    Buffer.from(JSON.stringify(message)),
    { persistent: true }
  );
}

async function subscribeToMediaEvents(routingKey, callback) {
  const channel = await connect();
  
  // Create a queue for this service
  const { queue } = await channel.assertQueue('', { exclusive: true });
  
  // Bind to the exchange with the routing key
  await channel.bindQueue(queue, 'media-events', routingKey);
  
  // Consume messages
  return channel.consume(queue, (msg) => {
    if (msg !== null) {
      try {
        const content = JSON.parse(msg.content.toString());
        callback(content, msg);
        channel.ack(msg);
      } catch (error) {
        logger.error(`Error processing message: ${error.message}`);
        channel.nack(msg, false, true);
      }
    }
  });
}
```

2. **Publish events from the Media Management Service**:

```javascript
// In processing-pipeline/src/index.js
const { publishMediaEvent } = require('../../shared/rabbitmq-client');

// After asset processing is complete
await publishMediaEvent('asset.processed', {
  assetId: asset._id,
  userId: asset.userId,
  status: 'ready',
  metadata: asset.metadata,
  thumbnailUrl: asset.thumbnailUrl,
  streamingUrl: asset.streamingUrl,
  timestamp: new Date().toISOString()
});
```

3. **Subscribe to events in other services**:

```javascript
// In another service that needs to know about processed assets
const { subscribeToMediaEvents } = require('../../shared/rabbitmq-client');

async function setupEventSubscriptions() {
  // Subscribe to asset processed events
  await subscribeToMediaEvents('asset.processed', async (message) => {
    logger.info(`Asset processed: ${message.assetId}`);
    
    // Handle the event
    // ...
  });
  
  logger.info('Event subscriptions set up');
}
```

#### Shared MongoDB Access

Configure MongoDB collections to be accessed by multiple services:

```javascript
// In shared/db-client.js
// Ensure consistent collection names across services

const COLLECTIONS = {
  ASSETS: 'media-assets',
  USERS: 'users',
  NOTIFICATIONS: 'notifications',
  STREAMING_SESSIONS: 'streaming-sessions',
  ANALYTICS: 'analytics'
};

module.exports = {
  // ... existing code ...
  COLLECTIONS
};
```

Use these collection names consistently across all services:

```javascript
const { COLLECTIONS } = require('../../shared/db-client');

// In content-management service
const asset = await db.collection(COLLECTIONS.ASSETS).findOne({ _id: assetId });

// In notification service
await db.collection(COLLECTIONS.NOTIFICATIONS).insertOne({ 
  userId, 
  message, 
  createdAt: new Date() 
});
```

#### Shared Redis Caching

Setup shared Redis caching for assets:

```javascript
// In shared/redis-client.js

// Asset cache methods
async function cacheAsset(assetId, assetData, ttlSeconds = 3600) {
  const client = getClient();
  const key = `asset:${assetId}`;
  
  await client.set(key, JSON.stringify(assetData), {
    EX: ttlSeconds
  });
}

async function getCachedAsset(assetId) {
  const client = getClient();
  const key = `asset:${assetId}`;
  
  const data = await client.get(key);
  return data ? JSON.parse(data) : null;
}

async function invalidateAssetCache(assetId) {
  const client = getClient();
  const key = `asset:${assetId}`;
  
  await client.del(key);
}

module.exports = {
  // ... existing code ...
  cacheAsset,
  getCachedAsset,
  invalidateAssetCache
};
```

### 5. Monitoring and Observability

#### End-to-End Health Checks

Add a comprehensive health check that monitors the whole system:

```javascript
// In status-checker/src/index.js

async function checkSystemHealth() {
  const results = {
    status: 'UP',
    timestamp: new Date().toISOString(),
    services: {}
  };
  
  // Check Media Management Services
  const mediaServices = [
    { name: 'api-gateway', url: 'http://api-gateway:3000/health' },
    { name: 'upload-manager', url: 'http://upload-manager:3001/health' },
    { name: 'processing-pipeline', url: 'http://processing-pipeline:3010/health' },
    { name: 'streaming-engine', url: 'http://streaming-engine:3002/health' },
    { name: 'content-management', url: 'http://content-management:3003/health' }
  ];
  
  // Check External Services
  const externalServices = [
    { name: 'notification-service', url: process.env.NOTIFICATION_SERVICE_URL + '/health' },
    { name: 'social-media-service', url: process.env.SOCIAL_MEDIA_SERVICE_URL + '/health' }
  ];
  
  const allServices = [...mediaServices, ...externalServices];
  
  await Promise.all(allServices.map(async (service) => {
    try {
      const response = await fetch(service.url, { timeout: 5000 });
      const data = await response.json();
      
      results.services[service.name] = {
        status: data.status,
        dependencies: data.dependencies
      };
      
      // Update overall status
      if (data.status === 'DOWN') {
        results.status = 'DOWN';
      } else if (data.status === 'DEGRADED' && results.status !== 'DOWN') {
        results.status = 'DEGRADED';
      }
    } catch (error) {
      results.services[service.name] = {
        status: 'DOWN',
        error: error.message
      };
      
      results.status = 'DOWN';
    }
  }));
  
  return results;
}
```

## Using the Dockerized Media Management Service

### 1. Using from Mobile Apps

Update mobile apps to connect to the API Gateway:

```swift
// Swift iOS App
let apiBaseURL = "https://api.yourdomain.com/api/media"

// Upload a video
func uploadVideo(url: URL) {
    // Get the file size
    let fileSize = try? FileManager.default.attributesOfItem(atPath: url.path)[.size] as? Int64 ?? 0
    
    // Initialize upload
    let initRequest = URLRequest(url: URL(string: "\(apiBaseURL)/uploads/init")!)
    // ... implementation ...
    
    // Upload chunks
    // ... implementation ...
    
    // Check status
    // ... implementation ...
}

// Stream a video
func streamVideo(assetId: String) {
    let streamRequest = URLRequest(url: URL(string: "\(apiBaseURL)/media/\(assetId)/stream")!)
    // ... implementation ...
}
```

### 2. Using from Other Services

Other services can interact with the Media Management Service:

```javascript
// Example: Notification Service using media asset data
async function notifyUserAboutMediaComment(assetId, commenterId, comment) {
  try {
    // Get asset data
    const asset = await mediaService.getAssetById(assetId);
    
    // Only notify if the commenter is not the asset owner
    if (asset.userId !== commenterId) {
      await sendNotification(asset.userId, {
        type: 'media_comment',
        title: 'New comment on your media',
        message: `${commenterName} commented: "${comment.substring(0, 50)}${comment.length > 50 ? '...' : ''}"`,
        data: { assetId }
      });
    }
  } catch (error) {
    logger.error(`Failed to notify about media comment: ${error.message}`);
  }
}
```

### 3. Scaling and Load Balancing

For production environments, add load balancing and scaling:

```yaml
# In docker-compose.prod.yml
version: '3.8'

services:
  # Scale processing pipeline for better throughput
  processing-pipeline:
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '1'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
      restart_policy:
        condition: on-failure
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 2
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s

  # Add a load balancer for better distribution
  nginx-lb:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx/nginx-lb.conf:/etc/nginx/nginx.conf:ro
      - ./config/nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api-gateway
    networks:
      - media-network
    restart: unless-stopped
```

Example Nginx load balancer configuration:

```nginx
# nginx-lb.conf
http {
    upstream media_api {
        server api-gateway:3000;
        # Add more api-gateway instances if scaling horizontally
    }
    
    server {
        listen 80;
        server_name api.yourdomain.com;
        
        # Redirect to HTTPS
        return 301 https://$host$request_uri;
    }
    
    server {
        listen 443 ssl;
        server_name api.yourdomain.com;
        
        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        
        location / {
            proxy_pass http://media_api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## Conclusion

By following this integration guide, you can successfully incorporate the Media Management Service into your existing microservices architecture. The service will leverage the shared infrastructure while providing specialized media handling capabilities.

Key benefits of this integration:
1. Reuse of existing infrastructure (RabbitMQ, MongoDB, Redis)
2. Seamless communication with other services
3. Consistent authentication and authorization
4. End-to-end monitoring and health checks
5. Scalable and resilient media handling
