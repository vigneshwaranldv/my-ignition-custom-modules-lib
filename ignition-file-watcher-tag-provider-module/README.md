# Ignition Custom Modules

> Work-In-Progress modules - MVP functioanlity is completed; adding more functionalities and pending some testing.

---

## Table of Contents
1. [Modules Overview](#modules-overview)
2. [Using the File Tag Provider](#using-the-file-tag-provider)
   - [How It Works](#how-it-works)
   - [Configure a CSV Source](#configure-a-csv-source)
   - [Configure an Excel Source](#configure-an-excel-source)
   - [Reading Tags in Scripts](#reading-tags-in-scripts)
3. [Real-World Examples](#real-world-examples)


---
## Modules Overview

| Module | Artifact | Scope | Ignition Min |
|--------|----------|-------|--------------|
| **File Tag Provider** | `Ignition-File-Tag-Provider.modl` | Gateway | 8.1.0 |

Modules are **free modules** (no Ignition licence tier required), work in trial mode, and are unsigned for development (see [Signing](#module-signing) for production).

---

## Using the File Tag Provider

The File Tag Provider registers a tag provider named **`[FileTags]`** in the Ignition tag browser. Tags are read from CSV or Excel files on the gateway host and refreshed automatically when the file changes.

### How It Works

```
CSV / Excel file  →  FileWatcherService (polls every N sec)
                  →  TagMappingEngine   (maps columns → tag paths)
                  →  ManagedTagProvider →  [FileTags] in Ignition
```

- **No gateway restart needed** to add/remove sources — just insert a row via script or the config REST endpoint.
- **Change detection** uses file `last-modified` timestamp — no unnecessary re-reads.
- **Quality codes**: `GOOD` when the file is read, `BAD_FAILURE` when the file is missing.

---

### Configure a CSV Source

**CSV file** (`/data/setpoints.csv`):

```csv
parameter,value,unit,low_limit,high_limit
reactor_temp,85.0,C,60,95
reactor_pressure,2.4,bar,1.5,3.5
coolant_flow,45.2,L/min,30,60
```

**Option A — via script (recommended for automation):**

```python
# Run this once from the Script Console or a Gateway Startup Script
import system

# 1. Register the file source
source = {
    "name":           "reactor-setpoints",
    "filePath":       "/data/setpoints.csv",   # absolute path on the gateway host
    "fileType":       "CSV",
    "tagProviderName": "FileTags",
    "tagPathPrefix":  "Reactor/Setpoints",     # tags appear under [FileTags]Reactor/Setpoints/
    "pollIntervalSecs": 30,
    "enabled":        True
}

resp = system.net.httpPost(
    "http://localhost:8088/main/data/file-tags/sources",
    contentType = "application/json",
    postData    = system.util.jsonEncode(source)
)

source_id = system.util.jsonDecode(resp.text)["id"]

# 2. Map the 'value' column to a tag for each row
mappings = {
    "mappings": [
        {"columnName": "value", "tagPath": "value", "dataType": None}
    ]
}
system.net.httpPost(
    "http://localhost:8088/main/data/file-tags/sources/{0}/mappings".format(source_id),
    contentType = "application/json",
    postData    = system.util.jsonEncode(mappings)
)
```

Tags created (with `tagPathPrefix = "Reactor/Setpoints"` and 3 data rows):

```
[FileTags]Reactor/Setpoints/row_0/value   →  85.0
[FileTags]Reactor/Setpoints/row_1/value   →  2.4
[FileTags]Reactor/Setpoints/row_2/value   →  45.2
```

**Option B — single-row CSV (no row_ prefix):**

For a CSV with only **one data row**, the tag path prefix is used directly:

```csv
temperature,pressure,flow
85.0,2.4,45.2
```

With `tagPathPrefix = "Live"` and mappings for `temperature`, `pressure`, `flow`:

```
[FileTags]Live/temperature  →  85.0
[FileTags]Live/pressure     →  2.4
[FileTags]Live/flow         →  45.2
```

---

### Configure an Excel Source

**Excel file** (`/data/process-data.xlsx`, Sheet: `OvenData`):

```python
source = {
    "name":            "oven-data",
    "filePath":        "/data/process-data.xlsx",
    "fileType":        "EXCEL_XLSX",
    "sheetName":       "OvenData",          # leave null to use the first sheet
    "tagProviderName": "FileTags",
    "tagPathPrefix":   "Oven",
    "pollIntervalSecs": 60
}
# (same POST call as above)
```

Supports `.xlsx` (POI OOXML), `.xls` (POI HSSF), formula cells, and integer formatting.

---

### Reading Tags in Scripts

Once configured, `[FileTags]` tags are readable via all standard Ignition tag APIs:

```python
# Read a single tag
val = system.tag.readBlocking(["[FileTags]Reactor/Setpoints/row_0/value"])[0].value
print("Reactor temp setpoint:", val)

# Read multiple tags at once
paths = [
    "[FileTags]Live/temperature",
    "[FileTags]Live/pressure",
    "[FileTags]Live/flow",
]
values = system.tag.readBlocking(paths)
temp, pressure, flow = [v.value for v in values]

# Use in an Expression tag
# Expression: {[FileTags]Live/temperature} > 90 ? "HIGH" : "NORMAL"

# Browse all tags under [FileTags]
results = system.tag.browse("[FileTags]")
for tag in results.getResults():
    print(tag["path"], tag["value"])
```

---

## Real-World Examples

### Setpoint Comparison (CSV vs Live Tags)

```python
# Compares live process values against file-based setpoints
# Run from a Perspective button or scheduled script

def check_setpoints():
    checks = [
        ("Temperature", "[default]Reactor/Temperature",       "[FileTags]Setpoints/row_0/value"),
        ("Pressure",    "[default]Reactor/Pressure",          "[FileTags]Setpoints/row_1/value"),
        ("Flow",        "[default]Reactor/CoolantFlow",       "[FileTags]Setpoints/row_2/value"),
    ]
    
    live_paths    = [c[1] for c in checks]
    target_paths  = [c[2] for c in checks]
    live_values   = [v.value for v in system.tag.readBlocking(live_paths)]
    target_values = [v.value for v in system.tag.readBlocking(target_paths)]
    
    deviations = []
    for name, _, _, live, target in zip(*(list(checks) + [live_values, target_values])):
        pct = abs(live - target) / max(abs(target), 1e-9) * 100
        if pct > 10:
            deviations.append("{0}: live={1:.1f}, target={2:.1f} ({3:.0f}% off)".format(
                name, live, target, pct
            ))
    
    if deviations:
        summary = system.llm.summarize("\n".join(deviations), maxWords=50)
        system.tag.writeBlocking(["[default]Dashboard/DeviationSummary"], [summary])
```

---

---

## Dev Environment Setup

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| **JDK 21** | 21.x | Build runtime (Kotlin 2.0 requires JDK 11+, but JDK 21 is recommended) |
| **Docker Desktop** | 4.x+ | Local Ignition gateway + Ollama containers |
| **Git** | any | Version control |
| Python 3 | 3.9+ | Module acceptance script (`infra/scripts/accept-module.sh`) |

> ⚠️ **JDK 25 is not compatible** with Kotlin 2.0 (version string parse issue). Use JDK 21.

---

### Project Structure

```
ignition-custom-modules/
├── modules/
│   ├── file-tag-provider/              # [FileTags] file-backed tag provider
│       ├── build.gradle.kts
│       └── src/
│           ├── main/kotlin/com/customignition/filetags/
│           │   ├── common/             # FileSource, ColumnMapping, TagRecord, TagUpdate, TagQuality
│           │   ├── db/                 # FileSourceDb — SQLite config store
│           │   ├── readers/            # CsvFileReader, ExcelFileReader
│           │   ├── engine/             # TagMappingEngine — column → tag path mapping
│           │   ├── watcher/            # FileWatcherService — scheduled polling
│           │   ├── provider/           # FileTagProvider — wraps ManagedTagProvider
│           │   └── gateway/            # GatewayHook, FileTagConfigServlet
│           └── test/                   # 33 unit tests
│
├── gradle/
│   ├── libs.versions.toml              # Version catalog (all dependency versions)
│   └── wrapper/                        # Gradle wrapper (gradlew / gradlew.bat)
│
├── infra/
│   ├── docker/
│   │   └── docker-compose.yml          # ignition-dev + ollama-dev containers
│   └── scripts/
│       ├── accept-module.sh            # Insert EULAS acceptance into gateway DB
│       ├── deploy-module.sh            # Upload .modl via gateway REST API
│       ├── wait-for-gateway.sh         # Poll until gateway is healthy
│       └── reset-gateway.sh            # Wipe and restart the gateway volume
│
├── .github/workflows/
│   └── ci.yml                          # GitHub Actions: test → build → integration test
│
├── build.gradle.kts                    # Root build — shared Kotlin/test config
├── settings.gradle.kts                 # Includes modules/system-llm, modules/file-tag-provider
├── gradle.properties                   # Gradle performance flags, skipModlSigning
└── TEST-RESULTS.md                     # Integration test results & evidence
```

---

---

## Module Reference

### File Tag Provider — Tag Path Rules

| CSV rows | tagPathPrefix | Column mapping tagPath | Resulting tag path |
|----------|---------------|------------------------|--------------------|
| 1 row | `"Live"` | `"temperature"` | `[FileTags]Live/temperature` |
| N rows | `"Batch"` | `"value"` | `[FileTags]Batch/row_0/value`, `row_1/value`, … |
| 1 row | `""` (empty) | `"flow"` | `[FileTags]flow` |

**Supported data types in column mappings** (leave `null` for auto-infer):
`"float"` · `"double"` · `"int"` · `"long"` · `"bool"` · `"boolean"` · `"string"`

---
