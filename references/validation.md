# Phase 6: Testing and validation

Goal: evidence that the new pipeline reproduces the agreed behaviour, with every difference explained.

## 1. Agree the rules before comparing

Write the parity rules into the design doc and get them agreed before you run the comparison:

| Class | Typical columns / checks | Rule |
| --- | --- | --- |
| Exact | Business columns, keys, grain, lineage columns | 100% match for sample keys minus exclusions |
| Rule-based | ETL timestamps, generated IDs | Rules (for example seeded rows keep old values; new rows fall within the run window; never null) |
| Special rows | Sentinel/redacted rows | Counted separately against the seed |
| Structure | Columns, types, order, nullability, properties, clustering | Identical, except approved changes |
| Accepted differences | Intended fixes, known prod defects | Explained one by one and signed off |

## 2. Freeze the comparison point

Production keeps changing. Compare against a **frozen copy**: read-only copies of the production tables taken at the first versions that include everything the POC processed (the watermark), restricted to the sample keys. Check that the watermark holds (source max unchanged before and after capture). Time travel to a single timestamp across many tables can fail or be inconsistent; per-table versions are safer.

## 3. Exclusions

Keep an exclusion table `(key, reason)` for keys that can't be compared fairly: for example keys changed by processes the POC doesn't run (compliance erasure in prod), test keys used for destructive tests, events after the watermark. Report exclusion counts with the results.

## 4. Tests to run

| Test | What it shows |
| --- | --- |
| Row-set parity per table | Keys only in prod, only in POC (target 0, or each explained) |
| Column parity on matched keys | Per-column differences |
| Duplicate keys | New targets have none; count existing ones in prod |
| Incremental run | A second run processes only new input; counts move as expected |
| Re-run / idempotency | Re-processing the same input changes nothing |
| Edge cohorts | Each targeted cohort behaves as designed |
| Special rows | Sentinel/redacted counts equal the seed |
| ETL timestamp rules | Rule-based columns satisfy their rules |
| Structure | Schema and table properties match |
| Compliance | Erasure/redaction via the planned mechanism works and persists after later runs; seeds don't re-insert erased rows |
| Protection | Full refresh is blocked where configured |
| No production writes | Table history on all production objects shows no commits by the POC user |
| Performance (indicative) | Run duration and cost at sample size; extrapolate with care |

## 5. Classify every difference

For each difference: count, example keys, cause, class. Classes: **POC bug** (fix), **prod defect** (new design fixes it; accepted difference), **intended change** (decision), **timing** (exclusion), **unexplained** (stays open; never fold into "accepted"). Don't add counts across overlapping categories.

## 6. Independent re-check

Re-check headline numbers with a different method (for example distinct-key anti-joins vs the original full-row comparison) before reporting. Report what you checked and what you couldn't.

## 7. Save it as code

All validation queries live in a validation notebook in the bundle, with a header that explains order, which cells write (snapshot copies, exclusions) and which only read. The design doc links to it; it doesn't embed long SQL.

## Output

Design doc section "Validation results": parity table per object, accepted differences with explanations, open items, sign-off column. Gate: the user reviews the results.
