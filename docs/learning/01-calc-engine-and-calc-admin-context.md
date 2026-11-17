# 01. The real calc engine and Calc Admin, and where each concept applies

> Read time: 40 minutes. No hands-on steps. Purpose: every concept in the labs maps to a real thing in NAIC. Facts below come from the NAIC architecture pages, the DCM proof of concept branch (`origin/feature/dcm-poc`, tip `b77dc71`, read-only), the earlier explainer reports and the Calc Admin documents. Where a shape was rebuilt from documentation rather than seen as source, it says so.

Sources used (all read-only):

- `calc-engine/docs/calc-engine-complete-architecture.html` (called ARCH here), `calc-engine-pipeline.html` (PIPE), `calc-admin-with-flyway.html` (ADMINFW).
- DCM branch files: `dcm/calc_engine/manifest.yml`, `dcm/calc_engine/sources/definitions/`, `scripts/refresh_dev_clone.sh`, `sql/clone_support/calc_dev_clone_procs.sql`, `data_changes/`, `plans/dcm_strategy_working_notes.md` (NOTES).
- `Analysis/2026-11-12_dcm-and-flyway-explainer/report.md`, `Analysis/2026-11-11_gitlab-ci-for-snowflake-explainer/report.md`.
- Calc Admin: `calculations-workflow-audit-tables.md`, `02-dheeraj-hybrid-tables-changes.md`, ADR-0026.

## What the calc engine is

The calc engine computes NAIC financial ratios and totals inside Snowflake. Its key design choice: **calculations are stored as data, and stored procedures turn that data into SQL and run it.**

- **Calculations as rows.** Tables in the schema `COMMON_INT.CALCULATION_SUPPORT` hold the definition of each calculation: `CALCULATIONS` (one row per calc, with a status of `DRAFT`, `ACTIVE` or `INACTIVE`), `FORMULAS`, `MAPPINGS`, `MAPPING_DETAILS`, `MAPPING_JOINS`, `FORMULA_VARIABLES`, `FORMULA_CATEGORIES`, `CALC_OUTPUT_COLUMNS`, `CALC_MERGE_KEYS`, and others. Run history is in `EXECUTIONS`; promotion history in `PROMOTION_LOG`.
- **Stored procedures that generate SQL.** About 34 procedures in INT as of 2026-11-11, for example `SP_REGISTER_CALCULATIONS`, `SP_UPDATE_DRAFT_CALCULATION`, `SP_ACTIVATE_DRAFT_CALCULATION` (promotes a draft and retires the calc it replaces), `SP_EXTRACT_CALCULATION_JSON`, `SP_BUILD_SQL_TEMPLATE`. `SP_RUN_CALCULATION` builds and runs the SQL for one calc. Many procedures use `EXECUTE AS OWNER`, meaning they run with the rights of the role that owns them, not the caller.
- **Other objects.** 5 functions, 1 view (`ENV_CONFIG`), 9 sequences (`SEQ_CALC_ID`, `SEQ_EXECUTION_ID`, and so on), 24 tables in total in INT (NOTES section 30, inventory of 2026-11-11).
- **Results.** Calculation output goes to `NAIC_<tier>_FINANCIAL_EDL.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE` (tier such as `INT`). That schema belongs to another team (EDP), which already runs its own Flyway for it.
- **Calculation states and approvals.** The engine knows `DRAFT`, `ACTIVE`, `INACTIVE`. Calc Admin (the web app and service on top) adds its own review statuses, covered below.
- **Redirects.** Procedures contain INT names (for example `COMMON_INT.CALCULATION_SUPPORT.`). A function `FN_APPLY_REDIRECTS` rewrites those names at run time, using a map from the view `ENV_CONFIG`. In a clone the map points INT names to the clone, so a calc run in a clone writes to the clone. This is the reason clone-safe deployment needs templating (Lab 5).
- **Streamlit.** A Streamlit app `CALCULATION_ENGINE` runs inside INT from a Snowflake Git repository object. It is out of scope for the sandbox. DCM lists Streamlit as a preview entity type.

> **The standing rule:** on NAIC, nobody except the calc owner runs calculations. That is why the sandbox has its own harmless `SP_COMPUTE_MINI_RESULT`.

## Old way and drift

**The old way.** A developer edits a very large file, `sql/procedures.sql` (about 6,700 lines, 31 `CREATE OR REPLACE PROCEDURE` statements), then pastes it into a Snowsight worksheet and runs it against INT. There is no CI pipeline file yet. `CREATE OR REPLACE` is safe to repeat, but it silently overwrites whatever is live, including a newer version someone else pasted.

**Drift** means the live Snowflake objects no longer match the files in Git. Evidence found in NAIC (earlier explainer, labelled by finding):

- Live objects with no DDL on `main`: `SP_PROMOTE_METADATA`, `SP_TOP_STATES`, and tables such as `FIN_CALCULATION_RESULT_DRAFT`, `DYNAMIC_FUNCTION_CONFIGS`, `PROMOTION_LOG`.
- `main` was behind INT: it lacked the redirect code (`FN_APPLY_REDIRECTS`, `SP_GET_ENV_REDIRECTS`, `ENV_CONFIG`). NOTES section 30 warns that deploying from `main` would revert that work.
- Two Calc Admin hybrid tables, `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT`, were created by hand in INT on 2026-11-03 and have no DDL in Git. They do not exist in QA or PROD.
- Mixed ownership: some objects owned by `ACCOUNTADMIN`, some by `SYSADMIN`, most by `CICD_PIPELINE`. Clones broke until everything was owned by one role.
- No audit trail of who deployed what. NOTES section 19: the deploy audit comes from GitLab, not Snowflake.
- A hot fix in Snowsight would be reverted by the next deployment from files.

Lab 1 lets you create each of these pains yourself in miniature.

## Target architecture lanes

The architecture pages split responsibilities into lanes. The rule, in the pages' words, is that every object, migration, identity and artifact has one named owner and that no object has overlapping DCM and Flyway ownership.

| Lane | Owns | Does not own | Tool |
|---|---|---|---|
| Platform | Accounts, databases, schemas (containers), warehouses, service users, roles, integrations | Procedure bodies, migration data | Terraform |
| DCM Calc Engine project | Standard tables, SQL and Python routines, views, sequences, approved grants, ordinary tasks | Hybrid tables, Flyway history, row migrations, runtime-generated functions | `snow dcm` |
| Flyway | Calc Admin lifecycle schema, hybrid tables and indexes, `FIN_CALCULATION` migrations, explicit renames and type changes, data changes, clone support | Procedure bodies owned by DCM | Flyway |
| Artifact pipeline | Python package and Streamlit files, SBOM, scan evidence | Database schema | CI jobs |
| Runtime policy | Approved objects created at run time in a separate schema | Anything under DCM or Flyway | Policy |

**Status of DCM, stated plainly.** Both DCM and Flyway are being implemented. The three target pages (ARCH, PIPE, ADMINFW) describe both tools. A working note dated 2026-11-11 (NOTES section 30) recorded that the DCM strategy was paused after a meeting, and the INT DCM project object `COMMON_INT.CALCULATION_SUPPORT.CALC_ENGINE_DCM` was dropped. The team has since confirmed that DCM is part of the design, so the INT project is to be re-adopted. The roadmap in ARCH names the phases: 1 restore Git and INT parity, 2 DCM core re-adoption in INT (exit evidence: an adoption-only plan, unchanged data and grants, an empty re-plan), 3 Flyway boundaries and clone bootstrap, 4 a minimal merged-result pipeline plus a manual INT deploy.

A useful fact from that episode: dropping the DCM project object left every object, row and grant in place (34 procedures, 5 functions, 24 tables, 1 view, 9 sequences; `CALCULATIONS` had 1,324 rows before and after). You will demonstrate the same behaviour in Lab 3.

> **VERIFIED:** dropping only the DCM project object leaves managed objects in place as unmanaged, but deletes the deployment history (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage, read 2026-11-16).

**What the DCM proof of concept proved** (branch `feature/dcm-poc`, NOTES sections 18 to 25):

- Adopting 70 existing objects into a DCM project changed no data. First clone deploy: 70 entities, 0 create, 70 alter, 0 drop; 69 were adoption only and 1 was a real change.
- After a deploy, the next plan reported no changes. Editing one procedure planned as exactly one change.
- Hybrid tables cannot be defined in DCM: `DEFINE HYBRID TABLE` fails with `unexpected 'HYBRID'` (so the Flyway lane owns them).
- Grants survive edits. A clone refresh takes about 5 minutes. A calc group run in a clone wrote 267,427 result rows to the clone and 0 to INT (done by the team, not repeated in this pack).

**The Flyway contract in the target design** (ARCH and ADMINFW): Flyway runs in pipeline jobs, never at application startup. One history table per migration boundary and environment. `validateOnMigrate=true`, `outOfOrder=false`, `baselineOnMigrate=false` in normal use, `cleanDisabled=true`, no automatic repair. Applied versions are immutable (fix forward). DDL migrations are small because Snowflake commits DDL independently. Data migrations use explicit transactions and row-count assertions.

**There is no universal rollback.** Snowflake DDL commits independently, and neither the DCM definitions nor Flyway history restore data.

## Who owns what

| Change type | Owner | Why |
|---|---|---|
| New database or schema | Platform (Terraform) | Container and access decisions |
| New or edited standard table, view, sequence, function | DCM | Desired-state diff and plan are a fit |
| Procedure body edit | DCM | Whole-object replace shown in the plan |
| Hybrid table, secondary index | Flyway | DCM does not support hybrid tables |
| Table rename, type change | Flyway (explicit), then DCM definition updated | DCM would see rename as drop plus create |
| Row data fix | Flyway (ledger and row-count assertion) | Neither DCM nor DDL handles rows |
| Grants on engine objects | DCM (`GRANT` statements) | Declared and reapplied |
| Grants on Flyway objects | Flyway migration | Same lane as the table |
| Clone support procedures | `snow sql` deployment (clone support), outside DCM and Flyway in the sandbox | Owner-rights procedures that create databases |

The ordering rule for normal changes: DCM plan and deploy first, then Flyway migrate, then a DCM re-plan that must be empty. For a rename: Flyway first, then the DCM definition.

### Calc Admin and its lifecycle tables

Calc Admin is the web UI and service where people draft, review and activate calculations. Its facts:

- **Live today (INT).** Two hybrid tables in the engine schema, created by hand on 2026-11-03, empty at creation, owner `CICD_PIPELINE`. `CALC_WORKFLOW_STATUS` has one row per calc with an enforced primary key `CALC_ID` and a status of `DRAFT`, `IN_REVIEW`, `APPROVED`, `ACTIVE` or `DISCARDED`. `CALC_AUDIT_EVENT` is append-only, with an enforced primary key `EVENT_ID` that defaults from a sequence `CALC_AUDIT_EVENT_SEQ`. The service role `FN_NAIC_CALC_ENGINE_NP` holds `INSERT`, `UPDATE`, `DELETE` on the status table and only `INSERT` on the audit table. The column list in the pack is reconstructed from documentation, not from the original DDL, which was not found in any repository.
- **A grant bug that happened.** The service role could not see the sequence until `USAGE` was granted. An insert that omits `EVENT_ID` fails without it. Lab 7 reproduces this.
- **Action-first audit.** Today the service calls the engine first and writes one audit row afterwards. If the audit write fails, an action happened without a record.
- **Target design (ADR-0026, status Proposed 2026-10-01).** The unit of review becomes the calc set, not a single calc. Five tables and two sequences: `CALC_REVISION`, `CALC_REVISION_MEMBER`, `CALC_VERSION`, `CALC_AUDIT_EVENT` (standard) and `CALC_OPEN_REVISION` (the only hybrid, enforcing one open revision per calc set and year and holding a row lock). Audit becomes **event-first**: write a `PENDING` event, call the engine, then complete the event once with the outcome. ADR-0026 rejects making every table hybrid because hybrid tables give up Fail-safe, UNDROP, replication and full Time Travel, which matter most for history.
- **Open tension.** Event-first needs one `UPDATE` on the audit event (`PENDING` to outcome), while the live grants allow insert and select only. Snowflake cannot narrow `UPDATE` to a column or to a pending row. Lab 7 shows both models without choosing a side. The choice between the two-hybrid-table branch design and the single-hybrid ADR design is an open stakeholder decision.
- **Why Flyway for it.** The old Postgres datasource was removed (US-67511), so there is no migration tool for the Snowflake lifecycle tables. DCM does not support hybrid tables. Flyway fits: versioned scripts, history table, checksum.
- **Baseline story.** INT already has the hand-made tables, so INT is baselined after verifying the live shape equals the scripts, and QA and PROD run the scripts for real (baseline needs an ADR or runbook approval because `baselineOnMigrate` stays false).
- **Why not run Flyway at app startup.** The target says application pods start after the migration job and a compatibility gate succeed; runtime only validates the schema version and never alters it. The Postgres-based `fpr-admin-service` does the opposite (Spring Boot runs Flyway at start with the app's own datasource, `out-of-order=true`, and `repair-on-migrate=true` in dev). Lab 7 contrasts the two.

### The existing Flyway at NAIC (EDP)

EDP already runs Flyway for `NAIC_<tier>_FINANCIAL_EDL.FIN_CALCULATION`, with a history table `flyway_bus_history_table` and a GitLab template. Concerns recorded by the earlier explainer: the Flyway image is tagged `latest` and the template follows `master` (both unpinned). Lab 6 covers pinning.

### CI/CD, as planned

The GitLab pipeline for the calc engine does not exist yet. The target (PIPE): static checks, a review clone per merge request, Flyway info and validate, a full DCM plan with an exact DROP policy, deploy, empty re-plan, smoke tests, cleanup that always runs, then a protected manual INT deploy, later QA and PROD with change approval. Identity target: GitLab OIDC to Snowflake workload identity federation, separate identities for review, INT, QA, PROD. Supply chain target: mirrored, version-pinned images, no `latest`, no downloads at job run time. Lab 6 rebuilds this on GitHub Actions.

## Real to miniature table

The sandbox is a small copy of the same shapes. Names are fixed so later labs and your assistant find them.

| Real thing in NAIC | Miniature in the sandbox | Lab where you meet it |
|---|---|---|
| Database `COMMON_INT` | `SBX_INT` | 02 (created), 03 |
| Schema `CALCULATION_SUPPORT` | `CALC_SUPPORT` (in `SBX_INT`, `SBX_QA`, `SBX_PROD`) | 03 |
| `NAIC_INT_FINANCIAL_EDL` (results, EDP) | `SBX_INT_FIN` with `FIN_CALCULATION` | 02, 07 |
| DCM project `CALC_ENGINE_DCM` | `CALC_ENGINE_DCM` (one per database) | 05, 06, 07 |
| Role `CICD_PIPELINE` | `SBX_CICD_PIPELINE` | 02, 03 |
| Role `COMMON_INT_CALC_DEV` | `SBX_CALC_DEV` | 02, 07 |
| Role `FN_NAIC_CALC_ENGINE_NP` | `SBX_FN_CALC_ENGINE_NP` | 02, 09 |
| Clone database `CALC_DEV_<ticket>` | `SBX_DEV_<TICKET>` | 07, 08 |
| Procedures `CREATE_CALC_DEV_CLONE`, `DROP_CALC_DEV_CLONE` | same names in `SBX_INT.CALC_DEV_SUPPORT` | 07 |
| View `ENV_CONFIG`, function `FN_APPLY_REDIRECTS` | same names | 05, 07 |
| Hybrid tables `CALC_WORKFLOW_STATUS`, `CALC_AUDIT_EVENT` | same names (hybrid in Mode H, standard stand-ins in Mode S) | 06, 09 |
| Ledger table `DATA_CHANGE_LOG` and `data_changes/V...` scripts | `DATA_CHANGE_LOG` and Flyway migration `V3` | 06 |
| `SP_RUN_CALCULATION` (never called on NAIC) | `SP_COMPUTE_MINI_RESULT` | 07 |
| `SP_REGISTER_CALCULATIONS` | `SP_REGISTER_CALCULATION` | 03, 05 |
| `SP_ACTIVATE_DRAFT_CALCULATION` | `SP_ACTIVATE_DRAFT_CALCULATION` (mini) | 05, 09 |
| `SP_ADD_CATEGORY` | `SP_ADD_CATEGORY` | 03, 05 |
| Tables `CALCULATIONS`, `FORMULAS`, `FORMULA_CATEGORIES`, `EXECUTIONS` | same names, few columns | 03 onward |
| Sequences `SEQ_CALC_ID` and others | `SEQ_CALC_ID`, `SEQ_FORMULA_ID`, `SEQ_EXECUTION_ID`, `SEQ_WORKFLOW_ID`, `SEQ_AUDIT_EVENT_ID` | 03, 05, 06 |
| `scripts/refresh_dev_clone.sh` | `scripts/refresh_dev_clone.sh` | 07 |
| Planned GitLab pipeline | Six GitHub workflows | 08 |
| Terraform lane | `snowflake-sandbox-bootstrap.sql` | 02 |
| EDP Flyway for `FIN_CALCULATION` | `FIN_CALCULATION_RESULT_BASE` created by the bootstrap | 02, 07 |
| Streamlit `CALCULATION_ENGINE` | Out of scope (mentioned only) | none |
| Manifest targets `INT` and `CLONE` with `calc_db` templating | `dcm/calc_engine/manifest.yml` with `INT`, `QA`, `PROD`, `CLONE` | 05, 07 |
| Safety placeholder `CALC_DEV_SET_ON_CLI` | `SBX_DEV_SET_ON_CLI` (must never exist) | 07, 10 |

## Reading list (the real sources)

In this order, if you want to read the originals later:

1. `calc-engine/docs/calc-engine-complete-architecture.html`: sections on lanes, deployment ownership, clones, rename protocol, failure drills.
2. `calc-engine/docs/calc-engine-pipeline.html`: the job flow and pipeline types.
3. `calc-engine/docs/calc-admin-with-flyway.html`: the lifecycle schema, event-first bracket, Flyway rules.
4. `Analysis/2026-11-12_dcm-and-flyway-explainer/report.md`: plain-language explainer with evidence labels.
5. Branch `origin/feature/dcm-poc`: `dcm/calc_engine/manifest.yml`, `sql/clone_support/calc_dev_clone_procs.sql`, `scripts/refresh_dev_clone.sh`, `plans/dcm_strategy_working_notes.md`.
6. `Calc-admin/calculations-workflow-audit-tables.md` and `calc-admin-gov` ADR-0026.
7. `Fpr-admin/fpr-admin-service/src/main/resources/db/migration/` and `application.properties` for the Postgres Flyway precedent.

## Open items for file 13

TC ids used in this file: none. All claims are from NAIC sources read by the authors or from the Snowflake page cited.

## Next

Go to [02-accounts-and-setup.md](02-accounts-and-setup.md).
