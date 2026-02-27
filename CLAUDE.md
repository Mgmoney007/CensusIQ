# CensusIQ — Claude Code Memory
Triton Benefits & HR Solutions | Employee Census Generator
Reduces manual census processing from 2.5 hours → 4 minutes

---

## Stack
- **Backend:** Python 3.11 + Flask, runs on port 5050
- **Excel/CSV:** pandas + openpyxl
- **PDF (text-based):** pdfplumber
- **PDF (image/scanned):** pdf2image + pytesseract (OCR)
- **Frontend:** Single-file vanilla JS in `templates/index.html` — no bundler, no framework
- **State:** In-memory `session_data` dict — no database

## Run & Test
```bash
python app.py              # launches browser at http://localhost:5050
python -c "from app import X; ..."   # quick unit tests
```

## File Structure
```
censusiq/
├── app.py                 # ALL backend logic + routes (single file)
├── templates/index.html   # ENTIRE frontend (single file)
├── requirements.txt
├── uploads/               # temp uploaded files
└── exports/               # generated .xlsx and .zip files
```

---

## Architecture — app.py Sections (in order)

### Session State
All runtime data lives in one dict — never add a database:
```python
session_data = {
    "files": [],           # {name, type, path}
    "records": [],         # merged normalized employee records
    "issues": [],          # validation flags
    "merge_log": [],       # cross-source merge activity
    "company_name": "",    # user-set group/client name
    "detected_company": "",# auto-detected from carrier invoice OCR
    "invoice_meta": {},    # invoice number, period, customer no.
    "invoice_records": [], # raw records from PDFs
    "census_records": [],  # raw records from spreadsheets
    "reconcile": [],       # computed reconcile rows
}
```

### Normalization Maps (do not duplicate — always use these)
- `RELATIONSHIP_MAP` → normalizes to: `Subscriber`, `Spouse`, `Child`
- `GENDER_MAP` → normalizes to: `M`, `F`
- `COVERAGE_MAP` → normalizes to: `Single`, `2 Adult`, `Parent/CH`, `Fam`, `WC/WO/WP`
- `STATUS_MAP` → normalizes to: `F`, `P`, `C`, `R`, `S`
- `TIER_NORMALIZE` → used by reconcile engine for tier comparison

### Key Functions
| Function | Purpose |
|---|---|
| `guess_column_mapping(columns)` | Regex fuzzy-match raw headers → standard fields |
| `parse_excel_csv(filepath, filename)` | Parses .xlsx/.xls/.csv, auto-detects header row |
| `detect_carrier_invoice(text)` | Returns 'UHC', 'AETNA', 'CIGNA', 'BCBS', or None |
| `parse_uhc_invoice_ocr(filepath, filename)` | Full OCR pipeline for UHC scanned invoices |
| `parse_pdf(filepath, filename)` | Router: pdfplumber for text PDFs, OCR for image PDFs |
| `normalize_record(rec)` | Applies all normalization maps to a raw record |
| `merge_records(records)` | Deduplicates across sources using last\|first\|dob\|ssn key |
| `validate_records(records)` | Returns issues list with severity: error/warning/info |
| `build_triton_census(records, company_name)` | Builds Triton-template Excel (legend + data rows) |
| `build_carrier_file(records, carrier)` | Builds carrier-specific Excel (BCBS/Aetna/UHC/Cigna) |
| `build_reconcile(all_records)` | Compares invoice vs census records, returns diff rows |
| `fuzzy_name_key(last, first)` | 6-char last + 4-char first for OCR-tolerant matching |

### Routes
```
GET  /              → index.html
POST /upload        → save files to uploads/, update session_data["files"]
POST /process       → parse all files, normalize, merge, validate, reconcile
POST /company       → set session_data["company_name"]
GET  /records       → return session_data["records"] + issues
POST /records/update → edit single field, re-validate
GET  /reconcile     → return session_data["reconcile"] + invoice_meta
GET  /export/<carrier> → download single .xlsx (triton/BCBS/Aetna/UHC/Cigna)
POST /export/all    → download .zip of selected carriers
POST /reset         → clear all session_data
```

---

## UHC Invoice OCR Pipeline
UHC invoices are **image-based PDFs** (scanned) — pdfplumber returns 0 chars.

OCR flow in `parse_uhc_invoice_ocr()`:
1. `convert_from_path()` at 200 DPI → PIL images
2. `pytesseract.image_to_string()` per page
3. Company name: extracted from first line of page 3+ (strip "Page X of Y")
4. Employee rows: regex `\d{6,7}\s*[\|l]\s*([A-Z]...)\s+Lib[A-Z0-9]...([A-Z]{1,3})\s+A`
5. Each employee appears **twice** per invoice (two plan components) — dedup by `last|first` key
6. Coverage codes: `E`=Single, `ES`=2 Adult, `ESC`=Fam, `EC`=Parent/CH
7. Premium total: last dollar amount on first of the two lines per employee

Carriers not yet OCR-supported (only UHC is): BCBS, Aetna, Cigna
When adding a new carrier OCR parser, follow `parse_uhc_invoice_ocr()` as the template.

---

## Reconcile Engine
`build_reconcile(all_records)` compares records tagged `_source_type='invoice'` vs `'census'`.

Match statuses (sorted in this order in output):
1. `mismatch` — same employee, different coverage tier → **error**
2. `invoice_only` — on invoice, not in census → **error**
3. `census_only` — in census, not on invoice → **warning**
4. `warning` — matched but name spelling differs (OCR typo)
5. `matched` — clean match on name + tier

Matching uses `fuzzy_name_key()` — tolerates OCR typos like "Strang" vs "Strong".

---

## Frontend (index.html) — Panels & Flow
Navigation order: Upload → Process → Review → Reconcile → Records → Export

| Panel ID | Purpose |
|---|---|
| `panel-upload` | Company name input + drag-drop file zone |
| `panel-process` | 6-step animated processing visualization |
| `panel-review` | Stats cards, issues list, merge log, parse logs |
| `panel-reconcile` | Side-by-side invoice vs census comparison table |
| `panel-records` | Editable employee records table |
| `panel-export` | Carrier selection + download buttons |

Design system:
- Colors: `--navy` `--teal` (#00C9A7) `--amber` (#F5A623) `--red` `--white`
- Fonts: DM Serif Display (headings), Sora (body), DM Mono (code/data)
- All state in JS vars: `allRecords`, `allIssues`, `reconcileData`, `companyName`

---

## Carrier Export Formats

| Carrier | Header Color | Key Column Differences |
|---|---|---|
| Triton | `#0D1B2A` navy | Legend block rows 1-18, example rows 21-24, data from row 26 |
| BCBS | `#003087` | Standard order: last, first, dob, gender, ssn... |
| Aetna | `#7B0C2A` | Relationship before DOB, "Coverage Tier" not "Plan" |
| UHC | `#003366` | "Subscriber Last", "Tax ID" for SSN, "Member Type" for relationship |
| Cigna | `#006B50` | "Birth Date", "Benefit Plan", no emp_status column |

---

## Standard Field Names (always use these internal names)
`last_name`, `first_name`, `relationship`, `gender`, `dob`, `state`, `zip`,
`plan_election`, `emp_status`, `hire_date`, `term_date`, `ssn`, `email`,
`phone`, `salary`, `hours_per_week`, `waive_reason`

Internal metadata fields prefixed with `_`: `_source`, `_source_type`,
`_carrier`, `_coverage_code`, `_premium_total`, `_invoice_period`

---

## Rules — Never Break These
- **Always save** plans to /PLAN.md in the project root before executing
- **Never split** `app.py` into multiple files — keep all backend in one file
- **Never split** `index.html` into separate CSS/JS files — keep frontend in one file
- **Never add a database** — `session_data` dict is intentional for desktop use
- **Always use** existing normalization maps — don't create parallel maps
- **All new routes** go in the `# ROUTES` section of app.py
- **Always tag** records with `_source_type: 'invoice'` or `'census'` in parse functions
- **Never use** PyMuPDF — always use pdfplumber (text) or pdf2image+pytesseract (image)
- **Test new functions** with `python -c "from app import X; ..."` before wiring to routes

---

## Planned / Not Yet Built
- Multi-group management (currently one group per session)
- Aetna, BCBS, Cigna invoice OCR parsers
- Reconcile Excel export (currently CSV only)
- DOB and hire date columns in reconcile table from invoice side
- `rapidfuzz` upgrade for name matching (currently regex-based)
