# Running and Troubleshooting Guide

This guide provides instructions for running your Media Management Service and troubleshooting common issues that might arise during development and testing.

## Running the Services

### Running Individual Services

Each service can be run independently in development mode:

```bash
# Start Upload Manager
cd upload-manager
npm run dev

# Start Processing Pipeline
cd processing-pipeline
npm run dev

# Start Streaming Engine
cd streaming-engine
npm run dev

# Start Content Management
cd content-management
npm run dev
```

### Running All Services Together

From the project root, use the concurrently scripts to run multiple services:

```bash
# Run all services in development mode
npm run dev

# Or run specific services together
npm run dev:upload dev:processing
```

## Testing Your Setup

### 1. Verifying Prerequisites

Ensure all dependencies are running correctly:

```bash
# Check MongoDB
mongosh --eval "db.runCommand({ping:1})"

# Check Redis 
redis-cli ping

# Check RabbitMQ
rabbitmqctl status | grep -A 2 "Uptime"

# Check MinIO
curl -s http://localhost:9000/minio/health/live
```

### 2. Testing Upload Service

You can test the upload service using curl:

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

For a more user-friendly testing approach, create an HTML file with a UI for uploading files:

```html
<!-- test-upload.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Media Upload Test</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
    .container { border: 1px solid #ddd; padding: 20px; border-radius: 5px; }
    progress { width: 100%; height: 20px; }
    #status { background: #f5f5f5; padding: 10px; border-radius: 5px; white-space: pre-wrap; }
    button { padding: 10px 15px; background: #4CAF50; color: white; border: none; border-radius: 5px; cursor: pointer; }
    input, textarea { width: 100%; padding: 8px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 5px; }
  </style>
</head>
<body>
  <h1>Media Upload Test</h1>
  <div class="container">
    <h2>Upload Settings</h2>
    <div>
      <label for="title">Title:</label>
      <input type="text" id="title" value="Test Upload">
    </div>
    <div>
      <label for="description">Description:</label>
      <textarea id="description">Test upload description</textarea>
    </div>
    <div>
      <label for="fileInput">Select File:</label>
      <input type="file" id="fileInput">
    </div>
    <button onclick="uploadFile()">Start Upload</button>
    <h3>Progress:</h3>
    <progress id="progress" value="0" max="100"></progress>
    <div id="progressText">0%</div>
    <h3>Status:</h3>
    <pre id="status">No upload in progress</pre>
  </div>

  <script>
    let uploadId = null;
    let statusCheckInterval = null;

    async function uploadFile() {
      const fileInput = document.getElementById('fileInput');
      const file = fileInput.files[0];
      
      if (!file) {
        alert('Please select a file');
        return;
      }
      
      // Metadata from form inputs
      const title = document.getElementById('title').value || 'Untitled';
      const description = document.getElementById('description').value || '';
      
      // Reset UI
      document.getElementById('progress').value = 0;
      document.getElementById('progressText').textContent = '0%';
      document.getElementById('status').textContent = 'Initializing upload...';
      
      try {
        // 1. Initialize upload
        const initResponse = await fetch('http://localhost:3001/api/uploads/init', {
          method: 'POST',
          headers: {'Content-Type': 'application/json'},
          body: JSON.stringify({
            filename: file.name,
            fileSize: file.size,
            fileType: file.type,
            metadata: { title, description }
          })
        });
        
        if (!initResponse.ok) {
          const errorData = await initResponse.json();
          throw new Error(`Init failed: ${errorData.error || initResponse.statusText}`);
        }
        
        const initData = await initResponse.json();
        uploadId = initData.uploadId;
        const chunkSize = initData.chunkSize;
        
        updateStatus(`Upload initialized. ID: ${uploadId}`);
        
        // 2. Upload chunks
        const totalChunks = Math.ceil(file.size / chunkSize);
        updateStatus(`Uploading ${totalChunks} chunks...`);
        
        for (let i = 0; i < totalChunks; i++) {
          const start = i * chunkSize;
          const end = Math.min(file.size, start + chunkSize);
          const chunk = file.slice(start, end);
          
          const formData = new FormData();
          formData.append('chunk', chunk);
          formData.append('chunkNumber', i);
          formData.append('totalChunks', totalChunks);
          
          const uploadResponse = await fetch(`http://localhost:3001/api/uploads/${uploadId}/chunks`, {
            method: 'POST',
            body: formData
          });
          
          if (!uploadResponse.ok) {
            const errorData = await uploadResponse.json();
            throw new Error(`Chunk upload failed: ${errorData.error || uploadResponse.statusText}`);
          }
          
          const uploadData = await uploadResponse.json();
          
          // Update progress
          const progress = Math.floor((uploadData.received / uploadData.total) * 100);
          document.getElementById('progress').value = progress;
          document.getElementById('progressText').textContent = `${progress}% (${uploadData.received}/${uploadData.total} chunks)`;
          
          updateStatus(`Chunk ${i+1}/${totalChunks} uploaded`);
        }
        
        updateStatus('All chunks uploaded. Starting processing...');
        
        // 3. Start status checking
        checkUploadStatus();
        statusCheckInterval = setInterval(checkUploadStatus, 2000);
        
      } catch (error) {
        updateStatus(`Error: ${error.message}`);
        console.error('Upload error:', error);
      }
    }
    
    async function checkUploadStatus() {
      if (!uploadId) return;
      
      try {
        const response = await fetch(`http://localhost:3001/api/uploads/${uploadId}`);
        
        if (!response.ok) {
          clearInterval(statusCheckInterval);
          const errorData = await response.json();
          throw new Error(`Status check failed: ${errorData.error || response.statusText}`);
        }
        
        const statusData = await response.json();
        
        // Show full status data
        document.getElementById('status').textContent = JSON.stringify(statusData, null, 2);
        
        // Update progress if available
        if (statusData.progress !== undefined) {
          document.getElementById('progress').value = statusData.progress;
          document.getElementById('progressText').textContent = `${statusData.progress}%`;
        }
        
        // Check if process is complete
        if (['complete', 'processed', 'failed', 'processing_failed'].includes(statusData.status)) {
          clearInterval(statusCheckInterval);
          
          if (statusData.assetId) {
            updateStatus(`Processing complete! Asset ID: ${statusData.assetId}\n\n${JSON.stringify(statusData, null, 2)}`);
          } else if (statusData.error) {
            updateStatus(`Error: ${statusData.error}\n\n${JSON.stringify(statusData, null, 2)}`);
          }
        }
        
      } catch (error) {
        clearInterval(statusCheckInterval);
        updateStatus(`Error checking status: ${error.message}`);
        console.error('Status check error:', error);
      }
    }
    
    function updateStatus(message) {
      const statusElem = document.getElementById('status');
      statusElem.textContent = `${new Date().toLocaleTimeString()}: ${message}\n${statusElem.textContent}`;
    }
  </script>
</body>
</html>
```

Save this file and open it in your browser to test uploads.

### 3. Generating a Test JWT Token

To test authenticated endpoints, you can generate a JWT token:

```bash
# From project root
npm run token test-user

# Or directly run the script
node scripts/generate-token.js test-user admin
```

## Troubleshooting Common Issues

### Connection Issues

If services can't connect to one another, check these common issues:

#### MongoDB Connection Issues

```
Error: Failed to connect to MongoDB: MongoNetworkError: connect ECONNREFUSED 127.0.0.1:27017
```

**Solutions:**
1. Check if MongoDB is running: `sudo systemctl status mongod`
2. If not running, start it: `sudo systemctl start mongod`
3. Verify MongoDB port: `sudo netstat -tulpn | grep 27017`
4. Check MongoDB logs: `sudo journalctl -u mongod.service -f`

#### Redis Connection Issues

```
Error: Redis connection to 127.0.0.1:6379 failed - connect ECONNREFUSED 127.0.0.1:6379
```

**Solutions:**
1. Check if Redis is running: `sudo systemctl status redis-server`
2. If not running, start it: `sudo systemctl start redis-server`
3. Verify Redis is accepting connections: `redis-cli ping`
4. Check Redis logs: `sudo journalctl -u redis-server.service -f`

#### RabbitMQ Connection Issues

```
Error: connect ECONNREFUSED 127.0.0.1:5672
```

**Solutions:**
1. Check if RabbitMQ is running: `sudo systemctl status rabbitmq-server`
2. If not running, start it: `sudo systemctl start rabbitmq-server`
3. Verify RabbitMQ status: `rabbitmqctl status`
4. Check RabbitMQ logs: `sudo journalctl -u rabbitmq-server.service -f`

#### MinIO Connection Issues

```
Error: The specified bucket does not exist
```

**Solutions:**
1. Check if MinIO is running: `ps aux | grep minio`
2. Verify the bucket exists: `mc ls local/`
3. Create the bucket if needed: `mc mb local/media-assets`

### Upload Issues

#### Chunks Not Being Saved

**Problem:** Uploads start but chunks are not being saved

**Solutions:**
1. Check if the temporary directory exists and has proper permissions: 
   ```bash
   ls -la temp/uploads
   sudo chmod -R 755 temp/uploads
   ```
2. Check if the upload directory is writable by the current user:
   ```bash
   touch temp/uploads/test.txt
   rm temp/uploads/test.txt
   ```
3. Check storage space: `df -h`

#### Maximum File Size Exceeded

**Problem:** "File too large" error when uploading

**Solution:** Modify the `MAX_FILE_SIZE` in the upload-manager's environment file:
```bash
# In env/upload-manager.env
MAX_FILE_SIZE=10737418240  # Increase to 10GB
```

#### Upload Expires Before Completion

**Problem:** Upload session expires before the upload is complete

**Solution:** Increase the upload expiration time:
```bash
# In env/upload-manager.env
UPLOAD_EXPIRE_HOURS=48  # Increase to 48 hours
```

### Processing Issues

#### FFmpeg Not Found

**Problem:** Processing fails with "FFmpeg not found" errors

**Solutions:**
1. Verify FFmpeg is installed: `ffmpeg -version`
2. If not installed, install it: `sudo apt install ffmpeg`
3. Check if FFmpeg is in the PATH: `which ffmpeg`

#### Processing Queue Issues

**Problem:** Files upload but never start processing

**Solutions:**
1. Check if RabbitMQ queue exists: `rabbitmqctl list_queues`
2. Check if the processing service is connecting to RabbitMQ correctly
3. Check processing-pipeline logs for errors

#### Insufficient Disk Space

**Problem:** Processing fails with disk space errors

**Solution:** Free up disk space or mount additional storage:
```bash
# Check disk space
df -h

# Clean up temporary files
npm run cleanup
```

### Streaming Issues

#### CORS Errors When Streaming

**Problem:** Browser console shows CORS errors when trying to stream media

**Solutions:**
1. Verify CORS settings in the streaming-engine's environment file:
   ```bash
   # In env/streaming-engine.env
   CORS_ORIGIN=*  # For development
   ```
2. Check that the MinIO bucket has proper CORS settings:
   ```bash
   mc admin config set local cors='{"allow_origins": ["*"]}'
   ```

#### JWT Token Issues

**Problem:** Authentication failures with "Invalid token" errors

**Solutions:**
1. Verify the JWT_SECRET is the same across services
2. Check if the token is expired (regenerate a new one)
3. Ensure the correct authorization header format: `Bearer YOUR_TOKEN`

### Logging and Debugging

To get more detailed logs for debugging:

1. Change log level to debug:
   ```bash
   # In the service's .env file
   LOG_LEVEL=debug
   ```

2. Check service logs:
   ```bash
   # Check specific service logs
   cat upload-manager/logs/upload-manager.log
   
   # Check error logs
   cat upload-manager/logs/error.log
   ```

3. Enable more verbose RabbitMQ logging:
   ```bash
   # Set RabbitMQ to debug mode
   rabbitmqctl set_log_level debug
   
   # View logs
   rabbitmqctl log_tail
   ```

## Performance Tuning

### Node.js Memory Limits

If processing larger files causes memory issues, adjust the Node.js memory limit:

```bash
# Start the service with increased memory limit
NODE_OPTIONS="--max-old-space-size=4096" npm run dev:processing
```

### Handling High Concurrency

For handling multiple uploads simultaneously, tune these settings:

```bash
# In env/upload-manager.env
RATE_LIMIT_MAX=200  # Increase API rate limit
RATE_LIMIT_WINDOW_MS=60000

# In env/processing-pipeline.env
CONCURRENT_PROCESSING=4  # Process more files simultaneously
```

### Redis Performance

For better Redis performance with many uploads:

```bash
# Modify Redis configuration
sudo nano /etc/redis/redis.conf

# Add/modify these lines
maxmemory 1gb
maxmemory-policy allkeys-lru
```

## Cleanup Procedures

### Manual Cleanup

To manually clean up temporary files and incomplete uploads:

```bash
# Run the cleanup script
npm run cleanup

# Or with custom age threshold (in hours)
TEMP_FILE_MAX_AGE_HOURS=12 npm run cleanup
```

### Setting Up Automatic Cleanup

To set up automatic cleanup using cron:

```bash
# Edit crontab
crontab -e

# Add this line to run cleanup every hour
0 * * * * cd /path/to/your/project && npm run cleanup >> logs/cleanup.log 2>&1
```

## Database Management

### Viewing Media Assets in MongoDB

```bash
# Connect to MongoDB
mongosh

# Switch to media-service database
use media-service

# View all media assets
db["media-assets"].find().pretty()

# Find asset by ID
db["media-assets"].findOne({_id: "asset_123456"})

# Count assets
db["media-assets"].countDocuments()
```

### Cleaning Up MongoDB

```bash
# Remove failed assets older than 7 days
db["media-assets"].deleteMany({
  status: "failed", 
  createdAt: {$lt: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)}
})
```

## Health Monitoring

If you implemented the status-checker service, you can monitor the health of all services:

```bash
# Start the status checker
cd status-checker
npm run dev

# Check the dashboard at http://localhost:3006
```
