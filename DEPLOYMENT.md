# Deployment

CRM AI runs as three pieces, on free tiers that don't expire the way the old
Railway trial did:

| Piece | Host | Notes |
|---|---|---|
| Postgres | **Neon** (free) | Permanent free tier; autosuspends when idle, wakes on connect. |
| Backend (`/backend`, Express + Prisma) | **Render** (free web service) | Auto-deploys on push to `main` via `render.yaml`. Health: `/health`. Spins down when idle — kept warm by an UptimeRobot ping. |
| Frontend (`/frontend`, Next.js) | **Vercel** (free) | Auto-deploys on push to `main`. |

Pushing to `main` redeploys backend (Render) and frontend (Vercel) automatically.

## First-time setup

### 1. Database — Neon
1. Sign in at https://console.neon.tech → **New Project** (name it `crm-ai`, region near you).
2. On the project dashboard, click **Connect** and copy the **Pooled connection** string
   (it contains `-pooler` in the host). This is your `DATABASE_URL`.
   - Prisma needs `?sslmode=require` on the end; Neon's string already includes it.

### 2. Backend — Render
1. At https://dashboard.render.com → **New → Blueprint**, connect the `jbass-dev/crm-ai` repo.
   Render reads `render.yaml` and creates the `crm-ai-backend` web service.
2. When prompted, fill the `sync: false` env vars:
   - `DATABASE_URL` = the Neon pooled string from step 1
   - `OPENAI_API_KEY` = your Groq key (`gsk_...`)
   - `CORS_ORIGIN` = your Vercel URL (set after step 3; use `*` temporarily, then tighten)
   - `JWT_SECRET` is auto-generated; `OPENAI_BASE_URL` and `OPENAI_MODEL` are preset for Groq.
3. Deploy. The start command runs `prisma db push` (creates the schema on the fresh Neon DB)
   then boots the server. Note the service URL, e.g. `https://crm-ai-backend.onrender.com`.
4. (Optional) Seed demo data: Render → the service → **Shell** → `node prisma/seed.js`.

### 3. Frontend — Vercel
1. At https://vercel.com/new, import `jbass-dev/crm-ai`.
2. Set **Root Directory** = `frontend` (Vercel auto-detects Next.js).
3. Add env var `NEXT_PUBLIC_API_URL` = the Render backend URL from step 2.
4. Deploy. Note the URL, e.g. `https://crm-ai.vercel.app`.
5. Go back to Render → backend env → set `CORS_ORIGIN` to that exact Vercel URL → redeploy.

### 4. Keep the backend warm — UptimeRobot
Render free services sleep after ~15 min idle (a ~30s cold start on the next hit).
1. At https://uptimerobot.com → **Add New Monitor** → HTTP(s).
2. URL = `https://<your-render-backend>.onrender.com/health`, interval 5 minutes.
This is the same trick StoryForge uses.

## Environment variables (reference)

**Backend (Render):** `DATABASE_URL`, `JWT_SECRET`, `OPENAI_API_KEY`, `OPENAI_BASE_URL`
(`https://api.groq.com/openai/v1`), `OPENAI_MODEL` (`openai/gpt-oss-120b`), `CORS_ORIGIN`.
Render sets `PORT` automatically; the app reads it.

**Frontend (Vercel):** `NEXT_PUBLIC_API_URL` = backend URL.

## Notes
- File uploads (`/backend/uploads`) are ephemeral on Render's free tier — fine for a demo,
  but they reset on redeploy. Move to Cloudflare R2 / S3 if you need durable uploads.
- Local dev is unchanged: `docker-compose up` still runs Postgres + backend + frontend.
- The `railway.json` files are legacy and unused by Render/Vercel; kept for reference.
