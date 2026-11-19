# Phases 7-9: Fix and re-test, findings and sign-off, production readiness

## 1. Observation log

Log every finding as O#: what was found, impact, evidence (query, file, line), recommendation, owner, status (open / fixed / accepted / deferred). Include findings about the existing job (defects, security concerns, unused logic), not only about the POC.

## 2. Fix and re-test loop

1. Present open issues grouped by type (POC bug, design gap, prod defect, unexplained).
2. For each, the user decides: **fix**, **accept** (as a documented difference), or **defer** (becomes a production gate).
3. Fix in code (never in ad-hoc cells), re-run the affected tests plus a regression of the headline parity checks, and update the observation and results.
4. If a fix changes a design decision, update the decision log and say so.

## 3. Findings and sign-off package

The design doc should let a reviewer decide quickly:

* **Summary at the top:** objective, conclusion, next steps. State the conclusion honestly, including scale ("proven on N sample keys") and conditions.
* **Index and reading guide** (who should read what).
* **Results** with accepted differences and a sign-off column.
* **Proven / not proven** list.
* **Decisions requested** from reviewers, with owners.
* **Code and assets** (links) and **code mapping** (old scripts/configs -> new files).
* Keep long SQL out of the doc; link to the validation notebook.

Before sharing: re-check headline numbers independently; check links resolve; check numbers agree across sections; check that reviewers have read access to the doc and code. Don't grant access to POC tables with sensitive data without approval.

## 4. Production gates (G#)

Tailor to the job; typical gates:

| Area | Gate |
| --- | --- |
| Compliance | Erasure/redaction implemented for pipeline-owned tables and registered with the compliance process; hard delete and vacuum tested |
| Protection | Full refresh blocked on stateful tables |
| Behaviour sign-off | Data owner signs off accepted differences and intended changes |
| Open items | Unexplained differences traced |
| Consumers | Owners confirm impact (object types, names, streaming/CDF reads) |
| Scale | Full-volume run: duration, cost, cluster/serverless sizing |
| Quality and monitoring | Expectations, alerts, latency/freshness monitoring |
| CI/CD | Bundle deployed through CI; config and onboarding versioned; logic kept in sync with the old job until cutover |
| Shadow run | New pipeline runs alongside the old job with periodic parity checks |
| Cutover and rollback | Name switch or view swap plan; old job paused but restartable; checkpoints and old tables kept read-only for the rollback window |
| Decommission | Old job, configs and tables retired after the rollback window |

## 5. Cleanup

List every POC object (tables, views, volumes, pipelines, jobs, files) and confirm with the user before dropping anything; keep what an audit or sign-off still needs.

## 6. Lessons for the next migration

At the end, ask the user whether any learnings (framework mappings, sdp-meta quirks, validation shortcuts) should be added to this skill's references, so the next migration benefits. Add them as guidance, not as rules for every job.
