# DevOps Assessment — Implementation Documentation

## Levels completed
- [x] Level 1 — Containerization & Application Infrastructure
- [x] Level 2 — Object Storage Integration
- [ ] Level 3 — Reverse Proxy Implementation
- [ ] Level 4 — CI/CD, Deployment, Rollback & Security

## Prerequisites
- Docker Desktop (or Docker Engine + Compose v2) installed and running
- Git

## Repository structure (infra additions on top of the provided app)
```
.
├── backend/
│   ├── Dockerfile          # production image: python:3.12-slim + uvicorn
│   └── app/...              # provided application code (untouched)
├── frontend/
│   ├── Dockerfile          # multi-stage: node build -> nginx runtime
│   ├── nginx.conf           # SPA routing + static asset caching
│   └── src/...               # provided application code (untouched)
├── docker-compose.yml       # local dev stack: postgres, localstack, backend, frontend
├── .env.example             # template for required environment variables
└── .gitignore               # excludes real .env files from version control
```

## Environment configuration
Copy the template and fill in values (no real secrets needed — everything runs locally against LocalStack and a throwaway Postgres container):
```bash
cp .env.example .env
```

Required variables (consumed by `backend/app/config.py` and the compose file):

| Variable | Purpose |
|---|---|
| `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Postgres credentials for the local container |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Fake credentials required by boto3; LocalStack does not validate them (`test`/`test` is the standard convention) |
| `AWS_REGION` | Region tag for the S3 client (`us-east-1`) |
| `S3_BUCKET_NAME` | Bucket name; auto-created by the backend on startup |

`DB_HOST`/`DB_PORT` and the S3 endpoint URLs are set directly in `docker-compose.yml` (they're infrastructure wiring, not secrets, so they don't belong in `.env`).

## Running the application locally
```bash
docker compose up -d --build
```

Check everything came up healthy:
```bash
docker compose ps
```
All four containers (`postgres`, `localstack`, `backend`, `frontend`) should show a healthy/running state.

## Stopping the application
```bash
docker compose down          # stops containers, keeps data (named volumes persist)
docker compose down -v       # stops containers AND wipes Postgres/LocalStack data — use for a clean-slate test
```

**Important**: always bring the whole stack down/up together (`docker compose down && docker compose up -d`) rather than restarting individual services. Restarting only some containers can leave the backend holding stale connections to a Postgres/LocalStack instance that was recreated underneath it.

## Application URLs
| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend API root | http://localhost:8000 |
| Backend interactive docs | http://localhost:8000/docs |

## Health-check endpoints
| Endpoint | Verifies |
|---|---|
| `GET /api/health` | Backend process is up |
| `GET /api/health/db` | Postgres connectivity (`SELECT 1`) |
| `GET /api/health/s3` | S3/LocalStack bucket reachability |
| `GET /api/health/full` | All of the above combined |

## Docker / Compose implementation notes

**Backend (`backend/Dockerfile`)**
- Base image: `python:3.12-slim`.
- Runs via `uvicorn app.main:app --host 0.0.0.0 --port 8000` — a production ASGI server, no `--reload`.
- All configuration (DB credentials, S3 endpoints, CORS origins) is injected via environment variables at container runtime, not baked into the image.
- Built-in `HEALTHCHECK` hits `/api/health`.

**Frontend (`frontend/Dockerfile`)**
- Multi-stage build: `node:20-alpine` builds the production Vite bundle (`npm run build`, which runs `tsc -b && vite build`), then `nginx:1.27-alpine` serves the static output — no Node.js or dev server present in the final image.
- `nginx.conf` adds `try_files $uri $uri/ /index.html;` so React Router's client-side routes don't 404 on a hard refresh.

**Compose (`docker-compose.yml`)**
- `postgres`: persists data to a named volume (`postgres_data`) so data survives container restarts.
- `backend` `depends_on: postgres` / `localstack` with `condition: service_healthy` — the backend won't start until both dependencies pass their own healthchecks.
- `restart: unless-stopped` on every service for resilience against unexpected crashes.
- No ports are published for `postgres` — only reachable inside the Docker network, by service name, from `backend`.

## Object storage (Level 2) implementation notes
- LocalStack (`localstack/localstack:3`) provides an S3-compatible API locally; its port is published to the host (`4566:4566`) as well as reachable inside the Docker network as `localstack:4566`.
- The bucket named by `S3_BUCKET_NAME` is created automatically on backend startup (`storage_service.ensure_bucket()` in `app/services/storage.py`) — no manual bucket-creation step is required.
- `storage.py` uses **two separate S3 clients** to solve the fact that the backend and the browser reach LocalStack from different network contexts:
  - An internal client configured with `S3_ENDPOINT_URL=http://localstack:4566` for all backend-side operations (upload, delete, health checks) — this hostname only resolves inside the Docker network.
  - A public/presign client configured with `S3_PUBLIC_ENDPOINT_URL=http://localhost:4566`, used only when generating presigned download URLs (`GET /api/files/{id}`), since those URLs are opened directly by the browser running on the host machine.
- This is entirely environment-variable driven — no application code was modified.

## Verification steps performed
1. `docker compose down -v && docker compose up -d --build` from a clean state.
2. Confirmed all four containers report healthy via `docker compose ps`.
3. Hit all four health endpoints directly with `curl`, confirmed `200 OK` and `"status":"ok"` on each.
4. Through the frontend UI at `http://localhost:5173`:
   - Confirmed the dashboard shows Database and Storage as connected.
   - Confirmed the 7 seeded items appear on the Items page.
   - Uploaded a file, confirmed it appears in the Files list.
   - Opened/downloaded the uploaded file successfully (exercises the presigned URL flow).
   - Deleted the file, confirmed it disappeared from the list and `/api/health/full` still reports healthy.
5. Ran `docker compose restart postgres backend` and confirmed previously created items/files were still present afterward, verifying the named volume persists data across restarts.

## Troubleshooting
- **Frontend container stuck in `Restarting`, logs show `nginx: [emerg] unknown directive ...`**: `frontend/nginx.conf` has malformed content (e.g. a stray line). Recreate the file with clean content and rebuild: `docker compose build frontend && docker compose up -d frontend`.
- **`/api/health/db` or `/api/health/s3` report disconnected after only some containers were restarted**: the backend is holding stale connections to a dependency container that got recreated. Bring the whole stack down and up together: `docker compose down && docker compose up -d`.
- **Changes to `.env` don't seem to take effect**: environment variables consumed at container *runtime* (DB/S3 credentials) just need `docker compose up -d` again. Frontend build-time variables (like `VITE_API_BASE_URL`, used starting Level 3) require a rebuild (`docker compose build frontend`), not just a restart, since Vite bakes them into the static bundle at build time.
- **Port already in use on `5173` or `8000`**: another process on the host is using that port; stop it or change the published port mapping in `docker-compose.yml`.
