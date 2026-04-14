# Executive Summary — CDPEAD v1.0

## Canadian Defence Procurement Engineering Analytics Dashboard

---

**Problem:** Canada's defence procurement portfolio — currently valued at over $135B across 12+ major programs — lacks a unified, transparent engineering analytics tool for systematic risk assessment. Cost growth averaging 55%+ across the portfolio and schedule delays averaging 40+ months remain difficult to quantify, compare, and act upon without structured analytical infrastructure.

**Solution:** CDPEAD is an open-source engineering analytics platform that provides quantitative, model-driven insight into procurement program performance, supplier dependency risk, and budget allocation patterns. It uses a transparent Procurement Risk Index (PRI) built on five normalized, weighted factors with fully documented mathematical justification.

**Key Findings (from current dataset):**
- Portfolio-level average cost growth exceeds 55%, driven primarily by the Canadian Surface Combatant program (~195% growth from original estimates)
- Naval programs represent the largest share of both spending and cost growth
- Supplier concentration is elevated: Irving Shipbuilding and Lockheed Martin account for a significant share of total portfolio value
- Foreign dependency varies widely — from ~0% (DCOMM) to ~82% (FFCP) — with sovereignty implications
- The PRI model identifies CSC, JSS, and ACSV as the highest-risk programs when all five factors are weighted

**For Whom:** DND technical authorities, DGLEPM project officers, defence systems engineers, Parliamentary Budget Office analysts, and parliamentary committee technical staff.

**Not For:** Political commentary, media consumption, or public opinion shaping. This is an engineering tool.

---

# 5-Slide Technical Pitch Outline

## Slide 1 — Problem Statement
- Canada manages $135B+ in active defence procurement
- No unified analytical tool exists for systematic risk quantification
- AG and PBO reports flag the same issues repeatedly with no systemic tracking
- Decision-makers lack engineering-grade decision support

## Slide 2 — Solution Architecture
- Full-stack analytics platform: React → FastAPI → PostgreSQL
- Modular design: Program Tracker, Risk Engine, Supplier Analysis, Budget Analytics
- All data from public/open Canadian sources
- Designed for local deployment with cloud-ready architecture

## Slide 3 — Procurement Risk Index Model
- Composite weighted formula: PRI = w₁C + w₂S + w₃SC + w₄FD + w₅TM
- HHI-based supplier concentration (Competition Bureau standard)
- Technology maturity mapping from TRL groups
- Interactive weight adjustment with real-time sensitivity analysis
- Every assumption documented and justifiable

## Slide 4 — Key Analytical Outputs
- Cost growth vs. schedule slip scatter analysis (bubble = portfolio size)
- Risk tier distribution across portfolio (Low/Medium/High/Critical)
- Canadian content vs. foreign dependency by program
- Per-program radar decomposition of risk factors
- Domain-level budget allocation and growth trends

## Slide 5 — Roadmap & Value Proposition
- Stretch: Monte Carlo simulation, scenario modeling, PDF export
- Future: NLP-based PDF extraction for automated data refresh
- Value: Transparent, reproducible, engineering-grade analysis
- Differentiator: Not a media dashboard — an engineer's decision-support system
