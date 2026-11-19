# Phase 1: Analysis of the existing job

Goal: a complete, evidence-backed picture of what the job does today, including behaviour hidden in shared code, so the new design can reproduce it (or knowingly change it).

## 1. Locate every artefact

| Artefact | Where to look | Tools |
| --- | --- | --- |
| Orchestration | Airflow DAG: schedule, sensors, cross-DAG dependencies, retries and timeouts, SLAs, catchup/backfill settings, templated parameters (`{{ ds }}`, `execution_date`), XComs, pools; plus CI/CD that generates Databricks jobs | `regexAssetSearch`, repo files, GitHub connector if enabled |
| Databricks job definition | Tasks, notebook paths, parameters, clusters, schedules (in each environment) | `databricksApi` `/api/2.1/jobs/get` |
| Job config | YAML/JSON configs and their includes | `readAssetById`, `regexAssetSearch` |
| Shared framework code | Generic notebooks/libraries that config-driven tasks call (ingest, merge, transform, compliance, maintenance) | Resolve from the job's `notebook_task` paths; don't trust name mappings alone |
| Tables | Every source, intermediate and target table, plus properties, partitioning, clustering, CDF, retention | `readTable`, `DESCRIBE DETAIL`, `DESCRIBE HISTORY` |

Record which environment each fact comes from (prod vs nonprod paths can differ). Flag anything you couldn't verify.

## 2. Task-by-task inventory

For every task capture: input(s), output(s), read mode (batch / stream / CDF, starting version), transformation logic (with file and line), write mode (append / merge / overwrite), merge keys and partition columns, dedupe rule and ordering column, tie-breaks, lookback windows, trigger and batch limits, and side services that run alongside it.

## 3. Hidden-behaviour checklist

Shared framework code often decides the real behaviour. Check, for each task:

* **Dedupe granularity:** is in-batch dedupe on the same key as the merge? A mismatch can silently drop or keep rows.
* **Ordering and ties:** what wins when two events share a timestamp? Is it deterministic?
* **Change handling on reads:** options such as `ignoreChanges` / `skipChangeCommits`; what upstream operations actually happen (check source history).
* **Lookback windows** in fix-up or enrichment steps (data older than the window is never corrected).
* **Date-driven reprocessing:** tasks parameterised by run date (`{{ ds }}`) that overwrite or re-merge a partition, and how backfills are run today. SDP processes incrementally, so this needs an explicit design choice.
* **Non-data tasks:** API calls, file exports, notifications, external system triggers. Note their inputs, outputs and failure handling.
* **Idempotency and replays:** what happens if a batch is processed twice?
* **Late and out-of-order data**, cross-region or multi-source ordering.
* **Compliance processes** (erasure, pseudonymisation, redaction): what tables they touch, which API they use, when they run, what sentinel values they write.
* **Maintenance hooks:** vacuum, optimize, retention deletes, latency or monitoring calls.
* **Schema handling:** evolution, casts, null handling, nested/array explode logic.
* **Unused or dead logic** (joins whose columns are never selected, empty tables).

## 4. Data profile

Use the `data-sampling` skill. Keep scans scoped. Measure:

* Row counts and daily volumes per source and target.
* **Source retention vs target history depth.** If targets hold history the sources no longer have (backfills, short retention), history must be seeded, not rebuilt.
* Key uniqueness in targets (existing duplicate keys are common and affect parity).
* Sentinel / special rows (redacted, defaulted, test data) and how they are keyed.
* Table history: which operations write to each table and how often; vacuum/time-travel limits (they bound how far back a seed can read).

## 5. Consumers and ownership

* Downstream readers from lineage and query history (`databricks-sql` skill for system tables). Note whether anyone streams from or reads the change feed of a target, because that constrains object types.
* Owner, tier/criticality, SLA, schedule source (Airflow vs Databricks), and any note that the data may be unused.

## 6. Complexity assessment

Score each dimension low / medium / high with evidence; don't force a single tier.

| Dimension | Low | High |
| --- | --- | --- |
| Tasks and layers | Few tasks, one layer | Many tasks, several layers, fan-in |
| Logic | Column mapping, simple filters | Window logic, pivots, multi-table joins, fix-up steps |
| State and history | Rebuildable from sources | History older than source retention; backfills |
| Change semantics | Append only | Upserts, deletes, CDF consumers, SCD2 |
| Compliance | None | Erasure / redaction on targets |
| Repetition | One-off objects | Many objects with the same shape (favours metadata-driven sdp-meta) |
| Consumers | Few batch readers | Many readers, streaming/CDF readers, external systems |
| Volume | Small | Large, skewed, high frequency |

## Output of phase 1

* Design doc section "Existing design" (task inventory, data flow, tables, side services).
* Initial observations (O#) for defects, risks and oddities found, each with evidence.
* Complexity assessment with a short narrative.
* Open questions for the user. Gate: the user confirms the analysis is complete and correct.
