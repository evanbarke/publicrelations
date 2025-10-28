# SACRRA T702 Operations Guide

This document is written for Daly Credit Solutions stakeholders who run or maintain the SACRRA T702 monthly and daily extracts. It covers manual generation, automation, monitoring, and how to accommodate format changes.

---

## 1. Overview of file types

| Mode | Purpose | Record types produced | Typical schedule |
|------|---------|-----------------------|------------------|
| Monthly (`--mode monthly`) | Full portfolio exposure snapshot for SACRRA | Header (H), Detail (D), Trailer (T) | Business day following month end |
| Daily (`--mode daily`) | Paid-up transitions since last run | Detail (D). Optional header/trailer when requested. | Weekdays, early morning |

All runs create fixed-width output files and a JSON manifest summarising counts and hashes. Files are written to the directory specified by `--output-dir` (defaults to `out/`).

---

## 2. Manual file generation

1. Sign in to the DALY-BI server (or a workstation with VPN access and the deployment).
2. Open **Command Prompt** and activate the project environment:
   ```cmd
   cd C:\sacrra_t702
   .\.venv\Scripts\activate
   ```
3. Run one of the commands below.

### Monthly run example
```cmd
python -m sacrra_t702.cli generate ^
  --mode monthly ^
  --run-date 20250131 ^
  --config config\default.yaml ^
  --source db ^
  --output-dir out
```
- `--run-date` must be the reporting date in `CCYYMMDD` format.
- Use `--clients all` to include every client; omit to stay on the WFS portfolios only.
- Add `--limit 1000` to test on a smaller sample.

### Daily run example
```cmd
python -m sacrra_t702.cli generate ^
  --mode daily ^
  --run-date 20250131 ^
  --config config\default.yaml ^
  --source db ^
  --output-dir out
```
- Daily output is Detail-only by default. Add `--include-header-trailer-daily` if SACRRA requests header/trailer records.

When the command completes, review the manifest (e.g. `out/SRN_WFS_T702_M_20250131_01_01_manifest.json`) for counts and hashes.

---

## 3. Scheduling with Windows Task Scheduler

1. Launch **Task Scheduler** → *Create Task*.
2. **General tab**
   - Name: `SACRRA T702 Monthly`
   - Security options: *Run whether user is logged on or not*; use the service account with SQL permissions.
3. **Triggers tab**
   - Monthly job: run on the 1st business day at 06:00. Use “Monthly” trigger and tick all months. Add a task delay if upstream data loads finish later.
   - Daily job: create a separate task triggered daily at 05:00 (Mon–Fri).
4. **Actions tab**
   - Action: *Start a program*
   - Program/script: `C:\sacrra_t702\.venv\Scripts\python.exe`
   - Add arguments (monthly example):
     ```
     -m sacrra_t702.cli generate --mode monthly --run-date $(Get-Date -Format yyyyMMdd) --config C:\sacrra_t702\config\default.yaml --source db --output-dir C:\sacrra_t702\out
     ```
     Replace the PowerShell expression by a fixed date when testing; for production use a wrapper `.cmd` or PowerShell script to calculate the correct reporting date.
   - Start in: `C:\sacrra_t702`
5. **Conditions/Settings**
   - Uncheck *Stop the task if it runs longer than* to prevent premature termination on large files.
   - Enable *Run task as soon as possible after a scheduled start is missed*.

> Tip: For monthly reporting, create a short PowerShell wrapper (e.g. `scripts\run_monthly.ps1`) that computes the month-end date and calls the CLI. Point the scheduled task to `powershell.exe -File ...`.

---

## 4. Monitoring and validation checklist

- Confirm that the output directory contains both the `.txt` extract and `.json` manifest for every run.
- Cross-check record counts in the manifest against SACRRA acknowledgements.
- Archive generated files to secure storage (e.g. network share with date-based folders).
- Track task history in Task Scheduler and enable email/Teams alerts for failures.
- Maintain a run log spreadsheet capturing run date, runtime, record counts, and issues encountered.

---

## 5. Handling format or mapping changes

1. **Layout definition** – Update `sacrra_t702/layouts/t702_layout.yaml`. This file controls field order, widths, and padding. After editing:
   - Validate YAML syntax with `python -m py_compile` or by running a small generation.
   - Use `pytest` (see README) to ensure writer tests still pass.
2. **Transform rules** – Apply business logic updates in `sacrra_t702/transform.py`. Keep functions pure where possible and add unit tests in `tests/` to document expectations.
3. **Configuration mapping** – Adjust lookup tables or naming tokens in `config/default.yaml`.
4. **Database extraction** – Modify SQL queries in `sacrra_t702/data_access.py`. Coordinate changes with DBAs and ensure indexes support the query shape.
5. **Regression testing**
   - Generate sample files using `samples/sample_monthly.csv` before pointing at production data.
   - Compare field-level deltas against SACRRA specifications or previous outputs.

Document any approved format change in your change log and communicate the effective date to SACRRA.

---

## 6. Support contacts and escalation

| Area | Contact | Notes |
|------|---------|-------|
| Data issues (missing/incorrect fields) | Data engineering lead | Provide manifest and sample record IDs |
| Connectivity/authentication | Infrastructure / DBA team | Reference the `DB_*` environment variables |
| Code defects | Development team | Open a ticket with logs, manifest, and reproduction steps |

Keep this guide with the deployment artefacts so new operators can onboard quickly.
