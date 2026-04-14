# Contributing to CDPEAD

## How to Contribute

### Data Contributions

The highest-impact contributions are data updates from public sources. If you're adding or updating program data:

1. Include the source URL or document reference
2. Note the extraction date
3. Flag any assumptions or interpretations
4. Update the `source_note` field for affected programs

### Code Contributions

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Make your changes
4. Test locally (both frontend and backend)
5. Submit a pull request with a clear description

### Priority Areas

- Manual data extraction from PBO and AG reports
- Vendor registry expansion (new vendors, name variations)
- Additional procurement program entries
- Test coverage improvements
- Documentation updates

## Development Setup

```bash
# Backend
cd backend
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
python scripts/seed_db.py
uvicorn app.main:app --reload --port 8080

# Frontend
cd frontend
npm install
npm start
```

## Data Standards

- All monetary values in CAD millions
- Dates in ISO 8601 format (YYYY-MM-DD)
- Supplier names must match the vendor registry (or add a new entry)
- Every data point must have a cited public source
- No classified, restricted, or protected information

## Code Standards

- Python: Follow PEP 8, type hints encouraged
- JavaScript/React: Functional components, hooks pattern
- All risk calculations must match between frontend (riskEngine.js) and backend (risk_engine.py)
- New features should include tests
