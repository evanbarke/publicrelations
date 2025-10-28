### SACRRA T702 Reporting (Daly – WFS Portfolios)

This repository implements a configurable generator for SACRRA-compliant T702 reporting files for Daly Credit Solutions (WFS portfolios). It supports monthly full-exposure files and daily paid-up files, with validation, reconciliation, deterministic fixed-width output, and manifests.

#### Features
- Config-driven file naming and portfolio selection
- Fixed-width writer with layout defined in YAML (`T702` layout scaffolded; extend as mapping finalizes)
- Monthly (H/D/T) and Daily (D-only) modes
- Basic validations and reconciliation (record counts, optional numeric hash)
- Manifests for each run (counts, hashes, inputs)
- Sample data to run locally without database access


#### Documentation
- `DEPLOYMENT_GUIDE.md`: Windows deployment steps for DALY-BI.
- `OPERATIONS_GUIDE.md`: Run book for monthly/daily generation and scheduling.

#### Quickstart
1) Python 3.10+

2) Create a virtual environment and install dependencies:
```
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

3) Generate a monthly file from sample data (WFS clients only, default):
```
python -m sacrra_t702.cli generate \
  --mode monthly \
  --run-date 20250131 \
  --config config/default.yaml \
  --input samples/sample_monthly.csv \
  --output-dir out
```

4) Generate a monthly file for all clients (not just WFS):
```
python -m sacrra_t702.cli generate \
  --mode monthly \
  --run-date 20250131 \
  --config config/default.yaml \
  --input samples/sample_monthly.csv \
  --output-dir out \
  --clients all
```

5) Generate a daily file from sample data (paid-up only):
```
python -m sacrra_t702.cli generate \
  --mode daily \
  --run-date 20250131 \
  --config config/default.yaml \
  --input samples/sample_daily.csv \
  --output-dir out
```

Output files follow the pattern `[MemberCode]_[Portfolio]_[Layout]_[Frequency]_[CCYYMMDD]_[Version]_[SubVersion].txt` and are written to `out/` by default, alongside a JSON manifest.

#### Client Filtering
The generator supports filtering between WFS clients only or all clients:

- **WFS clients only (default)**: `--clients wfs` - Filters for Client IDs 900359 (WFS Visa AD), 900360 (WFS Loans AD), 900361 (WFS Storecard AD)
- **All clients**: `--clients all` - Includes all clients in the database

When using `--source db`, the filtering is applied at the SQL level for better performance. When using `--source csv`, the filtering is applied during processing.

#### Configuration
- `config/default.yaml` controls:
  - Member/Portfolio tokens for naming
  - Versioning and environment
  - Account type lookups and status mappings
  - Layout options and numeric formatting

Create a `.env` (see `.env.example`) or export env vars to provide DB credentials when connecting to live systems. The generator can run from CSV inputs while database integration is being finalized.

#### Layout
- The fixed-width column model is defined in `sacrra_t702/layouts/t702_layout.yaml`. It now separates `header`, `detail`, and `trailer` schemas. This is a v0 scaffold and must be aligned to: “20150608_Layout 700v2 for Evolution v 2 8 for Member” and addendums (L702 Prescription, Adverse Removal, NCR guideline). Update widths, positions, and conditionality (P/R/Z) as the mapping workbook matures.

#### Database Integration
- The `sacrra_t702/data_access.py` module abstracts data sourcing. It supports:
  - CSV inputs (for local testing)
  - SQL Server via `pyodbc` (env vars: `DB_SERVER`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `SQLSERVER_DSN`, optional `SQLSERVER_DRIVER`).

Populate SQL views per spec (Parties, Accounts, Balances, Transactions, StatusHistory, Assignments, Contacts, Addresses) and point the generator at those views with configurable SQL.

Run monthly from DB (WFS clients only, default):
```
. .venv/bin/activate
export DB_SERVER=... DB_DATABASE=... DB_USERNAME=... DB_PASSWORD=...
python -m sacrra_t702.cli generate \
  --mode monthly \
  --run-date 20250131 \
  --config config/default.yaml \
  --source db \
  --output-dir out
```

Run monthly from DB for all clients:
```
. .venv/bin/activate
export DB_SERVER=... DB_DATABASE=... DB_USERNAME=... DB_PASSWORD=...
python -m sacrra_t702.cli generate \
  --mode monthly \
  --run-date 20250131 \
  --config config/default.yaml \
  --source db \
  --output-dir out \
  --clients all
```

Run daily from DB (paid-up transitions, WFS clients only):
```
. .venv/bin/activate
export DB_SERVER=... DB_DATABASE=... DB_USERNAME=... DB_PASSWORD=...
python -m sacrra_t702.cli generate \
  --mode daily \
  --run-date 20250131 \
  --config config/default.yaml \
  --source db \
  --output-dir out
```

#### Testing
Run the unit tests for the fixed-width writer and naming utilities:
```
pytest -q
```

#### Project Structure
```
config/
  default.yaml
sacrra_t702/
  __init__.py
  cli.py
  config.py
  data_access.py
  generator.py
  transform.py
  validate.py
  utils.py
  writer.py
  layouts/
    t702_layout.yaml
samples/
  sample_monthly.csv
  sample_daily.csv
tests/
  test_writer.py
requirements.txt
README.md
```



