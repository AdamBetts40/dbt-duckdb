# VS Code + dbt + DuckDB Setup Guide

Assumes Windows 10/11. macOS/Linux differences are noted where they matter. DuckDB runs in-process, so there is no server, ODBC driver or cloud credentials to set up.

---

## 1. Prerequisites checklist

| Tool | Purpose |
|---|---|
| VS Code | Editor |
| Python 3.10 – 3.12 | Runs dbt and DuckDB |
| Git | Version control |

---

## 2. Install VS Code

1. Download from <https://code.visualstudio.com/> (System Installer, or User Installer if you lack admin rights).
2. Tick **Add "Open with Code" to context menus** and **Add to PATH**.

Or:

```powershell
winget install Microsoft.VisualStudioCode
```

---

## 3. Install Python

1. Download from <https://www.python.org/downloads/> (3.11 or 3.12 recommended).
2. Tick **"Add python.exe to PATH"** on the first installer screen.
3. Verify in a new terminal:

```powershell
python --version
pip --version
```

Or: `winget install Python.Python.3.12`

---

## 4. Install Git

```powershell
winget install Git.Git
git --version
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## 5. Install VS Code extensions

### Minimum (recommended for the demo)

| Extension | ID | Why |
|---|---|---|
| Python | `ms-python.python` | Lets VS Code find your `.venv` and run Python (Pylance is bundled with it) |
| Power User for dbt | `innoverio.vscode-dbt-power-user` | Run/compile buttons, model previews, lineage and autocomplete |

```powershell
code --install-extension ms-python.python
code --install-extension innoverio.vscode-dbt-power-user
```

> Power User for dbt is optional if you're happy running everything from the terminal, but it makes the demo much smoother.

### Optional extras

Install these later if you need them.

| Extension | ID | Why |
|---|---|---|
| SQLFluff | `dorzey.vscode-sqlfluff` | SQL linting/formatting. Needs its own config, so best for team projects with style rules |
| YAML | `redhat.vscode-yaml` | Schema validation and hints for `.yml` files |
| GitLens | `eamodio.gitlens` | Extra Git history and blame features beyond the built-in Git support |

```powershell
code --install-extension dorzey.vscode-sqlfluff
code --install-extension redhat.vscode-yaml
code --install-extension eamodio.gitlens
```

---

## 6. Create the project folder and virtual environment

```powershell
mkdir dbt-duckdb-demo
cd dbt-duckdb-demo
python -m venv .venv
.venv\Scripts\Activate.ps1
```

> macOS/Linux: `source .venv/bin/activate`

If PowerShell blocks activation, run once:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Open the folder in VS Code and select the interpreter:

```powershell
code .
```

`Ctrl+Shift+P` → **Python: Select Interpreter** → choose `.venv`.

---

## 7. Install dbt and DuckDB

With the venv active:

```powershell
python -m pip install --upgrade pip
pip install dbt-core dbt-duckdb
dbt --version
```

`dbt-duckdb` pulls in the `duckdb` Python package. Save your dependencies:

```powershell
pip freeze > requirements.txt
```

**Optional:** install the DuckDB CLI for querying the database file directly:

```powershell
winget install DuckDB.cli
```

---

## 8. Initialise the dbt project

```powershell
dbt init dbt_duckdb_demo
```

Choose `duckdb` when prompted for the adapter. This scaffolds the project folders and creates a profile in `C:\Users\<you>\.dbt\profiles.yml`.

> To keep the profile inside the project instead, create `profiles.yml` in the project root and run dbt with `--profiles-dir .`, or set the `DBT_PROFILES_DIR` environment variable.

---

## 9. Configure `profiles.yml`

```yaml
dbt_duckdb_demo:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: dev.duckdb
      threads: 4
```

Notes:

- `path` is a local file that DuckDB creates on first run. Use `:memory:` for a throwaway in-memory database.
- To read from external files (Parquet, CSV) or use extensions, add them in the profile:

```yaml
      extensions:
        - httpfs
        - parquet
```

- If you keep `profiles.yml` inside the repo, don't put secrets in it.

---

## 10. Test the connection

```powershell
cd dbt_duckdb_demo
dbt debug
```

You want **"All checks passed!"**. Common failures:

| Error | Fix |
|---|---|
| `dbt` not recognised | Venv isn't active (Section 6) |
| Profile not found | Profile name in `dbt_project.yml` must match the top-level key in `profiles.yml` |
| Database file locked | Only one process can write to a `.duckdb` file at a time. Close the CLI or any other connection |

---

## 11. Run the starter models

```powershell
dbt run
dbt test
```

Check the output by querying the database file:

```powershell
duckdb dev.duckdb "SHOW ALL TABLES;"
```

Or in Python:

```python
import duckdb
con = duckdb.connect("dev.duckdb")
print(con.sql("SHOW ALL TABLES").df())
```

---

## 12. Git setup

```powershell
git init
```

Create a `.gitignore`:

```
.venv/
target/
dbt_packages/
logs/
__pycache__/
*.pyc
*.duckdb
*.duckdb.wal
.env
```

Initial commit:

```powershell
git add .
git commit -m "Initial dbt + DuckDB scaffold"
```

---

## 13. Recommended VS Code settings

`Ctrl+Shift+P` → **Preferences: Open Workspace Settings (JSON)**:

```json
{
  "python.defaultInterpreterPath": ".venv\\Scripts\\python.exe",
  "files.associations": {
    "*.sql": "jinja-sql",
    "*.yml": "jinja-yaml"
  },
  "editor.rulers": [120],
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true
}
```

---

## 14. Final verification

- [ ] `code --version` works
- [ ] `python --version` shows 3.10–3.12
- [ ] `.venv` is selected as the interpreter in VS Code
- [ ] `dbt --version` lists `dbt-core` and the `duckdb` plugin
- [ ] `dbt debug` → all checks passed
- [ ] `dbt run` creates tables in `dev.duckdb`
- [ ] Project is under Git with `*.duckdb` ignored

You're now ready for the next step.
