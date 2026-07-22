# Image Review Workbench

This project has two workflows:

- **Local:** run the frontend, FastAPI API, and Celery worker as separate host processes. Redis can run as one small Docker service, so Redis does not need to be installed on the host.
- **Docker Compose:** run the frontend, API, Celery worker(s), and Redis together.

MongoDB is not run locally or in Compose. Both workflows use the MongoDB Atlas URI configured in `backend/.env`.

## Prerequisites

For the Docker workflow, install Docker Engine or Docker Desktop with the Compose plugin. The first build and startup also need internet access to pull container images, install packages, and download the model from Hugging Face. You also need a reachable MongoDB Atlas database.

For the local workflow, install Python 3.11 and Node.js 20.19+ (or 22.12+), in addition to Docker for Redis.

## One-time setup

Create the backend environment file:

```bash
cp backend/.env.example backend/.env
```

Edit `backend/.env` and replace `MONGO_URI` with the real Atlas connection string. If Atlas network access is restricted, allow this machine's public IP. Add a Gemini API key only if the chat feature is needed. Never commit or share `backend/.env`; it contains credentials.

Images to process belong in `backend/data/images/`. Generated thumbnails are written to `backend/data/thumbnails/`.

## Run each part individually (Prefered workflow)

The commands below use separate terminals.

### 1. Redis

No host installation is required:

```bash
docker compose up redis
```

Redis is then available to local processes at `localhost:6379`.

### 2. Backend API

The pinned NEU ResNet-18 model is downloaded from Hugging Face into `backend/models/` on the first API/worker startup and then runs locally on CPU. The model is not stored in Git; `backend/models/` is intentionally ignored. Keep internet access available for the first startup (or whenever that directory is empty). Create and activate a Python environment, then start FastAPI:

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

The API is at <http://localhost:8000>; health check: <http://localhost:8000/api/health>.

### 3. Celery worker

From a second terminal, using the same Python environment:

```bash
cd backend
source .venv/bin/activate
celery -A app.tasks.celery_app worker --loglevel=info --concurrency=4
```

Change `--concurrency=4` to control how many worker processes this worker starts.

### 4. Frontend

```bash
cd frontend
npm ci
npm run dev
```

The frontend is at <http://localhost:5173> and calls the API at `http://localhost:8000`.

## Run everything together (Docker Compose workflow)

After creating `backend/.env`:

```bash
docker compose up --build
```

This starts Redis, FastAPI, one Celery worker container, and Vite. Open <http://localhost:5173>. The first startup can take longer while the pinned model checkpoint is downloaded; later starts reuse `backend/models/` through the bind mount.

Stop the stack with:

```bash
docker compose down
```

If this project was previously run with the old local MongoDB service, remove that one stale container once with:

```bash
docker compose down --remove-orphans
```

The source folders are mounted into the containers, so frontend and backend edits reload during development. Generated thumbnails remain under `backend/data/thumbnails/`. Downloaded checkpoint files remain under `backend/models/`.

## Scale workers

Scale the number of Celery worker containers:

```bash
docker compose up --build --scale celery_worker=3
```

Each container uses four processes by default. Override that per-container concurrency when starting Compose:

```bash
CELERY_CONCURRENCY=2 docker compose up --build --scale celery_worker=3
```

That example provides three worker containers with two processes each.

## Useful individual Docker commands

```bash
docker compose up redis
docker compose up backend
docker compose up celery_worker
docker compose up frontend
docker compose logs -f celery_worker
```

Services automatically start their declared dependencies. The backend and worker both require Redis and the Atlas URI from `backend/.env`.

