---
description: Build and launch the current app locally with Docker Compose, then verify it is healthy
---

Use the `docker-local` skill to launch the application in the current directory:

1. Make sure Docker is running (`docker info`). If it isn't, ask the user to start Docker
   Desktop.
2. If `.env` is missing, create it from `.env.example` with freshly generated local-only
   values (`openssl rand -base64 32` for `JWT_SECRET`).
3. Run `docker compose up --build -d`, then wait for every service to report healthy.
4. Verify the API health endpoint and the frontend respond, as the skill describes.
5. Report the URLs (frontend, API, Swagger UI) and any seeded local login. If a service
   fails, show the last 50 lines of its logs and diagnose.
