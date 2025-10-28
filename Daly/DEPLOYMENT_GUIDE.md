# SACRRA T702 Deployment (Windows Server)

This guide walks you through deploying the SACRRA T702 generator onto the Excalibur (DALY-BI) Windows server. Python 3.10+ is already installed on the host.

## 1. Prepare the server workspace
1. Sign in to the server with an account that can copy files into `C:\sacrra_t702`.
2. Create the application directory if it does not exist:
   ```cmd
   mkdir C:\sacrra_t702
   ```
3. Copy the following folders and files from source control or your build drop into `C:\sacrra_t702`:
   - `sacrra_t702/`
   - `config/`
   - `Docs/` (optional reference specifications)
   - `Schemas/` and `Specs/` if you maintain them alongside code
   - `samples/` (useful for smoke tests without database access)
   - `requirements.txt`
   - `README.md`
   - `OPERATIONS_GUIDE.md`

   You can use `robocopy` from your workstation (replace `\share\path` with your source):
   ```cmd
   robocopy "\\share\path\sacrra_t702" "\\DALY-BI\C$\sacrra_t702\sacrra_t702" /E /R:3 /W:5
   robocopy "\\share\path\config" "\\DALY-BI\C$\sacrra_t702\config" /E /R:3 /W:5
   robocopy "\\share\path\Docs" "\\DALY-BI\C$\sacrra_t702\Docs" /E /R:3 /W:5
   robocopy "\\share\path\samples" "\\DALY-BI\C$\sacrra_t702\samples" /E /R:3 /W:5
   copy "\\share\path\requirements.txt" "\\DALY-BI\C$\sacrra_t702\"
   copy "\\share\path\README.md" "\\DALY-BI\C$\sacrra_t702\"
   copy "\\share\path\OPERATIONS_GUIDE.md" "\\DALY-BI\C$\sacrra_t702\"
   ```

## 2. Create an isolated Python environment
From an elevated **Command Prompt** on the server:
```cmd
cd C:\sacrra_t702
python -m venv .venv
.\.venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

> Tip: Keep the virtual environment under `C:\sacrra_t702\.venv` so scheduled tasks can activate it easily.

## 3. Configure database connectivity
Create or update the environment variables that the generator reads. You can set them at the **System** level (recommended) or in the scheduled task definition.

```cmd
setx DB_SERVER "DALY-BI.DCC.CORP"
setx DB_DATABASE "ExcaliburV4"
setx DB_USERNAME "<sql-login>"
setx DB_PASSWORD "<sql-password>"
setx SQLSERVER_DRIVER "ODBC Driver 18 for SQL Server"
```

If you prefer a DSN, set `SQLSERVER_DSN` instead of individual server credentials.

After setting system variables, open a fresh Command Prompt (variables are cached per session) and activate the virtual environment.

## 4. Smoke test the deployment
Run a lightweight command to verify connectivity and rendering:
```cmd
cd C:\sacrra_t702
.\.venv\Scripts\activate
python -m sacrra_t702.cli generate ^
  --mode monthly ^
  --run-date 20250131 ^
  --config config\default.yaml ^
  --source db ^
  --output-dir out ^
  --limit 10
```

Expected results:
- The command finishes without errors.
- A manifest JSON file and sample output file appear in `C:\sacrra_t702\out`.

For offline smoke tests, swap `--source csv --input samples\sample_monthly.csv`.

## 5. Prepare for operations
1. Review `OPERATIONS_GUIDE.md` for day-to-day run books (manual and scheduled execution).
2. Grant write permissions on `C:\sacrra_t702\out` for the service account that will run scheduled tasks.
3. Capture the exact virtual environment activation path (`C:\sacrra_t702\.venv\Scripts\python.exe`) for scheduling.

## 6. Post-deployment housekeeping
- Store a zipped copy of the deployed artefacts and the configuration used.
- Document the credential owner responsible for rotating the SQL password.
- Add the deployment location to your infrastructure inventory.
- Monitor disk space on `C:\` because T702 outputs can be large.

## Troubleshooting tips
| Symptom | Suggested action |
|---------|------------------|
| `pyodbc` import errors | Confirm that the "ODBC Driver 18 for SQL Server" is installed. Download from Microsoft if missing. |
| Authentication failures | Test connectivity with `Test-ODBCConnection` (PowerShell) or `sqlcmd`. Verify firewall and SQL logins. |
| Output directory permission issues | Ensure the executing user (service account) has Modify rights on `C:\sacrra_t702\out`. |
| Virtual environment not activating in scheduled task | Use the absolute path to `python.exe` inside `.venv\Scripts` in the task action, and specify `C:\sacrra_t702` as the Start in directory. |

With these steps the Windows server is ready for regular SACRRA T702 file generation.
