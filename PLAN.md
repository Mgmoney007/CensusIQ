# CensusIQ — Development Roadmap

## Context
CensusIQ is 85% feature-complete for core business logic but has zero production infrastructure. The app works well as a local desktop tool (Flask + in-memory state), but lacks git history, tests, logging, deployment config, and several planned features. This roadmap establishes the foundation for stability, completes planned features, and opens paths for growth.

**Current state:** app.py (1,187 lines) + index.html (1,553 lines) — fully working UHC OCR, 5 carrier exports, reconcile engine, 6-panel SPA frontend. Zero commits, zero tests, no .gitignore, no deployment config.

---

## Phase 0: Foundation (Day 1)
*Get version control and project hygiene in order before anything else.*

### 0.1 — Create `.gitignore`
- **File:** `.gitignore`
- Include: `__pycache__/`, `*.pyc`, `uploads/`, `exports/`, `.env`, `venv/`, `.pytest_cache/`, `*.log`, `logs/`, `Thumbs.db`
- **Complexity:** Small

### 0.2 — Initial git commit
- Stage: `.gitignore`, `CLAUDE.md`, `README.md`, `app.py`, `templates/`, `requirements.txt`
- Commit message: "Initial commit: CensusIQ v1.0 core"
- **Complexity:** Small

### 0.3 — Pin dependencies
- **File:** `requirements.txt`
- Replace all `>=` with `==` using currently installed versions
- Add `python-dotenv==1.1.0` and `waitress==3.0.2` (Windows-compatible production server)
- **Complexity:** Small

### 0.4 — Add `pyproject.toml`
- **File:** `pyproject.toml`
- Project metadata: name, version (1.0.0), Python >=3.11, description
- **Complexity:** Small

---

## Phase 1: Stability & Quality (Days 2-5)
*Make the app production-grade: logging, config, error handling, tests.*

### 1.1 — Configuration management (.env)
- **Files:** `app.py` (lines 26-35), `.env.example`
- Load config via `python-dotenv`: `SECRET_KEY`, `UPLOAD_FOLDER`, `EXPORT_FOLDER`, `MAX_CONTENT_LENGTH_MB`, `LOG_LEVEL`, `PORT`
- Replace hardcoded values in `app.py` with `os.getenv()` calls
- **Complexity:** Small

### 1.2 — Logging infrastructure
- **File:** `app.py` (top of file, ~lines 7-20)
- Add `logging` with `RotatingFileHandler` → `logs/censusiq.log`
- Log in: `parse_excel_csv()`, `parse_uhc_invoice_ocr()`, `merge_records()`, `build_reconcile()`, all routes
- Replace any `print()` calls with `app.logger`
- Add `logs/` to `.gitignore`
- **Complexity:** Medium

### 1.3 — Error handling improvements
- **File:** `app.py` (after routes section, ~line 1165)
- Add Flask error handlers: `@app.errorhandler(413)`, `@app.errorhandler(500)`, `@app.errorhandler(Exception)`
- Improve parse function error messages: return structured `{"file": name, "error": msg, "suggestion": hint}`
- Add timeout guard for OCR operations (60s per page)
- **Complexity:** Medium

### 1.4 — Health check endpoint
- **File:** `app.py` (routes section)
- `GET /health` → returns `{"status": "healthy", "version": "1.0.0", "ocr_available": bool}`
- **Complexity:** Small

### 1.5 — Production server setup
- **Files:** `wsgi.py`, `scripts/run_production.bat`
- `wsgi.py`: simple `from app import app` entry point
- Windows: `waitress-serve --port=5050 --threads=4 wsgi:app`
- Linux/Mac: `gunicorn --bind 0.0.0.0:5050 --workers 4 --timeout 120 wsgi:app`
- Conditionally skip browser auto-open when `FLASK_ENV=production`
- **Complexity:** Small

### 1.6 — Testing infrastructure
- **Files:** `tests/conftest.py`, `tests/test_normalization.py`, `tests/test_parsers.py`, `tests/test_merge.py`, `tests/test_validation.py`, `tests/test_reconcile.py`, `tests/test_routes.py`
- Add `pytest`, `pytest-cov` to requirements
- Fixtures: Flask test client, sample records, sample Excel/PDF generators
- Target: >70% coverage on core business logic
- **Complexity:** Large

### 1.7 — CI/CD pipeline (GitHub Actions)
- **File:** `.github/workflows/test.yml`
- Trigger on push to main + PRs
- Matrix: Python 3.11, 3.12
- Steps: install deps → run pytest with coverage
- **Complexity:** Medium

---

## Phase 2: Feature Completion (Days 6-10)
*Build out all planned-but-not-started features from CLAUDE.md.*

### 2.1 — Reconcile Excel export *(must-have)*
- **Files:** `app.py` (~line 808), `templates/index.html` (~line 1525)
- New function `build_reconcile_excel(reconcile_rows, invoice_meta)` using openpyxl
- Color-coded rows: red (mismatch), amber (warning), teal (matched)
- Invoice metadata header block + legend
- New route `GET /export/reconcile`
- Update frontend `exportReconcile()` to call new endpoint instead of client-side CSV
- **Complexity:** Medium

### 2.2 — Upgrade to rapidfuzz for name matching
- **File:** `app.py` (~line 830)
- Add `rapidfuzz>=3.0` to requirements
- Replace `fuzzy_name_key()` string-slicing with Levenshtein distance matching (threshold ~85%)
- Keep backward-compatible: same `build_reconcile()` interface
- Add tests comparing old vs new matching on OCR typos
- **Complexity:** Medium

### 2.3 — DOB/hire date from invoice side in reconcile
- **Files:** `app.py` (reconcile output dict), `templates/index.html` (reconcile table ~line 736)
- Extract DOB from UHC invoice OCR text (add regex pattern)
- Include `invoice_dob` and `invoice_hire_date` in reconcile output rows
- Show both invoice and census dates side-by-side in table
- **Complexity:** Small

### 2.4 — BCBS invoice OCR parser
- **File:** `app.py` (~line 350)
- New function `parse_bcbs_invoice_ocr()` following `parse_uhc_invoice_ocr()` as template
- Requires sample BCBS invoices to develop regex patterns
- Wire into `parse_pdf()` router via `detect_carrier_invoice()` (detection already works)
- **Complexity:** Large — *blocked until sample invoices available*

### 2.5 — Aetna invoice OCR parser
- Same structure as 2.4 for Aetna format
- **Complexity:** Large — *blocked until sample invoices available*

### 2.6 — Cigna invoice OCR parser
- Same structure as 2.4 for Cigna format
- **Complexity:** Large — *blocked until sample invoices available*

---

## Phase 3: UX Polish (Days 11-14)
*Accessibility, responsiveness, and performance for larger datasets.*

### 3.1 — Mobile responsiveness
- **File:** `templates/index.html` (CSS section, ~line 8)
- Add `@media (max-width: 768px)` breakpoints
- Collapsible sidebar → hamburger menu on mobile
- Stack grid layouts to single column
- Horizontal scroll for data tables
- **Complexity:** Medium

### 3.2 — Accessibility (WCAG 2.1 AA)
- **File:** `templates/index.html`
- Add `aria-label` to all interactive elements
- Add keyboard navigation: Tab order, Escape to close, arrow keys in tables
- Focus management on panel transitions
- Text alternatives for color-only indicators (status dots/pills)
- **Complexity:** Medium

### 3.3 — Table pagination / virtualization
- **File:** `templates/index.html` (~line 1152, `renderTable()`)
- Add pagination (50 rows/page) for records table — simpler than virtual scroll, no dependencies
- Page controls: prev/next, page indicator, rows-per-page selector
- Keep search/filter working across all pages
- **Complexity:** Medium

### 3.4 — Better empty states & error recovery
- **File:** `templates/index.html`
- Add empty state messages for each panel when no data loaded
- Add inline validation feedback (checkmarks, progress bars)
- Show actionable suggestions on parse failures
- **Complexity:** Small

### 3.5 — Keyboard shortcuts
- **File:** `templates/index.html` (~line 1550)
- `Ctrl+U` → Upload, `Ctrl+P` → Process, `Ctrl+E` → Export, `Ctrl+R` → Reconcile, `Ctrl+/` → Focus search
- Show shortcut legend in sidebar footer
- **Complexity:** Small

---

## Phase 4: Deployment & Distribution (Days 15-18)
*Make it deployable as a container or distributable as a desktop app.*

### 4.1 — Dockerization
- **Files:** `Dockerfile`, `docker-compose.yml`, `.dockerignore`
- Base: `python:3.11-slim` + `tesseract-ocr` + `poppler-utils`
- Expose port 5050, run via gunicorn
- Volumes: `uploads/`, `exports/`, `logs/`
- Health check: `curl -f http://localhost:5050/health`
- **Complexity:** Medium

### 4.2 — Cloud deployment (pick one)
- **Recommended: Fly.io** or **Render** — both support Docker, scale-to-zero, persistent volumes
- Create platform-specific config (`fly.toml` or `render.yaml`)
- Add deploy script to `scripts/`
- **Complexity:** Medium

### 4.3 — PyInstaller desktop packaging (Windows .exe)
- **Files:** `censusiq.spec`, `scripts/build_exe.bat`
- Bundle Flask app + templates + Tesseract binary as standalone `.exe`
- Auto-opens browser on launch (existing behavior)
- Test on clean Windows machine without Python
- **Complexity:** Large

---

## Phase 5: Growth Features (Days 19+)
*Features for scaling beyond single-user desktop use.*

### 5.1 — Multi-group management
- **File:** `app.py` (session_data restructure)
- Change `session_data` to hold multiple groups: `{"groups": {id: {...}}, "active_group": id}`
- Routes: `POST /groups/create`, `GET /groups`, `POST /groups/<id>/activate`, `DELETE /groups/<id>`
- Group selector in sidebar
- Session persistence to JSON file on disk (stays within "no database" rule)
- **Complexity:** Large

### 5.2 — User authentication (web deployment only)
- Add `flask-login` + simple user model
- Login/register pages, route protection middleware
- Only needed if deploying as multi-tenant web service
- **Complexity:** Large

### 5.3 — REST API for integrations
- `/api/v1/` namespace with upload, process, records, reconcile, export endpoints
- API key auth, rate limiting (`flask-limiter`)
- OpenAPI/Swagger documentation
- **Complexity:** Medium

### 5.4 — Batch processing
- Process multiple groups in one operation
- Background thread execution with progress polling
- Combined ZIP export across groups
- **Complexity:** Large

### 5.5 — Audit trail
- Log all user actions: uploads, edits, exports with timestamps
- Store in JSON log file (no database)
- Admin view in UI
- **Complexity:** Medium

---

## Critical Path (MVP to Production)

```
Phase 0 (all) → Phase 1.1-1.5 → Phase 4.1 → Deploy
       1 day        3 days         1 day
                                         = ~5 days to production-ready
```

## Verification Strategy

| Phase | How to verify |
|-------|--------------|
| 0 | `git log` shows initial commit, `git status` clean |
| 1 | `pytest tests/ -v --cov=app` passes >70%, `curl /health` returns 200 |
| 2 | Upload UHC invoice + census → reconcile shows DOB both sides, export downloads .xlsx |
| 3 | Chrome DevTools responsive mode at 375px → usable, axe scan → 0 critical violations |
| 4 | `docker-compose up` → app at localhost:5050, health check passes |
| 5 | Create 2 groups, switch between them, data stays isolated |
