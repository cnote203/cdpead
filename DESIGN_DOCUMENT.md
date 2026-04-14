# Canadian Defence Procurement Engineering Analytics Dashboard
## System Design Document v1.0

---

## 1. System Architecture

### 1.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  React + TypeScript SPA                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Program  │ │  Risk    │ │ Supplier │ │ Budget   │       │
│  │ Tracker  │ │  Index   │ │ Analysis │ │Analytics │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│  Recharts / D3.js Visualizations                             │
└─────────────────────────┬───────────────────────────────────┘
                          │ REST API (JSON)
┌─────────────────────────┴───────────────────────────────────┐
│                    APPLICATION LAYER                          │
│  FastAPI (Python 3.11+)                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Program  │ │  Risk    │ │ Supplier │ │ Budget   │       │
│  │  Router  │ │  Engine  │ │  Router  │ │  Router  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│  ┌──────────────────────────────────────────────┐           │
│  │         Risk Calculation Engine               │           │
│  │  - PRI Formula    - Monte Carlo Module        │           │
│  │  - HHI Calculator - Sensitivity Analyzer      │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────┬───────────────────────────────────┘
                          │ SQLAlchemy ORM
┌─────────────────────────┴───────────────────────────────────┐
│                    DATA LAYER                                 │
│  PostgreSQL 15+                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Programs │ │Suppliers │ │  Budget  │ │   Risk   │       │
│  │          │ │          │ │Snapshots │ │  Scores  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────┴───────────────────────────────────┐
│                    INGESTION LAYER                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                    │
│  │   CSV    │ │   Data   │ │  Normal- │                    │
│  │  Parser  │ │ Cleaner  │ │  izer    │                    │
│  └──────────┘ └──────────┘ └──────────┘                    │
│  Sources: DPR, DRDC, PBO, PSPC Open Data                    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer          | Technology              | Rationale                                      |
|----------------|-------------------------|-------------------------------------------------|
| Frontend       | React 18 + TypeScript   | Type safety, component modularity, ecosystem    |
| Visualization  | Recharts + D3.js        | Recharts for standard charts, D3 for custom     |
| API            | FastAPI (Python 3.11+)  | Async support, auto-docs, Pydantic validation   |
| ORM            | SQLAlchemy 2.0          | Mature, async-compatible, migration support      |
| Database       | PostgreSQL 15           | JSONB support, window functions, extensibility   |
| Migrations     | Alembic                 | Version-controlled schema evolution              |
| Data Pipeline  | Pandas + custom modules | Familiar in defence analytics community          |
| Testing        | pytest + React Testing  | Standard coverage tooling                        |

### 1.3 Design Decisions

**Why FastAPI over Django?**
- Lighter footprint for an analytics API
- Native async for future Monte Carlo parallelism
- Auto-generated OpenAPI docs align with engineering transparency requirement

**Why PostgreSQL over SQLite?**
- Window functions needed for time-series budget analysis
- JSONB for flexible metadata storage
- Production-path alignment for future cloud deployment

**Why Recharts as primary?**
- Native React integration reduces glue code
- D3 reserved for custom risk visualizations that Recharts cannot handle

---

## 2. Database Schema

### 2.1 Entity-Relationship Diagram

```
┌──────────────────┐       ┌──────────────────┐
│    programs       │       │    suppliers      │
├──────────────────┤       ├──────────────────┤
│ id (PK)          │       │ id (PK)          │
│ name             │       │ name             │
│ capability_domain│       │ country          │
│ description      │       │ is_canadian      │
│ initial_budget   │  1:N  │ tier             │
│ current_cost     ├───────┤ capabilities     │
│ cost_growth_pct  │       └──────────────────┘
│ initial_delivery │              │
│ current_delivery │              │ N:M
│ schedule_slip_pct│              │
│ status           │       ┌──────┴───────────┐
│ tech_maturity    │       │ program_suppliers │
│ created_at       │       ├──────────────────┤
│ updated_at       │       │ program_id (FK)  │
└──────┬───────────┘       │ supplier_id (FK) │
       │                   │ role (prime/sub)  │
       │ 1:N               │ contract_value   │
       │                   └──────────────────┘
┌──────┴───────────┐
│ budget_snapshots  │       ┌──────────────────┐
├──────────────────┤       │   risk_scores     │
│ id (PK)          │       ├──────────────────┤
│ program_id (FK)  │       │ id (PK)          │
│ fiscal_year      │       │ program_id (FK)  │
│ approved_budget  │       │ calculated_at    │
│ actual_spend     │       │ cost_growth_norm │
│ source           │       │ schedule_var_norm│
└──────────────────┘       │ supplier_conc    │
                           │ foreign_dep      │
                           │ tech_maturity_sc │
                           │ composite_pri    │
                           │ risk_tier        │
                           │ weights_used     │
                           └──────────────────┘
```

### 2.2 Table Definitions

```sql
-- Capability domains enum
CREATE TYPE capability_domain AS ENUM (
    'Land', 'Naval', 'Air', 'Cyber', 'Space', 'Joint'
);

-- Program status enum
CREATE TYPE program_status AS ENUM (
    'Definition', 'Implementation', 'Delayed', 'Delivered', 'Cancelled'
);

-- Technology maturity enum
CREATE TYPE tech_maturity AS ENUM (
    'Proven', 'Incremental', 'Developmental', 'Experimental'
);

CREATE TABLE programs (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    short_code      VARCHAR(20) UNIQUE,
    capability      capability_domain NOT NULL,
    description     TEXT,
    initial_budget  NUMERIC(14,2) NOT NULL,  -- CAD millions
    current_cost    NUMERIC(14,2) NOT NULL,
    cost_growth_pct NUMERIC(6,2) GENERATED ALWAYS AS (
        CASE WHEN initial_budget > 0
             THEN ((current_cost - initial_budget) / initial_budget) * 100
             ELSE 0 END
    ) STORED,
    initial_delivery DATE,
    current_delivery DATE,
    schedule_slip_months INTEGER GENERATED ALWAYS AS (
        EXTRACT(MONTH FROM AGE(current_delivery, initial_delivery))
        + EXTRACT(YEAR FROM AGE(current_delivery, initial_delivery)) * 12
    ) STORED,
    status          program_status DEFAULT 'Definition',
    tech_maturity   tech_maturity DEFAULT 'Proven',
    source_url      TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE suppliers (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    country         VARCHAR(100) NOT NULL,
    is_canadian     BOOLEAN GENERATED ALWAYS AS (country = 'Canada') STORED,
    headquarters    VARCHAR(255),
    tier            VARCHAR(50),  -- 'Prime', 'Tier 1', 'Tier 2'
    capabilities    TEXT[],
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE program_suppliers (
    id              SERIAL PRIMARY KEY,
    program_id      INTEGER REFERENCES programs(id) ON DELETE CASCADE,
    supplier_id     INTEGER REFERENCES suppliers(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL,  -- 'Prime', 'Subcontractor', 'Sub-tier'
    contract_value  NUMERIC(14,2),         -- CAD millions, NULL if unknown
    work_share_pct  NUMERIC(5,2),          -- % of contract value
    UNIQUE(program_id, supplier_id, role)
);

CREATE TABLE budget_snapshots (
    id              SERIAL PRIMARY KEY,
    program_id      INTEGER REFERENCES programs(id) ON DELETE CASCADE,
    fiscal_year     INTEGER NOT NULL,
    approved_budget NUMERIC(14,2),
    actual_spend    NUMERIC(14,2),
    source          VARCHAR(255),
    UNIQUE(program_id, fiscal_year)
);

CREATE TABLE risk_scores (
    id                  SERIAL PRIMARY KEY,
    program_id          INTEGER REFERENCES programs(id) ON DELETE CASCADE,
    calculated_at       TIMESTAMPTZ DEFAULT NOW(),
    cost_growth_norm    NUMERIC(5,4),  -- 0.0000 to 1.0000
    schedule_var_norm   NUMERIC(5,4),
    supplier_conc_norm  NUMERIC(5,4),
    foreign_dep_norm    NUMERIC(5,4),
    tech_maturity_score NUMERIC(5,4),
    composite_pri       NUMERIC(5,4),  -- Final Procurement Risk Index
    risk_tier           VARCHAR(20),   -- 'Low', 'Medium', 'High', 'Critical'
    weights_used        JSONB,         -- Store weight config for reproducibility
    UNIQUE(program_id, calculated_at)
);
```

---

## 3. Procurement Risk Index (PRI) — Engineering Model

### 3.1 Formula

```
PRI = w₁·C_norm + w₂·S_norm + w₃·SC_norm + w₄·FD_norm + w₅·TM_score
```

Where:
- `C_norm`  = Normalized cost growth factor
- `S_norm`  = Normalized schedule variance factor
- `SC_norm` = Normalized supplier concentration (HHI-based)
- `FD_norm` = Foreign dependency ratio
- `TM_score`= Technology maturity categorical score

### 3.2 Component Definitions

#### Cost Growth Factor (C_norm)
```
C_raw = (Current_Cost - Initial_Budget) / Initial_Budget × 100

C_norm = min(C_raw / C_max, 1.0)

C_max = 100%  (saturation point — growth beyond 100% still yields max score)
```

**Rationale**: Cost growth is the single most visible indicator of procurement distress. The 100% saturation point is derived from historical Canadian procurement data where programs exceeding this threshold (e.g., certain shipbuilding programs) were already in critical status.

#### Schedule Variance Factor (S_norm)
```
S_raw = (Current_Delivery - Initial_Delivery) in months

S_norm = min(S_raw / S_max, 1.0)

S_max = 120 months (10 years — saturation point)
```

**Rationale**: Defence procurements are long-cycle. A 10-year delay saturation aligns with the observation that programs exceeding this threshold are typically restructured or cancelled. Shorter programs should use a proportional scale: `S_max = max(120, planned_duration × 1.5)`.

#### Supplier Concentration (SC_norm) — Herfindahl-Hirschman Index
```
HHI = Σ(sᵢ²)

Where sᵢ = market share of supplier i (as decimal, 0–1)

SC_norm = min(HHI / 0.5, 1.0)
```

**Rationale**: HHI is the standard industrial concentration metric used by Competition Bureau Canada. An HHI of 0.25 indicates a monopoly equivalent (single supplier); 0.5 as saturation captures duopoly-or-worse scenarios. For programs with a single prime contractor (HHI = 1.0), the score saturates at 1.0, correctly flagging maximum concentration risk.

#### Foreign Dependency Ratio (FD_norm)
```
FD = Foreign_Contract_Value / Total_Contract_Value

FD_norm = FD  (already in [0,1] range)
```

**Rationale**: Direct ratio. A program where 100% of value flows to foreign suppliers scores 1.0. This aligns with ITB (Industrial and Technological Benefits) policy objectives but measures actual dependency, not policy compliance.

#### Technology Maturity Score (TM_score)
```
Categorical mapping:
  Proven       → 0.10
  Incremental  → 0.35
  Developmental→ 0.65
  Experimental → 0.90
```

**Rationale**: Based on a simplified Technology Readiness Level (TRL) grouping:
- Proven (TRL 8–9): Fielded systems with minor modifications
- Incremental (TRL 6–7): Demonstrated technology with integration risk
- Developmental (TRL 4–5): Lab-validated concepts requiring engineering
- Experimental (TRL 1–3): Basic research, high uncertainty

The non-linear spacing reflects the exponential increase in cost/schedule risk as TRL decreases (consistent with GAO and RAND findings on defence acquisition).

### 3.3 Weight Selection and Justification

| Factor              | Weight | Justification                                                   |
|---------------------|--------|-----------------------------------------------------------------|
| Cost Growth (w₁)    | 0.30   | Primary indicator of program health per PBO/AG reports          |
| Schedule Var (w₂)   | 0.25   | Strong correlation with cost growth; secondary but significant  |
| Supplier Conc (w₃)  | 0.20   | Structural risk — limits competitive pressure and alternatives  |
| Foreign Dep (w₄)    | 0.15   | Sovereignty risk — exposed to ITAR/EAR and foreign policy       |
| Tech Maturity (w₅)  | 0.10   | Foundational driver but partially captured in cost/schedule     |

**Sum of weights**: 0.30 + 0.25 + 0.20 + 0.15 + 0.10 = 1.00 ✓

**Rationale for ordering**: Weights derived from a review of Auditor General reports on defence procurement (2010–2024), where cost growth and schedule delays are consistently the top two findings. Supplier concentration is weighted third because it is a structural risk that compounds over time. Foreign dependency captures sovereignty-specific Canadian concerns. Technology maturity is weighted lowest because its effects propagate into cost and schedule (avoiding double-counting).

### 3.4 Risk Tier Classification

| PRI Range     | Tier     | Interpretation                                          |
|---------------|----------|---------------------------------------------------------|
| 0.00 – 0.25  | Low      | On track; standard management oversight sufficient      |
| 0.26 – 0.50  | Medium   | Emerging risks; increased monitoring recommended        |
| 0.51 – 0.75  | High     | Significant concerns; intervention analysis warranted   |
| 0.76 – 1.00  | Critical | Systemic issues; restructuring analysis recommended     |

### 3.5 Sensitivity Analysis Framework

For each program, compute:
```
∂PRI/∂wᵢ = Component_i_normalized_score

Tornado chart: Sort |∂PRI/∂wᵢ × Δwᵢ| for ±20% weight perturbation
```

This shows which risk factor most influences the composite score for each program, enabling targeted risk mitigation.

---

## 4. Data Pipeline Design

### 4.1 Pipeline Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Raw CSV /   │    │   Extract   │    │  Transform  │    │    Load     │
│  Manual Entry│───▶│  & Validate │───▶│ & Normalize │───▶│  to DB +   │
│              │    │             │    │             │    │  Calculate  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                   - Schema check     - Currency norm    - Upsert records
                   - Type coercion    - Date parsing     - Compute PRI
                   - Null handling    - HHI calculation  - Generate risk
                   - Dedup            - FD ratio         - Archive snapshot
```

### 4.2 Data Sources (Public/Open)

| Source                                    | Data Available                              | Format  | Update Frequency |
|-------------------------------------------|---------------------------------------------|---------|------------------|
| Defence Policy Review / SSE               | Program listings, budgets                   | PDF/Web | Annual           |
| Parliamentary Budget Officer              | Cost estimates, independent analysis        | PDF/CSV | Per-report       |
| PSPC Proactive Disclosure                 | Contract awards, supplier names             | CSV     | Quarterly        |
| DND Departmental Plans & Results          | Program status, milestones                  | PDF/Web | Annual           |
| Auditor General Reports                   | Audit findings, cost variances              | PDF     | Per-audit        |
| Status Report on Major Crown Projects     | Budget, schedule, status per program        | Web     | Annual           |

**Flag**: Most source data requires manual extraction from PDFs. A future enhancement would be an NLP-based PDF extraction pipeline.

### 4.3 Data Normalization Rules

- All monetary values in CAD millions (constant 2024 dollars where possible)
- Dates in ISO 8601 format
- Supplier names canonicalized (e.g., "GDLS-C" → "General Dynamics Land Systems - Canada")
- Program names matched to DND official nomenclature
- Missing values explicitly flagged as NULL with source annotation

---

## 5. Folder Structure

```
canadian-defence-procurement-analytics/
├── README.md
├── DESIGN_DOCUMENT.md
├── LICENSE
├── docker-compose.yml
├── .env.example
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── index.tsx
│   │   ├── components/
│   │   │   ├── ProgramTracker/
│   │   │   ├── RiskIndex/
│   │   │   ├── SupplierAnalysis/
│   │   │   └── BudgetAnalytics/
│   │   ├── hooks/
│   │   ├── types/
│   │   ├── utils/
│   │   └── data/
│   └── public/
│
├── backend/
│   ├── requirements.txt
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── models/
│   │   │   ├── program.py
│   │   │   ├── supplier.py
│   │   │   └── risk.py
│   │   ├── routers/
│   │   │   ├── programs.py
│   │   │   ├── risk.py
│   │   │   ├── suppliers.py
│   │   │   └── budget.py
│   │   ├── services/
│   │   │   ├── risk_engine.py
│   │   │   ├── hhi_calculator.py
│   │   │   └── monte_carlo.py
│   │   └── schemas/
│   │       └── responses.py
│   └── tests/
│
├── data/
│   ├── raw/
│   ├── processed/
│   └── pipeline/
│       ├── ingest.py
│       ├── clean.py
│       ├── normalize.py
│       └── load.py
│
├── docs/
│   ├── architecture.md
│   ├── risk_model.md
│   ├── data_dictionary.md
│   └── executive_summary.md
│
└── scripts/
    ├── seed_db.py
    └── generate_report.py
```

---

## 6. Modeling Assumptions — Explicit Registry

| ID   | Assumption                                                                 | Impact   | Mitigation                          |
|------|----------------------------------------------------------------------------|----------|-------------------------------------|
| A-01 | Budget figures are in nominal CAD unless otherwise stated                  | Medium   | Apply GDP deflator when available   |
| A-02 | Schedule dates use DND's publicly stated forecasts                        | Low      | Cross-reference PBO where available |
| A-03 | Supplier contract values may be incomplete (redacted for security)         | High     | Flag missing data; use partial HHI  |
| A-04 | Technology maturity is assessed qualitatively from public descriptions     | Medium   | Document rationale per program      |
| A-05 | PRI weights are fixed; real system would allow user-adjustable weights     | Medium   | Sensitivity analysis compensates    |
| A-06 | Foreign dependency uses contract value, not production location            | Low      | ITB offsets are not modeled         |
| A-07 | HHI is calculated at the prime contractor level only                      | Medium   | Sub-tier data often unavailable     |
| A-08 | Monte Carlo cost distributions assume log-normal (standard in cost est.)  | Low      | Industry standard assumption        |

---

*Document version: 1.0 | Author: Systems Engineering Team | Classification: UNCLASSIFIED*
