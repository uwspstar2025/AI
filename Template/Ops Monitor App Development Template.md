# Ops Monitor App Development Template

Use this document as a blueprint for creating a standalone local service monitor app.

The goal is an operator-friendly dashboard that can inspect, start, stop, restart, and recover a local multi-service stack from desktop or mobile.

## 1. Purpose

Build a standalone monitor app that stays available even when the main frontend, backend, worker, or model service is down.

The monitor should answer:

- Which services are online?
- Why did a service fail to start?
- Can I start, stop, restart, or recover services from one UI?
- Can I operate it safely from a phone?

## 2. Recommended Stack

- Next.js app, separate from the main product frontend.
- Auth.js / NextAuth for Google sign-in.
- Server routes for local service probes and shell actions.
- Local run configuration for project paths, commands, probes, ports, and log paths.
- ADRs in `doc/adr` for monitor decisions.

## 3. Monitored Services

Example service set:

| Service | Runtime | Default Probe | Default Port |
|---------|---------|---------------|--------------|
| Frontend | Next.js | `GET http://127.0.0.1:3000` | `3000` |
| Backend | FastAPI | `GET http://127.0.0.1:8000/api/v1/health` | `8000` |
| Worker | Background Python process | `pgrep -fl "market_cache_worker"` | none |
| Ollama | Local model server | `GET http://127.0.0.1:11434/api/tags` | `11434` |

HTTP services should use HTTP probes first and port-listen checks as fallback. Portless workers should use process probes.

## 4. Roles And Access

Required behavior:

- Users must sign in with Google.
- Viewer users can see status, logs, and insights.
- Operator users can start, stop, restart, Start all, Stop all, Restart all, and use Auto recover.
- Operator emails are configured with `MONITOR_OPERATOR_EMAILS`.

Suggested env:

```bash
AUTH_URL=http://localhost:3010
AUTH_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
MONITOR_ALLOWED_EMAILS=
MONITOR_GMAIL_ONLY=1
MONITOR_OPERATOR_EMAILS=your.email@gmail.com
```

## 5. Project Path Configuration

Use path defaults, but allow overrides from both `.env.local` and a local settings file.

```bash
MONITOR_WORKSPACE_ROOT=/Users/xing/Desktop/AI_Projects
MONITOR_FRONTEND_DIR=/Users/xing/Desktop/AI_Projects/stock-ai-front-end
MONITOR_BACKEND_DIR=/Users/xing/Desktop/AI_Projects/stock-ai-back-end
MONITOR_WORKER_DIR=/Users/xing/Desktop/AI_Projects/stock-ai-worker
```

Avoid hard-coding one historical project layout. Real local paths move.

For operator-friendly setup, expose these values in a Settings / Run Config panel and persist edits to `.monitor-config.local.json`.

## 6. Command Model

Use default commands plus env overrides, then allow the monitor UI to save local edits.

Default start command examples:

```bash
# Frontend
cd "$MONITOR_FRONTEND_DIR" && npm run dev

# Backend
cd "$MONITOR_BACKEND_DIR" && "$MONITOR_BACKEND_DIR/.venv/bin/python" -m uvicorn app:app --host 127.0.0.1 --port 8000

# Worker
cd "$MONITOR_WORKER_DIR" && PYTHONPATH="$MONITOR_BACKEND_DIR" "$MONITOR_BACKEND_DIR/.venv/bin/python" -m market_cache_worker

# Ollama
ollama serve
```

Default stop command examples:

```bash
lsof -tiTCP:3000 -sTCP:LISTEN | xargs kill
lsof -tiTCP:8000 -sTCP:LISTEN | xargs kill
pkill -f "market_cache_worker"
lsof -tiTCP:11434 -sTCP:LISTEN | xargs kill
```

Override env:

```bash
MONITOR_CMD_START_FRONTEND=
MONITOR_CMD_STOP_FRONTEND=
MONITOR_CMD_START_BACKEND=
MONITOR_CMD_STOP_BACKEND=
MONITOR_CMD_START_WORKER=
MONITOR_CMD_STOP_WORKER=
MONITOR_CMD_START_OLLAMA=
MONITOR_CMD_STOP_OLLAMA=
MONITOR_LOGIN_SHELL=true
```

Local UI config file:

```text
.monitor-config.local.json
```

This file should be ignored by git. The monitor should boot without it, merge it over defaults when present, and use it as the source of truth for service actions, probes, and log reads.

Important implementation notes:

- Run start commands in the background.
- Write service logs to `/tmp/monitor-<service>.log`.
- Do not rely on `source .venv/bin/activate && python`; prefer the explicit venv interpreter path.
- Force frontend `PORT=3000` so it does not inherit the monitor app’s `PORT=3010`.

## 7. Settings / Run Config UX

The monitor should include a Settings panel that lets operators change each app's required run configuration without editing code.

Editable global settings:

- Workspace root.
- Login shell mode.

Editable service settings:

- Project directory.
- Start command.
- Stop command.
- Probe type: HTTP or process.
- Health URL for HTTP probes.
- Port for listen checks.
- Process pattern for process probes.
- Log path.

Behavior:

- Viewer users can inspect settings but cannot save.
- Operator users can save settings.
- Saving writes `.monitor-config.local.json`.
- After saving, refresh statuses so the UI reflects the new probes.
- Keep the panel usable on mobile with stacked fields and full-width actions.

## 8. Preflight Checks

Before starting a service, run preflight checks.

Frontend checks:

- Project directory exists.
- `package.json` exists.
- `node_modules` exists.
- `npm` is available on PATH.

Backend checks:

- Project directory exists.
- `app.py` exists.
- `.venv/bin/python` exists.
- `.venv/bin/python -m uvicorn --version` succeeds.

Worker checks:

- Worker directory exists.
- Worker entrypoint exists.
- Backend `.venv/bin/python` exists.

Ollama checks:

- `ollama` is available on PATH.

If preflight fails:

- Return a non-success API response.
- Show a visible UI alert.
- Include a direct hint, such as recreating `.venv` or setting `MONITOR_BACKEND_DIR`.

## 9. Runtime Diagnostics

Starting a background command does not mean the service is healthy.

After a Start or Restart action:

- Wait briefly.
- Re-run the service probe.
- If the service is still down, return failure.
- Include probe details and the tail of `/tmp/monitor-<service>.log`.

The UI should show:

- Action failure alert.
- Preflight/runtime diagnostics.
- Raw output/log tail.
- A likely issue summary when possible.

## 10. Operator Controls

Per-service controls:

- Start
- Stop
- Restart
- View detail
- View logs
- Open service URL
- Auto recover

Stack-level controls:

- Refresh
- Start all
- Stop all
- Restart all

Recommended safety:

- Stop all asks for confirmation.
- Restart all asks for confirmation.
- Start all does not ask for confirmation because it only starts stopped services.

## 11. Bulk Action Ordering

Bulk actions should respect dependencies.

Start order:

```text
Ollama -> Backend -> Worker -> Frontend
```

Stop order:

```text
Frontend -> Worker -> Backend -> Ollama
```

Restart all:

- Confirm first.
- Restart sequentially.
- Keep going if one service fails.
- Show partial success in output.

Sequential bulk actions are slower than parallel actions, but easier to debug and less likely to collide on ports.

## 12. Auto Recover

Auto recover should be explicit and bounded.

Expected behavior:

- When enabled for a stopped service, immediately try to start it.
- Retry up to 3 times.
- Wait 5 seconds between attempts.
- Check status before each retry to avoid double-starting slow services.
- Keep checking enabled services every 60 seconds while the page is open.
- Use a cooldown between retry groups.

The visible label should be `Auto recover`, not just `Auto`.

## 13. Mobile UX

Mobile is a primary operations surface.

Required mobile behavior:

- Use service cards as the primary layout.
- Hide table mode on small screens.
- Use large tap targets.
- Show action labels under icons.
- Keep Start all / Stop all / Restart all easy to reach.
- Keep diagnostics above service cards.
- Avoid horizontal scrolling.

Recommended card states:

- Running service: green border and green left accent.
- Stopped service: red border and red left accent.
- Auto recover selected: visible selected state.

## 14. Diagnostics UX

Do not make the operator read raw logs first.

Show:

- Short error banner.
- Preflight/runtime checklist.
- Likely issue summary.
- Raw output below with a Clear button.
- Recent action history.
- Full log modal with latest likely issue extracted above raw logs.

Action history can be client-side and ephemeral for the local MVP.

Example entries:

```text
10:32 Backend start failed
10:34 Start all requested (3)
10:35 Frontend start succeeded
```

## 15. Ops Insights

Useful optional system insights:

- Port status for `3000`, `3010`, `8000`, `11434`.
- CPU load.
- Memory.
- Disk.
- GPU/thermal notes on macOS.
- Ollama loaded models via `/api/ps`.
- Rolling peak snapshot.

Keep insights collapsible so they do not crowd the main operator controls.

## 16. Logs

Each service should have a log viewer.

Implementation:

- Read the configured service log path, defaulting to `/tmp/monitor-<service>.log`.
- Tail a bounded number of lines.
- Show the log path on desktop.
- Extract likely error lines using keywords such as `error`, `failed`, `exception`, `enoent`, `eaddrinuse`, `permission`, `not found`, and `missing`.

## 17. Local Dev Server

Monitor app:

```bash
npm run dev
```

Recommended package script:

```json
{
  "dev": "bash scripts/free-monitor-port.sh && next dev --port 3010 --webpack"
}
```

Port helper behavior:

- Kill stale listeners on `3010`.
- Clear `.next/dev` so a killed Next server does not leave a corrupt dev cache.

## 18. Suggested File Structure

```text
stock-ai-monitor/
├── app/
│   ├── api/system-monitor/route.ts
│   ├── api/system-monitor/config/route.ts
│   ├── api/system-monitor/logs/route.ts
│   ├── api/system-monitor/insights/route.ts
│   ├── api/auth/[...nextauth]/route.ts
│   ├── login/page.tsx
│   ├── page.tsx
│   └── globals.css
├── components/
│   ├── monitor-client.tsx
│   └── monitor-icons.tsx
├── lib/
│   ├── service-links.ts
│   ├── monitor-prefs.ts
│   ├── run-config.ts
│   ├── insights-snapshot.ts
│   └── oauth-credentials.ts
├── scripts/
│   └── free-monitor-port.sh
├── doc/
│   ├── monitor-app-dev-template.md
│   └── adr/
├── auth.ts
├── middleware.ts
├── package.json
└── README.md
```

## 19. ADRs To Create

At minimum:

- Use ADRs for monitor decisions.
- Standalone monitor app.
- Local service control with command overrides.
- Preflight and runtime diagnostics.
- Mobile-first operator dashboard.
- Auto recover retry behavior.
- Start all.
- Stop all.
- Operator usability refinements.
- Local run config settings UI.

## 20. Acceptance Checklist

Before calling the monitor app ready:

- Google sign-in works.
- Viewer cannot run service actions.
- Operator can Start/Stop/Restart each service.
- Start all starts only stopped services.
- Stop all stops only running services and confirms first.
- Restart all confirms first.
- Auto recover starts immediately and retries up to 3 times.
- Preflight catches missing paths/dependencies.
- Runtime failure is visible when a service does not become healthy.
- Operator can edit service run configuration from Settings.
- `.monitor-config.local.json` is ignored by git.
- Logs are available per service.
- Mobile cards fit without horizontal scrolling.
- Running/offline states are clear at card level.
- `npx tsc --noEmit` passes.

## 21. Common Failure Modes

### Broken Python virtualenv

Symptom:

```text
zsh:1: command not found: python
```

or:

```text
no such file or directory: .venv/bin/python
```

Fix:

```bash
cd stock-ai-back-end
python3 -m venv --clear .venv
.venv/bin/python -m pip install -r requirements.txt
```

Then use `.venv/bin/python` explicitly in monitor commands.

### Port bind denied in sandbox

Symptom:

```text
Error: listen EPERM: operation not permitted 0.0.0.0:3000
```

Fix:

Run the service outside the restricted sandbox or through the approved monitor command path.

### Stale log tail

Symptom:

The log shows old successful requests, but the service is currently off.

Fix:

Trust the status probe first. Clear/restart the service log if needed.

### Custom command overrides

If a custom start command is set in `.env.local` or Settings, built-in path preflight cannot fully validate it. Show a warning and rely on log output.
