---
name: free-cloud-deploy
description: Deploys a Spring Boot + React + PostgreSQL app to free cloud hosting - Render (Docker web service for the API, static site for the React SPA) with a Neon serverless PostgreSQL database - using a render.yaml Blueprint, a prod Spring profile, and a user-performed account and secrets setup. Use when the user wants the app live on the internet at no cost.
---

# Free Cloud Deployment: Render + Neon

```
https://<app>-web.onrender.com ──► https://<app>-api.onrender.com ──► Neon PostgreSQL
   Render static site (free)          Render web service, Docker (free)    (free, serverless)
```

## Why this combination

| Need | Choice | Free-tier notes (check the providers' pricing pages, they change) |
|---|---|---|
| API (Docker) | Render web service | Sleeps after ~15 min idle, cold start 30–60 s, 512 MB RAM |
| SPA | Render static site | Free, global CDN, always on |
| PostgreSQL | Neon | Free tier doesn't expire; scales to zero. Render's free Postgres is deleted after 30 days. |
| Deploy trigger | Render auto-deploy from GitHub | Redeploys on push to `main` |

Alternatives if the user prefers: Koyeb (API), Vercel or Netlify (SPA), Supabase (Postgres).

## Who does what

The agent **prepares files and instructions**. The **user** creates accounts, connects
GitHub, and pastes secrets into dashboards. The agent never asks for, receives, or types
passwords, connection strings or API keys.

## Agent steps

1. Confirm the gates in `testing-quality` pass and the repo is pushed to GitHub. Pushing
   requires the user's go-ahead.
2. Copy [templates/render.yaml](templates/render.yaml) to the repo root and replace `<app>`.
3. Add `backend/src/main/resources/application-prod.yml`:
   ```yaml
   spring:
     flyway:
       locations: classpath:db/migration      # no seed data in prod
   logging:
     level:
       root: INFO
   ```
4. Implement the one-time admin bootstrap from `app-security` (`ADMIN_EMAIL` /
   `ADMIN_PASSWORD`), so the user can log in to an empty prod database.
5. Commit: `chore(deploy): add Render blueprint and prod profile`, then push if the user
   agrees.
6. Give the user the numbered steps below, filled in with the real app name.

## User steps (hand these over verbatim, with `<app>` filled in)

**A. Create the database (Neon)**
1. Sign up at https://neon.tech and create a project named `<app>`, choosing the region
   closest to your Render region.
2. Open **Connection Details** and pick the **Java / JDBC** format if it's offered. Note
   the host, database, user and password.
3. Build the JDBC URL: `jdbc:postgresql://<host>/<database>?sslmode=require`. Use the
   pooled host (it contains `-pooler`) if it's shown.

**B. Deploy on Render**
1. Sign up at https://render.com with your GitHub account.
2. Click **New → Blueprint**, select this repository, and Render will read `render.yaml`.
3. When prompted for the variables marked `sync: false`, enter:
   | Service | Key | Value |
   |---|---|---|
   | `<app>-api` | `DB_URL` | the JDBC URL from A3 |
   | `<app>-api` | `DB_USERNAME` | Neon user |
   | `<app>-api` | `DB_PASSWORD` | Neon password |
   | `<app>-api` | `FRONTEND_ORIGIN` | `https://<app>-web.onrender.com` |
   | `<app>-api` | `ADMIN_EMAIL` / `ADMIN_PASSWORD` | your first admin login |
   | `<app>-web` | `VITE_API_BASE_URL` | `https://<app>-api.onrender.com` |
4. Click **Apply**. The first deploy takes about 5–10 minutes. `JWT_SECRET` is generated
   automatically.
5. If Render gave the services different URLs (because the names were taken), update
   `FRONTEND_ORIGIN` and `VITE_API_BASE_URL` to the real URLs and redeploy both services.
6. Optional: in each service's settings, set auto-deploy to wait for CI checks to pass, if
   your Render plan offers that option.

## Verification (agent, after the user says it's deployed)

```bash
curl -fsS https://<app>-api.onrender.com/actuator/health     # allow ~60 s on a cold start
curl -fsS -o /dev/null -w "%{http_code}\n" https://<app>-web.onrender.com   # 200
```

Then ask the user to log in with the admin account and try one create/list flow. Report
both URLs and the cold-start caveat.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| API deploy fails "port not detected" | `server.port` isn't `${PORT:8080}` |
| API crashes with OOM | Missing `-XX:MaxRAMPercentage=75`, or Hikari pool too big |
| `Connection refused` / SSL error to DB | JDBC URL missing `?sslmode=require`, or the `postgres://` format was used instead of `jdbc:postgresql://` |
| Browser shows CORS error | `FRONTEND_ORIGIN` doesn't exactly match the site URL (scheme, no trailing slash) |
| Page refresh gives 404 | Static-site rewrite rule `/*` → `/index.html` is missing |
| Frontend calls `localhost` | `VITE_API_BASE_URL` was set after the build. Redeploy the static site. |
