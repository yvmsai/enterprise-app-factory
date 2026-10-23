---
name: docker-local
description: Containerizes a Spring Boot + React + PostgreSQL app and runs it locally with Docker Compose - multi-stage Dockerfiles, non-root images, nginx SPA serving with an /api proxy, health checks, .env handling, launch and verification steps. Use when packaging the app, running it locally, or debugging a container that won't start.
---

# Docker Local

Copy the templates and replace `<app>` with the app name:

| Template | Destination |
|---|---|
| [templates/backend.Dockerfile](templates/backend.Dockerfile) | `backend/Dockerfile` |
| [templates/frontend.Dockerfile](templates/frontend.Dockerfile) | `frontend/Dockerfile` |
| [templates/nginx.conf](templates/nginx.conf) | `frontend/nginx.conf` |
| [templates/docker-compose.yml](templates/docker-compose.yml) | `docker-compose.yml` |
| [templates/env.example](templates/env.example) | `.env.example` |

Also add `.dockerignore` files: `target/` for the backend, `node_modules/` and `dist/` for
the frontend.

## How it fits together

```
http://localhost:3000 ─► frontend (nginx) ── /api/* ──► backend:8080 ──► db:5432
                              serves SPA                  Spring Boot       PostgreSQL
```

The frontend calls relative `/api/...` URLs and nginx proxies them, so there's no CORS
locally and no API URL baked into the image.

## Launch procedure

1. `docker info` must succeed. If it doesn't, ask the user to start Docker Desktop.
2. If `.env` doesn't exist, copy `.env.example` to `.env` and replace the placeholders
   with local-only values: `JWT_SECRET=$(openssl rand -base64 32)` and a random
   `DB_PASSWORD`. Confirm `.env` is listed in `.gitignore`.
3. `docker compose up --build -d`
4. Wait for health, up to about 3 minutes, since the first Maven build is slow:
   ```bash
   until [ "$(docker inspect -f '{{.State.Health.Status}}' "$(docker compose ps -q backend)")" = "healthy" ]; do sleep 5; done
   ```
   Run this in the background or with a timeout so a crash doesn't hang the session.
5. Verify:
   ```bash
   curl -fsS http://localhost:8080/actuator/health        # {"status":"UP",...}
   curl -fsS -o /dev/null -w "%{http_code}\n" http://localhost:3000   # 200
   ```
6. Report the URLs:
   - App: http://localhost:3000
   - API: http://localhost:8080/api/v1
   - Swagger UI: http://localhost:8080/swagger-ui.html
   - Seeded local login (from `db/seed`), if one exists

## Troubleshooting

| Symptom | Check |
|---|---|
| Backend exits immediately | `docker compose logs backend --tail 50`. Usually a missing env var or a Flyway/entity mismatch. |
| `Schema-validation: missing column` | The entity and migration disagree. Fix them with a new migration. |
| Frontend 502 on `/api` | Backend not healthy yet, or the service name isn't `backend` |
| Port already in use | `lsof -i :3000` / `:8080` / `:5432`, then change the host port in Compose |
| Want a clean database | `docker compose down -v` (deletes local data, so ask first) |

## Image rules

- Multi-stage builds. Runtime images contain no build tools or sources.
- Run as a non-root user.
- Pin base images to a major version (`postgres:16-alpine`), never `latest`.
- JVM flag `-XX:MaxRAMPercentage=75`, so the API fits in small free-tier containers (512 MB).
