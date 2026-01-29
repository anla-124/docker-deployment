# Docker Deploy Guide (A–Z)

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
