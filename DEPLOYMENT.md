# Wasl AI — Production Deployment Guide
**Target Stack:** Render (Web Service & Static Site) + Supabase (PostgreSQL)  
**Cost:** $0.00 / Free Tier MVP

---

## 1. Architecture

```
                    INTERNET
                       │
                       ▼
             ┌───────────────────┐
             │ Render Static Site│
             │ React + Vite      │
             │ HTTPS             │
             └─────────┬─────────┘
                       │
                       │ HTTPS REST API
                       ▼
             ┌───────────────────┐
             │ Render Web Service│
             │ FastAPI           │
             │ Gunicorn/Uvicorn  │
             │ HTTPS             │
             └───────┬─────┬─────┘
                     │     │
                     │     └──────────► Gemini API
                     │
                     ▼
             ┌───────────────────┐
             │ Supabase          │
             │ PostgreSQL 16     │
             └───────────────────┘
```

- **Frontend:** Render Static Site running the minified React/TypeScript SPA build with `_redirects` client-side routing.
- **Backend:** Render Web Service running FastAPI under `Gunicorn` with `UvicornWorker` binding dynamically to `0.0.0.0:$PORT`.
- **Database:** Supabase managed PostgreSQL 16 using standard SQLAlchemy connections. No proprietary Supabase Auth/SDK needed.
- **AI Receptionist:** Google Gemini 1.5 Flash via server-side API keys.

---

## 2. Prerequisites

1. A GitHub or GitLab repository containing the Wasl AI codebase.
2. A free account on [Render.com](https://render.com).
3. A free account on [Supabase.com](https://supabase.com).
4. A Google Gemini API key from [Google AI Studio](https://aistudio.google.com/).

---

## 3. Supabase Setup

1. Log into your **Supabase Dashboard** and click **"New project"**.
2. Fill in the project details:
   - **Name:** `wasl-ai-db`
   - **Database Password:** Choose a strong password and save it securely.
   - **Region:** Select the region closest to your Render services (e.g. `Frankfurt (eu-central-1)` or `Oregon (us-west-1)`).
   - **Pricing Plan:** Free Tier.
3. Once the database is provisioned (approx. 2 minutes), navigate to **Project Settings -> Database**.
4. Locate the **Connection string** section:
   - Select **URI** mode.
   - If using connection pooling, select **Transaction Mode** (Port `6543`) or **Session Mode** (Port `5432`).
   - Format: `postgresql://postgres.[REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres`
   - Prepend `+psycopg2` so SQLAlchemy recognizes the driver:  
     `postgresql+psycopg2://postgres.[REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres`

---

## 4. Environment Variables

### Backend Web Service Environment Variables
| Variable | Value / Description | Example |
| :--- | :--- | :--- |
| `ENVIRONMENT` | `production` | `production` |
| `DEBUG` | `false` | `false` |
| `PROJECT_NAME` | `Wasl AI` | `Wasl AI` |
| `API_V1_STR` | `/api/v1` | `/api/v1` |
| `SECRET_KEY` | Strong random key (32+ chars) | Generate via `openssl rand -hex 32` |
| `DATABASE_URL` | Supabase connection URI | `postgresql+psycopg2://postgres.xyz:pwd@...` |
| `GEMINI_API_KEY` | Google AI Studio Key | `AIzaSy...` |
| `GEMINI_MODEL` | `gemini-1.5-flash` | `gemini-1.5-flash` |
| `CORS_ALLOWED_ORIGINS` | Render Frontend URL | `https://wasl-ai-frontend.onrender.com` |
| `ALLOWED_HOSTS` | `*` or specific Render domains | `*` |
| `ENABLE_API_DOCS` | `false` | `false` |
| `LOG_LEVEL` | `INFO` | `INFO` |

### Frontend Static Site Environment Variables
| Variable | Value / Description | Example |
| :--- | :--- | :--- |
| `VITE_API_URL` | Full URL of your Render Web Service | `https://wasl-ai-backend.onrender.com` |

---

## 5. Render Backend Setup (Web Service)

### Option A: Via Blueprint (`render.yaml`)
1. In Render Dashboard, click **New + -> Blueprint**.
2. Connect your Git repository. Render will detect `render.yaml`.
3. Set the required secret values (`DATABASE_URL`, `GEMINI_API_KEY`, `CORS_ALLOWED_ORIGINS`).

### Option B: Manual Setup via Dashboard
1. Click **New + -> Web Service**.
2. Connect repository.
3. Configure:
   - **Name:** `wasl-ai-backend`
   - **Region:** Same region as Supabase (e.g., Frankfurt)
   - **Branch:** `main`
   - **Root Directory:** `backend`
   - **Runtime:** `Python`
   - **Build Command:** `pip install --upgrade pip && pip install -r requirements.txt`
   - **Start Command:** `gunicorn -k uvicorn.workers.UvicornWorker -c gunicorn_conf.py app.main:app`
   - **Plan:** Free
4. Under **Advanced**:
   - **Health Check Path:** `/health`
5. Add the Environment Variables listed in Section 4.
6. Click **Create Web Service**.

---

## 6. Render Frontend Setup (Static Site)

1. In Render Dashboard, click **New + -> Static Site**.
2. Connect repository.
3. Configure:
   - **Name:** `wasl-ai-frontend`
   - **Branch:** `main`
   - **Root Directory:** `frontend`
   - **Build Command:** `npm install && npm run build`
   - **Publish Directory:** `dist`
4. Under **Redirects / Rewrites**:
   - Add a rewrite rule:
     - **Type:** `Rewrite`
     - **Source:** `/*`
     - **Destination:** `/index.html`
   *(Note: The repository also includes `frontend/public/_redirects` which Render automatically respects).*
5. Under **Environment Variables**:
   - Add `VITE_API_URL` with your backend URL (e.g. `https://wasl-ai-backend.onrender.com`).
6. Click **Create Static Site**.

---

## 7. Alembic Migrations

Once your Supabase database is online and your backend is configured, apply the database migrations:

### Running from Local Machine:
```bash
# Set database URL to Supabase instance
export DATABASE_URL="postgresql+psycopg2://postgres.[REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres"

# Execute migrations to head
cd backend
alembic upgrade head
```

### Verification:
```bash
alembic current
```
Output must show `a21_billing_subscriptions (head)`.

---

## 8. First Deployment & Verification

1. Check backend liveness:
   ```bash
   curl -i https://wasl-ai-backend.onrender.com/health
   # Returns: {"status":"healthy","app":"Wasl AI","environment":"production","version":"0.1.0"}
   ```
2. Check database readiness:
   ```bash
   curl -i https://wasl-ai-backend.onrender.com/ready
   # Returns: {"status":"ready","database":"connected","app":"Wasl AI"}
   ```
3. Open the frontend in browser: `https://wasl-ai-frontend.onrender.com`.

---

## 9. Post-Deployment Verification Checklist

1. **Authentication:** Register a new business owner account at `/register`.
2. **Business Creation:** Create a new business and complete the onboarding wizard.
3. **Free Trial:** Verify that 14-day Free Trial is provisioned under the Billing tab.
4. **Public Profile:** Visit `/business/{slug}` and verify branding and services display.
5. **AI Chat Receptionist:** Open the chat widget and converse in English or Arabic.
6. **Booking Flow:** Book a service appointment and verify reference code is generated.
7. **CRM & Inbox:** Verify the lead and conversation appear in Owner Dashboard.
8. **Direct Route Navigation:** Refresh on `/dashboard` or `/login` to verify no 404 errors occur.

---

## 10. Updating the Application

- Pushes to the `main` branch trigger automatic continuous deployments on Render.
- If a new database migration is added:
  1. Run `alembic upgrade head` against Supabase before or alongside deployment.
  2. Render will automatically build and restart services with zero configuration changes.

---

## 11. Database Backup & 12. Database Restore

Refer to [docs/DISASTER_RECOVERY.md](docs/DISASTER_RECOVERY.md) for full commands and procedures using `pg_dump` and `pg_restore`.

---

## 13. Troubleshooting

- **CORS Error on Frontend:**
  Ensure `CORS_ALLOWED_ORIGINS` in Render backend exactly matches your frontend URL (including `https://` and without trailing slash).
- **Backend Startup Fails (Fail-Fast):**
  Check Render Web Service logs. In production mode, Wasl AI fails fast if `SECRET_KEY` is shorter than 32 characters or if `DATABASE_URL` is missing.
- **Cold Starts:**
  On Render Free Tier, backend services spin down after 15 minutes of inactivity. The first incoming request may take 30-50 seconds to respond. This is expected on the free tier.

---

## 14. Rollback Strategy

1. In Render Dashboard, open **Events** -> select previous successful deploy -> click **Rollback to this deploy**.
2. If database rollback is needed, restore the previous dump via `pg_restore` as documented in `DISASTER_RECOVERY.md`.

---

## 15. Security Checklist

- [x] No plaintext passwords or JWT secrets in Git history.
- [x] `DEBUG=false` in production.
- [x] Sensitive parameters (`password`, `token`, `bearer`, `api_key`, `DATABASE_URL`) redacted in logs.
- [x] Strict tenant isolation enforced at ORM and service layer.
- [x] SlowAPI rate limiting enabled on `/auth/login`, `/auth/register`, and `/chat`.
- [x] `/docs` and `/redoc` disabled in production unless `ENABLE_API_DOCS=true`.
