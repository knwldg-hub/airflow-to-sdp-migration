# Phases 4-5: POC setup and build

## 1. Setup questions (ask before creating anything)

* **Target:** which catalog/schema for POC objects (existing sandbox or new); naming prefix to avoid collisions with real objects.
* **Sources:** read production sources read-only, use copies, or use nonprod sources? For streaming sources, read the existing landing/raw tables or the original stream?
* **Data handling:** may the POC use real (possibly sensitive) data as-is? Who approves, and when? Record approver and date for audit.
* **Scope:** all tables/tasks or a subset; sample size; time window.
* **Compute and deployment:** serverless or classic; bundle target (dev); who can see the POC assets.

## 2. Sample design

* Pick a deterministic sample of business keys (for example a hash-modulo on the main entity key), small enough for runs of minutes.
* Add **targeted cohorts** for every edge case found in analysis (multi-source keys, ties, deletes/delinks, orphans, sentinel rows, late data).
* Save the sample as a table (`<prefix>_sample_keys(key, cohort)`) so seed, replay and validation use exactly the same keys.
* Define T0 (seed point) and the replay window; check source retention and target time-travel limits before fixing T0.

## 3. Project layout (bundle)

```
<project>/
  databricks.yml            # pipeline + job, dev target first
  conf/onboarding.(yml|json) # sdp-meta onboarding, if used
  src/
    transforms.py           # business logic as plain functions
    pipelines/              # sdp-meta runner, SDP code (Python/SQL), views
  seed/                     # seed and sample notebooks
  validation/               # parity and test notebooks
  tests/                    # unit tests for transforms
```

Put everything in the bundle from the start, including onboarding config and the onboarding step. Use the `dabs` skill for bundle structure.

## 4. Build steps

1. Pipeline settings: serverless where possible; configuration parameters for catalog/schema/sample/T0. Table properties: `pipelines.reset.allowed = false` on tables that hold seeded or unrebuildable history; `delta.enableChangeDataFeed = true` where consumers need it.
2. sdp-meta (if used): pin the version; write the onboarding file; run onboarding into dataflow-spec tables; wire custom transform functions to the business-logic module. Record onboarding quirks in the design doc.
3. Plain SDP code for remaining layers (`spark-declarative-pipelines` skill). Use one-time (`once`) flows for seeds alongside streaming flows where the design seeds history.
4. Seed tables from production `VERSION/TIMESTAMP AS OF T0`, restricted to the sample keys. Include special/sentinel rows as the design specifies.
5. Run the first update; check the event log for warnings; review row counts by layer with the user.
6. Unit-test transform functions on small hand-built inputs, including edge cases (nulls, odd-length arrays, duplicates, ties).

## 5. Safety while building

* Writes only to the approved POC target. Before any DML on POC tables, load the `writing-sql` skill and confirm with the user.
* Never run a full refresh on seeded tables without confirmation.
* Track every created object (tables, views, volumes, pipelines, jobs) for the cleanup list.
* After each edit to code or the design doc, re-read it to confirm it landed.

## Output

POC built and running; design doc section "POC code and assets" lists every file and object with links. Gate: the user reviews the first run.
