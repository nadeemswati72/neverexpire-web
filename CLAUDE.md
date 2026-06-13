# NeverExpire — Project Guide for Claude Code

NeverExpire is a Flask + SQLAlchemy (SQLite) web app that helps a family track
documents that expire (passports, insurance, warranties, subscriptions,
medication, certificates, etc.) and get reminders before they do. Documents
can be added manually or by uploading a file, which is run through the
Anthropic API to extract title/dates/fields automatically.

## Tech stack

- **Backend:** Flask (blueprints), SQLAlchemy ORM, SQLite (`data/neverexpire.db`)
- **Templates:** Jinja2 + Bootstrap 5 (CDN), custom navy/gold theme in `neverexpire/web/static/style.css`
- **AI extraction:** Anthropic API (`neverexpire/extractor.py`, model = `claude-haiku-4-5`)
- **Images:** Pillow — image validation for uploads/avatars, diagonal watermarking
- **Secondary UI:** Streamlit mockup under `ui/` (mock data only, not wired to the Flask DB)
- **Env:** Python venv at `venv/`, config via `.env` (see `.env.example`)

## Directory map

```
neverexpire/
  config.py              # paths, secrets, REMINDER_DAYS_THRESHOLD, EXTRACTION_MODEL
  extractor.py            # Anthropic-based expiry/field extraction
  watermark.py            # diagonal "For NeverExpire Reminders Only" watermark
  pipeline.py / security.py
  db/
    base.py, session.py   # SQLAlchemy Base + SessionLocal
    models/                # User, Person, Document(+File/ExtractionRun), DocumentType
                           # (+FieldDefinition/Translation), FamilyRelationType,
                           # PersonRelationship, DocumentReminder, ReminderRule, AuditLog
    queries.py             # read-only helper queries (dashboard summary, filtered lists)
    repository.py          # document create/delete + helpers (parse_date, etc.)
    accounts.py            # user/person registration, family members, photo upload
    reminders.py           # generate_due_reminders
    init_db.py, seed.py, seed_demo.py, seed_extras.py, migrate_v2.py, migrate_v3.py
    admin_data.py, create_admin.py
  web/
    __init__.py            # create_app(): Flask app factory, before/teardown DB session,
                           # injects current_user/current_person into templates
    auth.py                 # current_user(), login_required decorator, session-based auth
    __main__.py             # `python -m neverexpire.web` entrypoint
    routes/
      auth.py, dashboard.py, documents.py, family.py, admin.py, pages.py
    templates/
      base.html, dashboard.html
      auth/ (login, register)
      documents/ (list, detail, add, review)
      family/ (list, add, detail)
      admin/ (dashboard, document_types, document_type_fields, family_relation_types)
      pages/ (about, roadmap)
    static/
      style.css            # navy/gold theme (all custom CSS lives here)
      avatars/             # uploaded profile photos (Person/User.photo_path)
data/
  neverexpire.db           # SQLite database
  uploads/<person_id>/     # uploaded document files + watermarked copies
  uploads/staging/         # temp files during AI extraction, before confirm
ui/                         # Streamlit mockup (mock_data.py, components/, pages/dashboard.py)
samples/                    # sample scan images for manual testing
```

## Running it locally

1. Activate the venv tools directly (no need to "activate" — call the venv's python/pip):
   `.\venv\Scripts\python.exe`, `.\venv\Scripts\pip.exe`
2. `.env` must have `ANTHROPIC_API_KEY` set (see `.env.example`); `config.py` raises at
   import time if it's missing.
3. First-time DB setup: `python -m neverexpire.db.init_db` then `python -m neverexpire.db.seed`
   (optionally `seed_demo` / `seed_extras` for demo data).
4. **Dev server (no auto-reload)** — see `.claude/skills/dev-server/SKILL.md` for the
   PID-file pattern used to start/stop/restart it. **You must restart the server after
   any code change** — `flask run` here is started without the reloader.

## Demo accounts

- `alice@neverexpire.test` / `Demo@1234` — regular user, "Alice Johnson" family
  (spouse Mark Johnson, children, etc.) with a varied mix of documents
  (expired/expiring/valid/no-expiry) across document types.
- `demoadmin@neverexpire.test` / `Demo@1234` — admin role (see `web/routes/admin.py`,
  `db/admin_data.py`).

## Theme & styling conventions

All custom CSS lives in `neverexpire/web/static/style.css`, built on top of Bootstrap 5.
CSS variables (`:root`):

```css
--navy:    #1A365D   /* navbar, headings, links, primary text accents */
--gold:    #D69E2E   /* primary buttons, active tab underline */
--danger:  #E53E3E   /* expired status */
--warning: #ED8936   /* expiring-soon status */
--success: #38A169   /* valid / all-good status */
--bg:      #F7FAFC
--text-secondary: #718096  /* "no expiry" / muted status */
```

Status colors are consistent everywhere (dashboard pills, document rows, status
badges, status dots): `expired` → danger, `expiring_soon` → warning, `valid` →
success, `no_expiry` → secondary/grey. The expiry "soon" cutoff is
`today + config.REMINDER_DAYS_THRESHOLD` (currently 30 days).

When asked for a visual redesign, match this existing palette and component
vocabulary (`.doc-row`, `.doc-card`, `.status-badge-*`, `.filter-tab`,
`.health-pill`, `.family-card`, `.avatar-*`) rather than introducing a new style
system — and verify changes with the visual-verify skill before reporting done.

## Key data-model notes

- `Document.extra_fields` is a JSON column; per-type custom fields are defined in
  `DocumentType.field_definitions` (`DocumentFieldDefinition`, with
  `display_order`, `data_type`, `applies_to_extraction`, `is_active`).
- `Document` cascades (`all, delete-orphan`) to `DocumentFile`,
  `DocumentExtractionRun`, and `DocumentReminder` — deleting a `Document` via
  `repository.delete_document` also removes its files from disk.
- `Person.photo_path` / `User.photo_path` store paths relative to `static/`
  (e.g. `avatars/person_8.svg`), set via `db/accounts.py:set_person_photo`
  (validates extension + Pillow `Image.verify()`, deletes the old file on replace).
- `DocumentFile.watermarked_file_path`, when present, is preferred by
  `documents.serve_file` over the original `file_path`.
- A document's status (`expired` / `expiring_soon` / `valid` / `no_expiry`) is
  *derived*, not stored — computed from `expiry_date` vs `today`/`soon_cutoff`
  in `db/queries.py` (`_apply_status_filter`, `get_dashboard_summary`) and in
  templates (`documents/list.html`, `documents/detail.html`).

## Available skills

- `.claude/skills/dev-server/SKILL.md` — start/stop/restart the local Flask dev
  server and read its logs.
- `.claude/skills/visual-verify/SKILL.md` — headless-browser screenshot pipeline
  for verifying UI/template/CSS changes against the running dev server.

## MCP servers (`.mcp.json`)

Two project-scoped MCP servers are configured (Claude Code will prompt to
approve them on load; both run via `npx` on demand, no global install needed):

- **playwright** (`@playwright/mcp`) — real browser automation (navigate,
  click, fill forms, screenshot, read console/network). Prefer this over the
  manual `visual-verify` HTML+Edge-headless pipeline when you need to actually
  *interact* with the app (e.g. log in, click "Delete Document" and confirm
  the dialog, submit the photo-upload form) rather than just render a page.
- **sqlite** (`mcp-server-sqlite-npx`) — direct read/write SQL access to
  `data/neverexpire.db`. Use for quick inspection/debugging (e.g. checking
  seeded demo data, document counts, expiry dates) instead of writing
  throwaway `python -c` scripts. Be careful with writes — this is the same DB
  the dev server uses.
