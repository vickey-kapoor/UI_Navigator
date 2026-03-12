---
name: smoke-test
description: Run a full UI Navigator smoke test — pytest suite + health check + optional WebPilot WS validation. Call this before committing or after significant changes.
---

Run the following checks in order and report a final pass/fail table.

## Check 1 — Import sanity
```bash
cd C:/Users/vicke/OneDrive/Documents/GitHub/UI_Navigator
python -c "from src.api.server import app; from src.agent.webpilot_handler import WebPilotHandler; from src.api.webpilot_models import WebPilotAction; print('imports OK')"
```

## Check 2 — Non-browser test suite
```bash
cd C:/Users/vicke/OneDrive/Documents/GitHub/UI_Navigator
python -m pytest tests/test_api.py tests/test_webpilot_api.py tests/test_sessions.py tests/test_clarifier.py -q --tb=short
```
Expected: all 42 tests passing.

## Check 3 — Server health (if running)
```bash
curl -sf http://localhost:8080/health
```
If not running, note "server offline — skipping live checks" and continue.

## Check 4 — WebPilot WS smoke test (if server is running)
Use the `webpilot-smoke-tester` subagent to run the live WS test.

## Output

Print a summary table:

```
UI Navigator Smoke Test — <date>
═══════════════════════════════════
 Check                  │ Status │ Notes
────────────────────────┼────────┼────────────────────────
 Import sanity          │  PASS  │
 Unit tests (42)        │  PASS  │ 42 passed in Xs
 Server health          │  PASS  │ version X.X.X
 WebPilot WS            │  PASS  │ action=navigate, XXms
═══════════════════════════════════
 Overall                │  PASS  │
```

If any check fails, list the specific error and suggest which file to look at.
