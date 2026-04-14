# Data Dictionary — CDPEAD v1.0

## Programs Table

| Field | Type | Description | Source | Example |
|-------|------|-------------|--------|---------|
| `id` | Integer (PK) | Auto-increment primary key | System | 1 |
| `name` | String(255) | Official DND program name | DND DPR | "Canadian Surface Combatant" |
| `short_code` | String(20) | Program abbreviation (unique) | DND | "CSC" |
| `capability` | Enum | Capability domain | DND SSE | Land, Naval, Air, Cyber, Space, Joint |
| `description` | Text | Brief program description | DND DPR | "15 combat vessels to replace..." |
| `initial_budget` | Decimal(14,2) | Original approved budget (CAD millions) | DND/PBO | 26200.00 |
| `current_cost` | Decimal(14,2) | Latest cost estimate (CAD millions) | PBO/DND | 77300.00 |
| `cost_growth_pct` | Decimal(6,2) | Computed: ((current - initial) / initial) × 100 | Calculated | 195.04 |
| `initial_delivery` | Date | Originally planned IOC/delivery date | DND DPR | 2031-01-01 |
| `current_delivery` | Date | Current forecast delivery date | DND DPR | 2035-01-01 |
| `schedule_slip_months` | Integer | Computed: months between initial and current delivery | Calculated | 48 |
| `status` | Enum | Current program phase | DND DPR | Definition, Implementation, Delayed, Delivered, Cancelled |
| `tech_maturity` | Enum | Technology readiness classification | Assessment | Proven, Incremental, Developmental, Experimental |
| `source_url` | Text | URL of primary data source | Manual | "https://www.pbo-dpb.ca/..." |
| `notes` | Text | Additional context or caveats | Manual | "PBO 2023 cost estimate" |
| `created_at` | Timestamp | Record creation time | System | 2025-01-01T00:00:00Z |
| `updated_at` | Timestamp | Last modification time | System | 2025-06-15T12:00:00Z |

## Suppliers Table

| Field | Type | Description | Source | Example |
|-------|------|-------------|--------|---------|
| `id` | Integer (PK) | Auto-increment primary key | System | 1 |
| `name` | String(255) | Canonical supplier name | PSPC/Manual | "Irving Shipbuilding" |
| `country` | String(100) | Country of headquarters | Manual | "Canada" |
| `is_canadian` | Boolean | Computed: country == "Canada" | Calculated | true |
| `headquarters` | String(255) | HQ city/province | Manual | "Halifax, NS" |
| `tier` | String(50) | Supplier classification | Manual | "Prime", "Tier 1", "Tier 2" |
| `capabilities` | Text[] | Areas of expertise | Manual | ["Shipbuilding", "Naval Architecture"] |

## Program-Supplier Mapping Table

| Field | Type | Description | Source | Example |
|-------|------|-------------|--------|---------|
| `id` | Integer (PK) | Auto-increment primary key | System | 1 |
| `program_id` | Integer (FK) | Reference to programs.id | System | 1 |
| `supplier_id` | Integer (FK) | Reference to suppliers.id | System | 1 |
| `role` | String(50) | Supplier role in program | PSPC/Manual | "Prime", "Combat Systems Integrator" |
| `contract_value` | Decimal(14,2) | Contract value (CAD millions), NULL if unknown | PSPC | 11790.00 |
| `work_share_pct` | Decimal(5,2) | Percentage of total program value | Estimated | 45.00 |

## Budget Snapshots Table

| Field | Type | Description | Source | Example |
|-------|------|-------------|--------|---------|
| `id` | Integer (PK) | Auto-increment primary key | System | 1 |
| `program_id` | Integer (FK) | Reference to programs.id | System | 1 |
| `fiscal_year` | Integer | Canadian fiscal year (ending March 31) | DND/PBO | 2024 |
| `approved_budget` | Decimal(14,2) | Approved budget for that FY (CAD millions) | DND DPR | 2500.00 |
| `actual_spend` | Decimal(14,2) | Actual expenditure (CAD millions) | GC InfoBase | 2100.00 |
| `source` | String(255) | Data source reference | Manual | "GC InfoBase FY2023-24" |

## Risk Scores Table

| Field | Type | Description | Source | Example |
|-------|------|-------------|--------|---------|
| `id` | Integer (PK) | Auto-increment primary key | System | 1 |
| `program_id` | Integer (FK) | Reference to programs.id | System | 1 |
| `calculated_at` | Timestamp | When this score was computed | System | 2025-06-15T12:00:00Z |
| `cost_growth_norm` | Decimal(5,4) | Normalized cost growth (0–1) | Calculated | 1.0000 |
| `schedule_var_norm` | Decimal(5,4) | Normalized schedule variance (0–1) | Calculated | 0.4000 |
| `supplier_conc_norm` | Decimal(5,4) | Normalized HHI (0–1) | Calculated | 0.5450 |
| `foreign_dep_norm` | Decimal(5,4) | Foreign dependency ratio (0–1) | Calculated | 0.3500 |
| `tech_maturity_score` | Decimal(5,4) | Technology maturity score (0–1) | Calculated | 0.6500 |
| `composite_pri` | Decimal(5,4) | Final Procurement Risk Index (0–1) | Calculated | 0.6235 |
| `risk_tier` | String(20) | Classification tier | Calculated | "Low", "Medium", "High", "Critical" |
| `weights_used` | JSON | Weight configuration at time of calculation | System | {"costGrowth": 0.30, ...} |

## Enumerations

### capability_domain
`Land` | `Naval` | `Air` | `Cyber` | `Space` | `Joint`

### program_status
`Definition` | `Implementation` | `Delayed` | `Delivered` | `Cancelled`

### tech_maturity
| Value | TRL Equivalent | PRI Score |
|-------|---------------|-----------|
| `Proven` | TRL 8–9 | 0.10 |
| `Incremental` | TRL 6–7 | 0.35 |
| `Developmental` | TRL 4–5 | 0.65 |
| `Experimental` | TRL 1–3 | 0.90 |

## Currency and Units

- All monetary values are in **Canadian Dollars (CAD), millions**
- Budget figures are **nominal** unless explicitly stated as constant-year dollars
- Dates use **ISO 8601** format (YYYY-MM-DD)
- Schedule slip is measured in **calendar months**
- HHI values range from **0 (perfect competition) to 1 (monopoly)**
- PRI values range from **0 (no risk) to 1 (maximum risk)**

## Data Provenance

Every record should include:
- Source URL or document reference
- Extraction date
- Confidence level (where applicable)
- Notes on any assumptions or interpretations made during data entry
