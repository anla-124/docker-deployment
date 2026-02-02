# Docker + Local Network Deployment Guide

This is the step-by-step playbook to make an app Docker-deploy-ready **from scratch** and verify it works on a fresh clone. It assumes a Node/Next-style stack like pdf-search.

---

## 1) Prepare the App (manual checklist)

1. Confirm how the app runs locally.
   - Identify commands in `package.json` (e.g., `npm run build`, `npm start`).
   - Note required Node version from `.nvmrc` or `package.json`.
2. List all required environment variables.
   - DB URLs, API keys, service account paths, etc.
3. Ensure there is a basic health check endpoint.
   - Prefer `/api/health` returning 200 OK.

---

## 2) Generate Docker Support (AI-first)

Use Claude Code with this prompt:

```
You are in a repo with no Docker support.
This is a Node/Next app.

Tasks:
1) Inspect package.json to determine build/run commands and Node version.
2) Create a multi-stage Dockerfile:
   - build stage: npm ci + npm run build
   - runtime stage: npm start
3) Add .dockerignore (node_modules, .env*, .next, dist, logs, tmp).
4) Create docker-compose.yml:
   - app service exposes port 3000
   - add any required dependencies (db/vector) if the app uses them
5) Create .env.template with placeholders for all required env vars.
6) Add a short RUNBOOK section in README with exact commands + health check.

Constraints:
- Do not include secrets.
- Use correct Node version from package.json / .nvmrc.
- App should start on port 3000.
- Add a simple /api/health endpoint if it doesn’t exist, or document a basic health check.
```

Expected files after this step:
- `Dockerfile`
- `docker-compose.yml`
- `.dockerignore`
- `.env.template`
- README RUNBOOK section (or `RUNBOOK.md`)

---

## 3) Human Verification (local)

0. Ensure Docker Desktop (or Docker Engine) is running.
1. Create env file:
   - `cp .env.template .env.local`
   - Fill in real values (do not commit secrets).
2. Start containers:
   - `docker compose up -d --build`
3. Health check:
   - `curl http://localhost:3000/api/health`
4. View logs if needed:
   - `docker compose logs -f app`
5. Stop:
   - `docker compose down`

---

## 4) Acceptance Test (fresh clone)

**Pass condition:** a fresh clone runs with only env/credentials setup and `docker compose up -d`, then health check succeeds.

Steps:
1. Commit and push all Docker files to GitHub.
2. Clone to a new folder.
3. Set up credentials:
   - `cp .env.template .env.local`
   - Fill in real values.
   - Add any required service account JSON files (e.g., `credentials/`).
4. Run:
   - `docker compose up -d --build`
5. Verify:
   - `curl http://localhost:3000/api/health`

**PASS**: container up + health check OK.  
**FAIL**: container won’t start or health check fails.

---

## 5) Troubleshooting (common fixes)

- Check logs:
  - `docker compose logs -f app`
- Rebuild clean:
  - `docker compose down && docker compose up -d --build`
- Verify env vars are set and ports match.
- Confirm credentials file path matches env var (e.g., `GOOGLE_APPLICATION_CREDENTIALS`).

---

## 6) Basic Docker Commands (reference)

- Build + start in background: `docker compose up -d --build`
- Start existing containers: `docker compose start`
- Stop without removing containers: `docker compose stop`
- Stop and remove containers: `docker compose down`
- Follow logs: `docker compose logs -f app`
- List running services: `docker compose ps`

Flag notes:
- `-d` runs containers in the background (detached).
- `--build` forces a rebuild of images before starting.

Meaning of up/down vs start/stop:
- **Up/Down** creates/removes containers (and builds images if needed).
- **Start/Stop** pauses/resumes existing containers created by `up`.

---

## 7) Share Local Dev Server on Company Wi‑Fi

If you want coworkers on the same Wi‑Fi to access your local dev server, you must bind the dev server to **0.0.0.0** and share your local IP.

Steps (Next.js example):
1. Update the dev script in `package.json`:
   - Change `"dev": "next dev"` to:
   - `"dev": "next dev -H 0.0.0.0 -p 3000"`
2. Run:
   - `npm run dev`
3. Find your local IP (e.g., `192.168.x.x`) and share:
   - `http://YOUR_IP:3000`

Notes:
- Make sure your laptop firewall allows inbound connections on port 3000.
- Everyone must be on the same Wi‑Fi/VLAN.
- If this is not Next.js, use the framework's equivalent "host 0.0.0.0" flag.

---

## 8) Set Up Google Login via Supabase

This app uses Supabase Auth with Google OAuth. After sign-in, Supabase handles the token exchange and session management automatically. The app code lives in `src/components/auth/oauth-buttons.tsx` (client) and `src/app/auth/callback/route.ts` (server callback).

### A. Create Google OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (or select an existing one).
3. Navigate to **APIs & Services > OAuth consent screen**.
   - Choose **External** user type (or Internal if using Google Workspace).
   - Fill in the required fields: App name, User support email, Developer contact.
   - Add scopes: `email`, `profile`, `openid`.
   - Add test users if the app is still in "Testing" publishing status.
4. Navigate to **APIs & Services > Credentials**.
5. Click **Create Credentials > OAuth client ID**.
   - Application type: **Web application**.
   - **Authorized redirect URIs** — add your Supabase callback URL:
     ```
     https://<your-supabase-project-ref>.supabase.co/auth/v1/callback
     ```
     You can find your project ref in the Supabase dashboard URL or under **Project Settings > General**.
6. Copy the **Client ID** and **Client Secret**.

### B. Enable Google Provider in Supabase

1. Open your Supabase project dashboard at [supabase.com/dashboard](https://supabase.com/dashboard).
2. Go to **Authentication > Providers**.
3. Find **Google** in the list and enable it.
4. Paste the **Client ID** and **Client Secret** from the previous step.
5. The **Callback URL (Redirect URL)** shown on this page is the one you added to Google Cloud in step A.5 above. Confirm they match.
6. Click **Save**.

### C. Configure Redirect URLs in Supabase

1. In the Supabase dashboard, go to **Authentication > URL Configuration**.
2. Set the **Site URL** to your app's base URL:
   - Local development: `http://localhost:3000`
   - Production: `https://your-production-domain.com`
3. Add **Redirect URLs** (these are where Supabase is allowed to redirect after login):
   ```
   http://localhost:3000/auth/callback
   https://your-production-domain.com/auth/callback
   ```
   The app's callback handler at `/auth/callback` exchanges the OAuth code for a session and redirects the user to `/dashboard`.

### D. Environment Variables

No extra environment variables are needed beyond the standard Supabase ones already in `.env.template`:

```
NEXT_PUBLIC_SUPABASE_URL=https://<your-project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<your-supabase-anon-key>
SUPABASE_SERVICE_ROLE_KEY=<your-supabase-service-role-key>
```

Google OAuth credentials are stored in the Supabase dashboard, not in your app's environment.

### E. Verify the Setup

1. Start the app (`npm run dev` or `docker compose up -d --build`).
2. Open `http://localhost:3000/login`.
3. Click **Log in with Google**.
4. Complete the Google sign-in flow.
5. You should be redirected to `/dashboard`.

If something goes wrong, check:
- The Supabase callback URL in Google Cloud matches exactly (no trailing slash mismatch).
- The redirect URLs in Supabase **Authentication > URL Configuration** include your app's `/auth/callback` path.
- Your Google OAuth consent screen has the test user added (if still in "Testing" mode).
- Browser console and `docker compose logs -f app` for error details.
- The `/auth/auth-code-error` page will display if the OAuth code exchange fails.

### F. Local Development Note

On the login page (`src/app/login/page.tsx`), an email/password login form is shown alongside the Google button when running on `localhost` / `127.0.0.1`. This is for convenience during local development only. In production, only the Google login button is displayed.
