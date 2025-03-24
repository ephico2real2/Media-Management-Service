# Media-Management-Service
Media Management Service


Upload Manager: Handles chunked, resumable file uploads
Processing Pipeline: Transcodes media to multiple formats/resolutions
Streaming Engine: Delivers content via adaptive streaming
Content Management: Organizes and catalogs your media

Key Features

✅ Vendor Independence: Uses MinIO as a self-hosted S3-compatible storage
✅ Resumable Uploads: Handles large files with pause/resume capability
✅ Adaptive Streaming: HLS/DASH support for optimal playback
✅ Scalable Architecture: Components can grow independently

Setup Instructions
Follow the step-by-step guide I've provided in the "Local Setup Guide" artifact. This covers:

Creating the proper folder structure
Installing all required dependencies
Setting up MinIO, MongoDB, Redis, and RabbitMQ locally
Configuring each service with proper environment variables
