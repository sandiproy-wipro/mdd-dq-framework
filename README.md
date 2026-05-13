# ⚡ DQ Framework — Databricks Notebook Version

A config-driven Data Quality framework built **natively for Databricks notebooks**.
No local Python packages. No `src/` imports. No pytest. Just `%run` and go.

---

## Project structure

```
dq-framework-notebooks/
│
├── notebooks/
│   ├── 00_dq_controller.py        ← ★ RUN THIS — orchestrates everything
│   ├── 01_dq_config_validator.py  ← config loading + validation logic
│   ├── 02_dq_rule_engine.py       ← all 16 rule type implementations
│   ├── 03_dq_dataset_loader.py    ← Delta / Parquet / CSV / JDBC loader
│   ├── 04_dq_results_writer.py    ← Delta persistence + run summary
│   └── 05_dq_tests.py             ← self-contained test suite (Run All to test)
│
└── config/
    ├── master_config.yml           ← central dataset registry
    └── datasets/
        ├── orders.yml              ← rules for the orders dataset
        ├── customers.yml
        ├── products.yml
        └── transactions.yml        ← disabled in master_config
```

---

## Quick start

### 1. Import into Databricks

**Option A — Git Repo (recommended)**
In your Databricks workspace go to **Repos → Add Repo** and clone this repository.
All notebooks and YAML configs will be available at:
```
/Workspace/Repos/<your-name>/dq-framework-notebooks/
```

**Option B — Upload manually**
Upload the `notebooks/` folder to a Workspace folder and the `config/` folder to
DBFS (`/dbfs/FileStore/dq-framework/config/`) or a Unity Catalog Volume.

### 2. Open the controller notebook

Open `notebooks/00_dq_controller.py` in your Databricks workspace.

### 3. Set the widget values

| Widget | Value to set |
|---|---|
| `repo_root` | `/Workspace/Repos/your-name/dq-framework-notebooks` |
| `master_config_path` | `config/master_config.yml` |
| `dataset_filter` | leave empty to run all datasets |
| `dry_run` | `false` |
| `abort_on_critical` | `true` |
| `results_table` | `main.dq_framework.dq_results` |
| `alerts_table` | `main.dq_framework.dq_alerts` |

### 4. Click Run All

The controller will:
1. Install `pyyaml`
2. `%run` the four helper notebooks
3. Validate all configs (aborts if any are broken)
4. Load each dataset as a Spark DataFrame
5. Run all DQ rules
6. Write results to Delta
7. Print a summary and fail the job if CRITICAL rules failed

---

## How `%run` links the notebooks

```
00_dq_controller
    %run 01_dq_config_validator   → defines load_all_configs(), DQConfigError, ...
    %run 02_dq_rule_engine        → defines RuleEngine, DQRuleResult
    %run 03_dq_dataset_loader     → defines load_dataset(), DatasetLoaderError
    %run 04_dq_results_writer     → defines ResultsWriter
```

All symbols become available in the controller's scope after each `%run`.
No imports of local modules or `sys.path` manipulation required.

---

## Config validation — what gets caught before data is touched

| Violation | Error | Behaviour |
|---|---|---|
| Duplicate dataset names in master_config | `DQConfigError` | Abort immediately |
| `config_file` path not found on disk | `DQDatasetConfigError` | Collected, all reported |
| `dataset.name` ≠ name in master_config | `DQDatasetConfigError` | Collected, all reported |
| Empty or missing `rules` block | `DQDatasetConfigError` | Collected, all reported |
| Invalid rule type | `DQDatasetConfigError` | Collected, all reported |
| Invalid severity | `DQDatasetConfigError` | Collected, all reported |
| Missing required rule field | `DQDatasetConfigError` | Collected, all reported |
| No enabled datasets | `DQConfigError` | Abort immediately |

**Key behaviour:** all dataset errors are collected before raising — you see every
broken config in one run, not one error per run.

---

## 16 supported rule types

| Type | Description | Key config fields |
|---|---|---|
| `not_null` | Column must have zero nulls | `column` |
| `unique` | Column combo must be unique | `columns` |
| `row_count` | Row count within bounds | `min` and/or `max` |
| `accepted_values` | Values from an allowed list | `column`, `values` |
| `regex` | Values match a regex | `column`, `pattern` |
| `range` | Numeric column within bounds | `column`, `min`/`max` |
| `date_range` | Date within bounds | `column`, `min`/`max` (use `"today"` for today) |
| `referential` | FK exists in reference table | `column`, `ref_table`, `ref_column` |
| `freshness` | Data updated within N hours | `column`, `max_hours` |
| `custom_sql` | Arbitrary SQL (use `{table}`) | `sql` |
| `completeness` | Null rate ≤ threshold | `column`, `max_null_rate` |
| `duplicate_count` | Duplicate groups ≤ threshold | `columns`, `max_duplicates` |
| `conditional_not_null` | Not-null when condition met | `column`, `condition_column`, `condition_value`, `condition_operator` |
| `mutual_exclusivity` | Exactly one column populated | `columns`, `allow_all_null` |
| `character_set` | Only allowed characters | `column`, `charset_name` or `allowed_chars` |
| `length_check` | String length within bounds | `column`, `min_length`/`max_length`, `count_mode`, `strip_before_check` |

---

## Adding a new dataset — zero code changes

1. Create `config/datasets/my_table.yml`:
```yaml
dataset:
  name: my_table          # must match master_config entry exactly
  source:
    type: delta
    path: main.myschema.my_table

rules:
  - id: my_table_pk_not_null
    description: "Primary key must not be null"
    type: not_null
    column: id
    severity: CRITICAL

  - id: my_table_status
    description: "Status must be valid"
    type: accepted_values
    column: status
    values: [ACTIVE, INACTIVE, PENDING]
    severity: WARNING
```

2. Add to `config/master_config.yml`:
```yaml
datasets:
  - name: my_table
    config_file: config/datasets/my_table.yml
    enabled: true
```

3. Run `00_dq_controller` — no notebook changes needed.

---

## Running the test suite

Open `notebooks/05_dq_tests.py` and click **Run All**.

The test notebook:
- `%run`s notebooks 01–03 to load all logic
- Runs config validation tests without Spark (pure Python)
- Runs all 16 rule type tests against synthetic `spark.createDataFrame` data
- Raises an exception if any test fails (so the run shows as red)

---

## Scheduling as a Databricks Job

1. Go to **Workflows → Create Job**
2. Add a task of type **Notebook**
3. Point to `notebooks/00_dq_controller`
4. Set parameters (widgets):
   ```
   repo_root          = /Workspace/Repos/your-name/dq-framework-notebooks
   master_config_path = config/master_config.yml
   abort_on_critical  = true
   dry_run            = false
   results_table      = main.dq_framework.dq_results
   alerts_table       = main.dq_framework.dq_alerts
   ```
5. Schedule as needed (e.g. daily after your ingestion pipeline)

---

## Results tables

| Table | Contents |
|---|---|
| `dq_results` | All rule results — run_id, dataset, rule, status, severity, row_count, details |
| `dq_alerts` | CRITICAL and WARNING failures only — for alerting pipelines |

Both tables are created automatically on first run.

Query latest results:
```sql
SELECT dataset_name, rule_id, severity, status, row_count, details
FROM main.dq_framework.dq_results
WHERE run_id = (SELECT MAX(run_id) FROM main.dq_framework.dq_results)
ORDER BY severity, status;
```
