# Data Pipeline — CDPEAD

## Overview

The CDPEAD data pipeline consists of automated ingestors for structured government data sources and a manual entry workflow for PDF-based sources.

## Pipeline Components

### pspc_ingest.py — PSPC Proactive Disclosure

**Source:** Government of Canada Open Data Portal
**URL:** https://open.canada.ca/data/en/dataset/d8f85d91-7dec-4fd1-8f59-b1c3e3e8f780
**Format:** CSV
**Update frequency:** Quarterly
**Automation:** Fully automated

Downloads PSPC contract disclosure data, filters for Department of National Defence contracts, normalizes vendor names using the vendor registry, and maps contracts to tracked programs.

**Output:** `data/processed/pspc_contracts.json`

### gc_infobase.py — GC InfoBase Spending Data

**Source:** Treasury Board of Canada Secretariat
**URL:** https://www.tbs-sct.canada.ca/ems-sgd/edb-bdd/index-eng.html
**Format:** JSON API
**Update frequency:** Annual (after tabling of Public Accounts)
**Automation:** Fully automated

Fetches DND departmental spending by program activity and fiscal year.

**Output:** `data/processed/dnd_spending.json`

### vendor_registry.py — Vendor Name Canonicalization

A lookup table mapping raw vendor names from PSPC data to canonical names and countries. Handles common variations, abbreviations, and corporate name changes.

**Example mappings:**
- "IRVING SHIPBUILDING INC" → Irving Shipbuilding (Canada)
- "GENERAL DYNAMICS LAND SYSTEMS CANADA INC" → General Dynamics Land Systems - Canada (Canada)
- "LOCKHEED MARTIN CANADA INC" → Lockheed Martin Canada (Canada)

### update_programs.py — Master Update Orchestrator

Runs all pipeline components in sequence, merges new data with existing records, flags conflicts, and triggers risk recalculation.

**Usage:**
```bash
# Full update
python data/pipeline/update_programs.py

# Preview changes without writing
python data/pipeline/update_programs.py --dry-run
```

## Scheduling

**Recommended frequency:**
- PSPC data: Quarterly (Jan, Apr, Jul, Oct)
- GC InfoBase: Annually (after Public Accounts, typically Dec)
- Manual data (PBO/AG): As reports are published

**Windows Task Scheduler:**
- Program: `python`
- Arguments: `data/pipeline/update_programs.py`
- Start in: `C:\Users\christopher\Documents\DefenseProcurment\cdpead`
- Trigger: Quarterly on the 15th

## Data Requiring Manual Entry

The following sources cannot be automated and require manual data extraction:

| Source | Data Type | Frequency | Effort |
|--------|-----------|-----------|--------|
| Parliamentary Budget Officer reports | Independent cost estimates | Per-report (2-4/year) | 30-60 min/report |
| Auditor General reports | Audit findings, cost variances | Per-audit (1-3/year) | 60-120 min/report |
| DND Departmental Results Report | Program status updates | Annual | 60-90 min |
| Strong, Secure, Engaged updates | Policy/investment changes | As published | 30-60 min |

Use the Admin interface (`?admin=true` URL parameter) or directly edit `data/processed/programs.json` for manual data entry.

## Known Limitations

1. **Vendor name matching** — PSPC vendor names don't follow a standard format. The vendor registry handles known variations but new vendors require manual addition.
2. **Contract-to-program mapping** — PSPC contract descriptions don't always reference the program name. High-value unmatched contracts are logged for manual review.
3. **Subcontractor data** — Subcontract values are frequently redacted in public disclosures. HHI calculations may underestimate concentration for programs with significant undisclosed sub-tier spending.
4. **Currency normalization** — All figures are in nominal CAD. Inflation adjustment requires manual application of GDP deflator.
5. **PDF extraction** — PBO and AG reports are published as PDF. No automated extraction is currently implemented.
