# UI Navigator — Claude Code Guide

## Project Overview

AI agent that controls a browser by: taking screenshots → sending to Gemini 2.5 Flash → parsing an action plan → executing via Playwright. Exposed as a FastAPI REST + WebSocket service, containerized, deployed to Cloud Run. Also includes a Chrome Extension (WebPilot) for real-tab browser control via a sidebar UI.

## Current Versions
- **Backend**: `1.4.0` (`src/api/server.py` → `_VERSION`)
- **Extension**: `1.1.0` (`webpilot-extension/manifest.json` + `sidebar/package.json`)
- **Rule**: bump both version numbers on every build/release

## Key Commands

```bash
# Install dependencies
pip install -r requirements.txt
playwright install chromium

# Configure environment
cp .env.example .env
# Edit .env — set GOOGLE_API_KEY to a real Gemini key

# Run tests (58 total — all passing)
python -m pytest tests/ -v

# Start dev server (bash — loads .env automatically)
cd /path/to/UI_Navigator
export $(grep -v '^#' .env | xargs) && python -m uvicorn src.api.server:app --host 0.0.0.0 --port 8080

# Build WebPilot extension (must rebuild after any sidebar change)
cd webpilot-extension/sidebar && npm install && npm run build

# Docker (local)
docker-compose up --build

# Deploy to Cloud Run
export GOOGLE_CLOUD_PROJECT=<project-id>
export GOOGLE_API_KEY=<key>
chmod +x deploy.sh && ./deploy.sh
```

## Architecture

```
src/
  agent/
    core.py               # UINavigatorAgent — main screenshot→plan→execute loop
    vision.py             # GeminiVisionClient — calls gemini-2.5-flash (emits gemini_latency_ms)
    planner.py            # ActionPlanner — parses Gemini JSON → ActionPlan (Pydantic), with retry
    webpilot_handler.py   # WebPilotHandler — single-action vision+plan for extension WS loop
    adk_agent.py          # ADK Agent + Runner + InMemorySessionService
  executor/
    actions.py            # ActionType enum, Action/ActionResult models
    browser.py            # PlaywrightBrowserExecutor — headless Chromium automation
  api/
    server.py             # FastAPI REST + WebSocket server (v1.4.0)
    models.py             # Shared Pydantic models (TaskRecord, NavigateRequest, TaskListResponse, …)
    store.py              # Abstract TaskStore + create_store() factory
    store_memory.py       # MemoryTaskStore (default)
    store_firestore.py    # FirestoreTaskStore (TASK_STORE=firestore)
    session_routes.py     # ADK session endpoints
    webpilot_routes.py    # WebPilot WS endpoint + session REST + POST /webpilot/tts
    webpilot_models.py    # WebPilotAction, WebPilotSession, InterruptionType, TTSRequest, message models
  metrics.py              # Cloud Monitoring fire-and-forget emission
  tracing.py              # OTel + Cloud Trace context manager
  storage.py              # GCS screenshot upload → 7-day signed URLs
  logging_config.py       # JSON structured logging
tests/
  test_agent.py           # 16 tests
  test_api.py             # 18 tests
  test_webpilot_api.py    # 7 tests
  test_sessions.py        # 10 tests
  test_clarifier.py       # 7 tests
  load/
    locustfile.py         # Locust load test scenarios
monitoring/
  setup_alerts.sh         # gcloud alert policy creation (3 policies)
```

## API Endpoints

- `POST /navigate` — start a task, returns `task_id`
- `GET /tasks` — list all tasks
- `GET /tasks/{task_id}` — poll status/result
- `DELETE /tasks/{task_id}` — cancel a running task
- `WS /ws/{task_id}` — stream step events in real time
- `POST /screenshot` — one-shot screenshot + Gemini analysis
- `POST /clarify` — get clarifying questions for an ambiguous task
- `GET /health` — health check
- `POST /sessions` — create ADK Chrome Extension session
- `POST /sessions/{id}/step` — send screenshot → get ActionPlan (ADK)
- `POST /sessions/{id}/events` — log client-side telemetry
- `DELETE /sessions/{id}` — end ADK session
- `POST /webpilot/sessions` — create WebPilot session
- `DELETE /webpilot/sessions/{id}` — end WebPilot session
- `WS /webpilot/ws/{session_id}` — WebPilot real-time action loop
- `POST /webpilot/tts` — Gemini TTS narration → base64 WAV audio

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `GOOGLE_API_KEY` | Yes | Gemini API key |
| `GOOGLE_CLOUD_PROJECT` | Deploy only | GCP project ID |
| `GOOGLE_CLOUD_REGION` | Deploy only | Cloud Run region (default: us-central1) |
| `BROWSER_HEADLESS` | No | Run Chromium headless (default: true) |
| `MAX_CONCURRENT_TASKS` | No | Semaphore limit (default: 5) |
| `BROWSER_WIDTH/HEIGHT` | No | Viewport size (default: 1280x800) |
| `TASK_STORE` | No | `memory` (default) or `firestore` |
| `GCS_BUCKET` | No | GCS bucket for screenshot uploads |

## Testing Notes

- `asyncio_mode = "auto"` is set in `pyproject.toml`
- 58 tests total: 16 agent + 18 API + 7 webpilot + 10 sessions + 7 clarifier (all passing)
- Run non-browser tests only: `python -m pytest tests/test_api.py tests/test_webpilot_api.py tests/test_sessions.py tests/test_clarifier.py -v`
- Integration tests (test_agent.py) spin up a real Chromium browser — slow, run separately

### SDK Image Format (google-genai >= 0.8)
- OLD (broken): `{"mime_type": "image/png", "data": bytes}` dict
- NEW (correct): `types.Part.from_bytes(data=bytes, mime_type="image/png")`

### server.py lifespan
- WebPilot handler init is guarded: `if os.environ.get("GOOGLE_API_KEY")` — safe to run tests without key

### webpilot_routes.py
- Confirmation flow: `_run_action_loop` reads confirm message DIRECTLY from WS (not via asyncio.Event) — outer loop is blocked inside the function and cannot process messages
- Pre-accept close: always `accept()` before `close()` so TestClient doesn't raise WebSocketDisconnect
- Interruption dispatch: classify → ABORT short-circuits (no Gemini), REDIRECT clears history, REFINEMENT merges intent
- Auto-retry: tracks MD5 hash of consecutive screenshots; `stuck=True` passed to handler after 3 identical frames

### webpilot_handler.py
- `thinking_budget=0` set on ALL Gemini calls — disables 2.5 Flash reasoning mode for fast step decisions
- `classify_interruption_type()`: checks REDIRECT keywords BEFORE ABORT so "Actually cancel" → REDIRECT (keeps test passing)
- `get_narration_audio()`: uses `gemini-2.5-flash-preview-tts` with Aoede voice

### App.jsx narration rules
- Narration fires ONLY when a new entry is added to actionLog (tracked via `prevLogLenRef`) — NOT on STEP_RESULT updates (avoids double-speak)
- Confirmation narration: `useEffect` on `status === "confirming"` speaks `pendingAction.narration`
- Stop narration: `prevStatusRef` detects `running/thinking → idle` transition
- `handleConfirm`: only dispatches `STOPPED` on denial — server drives state on confirmation

## Deployment Notes

- Secret Manager secret named `GOOGLE_API_KEY` must exist in GCP before Cloud Run deploy
- Cloud Run config: 2 vCPU, 2 GiB RAM, min 0 / max 5 instances, 300s timeout
- Monitoring: `monitoring/setup_alerts.sh` creates 3 alert policies in Cloud Monitoring

## WebPilot Chrome Extension (Phase 7)

```
webpilot-extension/
  manifest.json        # MV3 v1.1.0, sidePanel + activeTab + tabs + scripting + storage + alarms
  background.js        # SW: WS owner, screenshot capture, action execution, session retry
  content.js           # DOM executor: click (shadow DOM), type (React-compatible), scroll, key
  icons/               # 16/48/128px
  sidebar/
    package.json / vite.config.js   # React 18 + Vite (base: "./")
    index.html / main.jsx / App.jsx
    components/
      TaskInput.jsx       # Textarea + mic button (hold-to-speak)
      ActionLog.jsx       # Step log with ✓/✗ per action
      ConfirmCard.jsx     # Proceed/Cancel for irreversible actions
      StatusIndicator.jsx # Colored dot: green/amber/red/grey
    hooks/
      useVoiceInput.js    # SpeechRecognition, 3 restart attempts, final results only
      useVoiceOutput.js   # speechSynthesis, soft female voice (Google UK English Female preferred)
      useWebSocket.js     # chrome.runtime bridge to background.js
```

Build: `cd webpilot-extension/sidebar && npm install && npm run build`
Load: Chrome → `chrome://extensions` → Load unpacked → select `webpilot-extension/`

### Chrome Setup Required
- **Site access**: `chrome://extensions` → WebPilot → Details → Site access → **On all sites**
- **Mic access**: `chrome://settings/content/microphone` → allow the extension origin
- Keyboard shortcut: `Ctrl+Shift+A` (Mac: `Command+Shift+A`) opens sidebar

### Key Architecture Notes
- WS is owned by **background.js** (service worker), not the sidebar
- Sidebar communicates via `chrome.runtime.sendMessage` / `chrome.runtime.onMessage`
- Session created on install/startup, stored in `chrome.storage.session`, retried with backoff if backend down
- Keep-alive alarm every 25s prevents service worker from going dormant mid-task
- Auto-stop: **15 max steps** or **3 consecutive failures** → sends stop to server

### Key Fixes
- `navigate` → `chrome.tabs.update` (content scripts blocked on new pages)
- After navigate: `waitForTabLoad()` waits for tab `status === "complete"` + 1.5s settle (15s timeout)
- Content script re-injected via `chrome.scripting.executeScript` before each non-navigate action (handles post-navigation loss)
- `tabCapture` permission removed — it's restricted and blocks extension loading; use `captureVisibleTab` with `tabs` permission instead
- `vite.config.js` must have `base: "./"` — Chrome extensions require relative asset paths
- Voice mic: SpeechRecognition tried directly; `not-allowed` shown as actionable error in UI

## Completed Phases

| Phase | Description |
|---|---|
| Baseline MVP | Core agent loop, FastAPI, Docker |
| 1 — Hardening | Structured logging, HTTP middleware, VisionUnavailableError |
| 2 — Security & Auth | APIKeyMiddleware, RateLimitMiddleware, input validation, SSRF protection |
| 3 — Persistence | TaskStore abstraction, `GET /tasks`, hourly cleanup, Firestore backend |
| 4 — Observability | Cloud Monitoring metrics, OTel/Cloud Trace, alert policies |
| 5 — Scale & Polish | GCS screenshot upload, Locust load tests, deploy.sh enhancements |
| 6 — ADK Extension | MV3 sidepanel, ADK sessions, voice input, real tab control |
| 7 — WebPilot Extension | WS-driven single-action loop, confirmation flow, interrupt, voice narration |
| 7.1 — PRD Gap Fixes | InterruptionType classify, thinking_budget=0, narration sync, auto-retry, TTS endpoint |
