# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mycelia is a self-hosted AI memory and timeline system that captures ideas, thoughts, and conversations through voice, screenshots, or text. It's a local-first, modular system built with Deno, React, Python, and leverages MongoDB, Redis, and Kafka for data storage and processing.

## Repository Structure

This is a monorepo with three main components:

- `backend/` - Deno + Remix server with OAuth2 authentication, MCP integration, and RESTful APIs
- `frontend/` - Standalone React SPA with timeline visualization using D3.js
- `python/` - Audio processing pipeline (chunking, transcription, diarization, daemon)

## Common Commands

### Initial Setup

```bash
# Start infrastructure services (MongoDB, Redis, Kafka)
docker compose up -d

# Configure backend environment
cd backend
cp .env.example .env
# Edit .env with your settings

# Generate auth credentials (requires services running)
deno run -A --env server.ts token create
# Copy the printed MYCELIA_TOKEN and MYCELIA_CLIENT_ID into your .env
```

### Backend Development

```bash
cd backend

# Start development server (http://localhost:5173)
deno task dev

# Alternative: run server.ts directly with custom commands
deno run -A --env server.ts serve

# Generate OAuth tokens
deno run -A --env server.ts token create

# Validate a token
deno run -A --env server.ts token validate

# Start audio processing worker
deno run -A --env server.ts audio worker

# Lint code
deno lint

# Type check (Deno has built-in type checking)
deno check server.ts
```

### Frontend Development

```bash
cd frontend

# Start development server (http://localhost:3001)
deno task dev

# Build for production
deno task build

# Preview production build
deno task preview

# Type check
deno task type-check

# Lint
deno lint

# Run tests
deno task test
```

### Python Audio Processing

```bash
cd python

# Run the daemon (continuously imports and processes audio)
uv run daemon.py

# Run speech-to-text processor
uv run stt.py

# List files with ingestion errors
uv run manage_errors.py list

# Retry all errored files
uv run manage_errors.py retry-all

# Retry specific file
uv run manage_errors.py retry <file_id>
```

### Remote CLI Operations

```bash
cd backend

# Login to remote server
deno run --env -E='MYCELIA_*' --allow-net cli.ts login

# Import audio file to remote server
deno run --env -E='MYCELIA_*' --allow-net cli.ts audio import /path/to/file.wav

# MCP operations (timeline, MongoDB, Redis, GridFS)
deno run --env -E='MYCELIA_*' --allow-net cli.ts mcp list
deno run --env -E='MYCELIA_*' --allow-net cli.ts mcp call tech.mycelia.timeline -a '{"action": "recalculate", "all": true}'
deno run --env -E='MYCELIA_*' --allow-net cli.ts mcp call tech.mycelia.mongo -a '{"action": "find", "collection": "audio_chunks", "query": {}, "options": {"limit": 10}}'
```

## Architecture

### Backend Architecture (Deno + Remix)

**Tech Stack:**
- **Runtime**: Deno with Node compatibility
- **Framework**: Remix with React Router v7
- **Server**: Express + Vite dev server
- **Database**: MongoDB with GridFS for file storage
- **Cache**: Redis with Redlock for distributed locking
- **Queue**: BullMQ with Redis backend
- **Message Broker**: Kafka for event streaming
- **Auth**: Custom OAuth2 implementation with JWT tokens
- **Observability**: OpenTelemetry for metrics and tracing
- **Testing**: Deno standard library testing + Testcontainers

**Key Architectural Patterns:**

1. **Resource-Based Authorization System** (`app/lib/auth/`):
   - Policy-based access control with `allow`, `deny`, and `modify` effects
   - Resource definitions with Zod schemas for request/response validation
   - Middleware modifiers for complex authorization logic
   - Glob pattern matching for resource paths (uses minimatch)
   - All resources registered in `ResourceManager` with `extractActions()` for permission checks

2. **Model Context Protocol (MCP)** (`app/lib/mcp/`):
   - MCP server implementation for exposing resources to AI models
   - Tools registered for timeline, MongoDB, Redis, and GridFS operations
   - Compliance tests to ensure MCP specification adherence
   - Route handler at `/mcp` for external MCP clients

3. **Modular Timeline System** (`app/modules/`, `backend/app/core.ts`):
   - **Layer**: React components that render on timeline canvas (e.g., audio, events, map)
   - **Tool**: UI controls for timeline manipulation
   - **Config**: Registry of enabled layers and tools
   - Each module exports its own Layer/Tool implementations

4. **MongoDB Collections** (`app/lib/mongo/collections.ts`):
   - Regular collections: `api_keys`, `events`, `audio_chunks`, `transcriptions`, `diarizations`, `source_files`, `people`, `conversations`, `objects`
   - GridFS buckets: `audio-files`
   - Histogram collections: `histogram_<resolution>` for timeline aggregations
   - Auto-creates missing collections on startup via `ensureAllCollectionsExist()`

5. **Audio Processing Pipeline**:
   - Audio ingestion via `/api/audio/ingest` route
   - GridFS storage for audio files
   - BullMQ job queue for processing (chunking, VAD, transcription)
   - Python worker processes handle heavy lifting (see Python section)

6. **Remix Routes** (`app/routes/`):
   - `_dash.*` - Dashboard UI routes (Remix SSR)
   - `api.*` - RESTful API endpoints (resource ingestion, file operations)
   - `data.*` - Data endpoints for timeline items, audio chunks
   - `oauth.token` - OAuth2 token endpoint
   - `[.]well-known.*` - OAuth2 discovery endpoints
   - `mcp.tsx` - MCP protocol handler
   - `llm.chat.completions.tsx` - OpenAI-compatible LLM endpoint

**Import Conventions:**
- Use `@/` for app directory imports: `import { foo } from '@/lib/utils'`
- Use `#/` for project root imports: `import config from '#/config'`
- JSR packages: `import { Command } from '@cliffy/command'`
- npm packages: handled via Deno's npm compatibility

### Frontend Architecture (React SPA)

See `frontend/CLAUDE.md` for detailed frontend documentation. Key points:

- Standalone SPA with React Router v7
- D3.js-based timeline with zoom/pan
- Zustand for state management
- Modular layer system for timeline rendering
- API client with OAuth2 token exchange
- Settings stored in localStorage

### Python Audio Processing

**Purpose**: Background daemon and processors for audio ingestion pipeline

**Key Components:**

1. **daemon.py** - Main orchestrator:
   - Discovers new audio files from configured sources (Apple Voice Memos, Google Drive, local folders)
   - Ingests audio into MongoDB via backend API
   - Runs voice activity detection (VAD) on chunks
   - Continuous operation with batch processing (20 files per batch)
   - Error tracking with 2-hour retry delay
   - Logs to `~/Library/mycelia/logs/daemon.log`

2. **discovery.py** - File discovery:
   - `AppleVoiceMemosImporter`: Reads from `CloudRecordings.db`
   - `GoogleCloudImporter`: Scans Google Drive folders with timezone-aware timestamps
   - `LocalFilesystemImporter`: Watches local audio directory
   - Auto-detection with environment variable overrides

3. **chunking.py** - Audio processing:
   - Splits audio files into chunks using ffmpeg
   - Converts to Opus format for efficient storage
   - Uploads chunks to backend GridFS

4. **stt.py** - Speech-to-text:
   - Fetches audio chunks from backend
   - Sends to Whisper server (`STT_SERVER_URL`)
   - Stores transcriptions in MongoDB

5. **diarization.py** - Voice activity detection:
   - Uses pyannote-audio for speaker diarization
   - Detects speech segments in audio chunks

**Configuration** (`settings.py`):
- Auto-detects Apple Voice Memos and Google Drive paths on macOS
- Environment variables for custom paths and timezones
- Requires Full Disk Access on macOS for Voice Memos access

**Dependencies**:
- Managed by `uv` (Astral Python package manager)
- PortAudio required for `stt.py` (install via brew/apt)
- FFmpeg required for audio conversion

## Code Style and Conventions

### TypeScript (Backend & Frontend)

- **Strict mode enabled**: Use explicit types, avoid `any`
- **Interfaces vs Types**: Interfaces for object shapes, types for unions/primitives
- **Zod for validation**: Runtime validation for API inputs/outputs, URL params
- **No comments**: Code should be self-documenting with descriptive names
- **Import order**: React → Third-party → Internal → Types

### Python

- **uv for dependencies**: Use `uv run` to execute scripts
- **Type hints**: Use Python 3.12+ type annotations where applicable
- **Error handling**: Centralized error tracking in MongoDB via daemon

### Deno-Specific Patterns

- Use `deno.json` import maps instead of relative imports
- npm packages work via Deno's npm compatibility
- No `node_modules` directory (global cache)
- Use `deno task` for scripts, not npm scripts
- TypeScript support built-in, no separate compilation step

## Testing

### Backend Testing

Backend uses Deno standard library testing with Testcontainers for integration tests:

```bash
cd backend

# Run all tests
deno test -A

# Run specific test file
deno test -A app/lib/auth/tests/auth.test.ts

# Run tests with coverage
deno test -A --coverage=coverage/
```

Test files are co-located with source code in `tests/` subdirectories or `*.test.ts` files.

### Frontend Testing

Frontend uses Vitest:

```bash
cd frontend

# Run tests
deno task test

# Run with UI
deno task test:ui

# Run with coverage
deno task test:coverage
```

## Important Notes

### OAuth2 Flow

1. Generate client credentials: `deno run -A --env server.ts token create`
2. Client exchanges credentials for JWT via `/oauth/token`
3. JWT used for authenticated API requests
4. Frontend stores JWT in memory (6-hour expiry)

### Timeline Data Flow

1. Audio chunks stored in MongoDB with timestamps
2. Histogram collections pre-aggregate data by resolution (minute, hour, day, week, month, year)
3. Frontend queries histograms for visible time range
4. D3 scale maps timestamps to pixels
5. Zoom/pan updates timeline range store, triggers re-fetch

### Audio Processing Flow

1. Python daemon discovers new audio files
2. Daemon uploads via `/api/audio/ingest` endpoint
3. Backend stores in GridFS, creates `source_files` record
4. BullMQ job enqueued for chunking
5. Python worker processes chunks (VAD, transcription)
6. Results stored in `audio_chunks`, `transcriptions`, `diarizations` collections
7. Timeline invalidation triggers histogram recalculation

### MCP Integration

MCP (Model Context Protocol) allows AI models to access Mycelia resources:

- Exposed tools: timeline operations, MongoDB queries, Redis operations, GridFS file access
- Authentication via standard OAuth2 tokens
- Request/response use EJSON serialization for MongoDB types
- Tool schemas auto-generated from Zod definitions

### Environment Variables

Critical environment variables (see `backend/.env.example`):
- `MYCELIA_TOKEN` - API key for CLI operations
- `MYCELIA_CLIENT_ID` - OAuth2 client ID
- `SECRET_KEY` - JWT signing key
- `MONGO_URL` - MongoDB connection string
- `REDIS_PASSWORD` - Redis password
- `KAFKA_ADMIN_PASSWORD` - Kafka admin password
- `STT_SERVER_URL` - Whisper server endpoint
- `OTEL_EXPORTER_OTLP_ENDPOINT` - OpenTelemetry collector endpoint
