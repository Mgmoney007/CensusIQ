# CensusIQ — Employee Census Generator
### Triton Benefits & HR Solutions

## Quick Start

### First Time Setup (run once)
```bash
pip install -r requirements.txt
```

### Launch the App
```bash
python app.py
```
The app opens automatically in your browser at http://localhost:5050

---

## How It Works

**Upload** → Drop in any mix of:
- Excel spreadsheets (.xlsx, .xls)
- CSV exports (payroll, Salesforce, ADP, etc.)
- PDF invoices and carrier reports

**Process** → CensusIQ automatically:
- Detects messy/inconsistent column names
- Maps columns to the Triton standard schema
- Normalizes all date formats to MM/DD/YYYY
- Merges the same employee across multiple files (e.g. DOB on the PDF, plan election on the spreadsheet)
- Rebuilds dependent/subscriber relationships
- Validates and flags errors

**Review** → See exactly what was found, merged, and flagged

**Edit** → Fix issues inline — click any cell to edit directly

**Export** → One click for:
- Triton Benefits Master Census (.xlsx)
- BCBS Standard format
- Aetna RFP format
- UHC Group Census format
- Cigna Standard format
- All selected files bundled as a .zip

---

## The Core Problem This Solves

You receive:
- A payroll export missing DOBs
- A carrier invoice PDF with SSNs and DOBs but no plan elections
- An old spreadsheet with plan elections but inconsistent column names

CensusIQ matches the person across all three sources by name + DOB + SSN last 4, 
fills in missing fields from each source, then outputs one clean, complete file.

---

## Carrier Formats

| Carrier | Format | Columns |
|---------|--------|---------|
| Triton Master | .xlsx | All fields |
| BCBS | .xlsx | 11 standard |
| Aetna | .xlsx | 11 RFP standard |
| UHC | .xlsx | 11 group census |
| Cigna | .xlsx | 10 standard |

---

## File Structure
```
censusiq/
  app.py          ← Main app (run this)
  requirements.txt
  templates/
    index.html    ← The full UI
  uploads/        ← Temporary upload storage
  exports/        ← Generated export files
```

---

*Built for Triton Benefits & HR Solutions*
