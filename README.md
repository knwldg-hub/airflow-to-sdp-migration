# airflow-to-sdp-migration (Genie Code skill)

A Genie Code skill that works with a data engineer to migrate an Airflow-orchestrated Databricks job to **Lakeflow Spark Declarative Pipelines (SDP)**, using **sdp-meta**, plain SDP, or both, on a medallion (bronze / silver / gold) architecture. The skill file is [`SKILL.md`](./SKILL.md).

> **Disclaimer:** this is a field-built accelerator, not a supported Databricks product. It is provided as is, without official support or warranty. Review every recommendation and keep production read-only until sign-off.

## Objective

Migrating an Airflow job is rarely a 1:1 rewrite. Real behaviour hides in shared framework code, run-date reprocessing, fix-up steps, backfilled history and compliance processes. This skill gives Genie Code a **repeatable, evidence-based method** so every migration, simple or complex, is:

* **Analysed from evidence:** the actual DAG, job definition, configs, shared code and data profile, with file, line and query citations.
* **Designed deliberately:** sdp-meta vs plain SDP vs hybrid, object types, history seeding and orchestration are chosen with trade-offs, and every behaviour change is recorded as a decision (D#).
* **Proven, not assumed:** a POC on a deterministic sample with edge-case cohorts, validated against a frozen production snapshot with agreed parity rules and an independent re-check.
* **Ready for production:** observations (O#), production gates (G#) with owners, a shadow run, and cutover, rollback and cleanup plans.
* **Engineer in the loop:** every phase ends with a gate; nothing is created or changed without the user's decision, and production is never written to.

### Phases

| Phase | Goal | Reference |
| --- | --- | --- |
| 0. Intake | Pick the job, agree scope, depth and success criteria | `SKILL.md` |
| 1. Analysis | Understand the existing job and data, including hidden behaviour | [references/analysis.md](references/analysis.md) |
| 2. Design | Medallion layout, sdp-meta / SDP / hybrid, object types, history, orchestration, task mapping | [references/design.md](references/design.md) |
| 3. Design review | Design doc ready for sign-off (or "POC first") | [references/templates.md](references/templates.md) |
| 4-5. POC setup and build | Environment, sample, bundle, onboarding, pipeline code, seed | [references/poc-build.md](references/poc-build.md) |
| 6. Test and validate | Parity, incremental, idempotency, compliance and protection tests | [references/validation.md](references/validation.md) |
| 7-9. Fix, sign-off, production readiness | Fix / accept / defer loop, sign-off package, gates, cutover | [references/findings-and-production.md](references/findings-and-production.md) |

### Repo layout

```
SKILL.md                               principles, workflow, phase summaries (loaded by Genie Code)
references/analysis.md                 phase 1: artefacts, task inventory, hidden-behaviour checklist, complexity
references/design.md                   phase 2: medallion, sdp-meta vs SDP, object types, seeding, orchestration
references/templates.md                design doc outline, D#/O#/G# logs, parity table, gate message
references/poc-build.md                phases 4-5: setup questions, sample design, bundle layout, build steps
references/validation.md               phase 6: parity rules, frozen comparison, tests, classification
references/findings-and-production.md  phases 7-9: fix loop, sign-off package, production gates, cleanup
```

## Install

Genie Code loads a skill from a folder that contains `SKILL.md`, under `.assistant/skills/` in your user folder (user skill) or in the workspace root (workspace skill). The whole folder is needed, including `references/`. ([How Genie Code skills work](https://docs.databricks.com/aws/en/genie-code/skills/); the docs site has a cloud switcher for Azure and GCP.)

`SKILL.md` is at the root of this repo, so cloning the repo as a Git folder inside a skills folder installs the skill directly. Updating is just **Pull**.

**Folder name:** keep the default Git folder name, `airflow-to-sdp-migration`. It matches the skill `name` in the `SKILL.md` frontmatter.

### Option 1: for yourself (recommended first)

1. In Genie Code, click **Settings → Open skills folder**. This opens `/Users/<you>/.assistant/skills/`, creating it if needed.
2. In that folder, choose **Create → Git folder**, paste `https://github.com/knwldg-hub/airflow-to-sdp-migration.git`, keep the default folder name, and click **Create**.
3. Start a new Genie Code chat.

To update: open the Git folder, click **Pull**, then start a new chat.

### Option 2: for the whole workspace (admin)

1. As a workspace admin, open `Workspace/.assistant/skills/` (create `.assistant` and `skills` if they don't exist; folders starting with a dot may be hidden).
2. Choose **Create → Git folder**, paste `https://github.com/knwldg-hub/airflow-to-sdp-migration.git`, keep the default folder name, and click **Create**.
3. Make sure users can read the folder.

Users get the skill in their next Genie Code chat. To update everyone: open the Git folder and click **Pull**.

### Option 3: keep it in another folder and register it

Use this to keep the skill outside `.assistant/skills/`. Genie Code registers a folder that *contains* skill folders, not a skill folder itself, so give the clone its own parent folder:

1. Create a parent folder, for example `/Users/<you>/skills`, and clone this repo inside it, keeping the default name: `/Users/<you>/skills/airflow-to-sdp-migration`.
2. In Genie Code, click **Settings**, enter `/Users/<you>/skills` in the add-skill field, and click **Add folder**.
3. Make sure the skill's toggle is on, then start a new chat.

Don't register a broad folder such as your whole user folder; every subfolder with a `SKILL.md` would be picked up.

### Option 4: copy the files (if you can't create a Git folder)

On GitHub, click **Code → Download ZIP** and unzip it (the ZIP has no `.git` folder, so nothing extra is copied). From the unzipped folder, with the Databricks CLI:

```bash
databricks workspace import-dir . \
  "/Workspace/Users/YOUR_EMAIL/.assistant/skills/airflow-to-sdp-migration" \
  --overwrite
```

Replace `YOUR_EMAIL` with your workspace user name. Updates mean copying again.

**Check the install:** in a new Genie Code chat, ask "Is the airflow-to-sdp-migration skill installed?" If you just updated and see old behaviour, start a new chat or hard-refresh the browser tab.

## Use

Open Genie Code and say, for example:

* "Use the airflow-to-sdp-migration skill to migrate the Airflow DAG `<dag_name>` to SDP."
* "Help me assess whether Databricks job `<job name or ID>` should move to sdp-meta or plain SDP."
* "Resume the SDP migration of `<job>`." (the skill checks memories, past conversations and the design doc first)

You can also `@`-mention the skill to make sure Genie Code uses it.

## Prerequisites

* Genie Code enabled in the workspace.
* Read access to the existing job: Airflow DAG code (repo, Git folder or GitHub connector), Databricks job definition, configs and shared framework code.
* Read access to the source and target tables (SELECT, plus table history for seeding and validation).
* A POC catalog/schema you can write to, approved before the POC starts.
* Serverless or a cluster for SDP pipelines, plus a SQL warehouse or serverless for profiling and validation.
* If using sdp-meta: a pinned version available from your package mirror.

## Safety

* Production objects are treated as read-only; all writes go to the approved POC target.
* Destructive actions (DROP, DELETE, overwrite, full refresh) need explicit confirmation, even in the POC target.
* Real or sensitive data in the POC needs recorded approval (approver and date).
* Don't commit customer data, credentials, workspace URLs or IDs to this repo.

## Feedback

Open an issue in this repo with the job shape (number of tasks and tables, sdp-meta or plain SDP), your cloud, and what happened.
