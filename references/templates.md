# Templates

## Design doc outline (one notebook per job)

Keep section numbers stable once shared; reference items by ID.

| Part | Section | Content |
| --- | --- | --- |
| Top | Overview | Summary (objective, conclusion, next steps), key facts, index, reading guide |
| A. Design | 1. Existing design | Task inventory, data flow, tables, side services |
| | 2. New design | Medallion layout, pattern choice, objects, deployment |
| | 3. Task mapping | Old task -> new object/flow -> behaviour change |
| | 4. Design decisions | D# log |
| B. POC method | 5. Sample design | Sample keys, cohorts, T0, replay window |
| | 6. Validation approach | Parity rules, comparison point, exclusions |
| C. Results | 7. Validation results | Baseline, final parity, operational tests (compliance, incremental) |
| | Conclusion | Verdict, proven / not proven, decisions requested |
| D. Findings | 8. Observations | O# log |
| | 9. Recommendations | |
| | 10. Next steps and gates | G# with owners |
| Appendix | POC code and assets | Links to files, pipeline, job, objects |
| | Code mapping | Old scripts/configs -> new files |

## Decision log

| D# | Topic | Decision | Reason / evidence | Status (proposed / decided / closed) |
| --- | --- | --- | --- | --- |

## Observation log

| O# | Observation | Impact | Evidence | Recommendation | Owner | Status |
| --- | --- | --- | --- | --- | --- | --- |

## Parity results

| Object | Prod rows | POC rows | Only prod | Only POC | Column diffs | Status | Sign-off |
| --- | --- | --- | --- | --- | --- | --- | --- |

## Production gates

| G# | Gate | How | Owner | Status |
| --- | --- | --- | --- | --- |

## Gate message to the user (end of each phase)

* What was done and the evidence (links, counts).
* What changed since the last gate.
* Decisions needed now, each with options, trade-offs and a recommended default.
* What happens next once confirmed.
