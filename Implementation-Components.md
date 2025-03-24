# Enhanced Implementation Components

## Upload Manager Enhancements

### 1. Backpressure Handling for Chunk Assembly

```javascript
// In upload-manager/src/routes/upload-routes.js
// Enhanced chunk assembly function

async function assembleChunks(uploadDir, outputFilePath, totalChunks) {
  return new Promise((resolve, reject) => {
    const writeStream = fs.createWriteStream(outputFilePath);
    let currentChunk = 0;
    
    // Function to process next chunk
    function processNextChunk() {
      if (currentChunk >= totalChunks) {
        writeStream.end();
        return resolve();
      }
      
      const chunkPath = path.join(uploadDir, `chunk-${currentChunk}`);
      
      if (!fs.existsSync(chunkPath)) {
        return reject(new Error(`Missing chunk file: ${chunkPath}`));
      }
      
      const readStream = fs.createReadStream(chunkPath);
      
      // Handle error events
      readStream.on('error', (err) => {
        reject(new Error(`Error reading chunk ${currentChunk}: ${err.message}`));
      });
      
      // Process data with backpressure awareness
      readStream.on('data', (chunk) => {
        // Check if the write stream can accept more data
        const canContinue = writeStream.write(chunk);
        
        // If writeStream buffer is full, pause reading until drained
        if (!canContinue) {
          readStream.pause();
        }
      });
      
      // Resume reading when write buffer drains
      writeStream.on('drain', () => {
        readStream.resume();
      });
      
      // When chunk is fully read, clean up and move to next chunk
      readStream.on('end', () => {
        try {
          // Remove processed chunk
          fs.unlinkSync(chunkPath);
          currentChunk++;
          processNextChunk();
        } catch (err) {
          reject(new Error(`Error processing chunk ${currentChunk}: ${err.message}`));
        }
      });
    }
    
    // Start processing from first chunk
    processNextChunk();
    
    // Handle errors on the write stream
    writeStream.on('error', (err) => {
      reject(new Error(`Error writing assembled file: ${err.message}`));
    });
  });
}
```

### 2. Chunk Upload with Retry and Deduplication Support

```javascript
// Updated route handler in upload-manager/src/routes/upload-routes.js

// Upload chunk with deduplication support
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
    
    // Check if this chunk was already processed (deduplication)
    const chunkKey = `upload:${uploadId}:chunk:${chunkNumber}`;
    const chunkExists = await redisClient.exists(chunkKey);
    
    if (chunkExists) {
      logger.info(`Chunk ${chunkNumber} for upload ${uploadId} already processed (duplicate request)`);
      
      // Return success response for idempotency
      const uploadInfo = await redisClient.hGetAll(`upload:${uploadId}`);
      const receivedChunks = parseInt(uploadInfo.receivedChunks);
      
      return res.status(200).json({
        uploadId,
        chunkNumber,
        received: receivedChunks,
        total: parseInt(uploadInfo.totalChunks),
        duplicateChunk: true
      });
    }
    
    // Update received chunks
    const uploadInfo = await redisClient.hGetAll(`upload:${uploadId}`);
    let receivedChunks = parseInt(uploadInfo.receivedChunks) + 1;
    
    // Mark this chunk as processed
    await redisClient.set(chunkKey, '1', {
      EX: parseInt(process.env.UPLOAD_EXPIRE_HOURS || '24') * 60 * 60
    });
    
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
```

### 3. File Integrity Verification with Client-Provided Hash

```javascript
// Add to upload initialization endpoint in upload-manager/src/routes/upload-routes.js

router.post('/init', async (req, res) => {
  try {
    const { filename, fileSize, fileType, metadata, fileHash } = req.body;
    
    if (!filename || !fileSize || !fileType) {
      return res.status(400).json({ error: 'Missing required parameters' });
    }
    
    // MIME type validation
    const allowedMimeTypes = [
      'video/mp4', 'video/webm', 'video/ogg',
      'audio/mpeg', 'audio/ogg', 'audio/wav',
      'image/jpeg', 'image/png', 'image/gif'
    ];
    
    if (!allowedMimeTypes.includes(fileType)) {
      return res.status(400).json({ error: 'Unsupported file type' });
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
    const uploadData = {
      filename: sanitizedFilename,
      fileSize: String(fileSize),
      fileType,
      metadata: JSON.stringify(metadata || {}),
      status: 'initialized',
      createdAt: new Date().toISOString(),
      userId: req.user?.id || 'anonymous',
      totalChunks: Math.ceil(fileSize / config.chunkSize),
      receivedChunks: '0',
    };
    
    // If client provided a file hash, store it for verification later
    if (fileHash) {
      uploadData.expectedHash = fileHash;
    }
    
    await redisClient.hSet(`upload:${uploadId}`, uploadData);
    
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

// And add this to the processUpload function:

// Add hash verification
if (uploadInfo.expectedHash) {
  logger.info(`Verifying file hash for upload ${uploadId}`);
  
  const fileHash = await calculateFileHash(outputFilePath);
  
  if (fileHash !== uploadInfo.expectedHash) {
    throw new Error(`File hash mismatch. Expected: ${uploadInfo.expectedHash}, Got: ${fileHash}`);
  }
  
  logger.info(`File hash verified for upload ${uploadId}`);
}
```

## Content Management Service Enhancements

### 1. ObjectId Validation and Soft Delete Support

```javascript
// In content-management/src/routes/asset-routes.js

const { MongoClient, ObjectId } = require('mongodb');
const logger = require('../../../shared/logger');

// Get a single asset with ObjectId validation
router.get('/api/assets/:id', async (req, res) => {
  try {
    const assetId = req.params.id;
    
    // Validate ObjectId format
    let objectId;
    try {
      // If assetId is not in ObjectId format but in another format (like UUID),
      // we can skip this validation or adapt it
      if (assetId.match(/^[0-9a-fA-F]{24}$/)) {
        objectId = new ObjectId(assetId);
      } else {
        objectId = assetId; // Use as-is for string IDs
      }
    } catch (error) {
      return res.status(400).json({ error: 'Invalid asset ID format' });
    }
    
    // Query with soft delete support
    const asset = await db.collection('media-assets').findOne({ 
      _id: objectId,
      deletedAt: { $exists: false } // Only return non-deleted assets
    });
    
    if (!asset) {
      return res.status(404).json({ error: 'Asset not found' });
    }
    
    // Check access permission
    if (asset.access === 'private' && asset.userId !== req.user?.id) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    res.json({
      id: asset._id,
      title: asset.metadata?.title || 'Untitled',
      description: asset.metadata?.description || '',
      thumbnailUrl: asset.thumbnailUrl,
      streamingUrl: asset.streamingUrl,
      status: asset.status,
      createdAt: asset.createdAt,
      duration: asset.mediaInfo?.format?.duration || 0,
      fileType: asset.fileType,
      metadata: asset.metadata || {},
      variants: Object.keys(asset.variants || {}).map(profile => ({
        profile,
        width: asset.variants[profile].width,
        height: asset.variants[profile].height,
        bitrate: asset.variants[profile].bitrate,
        url: asset.variants[profile].url
      }))
    });
  } catch (error) {
    logger.error(`Error fetching asset: ${error.message}`);
    res.status(500).json({ error: 'Failed to fetch asset' });
  }
});

// Implement soft delete instead of hard delete
router.delete('/api/assets/:id', async (req, res) => {
  try {
    const assetId = req.params.id;
    
    // Validate ObjectId format
    let objectId;
    try {
      if (assetId.match(/^[0-9a-fA-F]{24}$/)) {
        objectId = new ObjectId(assetId);
      } else {
        objectId = assetId;
      }
    } catch (error) {
      return res.status(400).json({ error: 'Invalid asset ID format' });
    }
    
    // Get asset to check ownership
    const asset = await db.collection('media-assets').findOne({ 
      _id: objectId,
      deletedAt: { $exists: false }
    });
    
    if (!asset) {
      return res.status(404).json({ error: 'Asset not found' });
    }
    
    // Check ownership
    if (asset.userId !== req.user?.id && req.user?.role !== 'admin') {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    // Soft delete the asset by setting deletedAt timestamp
    const result = await db.collection('media-assets').updateOne(
      { _id: objectId },
      { 
        $set: { 
          deletedAt: new Date(),
          deletedBy: req.user?.id,
          status: 'deleted'
        } 
      }
    );
    
    if (result.modifiedCount === 0) {
      return res.status(500).json({ error: 'Failed to delete asset' });
    }
    
    // Log the deletion
    logger.info(`Asset ${assetId} soft deleted by user ${req.user?.id}`);
    
    res.status(204).end();
  } catch (error) {
    logger.error(`Error deleting asset: ${error.message}`);
    res.status(500).json({ error: 'Failed to delete asset' });
  }
});
```

### 2. Redis Caching for Asset Retrieval

```javascript
// In content-management/src/routes/asset-routes.js

// Get a single asset with Redis caching
router.get('/api/assets/:id', async (req, res) => {
  try {
    const assetId = req.params.id;
    const cacheKey = `asset:${assetId}`;
    
    // Get Redis client
    const redisClient = getRedisClient();
    
    // Try to get from cache first
    const cachedAsset = await redisClient.get(cacheKey);
    
    if (cachedAsset && process.env.ENABLE_REDIS_CACHE === 'true') {
      const asset = JSON.parse(cachedAsset);
      
      // Check access permission even for cached assets
      if (asset.access === 'private' && asset.userId !== req.user?.id) {
        return res.status(403).json({ error: 'Access denied' });
      }
      
      logger.debug(`Asset ${assetId} served from cache`);
      return res.json(asset);
    }
    
    // Validate ObjectId format if needed
    let objectId;
    try {
      if (assetId.match(/^[0-9a-fA-F]{24}$/)) {
        objectId = new ObjectId(assetId);
      } else {
        objectId = assetId;
      }
    } catch (error) {
      return res.status(400).json({ error: 'Invalid asset ID format' });
    }
    
    // Get from database
    const asset = await db.collection('media-assets').findOne({ 
      _id: objectId,
      deletedAt: { $exists: false }
    });
    
    if (!asset) {
      return res.status(404).json({ error: 'Asset not found' });
    }
    
    // Check access permission
    if (asset.access === 'private' && asset.userId !== req.user?.id) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    // Format response
    const formattedAsset = {
      id: asset._id,
      title: asset.metadata?.title || 'Untitled',
      description: asset.metadata?.description || '',
      thumbnailUrl: asset.thumbnailUrl,
      streamingUrl: asset.streamingUrl,
      status: asset.status,
      createdAt: asset.createdAt,
      duration: asset.mediaInfo?.format?.duration || 0,
      fileType: asset.fileType,
      metadata: asset.metadata || {},
      variants: Object.keys(asset.variants || {}).map(profile => ({
        profile,
        width: asset.variants[profile].width,
        height: asset.variants[profile].height,
        bitrate: asset.variants[profile].bitrate,
        url: asset.variants[profile].url
      })),
      // Include access info for cache validation
      access: asset.access,
      userId: asset.userId
    };
    
    // Cache the formatted response
    if (process.env.ENABLE_REDIS_CACHE === 'true') {
      await redisClient.set(cacheKey, JSON.stringify(formattedAsset), {
        EX: parseInt(process.env.ASSET_CACHE_TTL || '3600') // 1 hour default
      });
    }
    
    res.json(formattedAsset);
  } catch (error) {
    logger.error(`Error fetching asset: ${error.message}`);
    res.status(500).json({ error: 'Failed to fetch asset' });
  }
});

// Add cache invalidation on update
router.patch('/api/assets/:id', async (req, res) => {
  // ... existing update logic ...
  
  // Invalidate cache after update
  if (process.env.ENABLE_REDIS_CACHE === 'true') {
    await redisClient.del(`asset:${assetId}`);
  }
  
  // ... rest of the function ...
});

// Add cache invalidation on delete
router.delete('/api/assets/:id', async (req, res) => {
  // ... existing delete logic ...
  
  // Invalidate cache after soft delete
  if (process.env.ENABLE_REDIS_CACHE === 'true') {
    await redisClient.del(`asset:${assetId}`);
  }
  
  // ... rest of the function ...
});
```

## FFmpeg Helper Functions Enhancements

### 1. Dynamic Thumbnail Generation and Error Capturing

```javascript
// In processing-pipeline/src/utils/ffmpeg-utils.js

/**
 * Generate thumbnail from video with smart positioning
 * @param {string} inputPath - Path to video file
 * @param {string} outputDir - Directory to save thumbnail
 * @param {number} duration - Duration of video in seconds
 * @returns {Promise<string>} - Path to generated thumbnail
 */
function generateThumbnail(inputPath, outputDir, duration = null) {
  const thumbnailPath = path.join(outputDir, 'thumbnail.jpg');
  
  return new Promise(async (resolve, reject) => {
    try {
      // If duration not provided, get it from the video
      if (duration === null) {
        const mediaInfo = await analyzeMedia(inputPath);
        duration = parseFloat(mediaInfo.format.duration || 0);
      }
      
      // Calculate timestamp (10% into the video, but at least 3 seconds in)
      // and not more than 30 seconds in for very long videos
      const timestamp = Math.min(
        Math.max(duration * 0.1, 3),
        Math.min(duration * 0.2, 30)
      );
      
      // Format timestamp as HH:MM:SS
      const formattedTime = new Date(timestamp * 1000).toISOString().substr(11, 8);
      
      logger.info(`Generating thumbnail at ${formattedTime} for ${path.basename(inputPath)}`);
      
      // Store stderr output for debugging
      let stderrOutput = '';
      
      const process = spawn('ffmpeg', [
        '-i', inputPath,
        '-ss', formattedTime,
        '-vframes', '1',
        '-vf', 'scale=640:-1',
        '-q:v', '2',
        thumbnailPath
      ]);
      
      // Capture stderr for error reporting
      process.stderr.on('data', (data) => {
        stderrOutput += data.toString();
      });
      
      process.on('close', (code) => {
        if (code === 0) {
          resolve(thumbnailPath);
        } else {
          logger.error(`FFmpeg error output: ${stderrOutput}`);
          reject(new Error(`FFmpeg exited with code ${code}`));
        }
      });
      
      process.on('error', (err) => {
        logger.error(`FFmpeg process error: ${err.message}`);
        reject(err);
      });
    } catch (error) {
      logger.error(`Error in thumbnail generation: ${error.message}`);
      reject(error);
    }
  });
}
```

### 2. External Transcode Profile Configuration

```javascript
// In processing-pipeline/src/utils/profile-loader.js

const fs = require('fs-extra');
const path = require('path');
const logger = require('../../../shared/logger');

// Default profiles
const defaultProfiles = {
  '1080p': { width: 1920, height: 1080, videoBitrate: '5000k', audioBitrate: '192k' },
  '720p': { width: 1280, height: 720, videoBitrate: '2500k', audioBitrate: '128k' },
  '480p': { width: 854, height: 480, videoBitrate: '1000k', audioBitrate: '128k' },
  '360p': { width: 640, height: 360, videoBitrate: '600k', audioBitrate: '96k' }
};

/**
 * Load transcode profiles from config files or use defaults
 * @returns {Object} Transcode profiles
 */
function loadProfiles() {
  const profilesDir = process.env.PROFILES_DIR || path.resolve(__dirname, '../../../config/profiles');
  let profiles = { ...defaultProfiles };
  
  try {
    // Ensure the directory exists
    if (!fs.existsSync(profilesDir)) {
      logger.info(`Profiles directory not found at ${profilesDir}, using default profiles`);
      return profiles;
    }
    
    // Read all profile JSON files
    const files = fs.readdirSync(profilesDir).filter(file => file.endsWith('.json'));
    
    if (files.length === 0) {
      logger.info('No profile files found, using default profiles');
      return profiles;
    }
    
    // Load each profile file
    for (const file of files) {
      try {
        const filePath = path.join(profilesDir, file);
        const profileName = path.basename(file, '.json');
        const profileData = JSON.parse(fs.readFileSync(filePath, 'utf8'));
        
        // Validate required fields
        if (!profileData.width || !profileData.height || !profileData.videoBitrate) {
          logger.warn(`Profile ${profileName} is missing required fields, skipping`);
          continue;
        }
        
        // Add to profiles
        profiles[profileName] = profileData;
        logger.info(`Loaded transcode profile: ${profileName}`);
      } catch (error) {
        logger.error(`Error loading profile ${file}: ${error.message}`);
      }
    }
    
    logger.info(`Loaded ${Object.keys(profiles).length} transcode profiles`);
    return profiles;
  } catch (error) {
    logger.error(`Error loading profiles: ${error.message}`);
    return profiles;
  }
}

module.exports = {
  loadProfiles
};
```

Then update the FFmpeg transcoding function to use these profiles:

```javascript
// In processing-pipeline/src/utils/ffmpeg-utils.js

const { loadProfiles } = require('./profile-loader');

// Load profiles once on startup
const transcodingProfiles = loadProfiles();

/**
 * Transcode media to a specific profile
 * @param {string} inputPath - Path to input media file
 * @param {string} outputDir - Directory to save transcoded file
 * @param {string} profileName - Name of transcode profile
 * @param {boolean} isVideo - Whether the file is a video
 * @returns {Promise<string>} - Path to transcoded file
 */
function transcodeMedia(inputPath, outputDir, profileName, isVideo) {
  const profileSettings = transcodingProfiles[profileName];
  
  if (!profileSettings) {
    return Promise.reject(new Error(`Unknown profile: ${profileName}`));
  }
  
  const outputPath = path.join(outputDir, `${profileName}.mp4`);
  
  return new Promise((resolve, reject) => {
    // Store stderr output for debugging
    let stderrOutput = '';
    
    const args = [
      '-i', inputPath,
      '-c:a', 'aac',
      '-b:a', profileSettings.audioBitrate
    ];
    
    if (isVideo) {
      args.push(
        '-c:v', 'libx264',
        '-profile:v', 'main',
        '-preset', profileSettings.preset || 'medium',
        '-b:v', profileSettings.videoBitrate,
        '-vf', `scale=${profileSettings.width}:${profileSettings.height}:force_original_aspect_ratio=decrease,pad=${profileSettings.width}:${profileSettings.height}:(ow-iw)/2:(oh-ih)/2`,
        '-r', profileSettings.frameRate || '30'
      );
      
      // Add optional parameters if specified in the profile
      if (profileSettings.keyframeInterval) {
        args.push('-g', profileSettings.keyframeInterval);
      }
      
      if (profileSettings.tune) {
        args.push('-tune', profileSettings.tune);
      }
    }
    
    args.push(outputPath);
    
    logger.info(`Transcoding to ${profileName}: ${path.basename(inputPath)}`);
    
    const process = spawn('ffmpeg', args);
    
    // Capture stderr for error reporting
    process.stderr.on('data', (data) => {
      stderrOutput += data.toString();
    });
    
    process.on('close', (code) => {
      if (code === 0) {
        resolve(outputPath);
      } else {
        logger.error(`FFmpeg error output: ${stderrOutput}`);
        reject(new Error(`FFmpeg exited with code ${code}`));
      }
    });
    
    process.on('error', (err) => {
      logger.error(`FFmpeg process error: ${err.message}`);
      reject(err);
    });
  });
}
```

## Enhanced Video Player

```html
<!-- test-player.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Enhanced Media Player</title>
  <link href="https://unpkg.com/video.js/dist/video-js.css" rel="stylesheet">
  <script src="https://unpkg.com/video.js/dist/video.js"></script>
  <script src="https://unpkg.com/videojs-contrib-hls/dist/videojs-contrib-hls.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 20px;
      background-color: #f5f5f5;
    }
    .container {
      max-width: 800px;
      margin: 0 auto;
      background-color: white;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    .player-container {
      margin-top: 20px;
    }
    .video-js {
      width: 100%;
      height: 450px;
    }
    .controls {
      margin-top: 20px;
    }
    input, button, select {
      padding: 8px;
      margin-right: 10px;
      margin-bottom: 10px;
    }
    button {
      background-color: #4CAF50;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    button:hover {
      background-color: #45a049;
    }
    .status {
      margin-top: 20px;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 4px;
      background-color: #f9f9f9;
    }
    .error {
      color: #d32f2f;
      background-color: #ffebee;
      padding: 10px;
      border-radius: 4px;
      margin-top: 10px;
      display: none;
    }
    .profiles {
      margin-top: 15px;
    }
    .playback-stats {
      margin-top: 15px;
      font-size: 0.9em;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Enhanced Media Player</h1>
    
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

## Implementation of Health Check Endpoints

```javascript
// Add to each service's main file (e.g., index.js)

// Health check endpoint with detailed status
app.get('/health', async (req, res) => {
  try {
    const health = {
      status: 'UP',
      service: process.env.SERVICE_NAME || 'unknown-service',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version || '1.0.0',
      dependencies: {}
    };
    
    // Check MongoDB connection
    try {
      const db = getDb();
      await db.command({ ping: 1 });
      health.dependencies.mongodb = { status: 'UP' };
    } catch (error) {
      health.dependencies.mongodb = { 
        status: 'DOWN', 
        error: error.message,
        timestamp: new Date().toISOString()
      };
      health.status = 'DEGRADED';
    }
    
    // Check Redis connection
    try {
      const redisClient = getRedisClient();
      await redisClient.ping();
      health.dependencies.redis = { status: 'UP' };
    } catch (error) {
      health.dependencies.redis = { 
        status: 'DOWN', 
        error: error.message,
        timestamp: new Date().toISOString()
      };
      health.status = 'DEGRADED';
    }
    
    // Check RabbitMQ connection (if applicable)
    if (process.env.RABBITMQ_URL) {
      try {
        const channel = getChannel();
        health.dependencies.rabbitmq = { status: 'UP' };
      } catch (error) {
        health.dependencies.rabbitmq = { 
          status: 'DOWN', 
          error: error.message,
          timestamp: new Date().toISOString()
        };
        health.status = 'DEGRADED';
      }
    }
    
    // Check S3 connection (if applicable)
    if (process.env.S3_ENDPOINT) {
      try {
        const command = new HeadBucketCommand({ Bucket: s3Config.bucket });
        await s3Client.send(command);
        health.dependencies.s3 = { status: 'UP' };
      } catch (error) {
        health.dependencies.s3 = { 
          status: 'DOWN', 
          error: error.message,
          timestamp: new Date().toISOString()
        };
        health.status = 'DEGRADED';
      }
    }
    
    // Check disk space (if applicable)
    if (process.env.TEMP_UPLOAD_DIR || process.env.WORK_DIR) {
      try {
        const dirToCheck = process.env.TEMP_UPLOAD_DIR || process.env.WORK_DIR;
        const stats = fs.statfsSync(dirToCheck);
        const availableGB = (stats.bavail * stats.bsize) / (1024 * 1024 * 1024);
        
        health.dependencies.diskSpace = { 
          status: availableGB < 1 ? 'WARNING' : 'UP',
          availableGB: Math.round(availableGB * 100) / 100
        };
        
        if (availableGB < 0.5) {
          health.status = 'DEGRADED';
          health.dependencies.diskSpace.status = 'CRITICAL';
        } else if (availableGB < 1) {
          health.status = health.status === 'UP' ? 'WARNING' : health.status;
        }
      } catch (error) {
        health.dependencies.diskSpace = { 
          status: 'UNKNOWN', 
          error: error.message 
        };
      }
    }
    
    // Add memory usage
    const memoryUsage = process.memoryUsage();
    health.memory = {
      rss: `${Math.round(memoryUsage.rss / 1024 / 1024)} MB`,
      heapTotal: `${Math.round(memoryUsage.heapTotal / 1024 / 1024)} MB`,
      heapUsed: `${Math.round(memoryUsage.heapUsed / 1024 / 1024)} MB`,
      external: `${Math.round(memoryUsage.external / 1024 / 1024)} MB`,
    };
    
    // Add uptime
    health.uptime = `${Math.floor(process.uptime() / 60)} minutes`;
    
    // Return appropriate status code
    const statusCode = health.status === 'UP' ? 200 : 
                      health.status === 'WARNING' ? 200 :
                      health.status === 'DEGRADED' ? 200 : 503;
    
    res.status(statusCode).json(health);
  } catch (error) {
    logger.error(`Health check error: ${error.message}`);
    res.status(500).json({
      status: 'DOWN',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});
```
