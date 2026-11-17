# Glossary and reference (file 12)

> **File no:** 12 | **Kind:** reference, no lab | **Time:** use as needed | **Prerequisites:** none | **Read date for public docs facts:** 2026-11-16 | **Modes:** all

This file holds the glossary, cheat sheets per tool, a troubleshooting table and links to the public documentation. Facts marked TO-CONFIRM were not verified against a public page; the full list is in file 13.

Convention for this pack: each term below is also defined where it first appears in a lab. "In NAIC" ties a term to the real calc-engine and calc-admin work. NAIC names appear only as they are named in NAIC's own design pages.

## 1. Glossary

| Term | Meaning | In NAIC or in the sandbox |
|---|---|---|
| Account identifier | The name of a Snowflake account in the form `<organization>-<account>`. It builds the host `<ACCOUNT_IDENTIFIER>.snowflakecomputing.com`. | Always written as a placeholder in this pack. |
| `ACCOUNTADMIN` | The most powerful Snowflake role. Use it rarely, never as a default role. Only it can create resource monitors. | Bootstrap and the cost drill only. |
| Action-first audit | Do the action, then write the audit row. A failure between the two leaves an unaudited change. | Calc Admin today. Contrast with event-first. |
| Adoption | Putting an existing object under a tool's control without changing it. For DCM: write a matching `DEFINE`, plan, deploy. For Flyway: `baseline`. | DCM re-adoption of INT objects; baseline of the hand-made lifecycle tables. |
| Alias (DCM) | A human name for a deployment, such as `int-a1b2c3d`. | Pipeline deployments use `<env>-<short-sha>`. |
| Append-only | A table that is only added to, never edited or deleted by its writers. Enforced by grants. | `CALC_AUDIT_EVENT` for the app role. |
| `AUTO_SUSPEND` | Seconds of idleness after which a warehouse stops and stops costing credits. | 60 on `SBX_CALC_WH`. |
| Baseline (Flyway) | Telling Flyway that the database is already at version N. It records a marker and runs nothing. | Lab 2 and Lab 7. |
| Blast radius | How much damage a mistake can do. | Labs keep it at "one throwaway clone". |
| Callback (Flyway) | A script Flyway runs at lifecycle events. Limited in Community. | Not used. |
| Changeset (DCM plan) | The list of CREATE, ALTER and DROP entries a plan or deploy would apply. Empty means in sync. | Read by `check_plan_for_drops.py`. |
| Checksum | A number computed from a file. Flyway stores it and compares later. | Edited applied migrations fail `validate`. |
| CI/CD | Automatic checks (CI) and repeatable delivery (CD) driven by a pipeline. | GitLab at NAIC, GitHub Actions in the sandbox. |
| Clone (zero-copy) | A copy of a database, schema or table that shares storage with the source until data diverges. | `SBX_DEV_<TICKET>` ticket environments. |
| Concurrency group | GitHub's way to allow only one job at a time for a name. | The `resource_group` idea of GitLab. |
| Credit | The unit Snowflake bills compute in. A warehouse uses credits while running. | Trial: about $400 of free usage for 30 days. |
| DCM | Database Change Management. Snowflake's declarative way to describe objects and apply the difference. | Owns supported engine objects. |
| DCM project object | A schema-level Snowflake object that holds deployments and history for one environment. | `CALC_ENGINE_DCM` in each database. |
| DDL auto-commit | Snowflake commits DDL statements as they run, so they cannot be rolled back with the rest of a file. | Why half-failed migrations leave partial changes. |
| `DEFINE` | The statement used in a DCM definition file. It behaves like `CREATE OR ALTER`. | One object per file. |
| Deployment (DCM) | One applied change set, stored with its rendered definitions. Named `DEPLOYMENT$<n>`, optionally with an alias. | `SHOW DEPLOYMENTS IN DCM PROJECT`. |
| Digest (image) | An immutable content hash of a container image, written `sha256:...`. | Pins the Flyway image. |
| Drift | Live objects differ from what the files say, because someone changed them by hand. | Found by a full DCM plan, not by Flyway Community. |
| Drop check | A script that fails a pipeline if a plan contains a DROP. | `scripts/check_plan_for_drops.py`. |
| Environment (GitHub) | A named deployment target with protection rules and its own secrets. | `int`, `qa`, `prod`, `pr-clone`. |
| `EXECUTE AS OWNER` | A stored procedure runs with the rights of its owner, not the caller. | Clone procedures and engine procedures. |
| Event-first audit | Write a `PENDING` event, do the action, write the outcome. | ADR-0026 design; Lab 7 miniature. |
| Fail closed | A check that cannot decide treats that as a failure. | The helper scripts. |
| Fail forward | After applying a migration, fix problems by adding a new one, never by editing the old one. | Flyway rule. |
| Flyway | A migration tool that runs numbered SQL files in order and records them in a history table. | Owns hybrid tables, renames, data changes. |
| Flyway history table | The table in which Flyway records every migration. Default name `flyway_schema_history`. | `FLYWAY_HISTORY_CALC_SUPPORT` in the main project. |
| Hybrid table | A row-store table with an enforced primary key, row-level locks and secondary indexes. Not in trial accounts. | `CALC_WORKFLOW_STATUS`, `CALC_AUDIT_EVENT`. |
| `IGNORE HYBRID TABLES` | A clause on a schema clone that skips hybrid tables, which cannot be cloned at schema level. | Clone procedure. |
| Idempotent | Running it twice has the same result as running it once. | Data fixes use `WHERE ... <> new value`. |
| Index (secondary) | An extra structure that speeds lookups on columns. On hybrid tables created in `CREATE` or with `CREATE INDEX`. | `IDX_AUDIT_EVENT_TYPE`. |
| Jinja | The template language DCM uses in definition files (`{{var}}`, `{% if %}`). | `{{calc_db}}`. |
| JWT | A signed token. Key-pair login sends one. | Error 390144. |
| Key pair | A private key you keep and a public key you register with Snowflake. | Service user login. |
| Ledger (data changes) | A table listing the data changes applied. | `DATA_CHANGE_LOG`. |
| Lock timeout | How long a statement waits for a lock before failing. | Lab 7 sets 20 seconds. |
| Manifest (`manifest.yml`) | The file that names a DCM project's targets, owners and templating values. | `dcm/calc_engine/manifest.yml`. |
| Migration | A numbered SQL file that moves a database one step. | `V1` to `V8`. |
| Mode H, S, D | Pack flags: H hybrid tables available, S standard stand-ins, D DCM available (`D-off` if not). | Chosen in file 02. |
| MFA | Multi-factor authentication. Required for human sign-in in most accounts. | See file 02 and TC-G19. |
| OIDC | A standard for short-lived identity tokens. GitHub can issue one per job. | Optional stretch of Lab 6. |
| Out-of-order | Applying a migration whose number is lower than one already applied. Off by default. | `outOfOrder = false`. |
| Ownership | The one role that can alter, drop and grant on an object. | Pipeline role owns engine objects. |
| Pending (audit) | An event whose outcome has not been written. | Reconcile later. |
| Placeholder (Flyway) | A `${name}` token replaced inside migrations. | `${calc_db}`, `${app_role}`. |
| Plan (DCM) | The computed difference between definitions and live state. Executes no DDL. | `snow dcm plan`. |
| `PLAN DELTA` | A plan limited to changed definitions and what depends on them. Does not detect drift. | `snow dcm plan --delta`. |
| Pipeline | The automated sequence run on a change. | Six GitHub workflows. |
| Pinning | Naming an exact immutable version of a tool, image or action. | SHAs, tags and digests. |
| Project owner role | The role a DCM project runs as and that owns what it deploys. | `SBX_CICD_PIPELINE`. |
| Purge (DCM) | Drops everything a project manages. Non-production only. | Capstone cleanup. |
| Re-plan | A plan run right after a deploy. It must be empty. | Every pipeline ends with it. |
| Redirects | A map that rewrites source names to clone names inside procedures at run time. | `ENV_CONFIG`, `FN_APPLY_REDIRECTS`. |
| Repair (Flyway) | Removes failed history rows and realigns checksums to the files. | Used with care in Lab 8. |
| Repeatable migration | `R__<name>.sql`, re-applied whenever its checksum changes. | Procedure files in way 2. |
| Resource monitor | A credit budget with triggers that notify or suspend a warehouse. | `SBX_CALC_RM`. |
| Role | A set of privileges a user can use. | Four sandbox roles. |
| Runner | The machine that executes a workflow job. | `ubuntu-24.04`. |
| Secret (GitHub) | An encrypted value exposed to workflows, hidden in logs. | Private key, passphrase, account. |
| Service user | A non-human Snowflake user (`TYPE = SERVICE`). It cannot use a password. | `SBX_SVC_CICD`. |
| Snowflake CLI (`snow`) | The command-line tool for Snowflake. DCM commands need 3.24.0 or higher. | Pinned at 3.33.0. |
| Supply chain | Everything your pipeline downloads and runs. | Pinned and mirrored. |
| Target (DCM) | A named entry in the manifest: account, project, owner role, template set. | `INT`, `QA`, `PROD`, `CLONE`. |
| Templating | Filling variables into definition files per environment. | `calc_db`, `app_role`. |
| Terraform | Infrastructure as code. At NAIC it owns accounts, databases, schemas, roles, warehouses and integrations. | Replaced in the sandbox by bootstrap SQL. |
| Ticket | The short id of a piece of work, here the clone name suffix. | `LAB5A`, `PR12`. |
| Time Travel | Query or restore old data within a retention period. | Clones have 0 days. |
| TOML | The configuration format of `flyway.toml`. | Replaces `flyway.conf`. |
| Trial account | Snowflake's free first account: 30 days or until the credits are used. | No hybrid tables. |
| Unmanage | Make DCM stop managing an object without dropping it. | `SYSTEM$UNMANAGE_...` functions. |
| `validate` (Flyway) | Compares applied migrations with the files and fails on mismatch. | Run in `validate.yml`. |
| Versioned migration | `V<number>__<description>.sql`, applied once, in order. | `V1`, `V2`, and so on. |
| Warehouse | The compute that runs queries and costs credits while running. | `SBX_CALC_WH`. |
| Workload identity federation | Letting a platform's identity token log in to Snowflake with no stored key. | NAIC target. |

## 2. Cheat sheets

Each sheet lists only syntax seen on a public page (marked VERIFIED with the date) or taken from the sandbox files. Anything else is marked TO-CONFIRM.

### 2.1 `snow dcm` (Snowflake CLI 3.24.0 or higher)

> **VERIFIED:** DCM Projects need Snowflake CLI 3.24.0 or higher for the commands and options below; the CLI command list is `create, deploy, describe, drop, drop-deployment, list, list-deployments, plan, preview, purge, test` (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use and .../snowflake-cli/command-reference/dcm-commands/overview, read 2026-11-16).

| Task | Command |
|---|---|
| Version | `snow --version` |
| Test a connection | `snow connection test -c <name>` or with env variables `snow connection test -x` |
| Create the project object | `snow dcm create <db.schema.project> --target <T> --from <dir> [--if-not-exists]` |
| Plan | `snow dcm plan <project> --target <T> --from <dir> [-D "key='value'"] [--save-output] [--delta]` |
| Deploy | `snow dcm deploy <project> --target <T> --from <dir> [-D "key='value'"] [--alias <alias>] [--save-output]` |
| Variables | `-D "name='text'"` or `--variable`. Strings need inner single quotes, lists need brackets. Order of precedence: defaults, configuration, runtime. |
| Re-plan or re-deploy an old deployment | `--deployment <name_or_alias>` (quote `$` in `'DEPLOYMENT$1'`) |
| List and describe | `snow dcm list [--in database <db>]`, `snow dcm describe <project>` |
| Run expectations | `snow dcm test <project> --target <T> --from <dir>` (exit 0 pass, 1 fail) |
| Output files | `--save-output` recreates `out/` under the `--from` folder each time (file name TC-G04) |
| Role and connection | Always pass `-c <connection>` and `--role <role>`. Targets do not switch them. |
| Other commands | `preview`, `purge`, `drop`, `drop-deployment`, `list-deployments`: run `snow dcm <cmd> --help` (TC-G26) |

SQL forms (run in a worksheet):

```sql
CREATE DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;
SHOW DCM PROJECTS IN SCHEMA SBX_INT.CALC_SUPPORT;
SHOW DEPLOYMENTS IN DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;
SHOW ENTITIES IN DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;
SHOW GRANTS IN DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;
EXECUTE DCM PROJECT SBX_INT.CAP_DCM.CAP_DCM_PROJECT PURGE;       -- drops everything it manages, non-production only
DROP DCM PROJECT IF EXISTS SBX_INT.CAP_DCM.CAP_DCM_PROJECT;      -- objects stay, deployment history is deleted
GRANT CREATE DCM PROJECT ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CICD_PIPELINE;
GRANT PLAN ON DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM TO ROLE SBX_CALC_READER;
SELECT SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT('<project>', '<TYPE>', '<fqn>');          -- TC-10-01 for exact use
```

Privilege ladder on the project object: `READ` < `MONITOR` < `PLAN` < `OWNERSHIP`. `PLAN` runs as the project owner's role.

Definition file rules: only `DEFINE`, `GRANT`, `ATTACH`; one object per file; fully qualified names; each object defined once across all files; `/out/` in `.gitignore`; not supported in Jinja: `import`, `extends`, `include`.

Example shapes (grammar for procedures and sequences: TC-G03):

```sql
DEFINE TABLE SBX_INT.CAP_DCM.CALCULATIONS (CALC_ID NUMBER(38,0) NOT NULL, CALC_NAME VARCHAR(255), primary key (CALC_ID));
DEFINE SEQUENCE {{calc_db}}.{{calc_schema}}.SEQ_CALC_ID start with 2001 increment by 1 noorder;
GRANT SELECT ON TABLE {{calc_db}}.{{calc_schema}}.CALCULATIONS TO ROLE {{reader_role}};
```

Manifest keys: `manifest_version: 2`, `type: DCM_PROJECT`, `default_target`, `targets.<NAME>.{account_identifier, project_name, project_owner, templating_config}`, `templating.defaults`, `templating.configurations.<NAME>`.

### 2.2 Flyway (Community, 13.12.2)

> **VERIFIED:** Current stable 13.12.2 (2026-11-12). Community includes migrate, info, validate, baseline, repair, clean. Undo and dry run are Teams. Drift detection is Enterprise (https://documentation.red-gate.com/fd/feature-summary-165740620.html and .../flyway/reference/commands, read 2026-11-16).

| Task | Command |
|---|---|
| Version | `flyway -v` (Docker: `docker run --rm flyway/flyway:13.12.2 -v`) |
| Show status | `flyway info` |
| Check applied vs files | `flyway validate` |
| Apply pending | `flyway migrate` |
| Mark existing database | `flyway baseline -baselineVersion=<n> -baselineDescription="<text>"` |
| Fix history | `flyway repair` (same locations as `migrate`) |
| Drop everything | `flyway clean` (needs `cleanDisabled = false`, never in shared environments) |
| Run in Docker | `docker run --rm -v /abs/path/flyway:/flyway/project flyway/flyway:13.12.2 -workingDirectory=project migrate` |

Naming: `V<version>__<description>.sql` (two underscores), `R__<description>.sql`, `U<version>__...` is Teams. Versions may be `1`, `2`, `3_1`. Pack rule: lower-case snake_case descriptions.

Settings (TOML, case-sensitive; env var is `FLYWAY_<UPPER_SNAKE>`):

| Setting | Default | Pack value |
|---|---|---|
| `cleanDisabled` | `true` | `true` (set `false` only in a scratch project) |
| `validateOnMigrate` | `true` | `true` |
| `outOfOrder` | `false` | `false` |
| `baselineOnMigrate` | `false` | `false` |
| `table` | `flyway_schema_history` | `FLYWAY_HISTORY_CALC_SUPPORT` |
| `defaultSchema`, `schemas` | | `CALC_SUPPORT` |
| `locations` | | `filesystem:migrations/common,...hybrid` or `...standard` |
| `placeholders` | | `calc_db`, `calc_schema`, `app_role`, `reader_role` |

JDBC URL: `jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=<DB>&warehouse=<WH>&role=<ROLE>`. `db` and `warehouse` are mandatory. Key pair: add `&authenticator=snowflake_jwt&private_key_file=<absolute path>`.

Not verified here: `-outputType=json` (TC-G11), `-target=<version>` (TC-08-06), `-ignoreMigrationPatterns='*:pending'` (TC-08-10), default `baselineVersion` (TC-G09), per-environment placeholders in TOML (TC-G10), the JVM flag for encrypted keys (TC-G21).

Flyway on Snowflake: DDL commits immediately, so a failing multi-statement file leaves earlier statements applied. Parallel migrations are not available.

### 2.3 `snow sql`, connections and key pairs

```bash
snow connection test -c sbx_dev                                   # named connection from ~/.snowflake/connections.toml
snow sql -c sbx_dev -q "SELECT CURRENT_ROLE()"                     # one statement
snow sql -c sbx_dev -f scripts/smoke_clone.sql --database SBX_DEV_LAB5A --schema CALC_SUPPORT
snow sql -x -q "SELECT 1" --format json                            # temporary connection from environment variables
```

> **VERIFIED:** Environment variables for a temporary connection (`-x`): `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, `SNOWFLAKE_ROLE`, `SNOWFLAKE_WAREHOUSE`, `SNOWFLAKE_DATABASE`, `SNOWFLAKE_SCHEMA`, `SNOWFLAKE_AUTHENTICATOR`, `SNOWFLAKE_PRIVATE_KEY_FILE`, `SNOWFLAKE_PRIVATE_KEY_RAW`, `SNOWFLAKE_TOKEN`. The key passphrase comes from `PRIVATE_KEY_PASSPHRASE` (https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections, read 2026-11-16). `connections.toml` wins over `config.toml` for connections.

Key pair (VERIFIED, https://docs.snowflake.com/en/user-guide/key-pair-auth, read 2026-11-16):

```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes-256-cbc -inform PEM -out sbx_svc_cicd_rsa_key.p8      # encrypted, asks for a passphrase
openssl rsa -in sbx_svc_cicd_rsa_key.p8 -pubout -out sbx_svc_cicd_rsa_key.pub
openssl rsa -pubin -in sbx_svc_cicd_rsa_key.pub -outform DER | openssl dgst -sha256 -binary | openssl enc -base64     # local fingerprint
```

```sql
ALTER USER SBX_SVC_CICD SET RSA_PUBLIC_KEY = '<PUBLIC_KEY_BODY>';      -- body only, no BEGIN/END lines
ALTER USER SBX_SVC_CICD SET RSA_PUBLIC_KEY_2 = '<PUBLIC_KEY_BODY>';    -- second slot for rotation
ALTER USER SBX_SVC_CICD UNSET RSA_PUBLIC_KEY;
DESC USER SBX_SVC_CICD;                                                 -- RSA_PUBLIC_KEY_FP
```

The newer named form is `ALTER USER <u> ADD KEY PAIR <name> PUBLIC_KEY = '...'`, with `SHOW USER KEY PAIRS`, `ROTATE KEY PAIR`, `REMOVE KEY PAIR`. Never put a private key in the repository, in a chat, or in a file under version control.

### 2.4 Clones

```sql
CREATE DATABASE SBX_DEV_LAB5A CLONE SBX_INT DATA_RETENTION_TIME_IN_DAYS = 0;                       -- database clone
CREATE SCHEMA SBX_DEV_LAB5A.CALC_SUPPORT CLONE SBX_INT.CALC_SUPPORT IGNORE HYBRID TABLES;          -- schema clone without hybrid tables
CALL SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE('LAB5A');                                      -- guarded: ticket 1 to 40 letters or digits
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('LAB5A');                                        -- only SBX_DEV_* can be dropped
```

> **VERIFIED:** Hybrid tables can be cloned only at database level; a schema clone containing hybrid tables fails unless `IGNORE HYBRID TABLES` is appended (and the clause also works on database clones); `CREATE HYBRID TABLE ... CLONE` fails with `391411 (0A000): This feature is not supported for hybrid tables: 'CLONE'.` (https://docs.snowflake.com/en/user-guide/tables-hybrid-clone, read 2026-11-16). Clones lose the Time Travel history of the source.

Harmless negative tests for the guarded procedures: `'SBX_INT.X'`, `'NOPE; SELECT 1'`, `''`, `NULL`, a 41-letter string. All must be rejected before any SQL is built. Never test with a real database name.

Facts to remember: sequences in a clone continue from the source values; clones keep the source objects' owners, so ownership must be uniform before hand-off; the cloned Flyway history can claim a migration ran whose hybrid table was not cloned.

### 2.5 Grants and roles

```sql
GRANT ROLE SBX_CALC_DEV TO ROLE SBX_CICD_PIPELINE;                                                 -- role hierarchy (inheritance)
GRANT USAGE ON WAREHOUSE SBX_CALC_WH TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT USAGE ON DATABASE SBX_INT TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT USAGE ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT SELECT, INSERT ON TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT TO ROLE SBX_FN_CALC_ENGINE_NP;   -- append-only for the app
GRANT USAGE ON SEQUENCE SBX_INT.CALC_SUPPORT.SEQ_AUDIT_EVENT_ID TO ROLE SBX_FN_CALC_ENGINE_NP;      -- needed for a sequence default
GRANT OWNERSHIP ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
SHOW GRANTS TO ROLE SBX_FN_CALC_ENGINE_NP;
SHOW GRANTS ON TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT;
```

Notes: inside a DCM project, `GRANT OWNERSHIP` cannot use `COPY CURRENT GRANTS` or `REVOKE CURRENT GRANTS` and fails if the object has other grants not in the project (TC-G25). `CREATE HYBRID TABLE` is now a separate schema privilege; schema owners do not need it (`GRANT CREATE HYBRID TABLE ON SCHEMA ...`, TC-G14). A `CREATE OR REPLACE` on a procedure drops its grants. Bulk `GRANT OWNERSHIP ON ALL ... IN SCHEMA` needs `MANAGE GRANTS` (error 3001 in NAIC work).

Resource monitor (VERIFIED, https://docs.snowflake.com/en/sql-reference/sql/create-resource-monitor, read 2026-11-16):

```sql
USE ROLE ACCOUNTADMIN;
CREATE RESOURCE MONITOR SBX_CALC_RM WITH CREDIT_QUOTA = 20 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 50 PERCENT DO NOTIFY ON 80 PERCENT DO NOTIFY ON 90 PERCENT DO SUSPEND ON 100 PERCENT DO SUSPEND_IMMEDIATE;
ALTER WAREHOUSE SBX_CALC_WH SET RESOURCE_MONITOR = SBX_CALC_RM;
```

### 2.6 GitHub Actions snippets

```yaml
concurrency: { group: deploy-int, cancel-in-progress: false }      # one deploy at a time, never kill a running one
environment: int                                                    # waits for required reviewers
permissions: { contents: read }                                     # least privilege; add id-token: write only for OIDC
uses: snowflakedb/snowflake-actions@<FULL_SHA>                      # pin by commit SHA; resolve at lab time (TC-G22)
if: always()                                                        # cleanup runs after failure or cancel
```

### 2.7 Git, for you, the learner

The assistant never runs `git commit`, `git push`, `git pull` or `git fetch` unless you name that exact action in that request. The commands you use:

```bash
git status                       # safe, read only
git switch -c feature/LAB6A      # new branch
git add <files>                  # stage by name, not by dot
git commit -m "ci: add workflows"
git push -u origin feature/LAB6A
```

Branch names for tickets match `feature/<ticket>` where the ticket is 1 to 40 letters or digits.

## 3. Troubleshooting table

Errors seen in NAIC calc-engine work, plus the sandbox ones. A message is quoted only where a public page gives it. Otherwise the cause is described in plain words and the exact text is TO-CONFIRM. "NAIC" in the source column means the DCM branch notes or the clone procedure file (no real account or host names are reproduced here).

| Symptom or code | Where you see it | Cause | Fix or lesson | Source and status |
|---|---|---|---|---|
| TLS certificate error when `snow dcm` uploads files | Laptops on a corporate network with TLS inspection | The inspection proxy re-signs traffic to the Snowflake stage storage endpoint with a certificate authority that lacks a Key Usage field. Python 3.13 and later rejects it. Login traffic is not inspected. | Use a Python 3.12 environment for `snow`; point `REQUESTS_CA_BUNDLE` and `SSL_CERT_FILE` at the corporate CA bundle; ask the network team for a bypass or a reissued CA. Does not apply to a home laptop. | NAIC notes. VERIFIED internally, not a public-docs fact. |
| `unexpected 'HYBRID'` | `snow dcm plan` or `raw-analyze` | A `DEFINE HYBRID TABLE` is in the project. Hybrid tables are not in DCM's supported list. | Keep hybrid tables in Flyway. The `validate.yml` lint greps for it. | NAIC notes; docs list VERIFIED. |
| Error 392105 | `CREATE SCHEMA ... CLONE` | The schema contains hybrid tables and cannot be schema-cloned. | Add `IGNORE HYBRID TABLES` immediately after the clone source. Recreate the tables empty. | NAIC clone file. |
| `391411 (0A000): This feature is not supported for hybrid tables: 'CLONE'.` | `CREATE HYBRID TABLE ... CLONE` | Table-level clone of a hybrid table. | Clone the database, or skip hybrid tables and recreate them. | Public docs, VERIFIED 2026-11-16. |
| Error 1046 | `GET_DDL` inside a procedure | `GET_DDL` needs a constant object name. | One call per table with a literal name. | NAIC clone file. |
| Error 3001 | `GRANT OWNERSHIP ON ALL ... IN SCHEMA` | Bulk ownership needs `MANAGE GRANTS`. | Grant ownership object by object from `SHOW ... IN SCHEMA`. | NAIC notes. |
| Error 3645 (BCR-2395) | Ownership transfer inside an owner's-rights procedure | Ownership can go only to a role active in the session or by a role with `MANAGE GRANTS`. | Grant the developer role to the owner role (inheritance), as bootstrap grant 1 does. | NAIC notes. |
| Error 2003 | Granting on built-in procedures | `SHOW PROCEDURES IN SCHEMA` lists Snowflake built-ins. | Filter on `is_builtin = 'N'`. | NAIC clone file. |
| Error 1510 | `snow dcm plan -D calc_db=...` | A string template value has no SQL quotes. | `-D "calc_db='SBX_DEV_LAB5A'"`. | NAIC notes. |
| Plan fails: database `SBX_DEV_SET_ON_CLI` does not exist | `--target CLONE` without `-D` | The safety placeholder database does not exist on purpose. | Pass `-D "calc_db='...'"` and `-D "fin_db='...'"`. Exact text TC-10-05. | Pack design, NAIC notes. |
| Clone creation fails or hands over wrongly | Clone procedure | Objects in INT are owned by a role other than the owner role (for example created by `SYSADMIN`). | Keep every engine object owned by `SBX_CICD_PIPELINE`; create objects with that role active. | NAIC notes. |
| "Unsupported statement type USE" when running a procedure in a clone | Owner's-rights procedure | The call passed a warehouse that made the procedure run `USE WAREHOUSE`. | Set the warehouse on the session, not in the parameters. | NAIC notes. |
| Role cannot create a DCM project | `CREATE DCM PROJECT` | Role lacks `CREATE DCM PROJECT` on the schema. | `GRANT CREATE DCM PROJECT ON SCHEMA ... TO ROLE ...` (owners do not strictly need it). | Public docs VERIFIED; NAIC notes. |
| DCM ownership errors (cannot adopt, plan or deploy fails before changes) | `plan` / `deploy` | The project owner role must own the objects or hold the role that does. Objects owned by another role cannot be adopted until ownership is transferred or the roles are connected. Exact text not recorded. | Grant the other role to the project owner role and add `GRANT OWNERSHIP` in the definitions, or transfer ownership first. | Docs VERIFIED (lockout rule); message text TO-CONFIRM (TC-G25). |
| DCM plan shows extra grants | INT adoption | Schema future grants from the platform add grants to new objects. | Account for them in `access.sql` or leave them undeclared. | NAIC notes. |
| `JWT token is invalid.` with a UUID, vendor code 390144 | Any key-pair login | General wrapper. Specific causes: 394300 user not found (use `LOGIN_NAME`), 394301 missing times, 394302 issued more than 60 s ago (clock), 394303 expired, 394304 fingerprint mismatch, 394305 not RS256, 394306 signature invalid, 394307 wrong account. | An administrator reads the code with `SELECT JSON_EXTRACT_PATH_TEXT(SYSTEM$GET_LOGIN_FAILURE_DETAILS('<uuid>'), 'errorCode');`. Compare `DESC USER` fingerprints. | Public docs VERIFIED 2026-11-16. NAIC's own note on "expired key" is TO-CONFIRM (TC-10-02). |
| Password login refused for a service account | Service user | `TYPE = SERVICE` users can never use a password. Snowflake also blocks passwords for non-human users in accounts under the rollout phases. | Use a key pair, or workload identity. Trial accounts are exempt from the phases but service users still cannot use passwords. | Public docs VERIFIED. |
| Hybrid table creation fails in a trial | Probe in file 02 | Hybrid tables are not supported in trial accounts. | Use Mode S, or convert the trial. Exact text TC-G01. | Public docs VERIFIED; text TO-CONFIRM. |
| Sequence cannot be found or accessed | App role inserting into a table whose default is a sequence | The role lacks `USAGE` on the sequence. | Grant `USAGE` on the sequence (declare it in `grants/access.sql`). | NAIC Calc Admin notes; Lab 7 break it. |
| Flyway: connection works but migrations fail | `flyway info` or `migrate` | The JDBC URL has no `db` or no `warehouse`. | Both are mandatory in the URL. | Flyway docs VERIFIED. |
| Flyway `validate` fails on checksum | After editing an applied file | Stored checksum differs. | Undo the edit, or add a new migration. `repair` only if the change was deliberate. | VERIFIED; drill 2. |
| Flyway refuses to migrate after a failure | History has a failed row | A migration failed part-way; Snowflake DDL cannot roll back. | Clean the leftovers by hand, `repair`, fix the file or add a new one, migrate. | VERIFIED; drill 1. |
| Flyway `clean` refused | `flyway clean` | `cleanDisabled` is true by default. | Leave it. Set `false` only in a scratch project. | VERIFIED. |
| Flyway cannot read the key file in Docker | `flyway_ci.sh` | The container user cannot read a mode-600 file, or the passphrase did not reach the JVM. | See TC-08-02 and TC-G21. | TO-CONFIRM. |
| Workflow does not start: unable to resolve action | Any workflow | The placeholder `<FULL_SHA_...>` was not replaced. | Resolve and paste the 40-character SHAs (file 08, step 5.3). | Pack design. |
| Environment shows no required reviewers | GitHub, repository settings | Private repository on a plan without that feature. | Use a public repository, or a paid plan. | Docs VERIFIED; your plan TC-08-03. |
| `Waiting for review` forever | `deploy-int` | You are the only reviewer and "Prevent self-review" is on. | Untick it in the environment. | Docs VERIFIED. |
| Drop check fails | `pr-clone-check` | A definition was deleted, or a rename is being read as drop plus create. | Restore the definition, or do the explicit Flyway rename first. | Drill 3; Lab 4. |
| Re-plan is not empty | After deploy | Something changed outside DCM, a definition differs in a detail, or a Flyway step altered a DCM-owned object. | Read each change, fix the definition, or fix the owner of the object. | Lab 3, Lab 4. |
| Resource monitor does not trigger | Drill 9 | Usage is tracked with a delay, or the quota is not reached. | Wait, then `SHOW RESOURCE MONITORS`. | TC-10-07. |
| First sign-in asks for MFA | Snowsight in a new trial | Human sign-in requires MFA in most accounts. Duo is not available in trials. | Use an authenticator app or passkey. Details TC-G19. | Docs VERIFIED (Duo); prompt form TO-CONFIRM. |
| `snow` hangs at a prompt | `refresh_dev_clone.sh` run from a tool with no stdin | The confirm prompt reads nothing and answers no. | Run in a normal terminal or pass `--yes`. | NAIC notes. |

## 4. Public documentation links

Snowflake DCM Projects:
- Overview: https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-overview
- Use and privileges: https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use
- Files and manifest: https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-files
- Supported entities: https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities
- CI/CD: https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-cicd
- Manage, unmanage, purge: https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage
- `EXECUTE DCM PROJECT`: https://docs.snowflake.com/en/sql-reference/sql/execute-dcm-project
- Snowflake CLI DCM commands: https://docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dcm-commands/overview (and `/plan`, `/deploy`, `/create`, `/list`, `/describe`, `/test`)

Snowflake platform:
- Hybrid tables: https://docs.snowflake.com/en/user-guide/tables-hybrid ; limitations: .../tables-hybrid-limitations ; cloning: .../tables-hybrid-clone ; `CREATE HYBRID TABLE`: https://docs.snowflake.com/en/sql-reference/sql/create-hybrid-table
- New `CREATE HYBRID TABLE` privilege (BCR-2157): https://docs.snowflake.com/en/release-notes/bcr-bundles/2026_02/bcr-2157
- Trial accounts: https://docs.snowflake.com/en/user-guide/admin-trial-account
- Resource monitors: https://docs.snowflake.com/en/sql-reference/sql/create-resource-monitor and https://docs.snowflake.com/en/user-guide/resource-monitors
- Key-pair authentication: https://docs.snowflake.com/en/user-guide/key-pair-auth and .../key-pair-auth-troubleshooting
- Users and `TYPE = SERVICE`: https://docs.snowflake.com/en/user-guide/admin-user-management
- MFA and password rollout: https://docs.snowflake.com/en/user-guide/security-mfa-rollout
- Workload identity federation: https://docs.snowflake.com/en/user-guide/workload-identity-federation
- Snowflake CLI connections: https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections
- GitHub Action for the CLI: https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/github-action and https://github.com/marketplace/actions/snowflake-actions

Flyway:
- Snowflake driver: https://documentation.red-gate.com/flyway/reference/database-driver-reference/snowflake
- Feature summary (editions): https://documentation.red-gate.com/fd/feature-summary-165740620.html
- Commands: https://documentation.red-gate.com/flyway/reference/commands
- Docker: https://documentation.red-gate.com/flyway/reference/usage/flyway-docker
- TOML configuration: https://www.red-gate.com/hub/product-learning/flyway/using-toml-configuration-files-with-flyway
- Release notes: https://documentation.red-gate.com/flyway/release-notes-and-older-versions/release-notes-for-flyway-engine

GitHub:
- Environments: https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments
- Concurrency: https://docs.github.com/en/actions/how-tos/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs

## 5. Open items for file 13

TC-G01, TC-G03, TC-G04, TC-G09, TC-G10, TC-G11, TC-G14, TC-G19, TC-G21, TC-G22, TC-G25, TC-G26, TC-08-02, TC-08-03, TC-08-06, TC-08-10, TC-10-01, TC-10-02, TC-10-05, TC-10-07.

## 6. Next

Continue to [the open questions and verification log](13-open-questions-and-verification-log.md).
