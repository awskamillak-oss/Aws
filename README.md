# Automation Platform

Web-based industrial automation platform (PLC function-block editor, HMI designer,
tag database, Modbus TCP / BACnet IP & MS/TP). Target: Raspberry Pi 5.

## Architecture

```
 Browser (React)                      ┌─────────────── Raspberry Pi ────────────────┐
 ┌────────────────┐  HTTP / (WS later)│ ┌──────────────┐   IPC: JSON lines over     │
 │ frontend/      │ ─────────────────▶│ │ backend/     │   127.0.0.1:7001           │
 │ PLC editor     │                   │ │ FastAPI :8000│ ─────────────┐             │
 │ HMI designer   │                   │ └──────┬───────┘              ▼             │
 │ Tag DB, Comms  │                   │        │ writes config  ┌──────────────┐    │
 └────────────────┘                   │        ▼                │ engine/      │    │
                                      │ ┌──────────────┐ reads  │ scan cycle   │──▶ Modbus TCP
                                      │ │ SQLite (WAL) │◀───────│ tag table    │──▶ BACnet IP/MSTP
                                      │ │ data/*.db    │ config │ drivers      │    │
                                      │ └──────────────┘        └──────────────┘    │
                                      └─────────────────────────────────────────────┘
```

**Key rule: the engine is independent.** It is a separate OS process (separate
systemd service on the Pi). If the backend or browser crashes, restarts, or is
upgraded, the engine keeps scanning and controlling I/O. The backend only talks
to it through the localhost IPC socket.

| Folder | Role |
|---|---|
| `engine/` | Real-time control process. `runtime.py` = cyclic scan (inputs → logic → outputs), `tag_table.py` = live values in memory, `ipc_server.py` = localhost command API, `drivers/` = protocol drivers (later), `logic/` = function-block execution (later). |
| `backend/` | FastAPI web server. REST API for the UI, stores configuration in SQLite, proxies runtime commands/values to the engine via `engine_client.py`. In production also serves the built React app. |
| `frontend/` | React + TypeScript (Vite). One folder per module in `src/features/`. Dev server proxies `/api` to the backend. |
| `shared/` | Code both Python processes use: `config.py` (env-overridable settings), `ipc.py` (wire protocol), `db.py` (SQLite connection + schema). |
| `tests/` | pytest tests, written alongside each component. |
| `data/` | Runtime data (SQLite DB). Created automatically, git-ignored. |

**Data ownership:** configuration (tags, programs, screens, devices) lives in
SQLite and is written by the backend. Live values live only in engine memory
and are read over IPC — not written to SQLite every scan (protects the Pi's SD card).

## Run locally (Windows + VS Code)

Prerequisites: Python 3.11+, Node.js 20+ LTS, VS Code.

1. Open this folder in VS Code (*File → Open Folder*). Install the recommended extensions when prompted.
2. *Terminal → Run Task → "Setup: Python venv + deps"*
3. *Terminal → Run Task → "Setup: npm install"*
4. *Ctrl+Shift+P → "Python: Select Interpreter"* → pick `.venv`.
5. Run and Debug panel (Ctrl+Shift+D) → choose **Full Stack** → F5.
6. Open http://localhost:5173

Or manually, in three terminals:

```powershell
.\.venv\Scripts\python -m engine                          # terminal 1
.\.venv\Scripts\python -m uvicorn backend.main:app --port 8000   # terminal 2
cd frontend; npm run dev                                   # terminal 3
```

Tests: *Run Task → "Test: pytest"* or `.\.venv\Scripts\python -m pytest -v`.
API docs (auto-generated): http://127.0.0.1:8000/docs

**Try the independence:** stop the Backend while the Engine runs, then start it
again — the engine's cycle count keeps climbing the whole time.

## Roadmap (one component at a time)

1. ✅ Skeleton: engine process, IPC, backend, UI shell, SQLite
2. Tag database (SQLite schema, CRUD API, engine loads tags, UI table)
3. Live data: WebSocket from backend, engine subscriptions
4. Modbus TCP driver
5. Function-block runtime + PLC FBD editor
6. HMI designer + runtime viewer
7. BACnet/IP, then BACnet MS/TP (RS-485 HAT on the Pi)
8. Raspberry Pi deployment: systemd units, auth, backups
