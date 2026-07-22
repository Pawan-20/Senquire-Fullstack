# Image Review Workbench

A React, FastAPI, Celery, Redis, and MongoDB Atlas application that processes and reviews batches of metal-surface images. A pretrained ResNet-18 classifier assigns one of six defect labels while the UI displays results live.

## Flow

1. React lists files in `backend/data/images/`. When the user creates a batch, FastAPI stores one batch and its queued image records in MongoDB.
2. The review page opens an SSE connection. After Redis confirms the subscription, React asks FastAPI to start processing, preventing early live events from being missed.
3. FastAPI submits three independent Celery tasks per image: classification, thumbnail generation, and metadata extraction. Redis is the broker and temporary result backend.
4. Celery uses a prefork pool. With `--concurrency=4`, four worker processes execute tasks concurrently; this is process-based parallelism, not application-level multithreading. Each process keeps its own model in memory.
5. A Celery chord waits for an image's three tasks. Its finalizer merges the results, updates MongoDB, and publishes a small JSON status event through Redis pub/sub.
6. FastAPI forwards events over SSE. React merges each event into local state and immediately displays status, prediction, confidence, and the thumbnail served over normal HTTP.
7. Redis pub/sub is non-durable, so React also fetches the MongoDB snapshot every second during processing. SSE is the low-latency path; polling recovers missed events.



## Why this architecture?

CPU-heavy inference should not occupy FastAPI request handlers. Celery separates long-running work from the API; Redis lets multiple worker processes or containers consume one queue; MongoDB remains the durable source of truth. The three independent operations per image can run concurrently, while the chord provides one clear persistence point. Worker concurrency can scale without changing the API or frontend.

SSE was chosen over WebSockets because this application has one narrow live requirement: the server sends progress to a consuming frontend. Client actions already use REST, there is no collaborative multi-user session, and no duplex channel is needed. SSE uses ordinary HTTP, the browser's native `EventSource`, and automatic reconnection.

## Limitations

1. Inference is CPU-only and every prefork child loads a separate ~128 MB checkpoint. Higher concurrency increases memory use and eventually causes CPU contention.
2. Redis pub/sub cannot replay missed events. One-second MongoDB reconciliation repairs the UI, but adds database traffic and cannot recreate every intermediate animation.
3. API and workers share local image/thumbnail paths. Scaling across hosts needs shared/object storage; the current open CORS policy and unauthenticated endpoints are also development-only.



## What can be improved

1. Add LLM guardrails: limit and validate questions, minimize supplied data, defend against prompt injection, redact sensitive metadata, ground answers, and add rate limits, timeouts, and output checks.
2. Improve processing speed by measuring the task mix, batching inference, using GPU acceleration where suitable, and routing inference separately from lightweight thumbnail/metadata work. Tune concurrency to available CPU and RAM.
3. Move files to shared object storage and use a durable event mechanism such as Redis Streams when replay is required. Add authentication, restricted CORS, observability, retry/dead-letter handling, and end-to-end latency metrics.



## Prerequisites

- Docker Engine/Desktop with the Compose plugin
- Python 3.11 and [Poetry](https://python-poetry.org/)
- Node.js 20.19+ (or 22.12+)
- A reachable MongoDB Atlas database
- Internet access for initial installs/model download



## One-time setup

```bash
cp backend/.env.example backend/.env
```

Set `MONGO_URI` in `backend/.env`. Add `GEMINI_API_KEY` only if chat is needed. Never commit this file.

```bash
cd backend
poetry install
cd ../frontend
npm ci
```



### Download the pretrained model before running

Download the model folder from **[[https://drive.google.com/drive/folders/1z6I9MI6835YDTUJWYmR-dA8XJmt2d3l5?usp=sharing](https://drive.google.com/drive/folders/1z6I9MI6835YDTUJWYmR-dA8XJmt2d3l5?usp=sharing)]** and place it in the backend. The structure of backend should now look like this : 

```text
backend/models/defect-classifier-resnet18/
├── pytorch_model.pth
└── .cache/                  # Hugging Face cache metadata, if supplied
```

The pretrained model is `[Seif-melz/defect-classifier-resnet18](https://huggingface.co/Seif-melz/defect-classifier-resnet18)`. The exact revision is pinned in `backend/.env.example`. If the checkpoint is missing, API/worker startup downloads it automatically from that pinned revision. `backend/models/` is ignored by Git.

In this workspace, `pytorch_model.pth` and its cache metadata have already been downloaded to the path above. Place source images in `backend/data/images/`; thumbnails are generated in `backend/data/thumbnails/`.

## Preferred workflow: run services individually

Use separate terminals.

### 1. Redis

```bash
docker compose up redis
```



### 2. FastAPI

```bash
cd backend
poetry run uvicorn app.main:app --reload --port 8000
```

API: [http://localhost:8000](http://localhost:8000); health: [http://localhost:8000/api/health](http://localhost:8000/api/health).

### 3. Celery

```bash
cd backend
poetry run celery -A app.tasks.celery_app worker --loglevel=info --concurrency=4
```

`--concurrency=4` starts four prefork child processes; adjust it for your CPU and memory.

### 4. Frontend

```bash
cd frontend
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

## Alternative: one Compose command

After creating `backend/.env`:

```bash
docker compose up --build
```

This starts Redis, FastAPI, one Celery worker container, and Vite. Source, data, and model directories are bind-mounted, so generated/downloaded files remain on the host. Open [http://localhost:5173](http://localhost:5173).

```bash
docker compose down
```

Scale workers or change per-container concurrency with:

```bash
CELERY_CONCURRENCY=2 docker compose up --build --scale celery_worker=3
```

Useful commands:

```bash
docker compose up redis
docker compose logs -f backend celery_worker
docker compose down --remove-orphans
```

---



## Entire architecture

```text
                         User's browser
                 +---------------------------+
                 | React + React Router      |
                 | Home / Review / Summary   |
                 +-------------+-------------+
                               |
             REST requests + SSE stream + static images
                               |
                 +-------------v-------------+
                 | FastAPI on localhost:8000 |
                 |                           |
                 | Folder API   Batch API    |
                 | Stream API   Chat API     |
                 | Summary API  Static files |
                 +--+---------+----------+---+
                    |         |          |
          async     |         |          | HTTPS text prompt
          reads/    |         |          |
          writes    |         |          v
                    |         |     +----------+
                    |         |     | Gemini   |
                    |         |     +----------+
                    |         |
           +--------v----+    | enqueue tasks / subscribe
           | MongoDB     |    |
           | Atlas       |    |
           | batches     |    |
           | images      |    |
           +--------^----+    |
                    |         v
         synchronous|   +-----------------------------+
         worker DB  |   | Redis in Docker             |
         updates    |   | Celery broker/result backend|
                    |   | live pub/sub channels       |
                    |   +--------------+--------------+
                    |                  |
                    |        tasks     | live events
                    |                  |
              +-----+------------------v---------+
              | Local Celery worker              |
              |                                  |
              | child 1: ResNet-18 model on CPU  |
              | child 2: ResNet-18 model on CPU  |
              | child 3: ResNet-18 model on CPU  |
              | child 4: ResNet-18 model on CPU  |
              |                                  |
              | classification / thumbnails /   |
              | metadata / image finalization    |
              +----------------+-----------------+
                               |
                     read/write filesystem
                               |
              +----------------v-----------------+
              | backend/data/images              |
              | backend/data/thumbnails          |
              | backend/models/... checkpoint    |
              +----------------------------------+
```

