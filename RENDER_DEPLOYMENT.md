# Deploying Vantora CRM to Render

Four Render resources, created in this order:

| # | Resource | Render type | Source |
|---|----------|-------------|--------|
| 1 | PostgreSQL 17 | PostgreSQL | managed |
| 2 | Redis-compatible cache | Key Value | managed |
| 3 | Spring Boot API | Web Service (Docker) | `backend/` |
| 4 | React/Vite UI | Static Site | `frontend/` |

Create **all four in the same region**. Internal URLs only work between services in the same region and account.

> **Read first: email is mandatory.** Registration only creates the user after the
> emailed verification link is opened, and login is refused until then. With
> `MAIL_ENABLED=false` nobody can ever sign up. You need a real SMTP provider
> (Brevo, Mailgun, SendGrid, Postmark, Amazon SES…). Mailpit is local-only.
> Render has restricted outbound SMTP on free instances (ports 25/465/587); confirm
> current rules in Render's docs, and if your provider offers port 2525 or an HTTPS
> API, check which one your plan allows. This repo only speaks SMTP.

---

## 1. PostgreSQL

Dashboard → **New → PostgreSQL**

| Field | Value |
|-------|-------|
| Name | `vantora-db` |
| Database / User | `vantora` / `vantora` (any) |
| Region | your choice (same for everything) |
| PostgreSQL Version | **17** |

After it is created, open it and copy from the **Connections** panel: *Hostname* (internal), *Port*, *Database*, *Username*, *Password*.

Build the JDBC URL yourself. Render gives `postgresql://…`, but Spring needs JDBC syntax and separate credentials:

```
SPRING_DATASOURCE_URL      = jdbc:postgresql://<internal-hostname>:5432/<database>
SPRING_DATASOURCE_USERNAME = <username>
SPRING_DATASOURCE_PASSWORD = <password>
```

If you ever connect from outside Render (e.g. from your laptop) use the **external** hostname and append `?sslmode=require`.

Flyway runs automatically at API startup (`V1__baseline.sql`; `ddl-auto: validate`). Nothing to run manually.

## 2. Key Value (Redis)

Dashboard → **New → Key Value**

| Field | Value |
|-------|-------|
| Name | `vantora-redis` |
| Region | same as above |
| Maxmemory policy | `allkeys-lru` (it is only used as a cache and rate-limit store) |

Copy the **Internal** URL (looks like `redis://red-xxxxxxxx:6379`). Use it as:

```
SPRING_DATA_REDIS_URL = <internal key value url>
```

Do **not** set `SPRING_DATA_REDIS_HOST/PORT` as well; the URL replaces them. Redis here is a cache and a rate-limit store with in-process fallback, so losing its data is harmless. If you use the *external* URL it begins with `rediss://` (TLS) and contains a password; Spring handles that from the URL alone.

## 3. Backend Web Service

Dashboard → **New → Web Service** → connect https://github.com/Abhishek102501/vantora-crm.

| Field | Value |
|-------|-------|
| Name | `vantora-api` |
| Region | same as above |
| Branch | `main` |
| Root Directory | `backend` |
| Runtime / Language | **Docker** |
| Dockerfile Path | `./Dockerfile` |
| Docker Build Command / Start Command | leave **blank** (the Dockerfile `ENTRYPOINT` starts the app) |
| Health Check Path | `/actuator/health` |
| Instance type | at least **512 MB** RAM (JVM is capped at 75 % of it); 1 GB is more comfortable |
| Auto-Deploy | on commit (or off for manual releases) |

Render injects `PORT` (default `10000`). `application.yml` now reads `server.port: ${SERVER_PORT:${PORT:8080}}`, so the app binds the right port with no extra setting. Do not set `PORT` yourself.

The first build downloads Maven dependencies and takes several minutes; the JVM takes ~30–60 s to start. If Render times out the health check on a small instance, raise the instance size.

### Backend environment variables

Set under **Environment**. Mark everything that is a credential as a secret; never commit these.

**Required**

| Variable | Value |
|----------|-------|
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://<internal-host>:5432/<db>` |
| `SPRING_DATASOURCE_USERNAME` | from step 1 |
| `SPRING_DATASOURCE_PASSWORD` | from step 1 |
| `SPRING_DATA_REDIS_URL` | from step 2 |
| `JWT_SECRET` | random, **≥ 32 bytes**; must not contain `change-me` / `must-be-replaced`. Generate: `openssl rand -base64 48` (or let Render *Generate* one). The app refuses to start otherwise. Rotating it logs everyone out. |
| `APP_CORS_ALLOWED_ORIGINS` | exact frontend origin, e.g. `https://vantora-web.onrender.com` (no trailing slash, no path; comma-separate multiple) |
| `APP_FRONTEND_BASE_URL` | same origin as above. Used to build links in verification / password-reset emails |
| `APP_PUBLIC_BASE_URL` | the API's own public URL, e.g. `https://vantora-api.onrender.com` |
| `COOKIE_SECURE` | `true` |
| `COOKIE_SAME_SITE` | `None` when frontend and API are on different `*.onrender.com` hosts (see §Cookies); `Lax` if they share a registrable domain via custom domains |
| `SERVER_FORWARD_HEADERS_STRATEGY` | `native` (see below) |
| `MAIL_ENABLED` | `true` |
| `MAIL_HOST` | your SMTP host |
| `MAIL_PORT` | provider port (e.g. `587`, `2525`) |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | SMTP credentials (secret) |
| `MAIL_AUTH` | `true` |
| `MAIL_STARTTLS` | `true` (for port 587/2525) |
| `MAIL_FROM` | a sender your provider has verified, e.g. `no-reply@yourdomain.com` |

**Optional / tuning** (defaults are fine to start)

`DB_POOL_SIZE` (15; reduce on small Postgres plans), `JWT_ACCESS_TOKEN_TTL` (15m), `JWT_REFRESH_TOKEN_TTL` (7d), `JWT_ACTION_TOKEN_TTL` (30m: lifetime of email links), `COOKIE_DOMAIN` (blank), `AUTH_RATE_LIMIT`, `MUTATION_RATE_LIMIT_*`, the `MAX_*` quotas, `REMINDER_CRON`, `TOKEN_CLEANUP_CRON`. See `.env.example`.

**Do not set**: `SPRING_PROFILES_ACTIVE=dev` (that profile enables demo seeding and SQL formatting), `SERVER_PORT`, `DEMO_ENABLED`.

**Why `SERVER_FORWARD_HEADERS_STRATEGY=native`:** Render terminates TLS and proxies to your container. Without forwarded-header handling the app sees every request coming from the proxy's IP, so the login/register rate limiter (20/min per IP by default) would be shared by all users. `native` makes Tomcat honour `X-Forwarded-For`/`-Proto` from the platform's private-network proxy (verified locally: different forwarded IPs get separate rate-limit buckets). Local Docker Compose keeps `framework`; this is only for Render.

### File uploads

`STORAGE_PROVIDER=local` (the default) writes to the container filesystem, which Render **wipes on every deploy/restart** unless you attach a paid persistent disk (which also limits you to one instance). Options:

- **S3 / S3-compatible (recommended):** `STORAGE_PROVIDER=s3`, `AWS_S3_BUCKET`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_S3_ENDPOINT` for R2/Backblaze/MinIO. Startup fails fast if the bucket is empty.
- **Persistent Disk:** add a disk mounted at `/app/data/uploads` and set `STORAGE_LOCAL_PATH=/app/data/uploads`. Note the image runs as user `app`; if Render mounts the disk root-owned, uploads will fail with a permission error, so prefer S3.
- **Accept ephemeral files** for a demo; uploaded files will disappear.

## 4. Frontend Static Site

Dashboard → **New → Static Site** → same repository.

| Field | Value |
|-------|-------|
| Name | `vantora-web` |
| Branch | `main` |
| Root Directory | `frontend` |
| Build Command | `npm ci && npm run build` |
| Publish Directory | `dist` |

**Environment variables** (build-time; Vite inlines them, so changing one requires a rebuild):

| Variable | Value |
|----------|-------|
| `VITE_API_BASE_URL` | `https://vantora-api.onrender.com/api/v1` (your real API URL **including `/api/v1`**, no trailing slash) |
| `NODE_VERSION` | `22` (Vite 7 needs Node ≥ 20.19; the local Dockerfile uses 24) |

Leave `VITE_DEMO_MODE` unset.

**Redirects/Rewrites** (Static Site → *Redirects/Rewrites*): add one rule so React Router deep links (`/login`, `/verify-email`, `/reset-password`…) work on refresh:

| Source | Destination | Action |
|--------|-------------|--------|
| `/*` | `/index.html` | **Rewrite** |

Without this, opening the email verification link returns a 404.

The `frontend/Dockerfile` and `nginx.conf` are for Docker Compose only; they proxy `/api` to a host called `backend`, which does not exist on Render. Do not use them for the Static Site.

## 5. Order of operations (resolving the URL chicken-and-egg)

1. Create PostgreSQL and Key Value.
2. Create the backend Web Service with the variables above. For the not-yet-known frontend URL, put your intended name (`https://vantora-web.onrender.com`) in `APP_CORS_ALLOWED_ORIGINS` and `APP_FRONTEND_BASE_URL`. Deploy; wait for **Live**.
3. Check `https://<api>/actuator/health` returns `{"status":"UP"}`.
4. Create the Static Site with `VITE_API_BASE_URL` pointing at the API.
5. Render may append a random suffix if the name is taken. If the real frontend URL differs from your guess, correct `APP_CORS_ALLOWED_ORIGINS` and `APP_FRONTEND_BASE_URL` on the backend (this triggers a redeploy).
6. Smoke test (below).

## 6. CORS and cookies

- The backend allows only the origins in `APP_CORS_ALLOWED_ORIGINS`, with credentials, methods `GET POST PUT PATCH DELETE OPTIONS` and headers `Authorization, Content-Type, X-Request-Id`. A wrong value shows up in the browser console as a CORS error and in a `403` on the `OPTIONS` preflight.
- Auth model: a 15-minute access token (kept in memory, sent as `Authorization: Bearer`) plus a `HttpOnly` refresh cookie `mtcrm_refresh` scoped to `/api/v1/auth`.
- `*.onrender.com` is on the Public Suffix List, so `vantora-web.onrender.com` and `vantora-api.onrender.com` are **different sites**. The refresh cookie is therefore a third-party cookie and must be `Secure; SameSite=None` (`COOKIE_SECURE=true`, `COOKIE_SAME_SITE=None`). Verified locally that the API emits exactly that.
- Browsers that block third-party cookies (Safari by default, Chrome in some modes/incognito) will log users out on page reload. **The robust fix is custom domains under one parent**, e.g. `app.example.com` (static site) and `api.example.com` (API). Then set `COOKIE_SAME_SITE=Lax`, update `APP_CORS_ALLOWED_ORIGINS`/`APP_FRONTEND_BASE_URL` to `https://app.example.com`, and rebuild the frontend with `VITE_API_BASE_URL=https://api.example.com/api/v1`.

## 7. Smoke test

```bash
API=https://vantora-api.onrender.com
WEB=https://vantora-web.onrender.com

curl -s $API/actuator/health                       # {"status":"UP"}

# CORS preflight: expect Access-Control-Allow-Origin: $WEB and Allow-Credentials: true
curl -si -X OPTIONS $API/api/v1/auth/login \
  -H "Origin: $WEB" -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type" | head -15
```

Then in the browser: open `$WEB` → **Create workspace** → open the email → click the link (should land on `/verify-email`, not a 404) → sign in → dashboard loads → reload the page and confirm you stay signed in (proves the refresh cookie works).

## 8. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| Deploy fails: `JWT_SECRET is required` / `must contain at least 32 bytes` | Set `JWT_SECRET` as above. |
| `Connection refused` / `UnknownHost` to Postgres | Wrong host or different region; use the **internal** hostname, JDBC form (`jdbc:postgresql://…`). |
| `password authentication failed` | Username/password copied wrongly; set them as separate variables, not inside the URL. |
| `FlywayException` / `Schema-validation: missing table` | Pointing at a non-empty DB with other tables, or a manually altered schema. Use a fresh database. |
| Health check fails / "No open ports detected" | Make sure you did not hard-code `SERVER_PORT`; check logs for a startup exception; use a larger instance (OOM kills show as restarts). |
| Redis warnings in logs | Cache errors are logged and ignored (the app continues without cache). Check `SPRING_DATA_REDIS_URL` is the internal URL from the same region. |
| Browser: CORS error | `APP_CORS_ALLOWED_ORIGINS` must exactly equal the frontend origin (scheme + host, no slash). Redeploy after changing. |
| Login works, but reload logs you out | Cookie blocked: check `COOKIE_SECURE=true` and `COOKIE_SAME_SITE=None`, or move to same-site custom domains (§6). |
| Register succeeds but no email arrives / can't log in | SMTP not configured or blocked; check backend logs for mail errors. Without a delivered verification link no account exists. Links expire after `JWT_ACTION_TOKEN_TTL` (30m); use **Resend verification**. |
| `/verify-email` or other deep link 404s | Missing `/* → /index.html` rewrite on the Static Site. |
| Frontend calls `…onrender.com/api/v1` on itself or `localhost` | `VITE_API_BASE_URL` missing at build time. Set it and trigger **Clear build cache & deploy**. |
| 429 on login for everyone | `SERVER_FORWARD_HEADERS_STRATEGY` is not `native`, so all users share one IP bucket. |
| Uploaded files vanish after deploy | Local storage is ephemeral; use S3 or a disk (§3). |
| First request after idle is slow (free tier) | Free web services spin down; the JVM cold start takes about a minute. |

## 9. Local development is unchanged

`docker compose up --build` still uses Mailpit, the bundled Postgres/Redis and the Nginx proxy exactly as before. The only code changes made for Render were the `PORT` fallback in `application.yml` and a port-aware `HEALTHCHECK` in `backend/Dockerfile`; with neither `PORT` nor `SERVER_PORT` set, both still resolve to 8080.
