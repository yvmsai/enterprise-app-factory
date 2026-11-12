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

If the user has linked the project with the Neon CLI (`neon init` writes `DATABASE_URL` to
`.env`), derive `NEON_DB_URL`, `NEON_DB_USERNAME` and `NEON_DB_PASSWORD` from it into the
gitignored `.env` with a script that never prints the values. The `postgresql://` URL is not
a valid Spring datasource URL, and `channel_binding` isn't needed. Leave the local `DB_*`
values alone, and tell the user which `.env` key to copy into which Render variable. Ask
them not to paste connection strings into chat.

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
   **Each variable belongs to exactly one service.** `FRONTEND_ORIGIN` goes on the **API**
   (the Docker web service), even though it holds the website's URL. Users often put it on
   the static site because of its name. The static site needs only `VITE_API_BASE_URL`.
4. Click **Apply**. The first deploy takes about 5–10 minutes. `JWT_SECRET` is generated
   automatically.
5. If Render gave the services different URLs (because the names were taken), update
   `FRONTEND_ORIGIN` and `VITE_API_BASE_URL` to the real URLs and redeploy both services.
6. Optional: in each service's settings, set auto-deploy to wait for CI checks to pass, if
   your Render plan offers that option.

## Verification (agent, after the user says it's deployed)

A `200` from both URLs is not enough: check that the two services are wired together.

```bash
W=https://<app>-web.onrender.com; A=https://<app>-api.onrender.com
curl -fsS $A/actuator/health                                   # UP (allow ~60 s on a cold start)
curl -fsS -o /dev/null -w "%{http_code}\n" $W                   # 200
curl -fsS -o /dev/null -w "%{http_code}\n" $W/products          # 200: SPA rewrite works
# API URL baked into the SPA bundle
curl -s "$W$(curl -s $W | grep -o '/assets/index-[^"]*\.js' | head -1)" | grep -o 'https://[a-z0-9.-]*onrender.com' | sort -u
# CORS preflight from the site: expect 200 + access-control-allow-origin: $W
curl -s -o /dev/null -D - -X OPTIONS $A/api/v1/auth/login -H "Origin: $W" \
  -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: content-type" | grep -i '^HTTP\|allow-origin'
# Another origin must still be rejected: expect 403
curl -s -o /dev/null -w "%{http_code}\n" -X OPTIONS $A/api/v1/auth/login -H "Origin: https://example.com" -H "Access-Control-Request-Method: POST"
# DB reachable: a wrong login returns 401 problem-detail (not 500)
curl -s -X POST $A/api/v1/auth/login -H 'Content-Type: application/json' -d '{"email":"x@example.com","password":"wrong"}'
```

Don't test CORS with the API's own URL as `Origin`. Spring treats same-origin requests as
non-CORS and always lets them through, whatever `FRONTEND_ORIGIN` says.

Then load the site in a browser and submit a deliberately wrong login. Seeing "Invalid
email or password" proves the whole chain (browser → API → database). Ask the user to log
in with their admin account. Report both URLs and the cold-start caveat.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| API deploy fails "port not detected" | `server.port` isn't `${PORT:8080}` |
| API crashes with OOM | Missing `-XX:MaxRAMPercentage=75`, or Hikari pool too big |
| `Connection refused` / SSL error to DB | JDBC URL missing `?sslmode=require`, or the `postgres://` format was used instead of `jdbc:postgresql://` |
| Default `onrender.com` URLs return an instant 404 "Not Found" | No service exists under that name: the Blueprint wasn't applied, or Render added a suffix. Ask for the real URLs from the dashboard. |
| Preflight returns 403 "Invalid CORS request" for **every** origin, including `localhost` | `FRONTEND_ORIGIN` is empty on the API: it was put on the static site instead, or left blank in the Blueprint (`sync: false` creates the key with no value). Move it to the API and redeploy. |
| Browser shows CORS error for the site only | `FRONTEND_ORIGIN` doesn't match the site URL (wrong scheme, or Render's suffixed URL). `normalizeOrigins` already tolerates stray spaces, quotes and trailing slashes. |
| Page refresh gives 404 | Static-site rewrite rule `/*` → `/index.html` is missing |
| Frontend calls `localhost` | `VITE_API_BASE_URL` was set after the build. Redeploy the static site. |
