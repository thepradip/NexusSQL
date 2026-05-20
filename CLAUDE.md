# NexusSQL — Agent Instructions

## Git Identity — always use this, no exceptions

Every commit in this repo must be authored as:

```
name:  thepradip
email: pradiptivhale@gmail.com
```

Always set with `-c` flags — never touch global git config:

```bash
git -c user.name="thepradip" -c user.email="pradiptivhale@gmail.com" commit -m "..."
```

**No Co-Authored-By line. No other name or email. Ever.**

Use `/gpush` to commit and push — it enforces this automatically.

## Push rules (enforced by /gpush)

1. **Credential scan first** — grep all changed files for API keys, passwords, tokens. Stop if found.
2. **Tests must pass** — run `pytest` (backend) and health checks before committing. Fail = no push.
3. **No `.env` files** — never stage `.env`, only `.env.example` is allowed in the repo.
4. **Stage selectively** — never `git add .`; only add files relevant to the change.

## Project layout

```
backend/                    FastAPI SQL AI agent
frontend/                   React UI (Vite + Tailwind)
services/
  visualization_service/    Chart inference microservice (port 8011)
ariasql-package/            Pip-installable package (ariasql)
sqlas/                      SQLAS evaluation library
```

## Running locally

```bash
bash start.sh               # starts all three services
docker compose up --build   # Docker alternative
```

Service ports:

| Service | Port |
|---|---|
| visualization | 8011 |
| backend | 8000 |
| frontend (dev) | 5173 |
| frontend (Docker) | 80 |

## Key architecture notes

- `backend/tracing.py` calls the visualization service via HTTP (`visualization_client.py`)
- `VISUALIZATION_SERVICE_URL` defaults to `http://127.0.0.1:8011` locally; docker-compose overrides it to `http://visualization:8011`
- If the viz service is down, SQL answers still return — just no chart
- ReAct/agentic mode is toggled with the CPU icon in the UI or `force_agentic=True` in the API
