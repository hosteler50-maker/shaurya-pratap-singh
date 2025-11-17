# Copilot / AI Agent Instructions — Hostel Review Website

Short context
- This is a single-process Flask website whose primary data store is an Excel workbook: `data/hostels.xlsx` (managed by `openpyxl` in `app.py`).
- Sessions use filesystem storage via `Flask-Session` (session files live in `flask_session/`).
- Uploaded images are saved to `static/uploads/`.

Big picture (what matters to an agent)
- `app.py` is the canonical entry point and contains the full request handling and workbook-based persistence (functions: `load_hostels`, `load_reviews`, `add_review`, `create_user`, `save_hostel_image`, etc.).
- `models.py` contains SQLAlchemy models (User, Hostel, Review) and a migration helper `migrate_to_sqlite.py` exists to convert the workbook data to a DB — but this migration path requires adding/initializing Flask-SQLAlchemy (not wired in `app.py` by default).
- `templates/` and `static/` are standard Flask assets used by the UI; admin pages are implemented in templates named like `admin_*.html`.

Key developer workflows (how to run & verify)
- Run locally (dev): `python app.py` (debug True). The app bootstraps `data/hostels.xlsx` on first run via `ensure_data_file()`.
- Production run: `gunicorn app:app` (used in `DEPLOY_RENDER.md`). Set `SECRET_KEY` and `FLASK_ENV=production` in env.
- To migrate Excel -> SQLite (optional):
  - Add `Flask-SQLAlchemy` to `requirements.txt` and initialize `db` in `app.py` (configure `SQLALCHEMY_DATABASE_URI`).
  - Run: `python migrate_to_sqlite.py` (this script expects `app` and `db` to be importable and configured).
  - Verify with: `python check_db.py` (reads models via `app` app-context).

Project-specific conventions & pitfalls
- Primary datastore is a spreadsheet, not a DB: changing storage requires touching many functions in `app.py` (no thin repository layer).
- The `Reviews` sheet supports older (11-col) and normalized (17-col) formats — see `migrate_reviews_in_wb()` for exact mapping. Prefer preserving backward-compatibility when editing review parsing logic.
- Admin-authorized checks are simple: `is_admin()` compares `session['user_email']` to `ADMIN_EMAIL` (env var; default `hosteler50@gmail.com`). Modifying admin routes may require updating env or tests accordingly.
- Session storage is filesystem-based: tests or dev runs may leave many files in `flask_session/`. Clean that folder when needed.

Integration points & external dependencies
- openpyxl (XLSX storage): `data/hostels.xlsx` is read/written by `app.py` functions.
- Flask-Session (filesystem) persists session files under `flask_session/`.
- `keep-warm` workflow (`.github/workflows/keep-warm.yml`) pings `/health` — do not rename or remove that endpoint without updating the workflow.
- Deployment docs: `DEPLOY_RENDER.md` and `DEPLOY_PYTHONANYWHERE.md` describe platform-specific settings; mirror any changes to startup commands or requirements in those docs.

Practical editing guidance for an AI agent
- When changing data model / storage:
  - Do not remove workbook accessors (`load_hostels`, `load_reviews`) until a DB-backed path is fully implemented and tested.
  - If you add SQLAlchemy, update `requirements.txt`, add DB initialization to `app.py`, and ensure `migrate_to_sqlite.py` and `check_db.py` run in `app.app_context()`.
- When modifying routes that affect client UI, also update templates under `templates/` and static assets under `static/`.
- When changing secrets or env names, update `DEPLOY_RENDER.md` and `DEPLOY_PYTHONANYWHERE.md` accordingly.
- Avoid breaking the admin endpoints: `admin_migrate_reviews`, `admin_backup_workbook`, `admin_backups`, and `admin_backups_restore` perform file operations on `data/` — ensure proper path handling and permissions.

Files and locations an agent will frequently edit or inspect
- `app.py` (entry point, Excel-backed persistence, routes)
- `models.py` (SQLAlchemy models — optional path)
- `migrate_to_sqlite.py` (migration script; requires DB wiring)
- `check_db.py` (simple DB sanity script)
- `requirements.txt` (dependency list)
- `data/hostels.xlsx` (primary dataset) and `data/` (backups)
- `static/uploads/` (images)
- `.github/workflows/keep-warm.yml` (CI ping to `/health`)
- `DEPLOY_RENDER.md`, `DEPLOY_PYTHONANYWHERE.md` (deploy/run notes)

When to ask for human help
- If you plan to swap the storage layer (spreadsheet -> DB) ask the maintainer for migration policy and downtime windows.
- If you need missing secrets (production `SECRET_KEY`, `ADMIN_EMAIL`) or credentials for cloud storage, request them instead of hardcoding.

If anything is unclear or you'd like this trimmed/expanded into smaller actionable tasks (e.g., implement SQLAlchemy wiring, add tests, or update deploy docs), tell me which task to do next.
