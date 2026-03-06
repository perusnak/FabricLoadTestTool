# Fabric Load Test Tool

A set of Microsoft Fabric notebooks that **stress-test Power BI semantic models**
(datasets) by replaying DAX queries captured with
[Performance Analyzer](https://learn.microsoft.com/power-bi/create-reports/desktop-performance-analyzer)
and measuring response times under configurable concurrency.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Folder Structure](#folder-structure)
5. [Quick Start](#quick-start)
6. [Configuration Reference](#configuration-reference)
7. [Understanding the Output](#understanding-the-output)
8. [FAQ / Troubleshooting](#faq--troubleshooting)

---

## Overview

| Notebook | Purpose |
|---|---|
| **RunLoadTest.ipynb** | **Orchestrator** — configures the test, then launches one or more copies of *RunPerfScenario* in parallel. |
| **RunPerfScenario.ipynb** | **Worker** — connects to the semantic model via XMLA, replays every DAX query from a Performance Analyzer JSON, and writes a CSV log. |
| **Evaluation.ipynb** | **Analysis** — reads the CSV logs and produces summary tables + bar charts so you can compare scenarios (e.g. Import vs DirectQuery). |

### How a test run works

```
RunLoadTest  (parent)
  │
  ├──► RunPerfScenario  (thread 0)  ──► CSV log
  ├──► RunPerfScenario  (thread 1)  ──► CSV log
  └──► RunPerfScenario  (thread N)  ──► CSV log
                                          │
Evaluation  ◄─────────────────────────────┘
  └──► summary tables & charts
```

1. You open **RunLoadTest** and set the configuration (dataset name, concurrency,
   number of iterations, etc.).
2. It discovers all Performance Analyzer JSON files in the lakehouse and builds a
   DAG (Directed Acyclic Graph) of parallel notebook activities.
3. Fabric's `notebookutils.notebook.runMultiple()` launches one **RunPerfScenario**
   per virtual user, all running concurrently.
4. Each worker connects to the semantic model over XMLA, replays the DAX queries,
   and writes timing data to a CSV file.
5. After the test, you open **Evaluation** and point it at the log folder(s) to
   generate comparison charts.

---

## Architecture

```
Lakehouse (Files)
└── PerfScenarios/
    ├── Queries/                          ← Performance Analyzer JSON exports
    │   └── <workspace>/
    │       ├── PowerBIPerformanceData.json
    │       └── AnotherReport.json
    └── logs/                             ← CSV results written by RunPerfScenario
        └── <workspace>/
            └── <loadtestId>/
                ├── <loadtestId>_<user>_0.csv
                ├── <loadtestId>_<user>_1.csv
                └── error.log             ← created only on failure
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Microsoft Fabric workspace** | With a Lakehouse attached as the default. |
| **Semantic model** | A Power BI dataset (Import, DirectQuery, or Composite) accessible via XMLA endpoint. |
| **XMLA endpoint enabled** | Tenant or workspace admin must enable *XMLA read-only* (or read/write) in the Power BI admin portal. |
| **Performance Analyzer export** | At least one `.json` file captured from Power BI Desktop → Performance Analyzer → Export. |
| **Python libraries** (pre-installed on Fabric) | `sempy`, `pandas`, `matplotlib`, `seaborn`, `pyspark`. |

---

## Folder Structure

```
FabricLoadTestTool/
├── README.md                ← you are here
├── RunLoadTest.ipynb        ← orchestrator notebook
├── RunPerfScenario.ipynb    ← worker notebook (called by RunLoadTest or run standalone)
└── Evaluation.ipynb         ← analysis & visualisation notebook
```

---

## Quick Start

### Step 1 — Capture queries

1. Open your Power BI report in **Power BI Desktop**.
2. Go to **View → Performance Analyzer → Start recording**.
3. Interact with the report (switch pages, apply filters, etc.).
4. Click **Stop** → **Export** to save a `.json` file.

### Step 2 — Upload to the Lakehouse

Upload the exported JSON file(s) to:

```
/lakehouse/default/Files/PerfScenarios/Queries/<workspace_name>/
```

Replace `<workspace_name>` with the display name of your Power BI workspace
(e.g. `MRL GCTO POC`).

### Step 3 — Configure RunLoadTest

Open `RunLoadTest.ipynb` and edit the **CONFIG** section in Cell 3:

```python
load_test_name     = "My First Load Test"       # descriptive label
dataset            = "Sales Report"              # semantic model name
xmla_endpoint      = "powerbi://api.powerbi.com/v1.0/myorg/My%20Workspace"
workspace          = "My Workspace"
home_workspace     = "My Workspace"              # where this notebook lives
concurrent_threads = 5                           # virtual users
delay_sec          = 2                           # think-time between queries
iterations         = 3                           # how many times each user replays
```

### Step 4 — Run the test

Run all cells in `RunLoadTest.ipynb`.  The output will show:

```
running load test
load test complete
```

CSV files appear in `/lakehouse/default/Files/PerfScenarios/logs/<workspace>/<loadtestId>/`.

### Step 5 — Analyse results

Open `Evaluation.ipynb` and update the **CONFIGURATION** section to point at
the log folder(s) you want to compare:

```python
folder_specs = [
    ("My Workspace", "My First Load Test-20260305-143000", "5 users"),
    ("My Workspace", "My First Load Test-20260305-150000", "10 users"),
]
OUTPUT_MODE = "B"   # "A" for aggregate, "B" for per-user detail
```

Run the cell to see summary tables and charts.

---

## Configuration Reference

### RunLoadTest.ipynb — Cell 3

| Variable | Type | Description |
|---|---|---|
| `load_test_name` | `str` | Human-readable label written into every log row. |
| `dataset` | `str` | Power BI semantic model (dataset) name. |
| `xmla_endpoint` | `str` | XMLA endpoint URL.  Set to `None` to use the current workspace. |
| `workspace` | `str` | Display name of the workspace hosting the dataset. |
| `home_workspace` | `str` | Workspace where RunLoadTest + RunPerfScenario notebooks live. |
| `concurrent_threads` | `int` | Number of parallel virtual users. |
| `delay_sec` | `int` | Think-time in seconds between queries. |
| `iterations` | `int` | Number of times each user replays the full query set. |
| `users` | `list[str]` | Optional list of UPNs for RLS impersonation. Leave `[]` to run as yourself. |

### RunPerfScenario.ipynb — Cell 1 (Parameters)

All variables from RunLoadTest are passed automatically.  For **standalone** runs
you can edit Cell 1 directly — see the comments in-line.

### Evaluation.ipynb

| Variable | Type | Description |
|---|---|---|
| `folder_specs` | `list[tuple]` | List of `(workspace, folder, label)` tuples to compare. |
| `warmup_ratio` | `float` | Fraction of initial rows to discard (0–1). |
| `OUTPUT_MODE` | `str` | `"A"` = aggregate per folder, `"B"` = per-user detail. |

---

## Understanding the Output

### CSV columns (written by RunPerfScenario)

| Column | Description |
|---|---|
| `loadtest_id` | Unique run identifier. |
| `model` | Semantic model name. |
| `concurrent_threads` | How many virtual users ran in parallel. |
| `iterations` | Total iterations configured. |
| `delay_sec` | Think-time setting. |
| `query_number` | 1-based index of the query within one iteration. |
| `visual_name` | Power BI visual title that generated this DAX query. |
| `visual_id` | Internal visual identifier. |
| `iteration` | 0-based iteration index. |
| `query` | Full DAX query text. |
| `rows` | Number of rows returned. |
| `duration` | Wall-clock query duration in **seconds**. |
| `start_time` | Unix timestamp when the query started. |
| `start_time_dt` | Human-readable start time. |
| `customdata` | Value passed to the `CUSTOMDATA()` DAX function (for RLS). |
| `effective_username` | UPN used for RLS impersonation (or `None`). |
| `thread_id` | Virtual user index (0, 1, 2, …). |

### Evaluation charts

- **Mode A** — one grouped bar chart with avg, p50, p90, p95, p99, max per scenario.
- **Mode B** — individual bar charts per metric (one bar per user per scenario),
  plus a combined chart averaging across users.

---

## FAQ / Troubleshooting

### Q: How do I increase concurrency beyond 4?
Edit Cell 2 in RunLoadTest to request more vCores:
```python
%%configure -f
{ "vCores": 16 }
```

### Q: The test fails with a timeout
Increase `timeoutPerCellInSeconds` (per-cell) or `timeoutInSeconds` (global) in
the DAG definition inside RunLoadTest Cell 3.

### Q: I want to test Row-Level Security (RLS)
Populate the `users` list in RunLoadTest with real UPN addresses.  Each virtual
user thread cycles through the list round-robin.  The impersonated identity is
passed via `EffectiveUserName` in the XMLA connection string.

### Q: Can I run RunPerfScenario by itself?
Yes.  Edit Cell 1 (the parameters cell) with your desired values and run all
three cells.  No parent notebook is needed.

### Q: Where do I get the XMLA endpoint?
In the Power BI Service: **Workspace settings → Premium → Workspace connection**.
Copy the URL that starts with `powerbi://api.powerbi.com/...`.

---

## License

Internal use only.
