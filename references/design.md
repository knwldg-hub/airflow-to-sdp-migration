# Phase 2: Target design

Goal: a design the user understands and chooses, with every behaviour change made explicit. Present options with trade-offs and a recommended default; record decisions as D# (decision, reason, evidence, status).

## 1. Medallion layout

| Layer | Purpose | Good practice |
| --- | --- | --- |
| Bronze (raw) | Land source data as received | Append-only; minimal transformation; keep source lineage (source name, offsets/keys, ingest timestamp); one table per source shape, with one flow per source instance (for example per region) rather than one task each; replayable |
| Silver (refined) | Clean, conform, deduplicate, key | Explicit keys and sequencing; deterministic tie-breaks; data quality expectations; CDC (SCD1/SCD2) via AUTO CDC where rows are upserted |
| Gold (business) | Consumer-facing shapes | Joins, pivots, aggregates; object type driven by consumers; stable published names |

Map every existing table to a layer. If the old job mixes layers (for example business logic in ingestion, or fix-up steps that rewrite silver from gold), propose a cleaner placement and record the behaviour change.

## 2. Choosing sdp-meta, SDP, or both

These are criteria to weigh, not rules.

| Consider sdp-meta when | Consider plain SDP when |
| --- | --- |
| Many objects share the same shape (ingest, parse, upsert by key) | Few objects, or each is bespoke |
| Teams want config-driven onboarding of new sources/tables | Logic is complex (multi-table joins, pivots, windows) and reads better as code |
| Standard bronze/silver features are enough (append flows, AUTO CDC, expectations, custom transform hooks) | You need features or control sdp-meta doesn't expose |

A **hybrid** (sdp-meta for repetitive bronze/silver, plain SDP for gold or bespoke steps) in one pipeline is common. Prefer one pipeline when the same team owns the job and the layers share a cadence; split when ownership, cadence or failure isolation differ.

Before recommending sdp-meta, check: the version to pin, whether it's available from the internal package mirror, and whether the features the job needs are supported in that version. Read its docs/code for the version used; record onboarding quirks as you find them.

Keep business logic in plain, testable functions (one module), called from sdp-meta custom transform hooks or SDP code, so it can be unit-tested and reviewed independently of the framework.

## 3. Object types

| Need | Typical choice |
| --- | --- |
| Incremental append or CDC from a stream | Streaming table (with AUTO CDC for upserts) |
| Full recompute over a bounded set is acceptable | Materialized view |
| Consumers stream from or read the change feed of the target | Streaming table with change data feed enabled. (A materialized view's change feed is in Beta and can only be read from a different pipeline; check current docs before relying on it.) |
| Keep existing published names while internals change | Views over internal tables |
| Rows that must not be collapsed by a key (for example many rows sharing a redacted sentinel key) | Keep them outside the CDC target and union them in a view, or key them differently |

Confirm consumers (phase 1) before choosing. Use the `spark-declarative-pipelines` skill for API details.

## 4. History and seeding

* If sources retain full history and rebuild cost is acceptable: rebuild.
* Otherwise **seed** targets from existing tables as of a fixed point T0, then replay sources from slightly before T0 (overlap is safe when upserts are idempotent).
* Seeding relies on time travel: check vacuum/retention so the T0 version is still readable when you seed.
* Decide how ETL timestamps (insert/update) behave for seeded vs new rows.
* **Date-driven jobs:** if the old job re-processes partitions by run date or relies on Airflow backfills, choose how the new design handles reprocessing: incremental streaming plus AUTO CDC (late data handled by sequencing), materialized views (recompute), or a targeted reprocess (resetting selected flows' checkpoints, or a controlled full refresh where history allows). Record the choice as a decision.

## 5. Compliance and operations

* Erasure / redaction: how it will work on pipeline-owned tables (usually SQL DML on the pipeline targets), how it stays applied after later runs, and how new tables are registered with the compliance process.
* Protect stateful tables from full refresh with the table property `pipelines.reset.allowed = false` when history can't be rebuilt (it blocks full refresh, not incremental writes).
* Monitoring, alerting, data quality expectations, maintenance (predictive optimization vs explicit jobs), latency reporting.
* Cutover: target names (existing external tables vs new pipeline-owned tables), consumer impact, rollback.

## 6. Orchestration

* Replace the Airflow DAG with a **Lakeflow Job**: a pipeline task for the SDP pipeline, plus job tasks for anything that isn't a data transformation (API calls, exports, notifications, external triggers).
* Map Airflow features: schedule -> job schedule or continuous pipeline; sensors and cross-DAG dependencies -> job triggers (table update, file arrival) or task dependencies; retries/timeouts -> task settings; SLAs -> job duration thresholds and notifications; templated parameters -> job/pipeline parameters.
* Many sequential tasks in the DAG often collapse into one pipeline update, because SDP resolves dependencies between datasets itself.
* Plan how Airflow is retired for this job (DAG paused or removed, alerts moved), and keep it restartable for rollback.

## 7. Task mapping

Produce a table: every existing task -> new object/flow -> logic location -> behaviour change (none / intended fix / accepted difference). Steps that only exist to patch earlier defects (fix-ups, re-merges) can often be removed when the new design fixes the root cause; record each as a decision.

## Output of phase 2

Design doc sections "New design", "Task mapping", "Design decisions". Gate: the user picks options and agrees the design (or agrees to build the POC first and review design and results together).
