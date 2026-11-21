# ViralCut AI

> AI-powered YouTube-to-short-form-video pipeline. Paste a YouTube URL, get viral-ready clips with captions, smart cropping, and metadata — ready to upload to TikTok, Reels, and Shorts.

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start (Docker)](#quick-start-docker)
- [Local Development (Windows)](#local-development-windows)
- [GPU Acceleration](#gpu-acceleration)
- [Environment Variables](#environment-variables)
- [Database Migrations](#database-migrations)
- [API Endpoints](#api-endpoints)
- [Production Deployment](#production-deployment)
- [Project Structure](#project-structure)

---

## Features

- 🎬 **YouTube Download** – paste any YouTube URL; yt-dlp fetches the video.
- 🗣️ **Transcription** – word-level timestamps via Faster Whisper (GPU-accelerated).
- 🧠 **AI Clip Detection** – GPT-4o analyses the transcript to identify viral-worthy segments.
- 🎯 **Smart Cropping** – MediaPipe face/pose detection keeps subjects centred in 9:16.
- ✨ **Animated Captions** – karaoke-style word highlights, keyword emphasis, speaker labels.
- 📊 **Viral Score** – each clip is scored for shareability.
- 📤 **Export** – MP4 / WebM / GIF in 720p / 1080p / 4K.

---

## Architecture

```
┌─────────────┐    ┌──────────────┐    ┌────────────┐
│  Next.js UI │───▶│  FastAPI API  │───▶│ PostgreSQL │
│  (port 3000)│    │  (port 8000)  │    │  (port 5432)│
└─────────────┘    └──────┬───────┘    └────────────┘
                          │
                   ┌──────▼───────┐    ┌────────────┐
                   │ Celery Worker │───▶│   Redis    │
                   │  (processing) │    │ (port 6379) │
                   └──────────────┘    └────────────┘
```

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Docker & Docker Compose | 24+ | Recommended for quick start |
| Python | 3.11+ | For local backend dev |
| Node.js | 20+ | For local frontend dev |
| FFmpeg | 6+ | Required for video processing |
| PostgreSQL | 16 | Docker provides this |
| Redis | 7 | Docker provides this |
| NVIDIA GPU (optional) | CUDA 12+ | For fast Whisper inference |

---

## Quick Start (Docker)

```bash
# 1. Clone the repo
git clone https://github.com/your-org/viralcut-ai.git
cd viralcut-ai

# 2. Create your .env from the template
cp .env.example .env
# Edit .env → set OPENAI_API_KEY and SECRET_KEY

# 3. Build and start all services
docker compose up --build -d

# 4. Run database migrations
docker compose exec backend alembic upgrade head

# 5. Open the app
#    Frontend → http://localhost:3000
#    API docs → http://localhost:8000/docs
```

To stop:

```bash
docker compose down          # stop containers
docker compose down -v       # stop & delete volumes (data!)
```

---

## Local Development (Windows)

### 1. Install system dependencies

```powershell
# Install FFmpeg via winget (or download from https://ffmpeg.org)
winget install Gyan.FFmpeg

# Verify
ffmpeg -version
```

### 2. Set up PostgreSQL & Redis

The easiest way is to run only the infra services via Docker:

```powershell
docker compose up postgres redis -d
```

### 3. Backend

```powershell
cd backend

# Create virtual environment
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# Install dependencies
pip install -r requirements.txt

# Create .env
copy .env.example .env
# Edit .env → set OPENAI_API_KEY, SECRET_KEY

# Run migrations
alembic upgrade head

# Start the API server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### 4. Celery Worker (separate terminal)

```powershell
cd backend
.\.venv\Scripts\Activate.ps1
celery -A app.workers.celery_app worker --loglevel=info --pool=solo
```

> **Note:** Use `--pool=solo` on Windows; `prefork` is not supported.

### 5. Frontend

```powershell
cd frontend

npm install
npm run dev
# → http://localhost:3000
```

---

## GPU Acceleration

ViralCut AI uses **Faster Whisper** which supports CUDA-accelerated inference.

### NVIDIA GPU Setup

1. Install [CUDA Toolkit 12.x](https://developer.nvidia.com/cuda-downloads)
2. Install [cuDNN 9.x](https://developer.nvidia.com/cudnn)
3. Set environment variables:

```bash
WHISPER_DEVICE=cuda
WHISPER_COMPUTE_TYPE=float16   # or int8 for lower VRAM
```

### Docker with GPU

Add the NVIDIA runtime to the `worker` service in `docker-compose.yml`:

```yaml
worker:
  # ...existing config...
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

Also install [`nvidia-container-toolkit`](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

### CPU-only mode

The defaults work without a GPU:

```bash
WHISPER_DEVICE=auto           # falls back to cpu
WHISPER_COMPUTE_TYPE=float32  # int8 also works on CPU
```

---

## Environment Variables

See [`.env.example`](.env.example) for all variables. Key ones:

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | — | **Required.** Your OpenAI API key |
| `SECRET_KEY` | (insecure default) | JWT signing key – change in production |
| `DATABASE_URL` | `postgresql+asyncpg://viralcut:viralcut_secret@localhost:5432/viralcut` | Async DB connection |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis connection |
| `WHISPER_MODEL_SIZE` | `base` | Whisper model: `tiny`, `base`, `small`, `medium`, `large-v3` |
| `STORAGE_BACKEND` | `local` | `local` or `s3` |
| `MAX_VIDEO_DURATION_MINUTES` | `120` | Max input video length |

---

## Database Migrations

```bash
# Create a new migration after model changes
alembic revision --autogenerate -m "describe your change"

# Apply all pending migrations
alembic upgrade head

# Rollback one migration
alembic downgrade -1

# View migration history
alembic history --verbose
```

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/projects` | Create project from YouTube URL |
| `GET` | `/api/projects` | List all projects |
| `GET` | `/api/projects/{id}` | Get project with clips & jobs |
| `DELETE` | `/api/projects/{id}` | Delete project and all data |
| `GET` | `/api/projects/{id}/clips` | List clips for a project |
| `GET` | `/api/clips/{id}` | Get clip details with captions |
| `PATCH` | `/api/clips/{id}` | Update clip metadata |
| `POST` | `/api/clips/{id}/export` | Export clip to file |
| `GET` | `/api/jobs/{id}` | Get job status |
| `WS` | `/ws/progress/{project_id}` | Real-time progress updates |
| `GET` | `/health` | Health check |

Full OpenAPI docs available at `http://localhost:8000/docs` when the server is running.

---

## Production Deployment

### Security Checklist

- [ ] Set a strong random `SECRET_KEY`
- [ ] Set `APP_ENV=production` and `DEBUG=false`
- [ ] Use a managed PostgreSQL instance (or proper backups)
- [ ] Use a managed Redis instance
- [ ] Set `CORS_ORIGINS` to your actual frontend domain
- [ ] Put the API behind a reverse proxy (nginx / Caddy / Traefik) with HTTPS
- [ ] Store media on S3-compatible storage (`STORAGE_BACKEND=s3`)

### Recommended Stack

```
                        ┌─── HTTPS ───┐
                        │   Caddy /   │
  Internet  ──────────▶│   nginx     │
                        └──┬─────┬───┘
                           │     │
                   :3000   │     │  :8000
              ┌────────────┘     └────────────┐
              ▼                               ▼
        ┌──────────┐                   ┌────────────┐
        │ Frontend │                   │  Backend   │
        │ (Next.js)│                   │  (FastAPI) │
        └──────────┘                   └─────┬──────┘
                                             │
                                      ┌──────▼──────┐
                                      │   Worker    │
                                      │  (Celery)   │
                                      └─────────────┘
```

### Docker Production Build

```bash
# Build with production settings
docker compose -f docker-compose.yml build

# Run in detached mode
docker compose up -d

# Scale workers for more throughput
docker compose up -d --scale worker=4
```

---

## Project Structure

```
viralcut-ai/
├── backend/
│   ├── alembic/                 # DB migrations
│   │   ├── env.py
│   │   ├── script.py.mako
│   │   └── versions/
│   ├── app/
│   │   ├── api/
│   │   │   └── routes/          # FastAPI route handlers
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   └── models.py        # SQLAlchemy ORM models
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   └── schemas.py       # Pydantic request/response schemas
│   │   ├── services/            # Business logic
│   │   ├── utils/               # Helpers
│   │   ├── workers/             # Celery tasks
│   │   ├── config.py            # Pydantic Settings
│   │   └── database.py          # SQLAlchemy engine & session
│   ├── alembic.ini
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   ├── public/
│   └── src/
│       ├── app/                 # Next.js app router
│       ├── components/          # React components
│       ├── hooks/               # Custom React hooks
│       ├── lib/                 # API client, utilities
│       ├── stores/              # State management
│       └── types/               # TypeScript types
├── storage/
│   ├── uploads/
│   ├── processed/
│   ├── thumbnails/
│   └── exports/
├── docker-compose.yml
├── Dockerfile.backend
├── Dockerfile.frontend
├── .env.example
└── README.md
```

---

## License

MIT
