# CDPEAD — Canadian Defence Procurement Engineering Analytics Dashboard

An open-source engineering analytics platform for quantitative analysis of Canadian defence procurement programs.

![Classification](https://img.shields.io/badge/Classification-UNCLASSIFIED-2ea44f)
![Data](https://img.shields.io/badge/Data-Open%20Source%20Only-blue)
![License](https://img.shields.io/badge/License-MIT-yellow)
![Stack](https://img.shields.io/badge/Stack-React%20%2B%20FastAPI%20%2B%20PostgreSQL-orange)

---

## Overview

CDPEAD is a decision-support system built for defence systems engineers, DND technical authorities, DGLEPM project officers, and parliamentary technical staff. It provides structured, data-driven insights into major Canadian defence procurement programs through quantitative risk modeling, supplier dependency analysis, and budget allocation tracking.

This is **not** a political commentary tool. It is an engineering analytics platform built on transparent mathematical models, explicit assumptions, and reproducible calculations.

### Who This Is For

- DND Technical Authorities evaluating program health
- DGLEPM Project Officers tracking cost and schedule baselines
- Defence Systems Engineers assessing supplier and technology risk
- Parliamentary Budget Office analysts requiring structured procurement data
- Risk and Cost Analysts performing portfolio-level assessments

---

## Core Modules

### A. Program Performance Tracker

Tracks 12+ major procurement programs across Land, Naval, Air, Cyber, and Space domains. Provides sortable, filterable, searchable program views with drill-down detail including supplier breakdown and risk radar decomposition.

**Tracked metrics per program:**
- Initial approved budget vs. current estimated cost
- Cost growth percentage
- Initial delivery date vs. current forecast
- Schedule slippage (months)
- Prime contractor and known subcontractors
- Program status (Definition → Implementation → Delayed → Delivered)
- Technology maturity classification

### B. Procurement Risk Index (PRI) Engine

A transparent, weighted composite risk model with interactive sensitivity analysis.

```
PRI = w₁·C_norm + w₂·S_norm + w₃·SC_norm + w₄·FD_norm + w₅·TM_score
```

| Factor | Weight | Component |
|--------|--------|-----------|
| Cost Growth | 0.30 | `min(cost_growth% / 100, 1.0)` |
| Schedule Variance | 0.25 | `min(slip_months / 120, 1.0)` |
| Supplier Concentration | 0.20 | `min(HHI / 0.5, 1.0)` |
| Foreign Dependency | 0.15 | `foreign_value / total_value` |
| Technology Maturity | 0.10 | Categorical: Proven→0.10 ... Experimental→0.90 |

All weights are justified in the design document. The dashboard provides interactive weight adjustment with real-time sensitivity analysis (±20% perturbation tornado charts).

**Risk Tier Classification:**

| PRI Range | Tier | Interpretation |
|-----------|------|---------------|
| 0.00–0.25 | Low | Standard oversight sufficient |
| 0.26–0.50 | Medium | Increased monitoring recommended |
| 0.51–0.75 | High | Intervention analysis warranted |
| 0.76–1.00 | Critical | Restructuring analysis recommended |

### C. Supplier Dependency Analysis

- Herfindahl-Hirschman Index (HHI) concentration metrics at program and portfolio level
- Canadian vs. foreign supplier classification with value breakdown
- Prime contractor / subcontractor mapping
- Country-level spending distribution
- Complete supplier registry with cross-program role tracking

### D. Budget Allocation Analytics

- Spending by capability domain (Land, Naval, Air, Cyber, Space)
- Distribution by prime contractor
- Initial budget vs. current cost comparison per program
- Domain-level cost growth tracking
- Portfolio concentration metrics

### E. Methodology & Documentation

Complete in-platform documentation of all formulas, weight justifications, normalization methods, data sources, assumptions registry, and architectural decisions. Every modeling choice is visible and auditable.

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | React 18 + TypeScript | Component modularity, type safety, ecosystem |
| Visualization | Recharts + D3.js | Standard charts + custom risk visualizations |
| Backend | FastAPI (Python 3.11+) | Async support, auto OpenAPI docs, Pydantic validation |
| Database | SQLite (dev) / PostgreSQL 15 (prod) | Zero-config local dev; production-ready with window functions |
| ORM | SQLAlchemy 2.0 | Async-compatible, migration support |
| Data Pipeline | Pandas + custom ETL | Automated PSPC ingestion + manual entry support |
| Containerization | Docker Compose | One-command full-stack deployment |

---

## Data Sources

All data is derived from publicly available Canadian government sources.

| Source | Data Available | Format | Automation |
|--------|---------------|--------|------------|
| PSPC Proactive Disclosure | Contract awards, vendor names, values | CSV | ✅ Automated |
| GC InfoBase (Treasury Board) | DND spending by program activity | JSON API | ✅ Automated |
| Government of Canada Buyandsell | Tender data, solicitations | Structured | ✅ Automated |
| DND Departmental Plans & Results | Program status, budgets, milestones | HTML/PDF | ⚠️ Semi-automated |
| Parliamentary Budget Officer | Independent cost estimates | PDF | ❌ Manual extraction |
| Auditor General Reports | Audit findings, variance analysis | PDF | ❌ Manual extraction |
| Strong, Secure, Engaged (SSE) | Policy framework, planned investments | PDF | ❌ Manual extraction |

> **Note:** The majority of Canadian defence procurement data resides in PDF documents and requires manual extraction. The automated pipeline covers PSPC contract data (quarterly) and GC InfoBase spending data (annual). A future enhancement would be an NLP-based PDF extraction pipeline.

---

## Quick Start

### Option A — Local Development (No Docker)

**Prerequisites:** Node.js 18+, Python 3.11+

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/cdpead.git
cd cdpead

# Backend
cd backend
pip install -r requirements.txt
python scripts/seed_db.py
uvicorn app.main:app --reload --port 8080

# Frontend (new terminal)
cd frontend
npm install
npm start
```

Dashboard opens at `http://localhost:3000`. API at `http://localhost:8080`.

### Option B — Docker Compose (One Command)

**Prerequisites:** Docker Desktop

```bash
git clone https://github.com/YOUR_USERNAME/cdpead.git
cd cdpead
docker-compose up --build
```

Dashboard available at `http://localhost`.

### Option C — Frontend Only (Static Demo)

If you just want to view the dashboard without the backend:

```bash
cd frontend
npm install
npm start
```

The dashboard will display cached data with a "showing cached data" indicator.

---

## Data Pipeline

### Automated Updates

```bash
# Run the full data pipeline
python data/pipeline/update_programs.py

# Dry run (see what would change without writing)
python data/pipeline/update_programs.py --dry-run

# Individual pipeline components
python data/pipeline/pspc_ingest.py      # PSPC contract data
python data/pipeline/gc_infobase.py      # GC InfoBase spending data
```

### Scheduling

On Windows, use Task Scheduler to run quarterly:
```
Action: python data/pipeline/update_programs.py
Trigger: Quarterly (Jan 15, Apr 15, Jul 15, Oct 15)
```

On Linux/macOS:
```bash
# crontab -e
0 6 15 1,4,7,10 * cd /path/to/cdpead && python data/pipeline/update_programs.py >> data/pipeline/update_log.txt 2>&1
```

### Manual Data Entry

For data from PBO reports, AG audits, and policy documents, use the Admin interface:

1. Navigate to the dashboard with `?admin=true` in the URL
2. Use the Add/Edit Program form to enter or update program data
3. Each entry requires a Source URL and Notes field for provenance tracking

---

## Project Structure

```
cdpead/
├── README.md
├── DESIGN_DOCUMENT.md
├── LICENSE
├── docker-compose.yml
├── .env.example
├── .gitignore
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── public/
│   │   └── index.html
│   └── src/
│       ├── App.jsx                    # Main application shell
│       ├── index.js                   # Entry point
│       ├── components/
│       │   ├── ProgramTracker/        # Module A
│       │   ├── RiskIndex/             # Module B
│       │   ├── SupplierAnalysis/      # Module C
│       │   ├── BudgetAnalytics/       # Module D
│       │   └── Admin/                 # Admin data entry
│       ├── data/
│       │   ├── programs.json          # Static program data
│       │   └── dataLoader.js          # API/static data loader
│       ├── utils/
│       │   ├── riskEngine.js          # PRI calculation engine
│       │   └── formatters.js          # Currency/percent formatting
│       ├── hooks/
│       └── types/
│
├── backend/
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .env.example
│   ├── app/
│   │   ├── main.py                    # FastAPI application
│   │   ├── config.py                  # Settings and configuration
│   │   ├── models/
│   │   │   ├── program.py             # SQLAlchemy models
│   │   │   ├── supplier.py
│   │   │   └── risk.py
│   │   ├── routers/
│   │   │   ├── programs.py            # /api/programs endpoints
│   │   │   ├── risk.py                # /api/risk endpoints
│   │   │   ├── suppliers.py           # /api/suppliers endpoints
│   │   │   └── budget.py              # /api/budget endpoints
│   │   ├── services/
│   │   │   ├── risk_engine.py         # PRI calculation service
│   │   │   ├── hhi_calculator.py      # HHI computation
│   │   │   └── monte_carlo.py         # Monte Carlo simulation
│   │   └── schemas/
│   │       └── responses.py           # Pydantic response models
│   ├── scripts/
│   │   └── seed_db.py                 # Database seeding
│   └── tests/
│
├── data/
│   ├── raw/                           # Downloaded source data
│   ├── processed/                     # Cleaned, normalized data
│   │   └── programs.json              # Master program dataset
│   └── pipeline/
│       ├── README.md                  # Pipeline documentation
│       ├── pspc_ingest.py             # PSPC contract downloader
│       ├── gc_infobase.py             # GC InfoBase API client
│       ├── vendor_registry.py         # Vendor name canonicalization
│       └── update_programs.py         # Master update orchestrator
│
├── docs/
│   ├── DESIGN_DOCUMENT.md             # Full system design
│   ├── EXECUTIVE_SUMMARY.md           # One-page overview
│   ├── architecture.md                # Architecture details
│   ├── risk_model.md                  # PRI model documentation
│   └── data_dictionary.md             # Field definitions
│
├── scripts/
│   └── schedule_update.py             # Scheduled pipeline runner
│
└── nginx/
    └── nginx.conf                     # Reverse proxy configuration
```

---

## Engineering Design Decisions

1. **Transparent risk model** — No arbitrary scores. Every weight has documented rationale derived from Auditor General and PBO report analysis patterns.

2. **HHI for supplier concentration** — Standard industrial metric used by Competition Bureau Canada, not a custom invention. Provides internationally recognized concentration measurement.

3. **Saturation normalization** — Cost growth saturates at 100%, schedule at 120 months. Programs beyond these thresholds are already in structural distress — additional growth doesn't meaningfully change risk classification.

4. **Technology maturity mapping** — Categorical scores derived from TRL groupings with non-linear spacing reflecting the exponential increase in cost/schedule risk as TRL decreases (consistent with GAO and RAND findings on defence acquisition).

5. **Explicit assumptions registry** — Every modeling assumption is documented with an impact rating and mitigation strategy. No hidden parameters.

6. **Dual-mode data layer** — Frontend works with static JSON (demo mode) or live API (production mode) with automatic fallback. The dashboard never breaks due to backend unavailability.

7. **SQLite default, PostgreSQL production** — Zero-config local development with a clear migration path to production database. No Docker dependency for getting started.

---

## API Documentation

When the backend is running, full interactive API documentation is available at:

- **Swagger UI:** `http://localhost:8080/docs`
- **ReDoc:** `http://localhost:8080/redoc`

### Key Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/health` | Health check |
| `GET` | `/api/programs` | List programs (filterable) |
| `GET` | `/api/programs/{id}` | Program detail with suppliers |
| `GET` | `/api/risk/scores` | All risk scores |
| `POST` | `/api/risk/calculate` | Recalculate with custom weights |
| `GET` | `/api/suppliers` | Supplier registry |
| `GET` | `/api/suppliers/concentration` | HHI metrics |
| `GET` | `/api/budget/by-domain` | Domain spending |
| `GET` | `/api/budget/by-contractor` | Contractor spending |

---

## Programs Tracked

| Code | Program | Domain | Status |
|------|---------|--------|--------|
| CSC | Canadian Surface Combatant | Naval | Implementation |
| FFCP | Future Fighter Capability (F-35) | Air | Implementation |
| AOPS | Arctic & Offshore Patrol Ships | Naval | Implementation |
| JSS | Joint Support Ship | Naval | Delayed |
| FWSAR | Fixed-Wing Search & Rescue (CC-295) | Air | Implementation |
| ACSV | Armoured Combat Support Vehicle | Land | Implementation |
| RPAS | Remotely Piloted Aircraft System | Air | Definition |
| LVM | Logistics Vehicle Modernization | Land | Implementation |
| STTC | Strategic Tanker Transport | Air | Definition |
| DCOMM | Digital Communications on the Move | Cyber | Implementation |
| PSM | Patrol Ship Modernization | Naval | Delayed |
| GBAD | Ground-Based Air Defence | Land | Definition |

---

## Roadmap

- [x] Program Performance Tracker with sorting/filtering
- [x] Procurement Risk Index with interactive weights
- [x] Supplier Dependency Analysis with HHI
- [x] Budget Allocation Analytics
- [x] Full methodology documentation
- [x] FastAPI backend with REST API
- [x] PSPC automated data pipeline
- [ ] Monte Carlo cost growth simulation
- [ ] Scenario modeling (budget cut stress tests)
- [ ] PDF report generation and export
- [ ] NLP-based PDF extraction pipeline
- [ ] Time-series budget tracking with annual snapshots
- [ ] GraphQL API alternative

---

## Contributing

Contributions are welcome, particularly in these areas:

1. **Data collection** — Manual extraction from PBO and AG reports
2. **Pipeline improvements** — Better vendor name matching, new data sources
3. **Additional programs** — Adding procurement programs not yet tracked
4. **Visualization** — New chart types or analytical views
5. **Testing** — Unit and integration test coverage

Please ensure all data contributions cite their public source and include extraction date.

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

## Disclaimer

This platform uses only publicly available data from Canadian government sources. No classified, restricted, or protected information is included or generated. All cost figures, schedule dates, and supplier relationships are derived from open-source publications. Where data is unavailable or redacted, it is explicitly flagged.

This is an independent analytical tool and is not affiliated with, endorsed by, or representative of the Department of National Defence, the Canadian Armed Forces, or the Government of Canada.

---

*Built as an engineering analytics platform. Professional. Technical. Analytical. Non-political.*
