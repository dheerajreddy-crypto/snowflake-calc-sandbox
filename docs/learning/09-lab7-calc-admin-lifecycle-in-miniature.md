# Lab 7: the calc-admin lifecycle in miniature (file 09)

> **Lab no:** 7 | **File no:** 09 | **Time:** 2 to 2.5 h | **Prerequisites:** file 06 (Lab 4); file 08 (Lab 6) is optional | **Modes supported:** H or S, D (standard stand-ins run the whole lab, with a table of what differs) | **Leaves behind:** app-role grants verified; `V5` applied (Mode H) or a documented no-op (Mode S); `SP_ACTIVATE_DRAFT_CALCULATION` v2 with the event-first bracket deployed through DCM; audit rows exist.

## 1. Goal

Learn the calc-admin side of the platform by building its lifecycle objects in miniature and using them with least-privilege roles.

By the end you can:

- show that an app role with DML only can read and write rows but cannot change the schema;
- explain what a hybrid table adds over a standard table (enforced primary key, row-level locks) and prove it, or, in Mode S, prove the contrast;
- explain why the application must not run migrations when it starts, and what NAIC's pipeline-first design does instead;
- tell the "baseline INT, create QA and PROD for real" story and check the shape that makes a baseline safe;
- run an event-first audit bracket (PENDING event, engine call, COMPLETE event) and compare it with today's action-first approach;
- compare the live two-table model with the proposed five-table design in ADR-0026 in plain terms.

## 2. Why this matters in calc-engine and calc-admin

> **Real world (NAIC).** Calc Admin is the application that lets people draft, review, approve and activate calculations. The engine only knows calc states `DRAFT`, `ACTIVE` and `INACTIVE`. Calc Admin adds its own workflow states (`DRAFT`, `IN_REVIEW`, `APPROVED`, `ACTIVE`, `DISCARDED`) in its own tables, and records an audit trail.
>
> Two hybrid tables, `CALC_WORKFLOW_STATUS` (one row per calc) and `CALC_AUDIT_EVENT` (append-only), were created by hand in Snowsight in INT on 2026-11-03, in the engine's schema `COMMON_INT.CALCULATION_SUPPORT`, owned by `CICD_PIPELINE`. They do not exist in QA or PROD, and there is no DDL for them in Git. The runtime role `FN_NAIC_CALC_ENGINE_NP` (used by the service identity) has DML only: all of `INSERT`, `UPDATE`, `DELETE` on the status table and `INSERT` only on the audit table, because the audit trail is append-only. On the first day the role could not see the sequence that feeds `EVENT_ID` until `USAGE` was granted.
>
> The target design (`calc-admin-with-flyway.html`) puts a dedicated lifecycle schema under Flyway, run by a pipeline job with an owner identity. The application role gets DML only and never runs migrations. ADR-0026 (status Proposed) goes further: five tables and two sequences, one hybrid lock table, and an event-first audit.
>
> The other Calc Admin precedent is fpr-admin, a PostgreSQL service that runs Flyway itself when Spring Boot starts. You will contrast the two.
>
> Note on sources: the original DDL file for the two live tables was not found in any repository. The shapes in the sandbox are reconstructed from documentation, not copied.

## 3. Prerequisites and state check

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`, warehouse `SBX_CALC_WH`).

```sql
SHOW TABLES LIKE 'CALC_%' IN SCHEMA SBX_INT.CALC_SUPPORT;
```

You should see `CALC_WORKFLOW_STATUS`, `CALC_AUDIT_EVENT` and `CALC_CATEGORIES`. In the `is_hybrid` column the first two say `Y` in Mode H and `N` in Mode S. (The column name is lower-case in `SHOW` output.)

```sql
SELECT "version", "description", "success"
FROM SBX_INT.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT ORDER BY "installed_rank";
```

You should see versions 1 to 4, all successful. (Column names: see TC-08-08.)

```sql
SHOW GRANTS TO ROLE SBX_FN_CALC_ENGINE_NP;
```

You should see `SELECT, INSERT, UPDATE, DELETE` on `CALC_WORKFLOW_STATUS`, and only `SELECT` and `INSERT` on `CALC_AUDIT_EVENT`.

If the tables are missing, finish Lab 4 first (file 06). If the grants differ, read file 06 step "grants in the migrations", fix with a new migration, never by editing an applied one.

## 4. Concepts

**Lifecycle objects.** The tables a UI uses to track where each calculation is in its review (status, who did what, when). They are separate from the engine's content tables. In the sandbox: `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT`.

**Hybrid table.** A Snowflake table stored by row, made for many small reads and writes. It has a mandatory, enforced primary key, row-level locks, and secondary indexes. Standard tables are stored by column, are fast for scans, lock at coarser levels, and do not enforce primary or unique keys (only `NOT NULL`).

**Row-level locking.** Two sessions can update two different rows at the same time. On a standard table, an update to the table blocks other updates to it until the first transaction ends.

> **VERIFIED:** Hybrid tables use a row store and row-level locking, unlike partition-level or table-level locking for standard tables; the primary key is mandatory and enforced; secondary indexes can be created with `CREATE INDEX` (https://docs.snowflake.com/en/user-guide/tables-hybrid and .../tables-hybrid-limitations and .../sql-reference/sql/create-hybrid-table, read 2026-11-16).

**Least privilege.** Each identity gets only the rights it needs. Here three roles:

| Role | Used by | Can |
|---|---|---|
| `SBX_CICD_PIPELINE` | migration pipeline, owner | create, alter, drop. Owns the tables and sequences. |
| `SBX_FN_CALC_ENGINE_NP` | the application at run time | DML only. No DDL. On the audit table, `INSERT` and `SELECT` only. |
| `SBX_CALC_READER` | reporting | `SELECT` only. |

**Owner's rights and caller's rights.** A stored procedure declared `EXECUTE AS OWNER` runs with the owner's privileges, not the caller's. A caller who may only call the procedure can cause writes the caller could not do directly. That is both the reason to use it (controlled writes) and a risk (the procedure must validate its inputs).

**Action-first audit (today).** The application calls the engine, then writes an audit row. If the audit write fails after the engine call, the change has no record.

**Event-first audit (ADR-0026).** The application writes a `PENDING` event first, then calls the engine, then completes the event. If the process dies in the middle, a `PENDING` event with no outcome remains, which says "something may have happened here, reconcile". Nothing is silently unrecorded.

**Migrations at startup versus in a pipeline.** In fpr-admin the application itself runs Flyway when it starts. In the target for Calc Admin, a pipeline job runs Flyway under an owner identity before the application starts. The application only checks that the schema it expects is there.

**Baseline.** Telling Flyway "the objects up to version N already exist, start counting after N". It is how a database created by hand is adopted. It is only safe if the live shape equals what the scripts up to N would have created.

### 4.1 The sandbox objects

```
 SBX_INT.CALC_SUPPORT
   SEQ_WORKFLOW_ID ---default--->  CALC_WORKFLOW_STATUS   (hybrid in Mode H)
   SEQ_AUDIT_EVENT_ID -default-->  CALC_AUDIT_EVENT       (hybrid in Mode H, append-only for the app role)
   CALCULATIONS (DCM)   SP_ACTIVATE_DRAFT_CALCULATION (DCM, EXECUTE AS OWNER)

 owner role      SBX_CICD_PIPELINE   : DDL, owns all of the above
 app role        SBX_FN_CALC_ENGINE_NP: DML only
 read role       SBX_CALC_READER      : SELECT only
```

### 4.2 What differs in Mode S

| Behaviour | Mode H (hybrid) | Mode S (standard stand-ins) |
|---|---|---|
| Duplicate primary key | second insert fails | both rows are stored (PK is not enforced) |
| Locking in two sessions | different rows do not block each other | any update on the table waits for the other transaction |
| Secondary index | `V5` creates `IDX_AUDIT_EVENT_TYPE` | `V5` is a documented no-op |
| Everything else in this lab | same | same |

> **TO-CONFIRM [TC-09-01]:** The exact error text for a duplicate key on a hybrid table. The docs say the key is enforced. How to check at lab time: run step 5.3 in Mode H and paste the message. Record the result in file 13.

## 5. Step-by-step

### Step 5.1 Act as the app role and see what it may do

Run in: **Snowsight worksheet**. Use one worksheet for all of 5.1 to 5.3. Your user holds all four sandbox roles, so you can switch.

```sql
USE ROLE SBX_FN_CALC_ENGINE_NP;
USE WAREHOUSE SBX_CALC_WH;
SELECT CURRENT_ROLE();          -- SBX_FN_CALC_ENGINE_NP

-- allowed: DML on the status table
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (CALC_ID, ADMIN_STATUS, UPDATED_BY, UPDATED_AT)
SELECT 2001, 'DRAFT', CURRENT_USER(), CURRENT_TIMESTAMP();

UPDATE SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS
SET ADMIN_STATUS = 'IN_REVIEW', UPDATED_AT = CURRENT_TIMESTAMP()
WHERE CALC_ID = 2001;

-- allowed: append to the audit table
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT (CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR, EVENT_AT)
SELECT 2001, 'SUBMIT', 'COMPLETE', CURRENT_USER(), CURRENT_TIMESTAMP();

SELECT * FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT ORDER BY EVENT_ID DESC LIMIT 5;
```

You should see: both inserts succeed, the update affects 1 row, and the audit table has your row with an `EVENT_ID` that came from `SEQ_AUDIT_EVENT_ID` without you supplying it.

Now try what the app role must not do. Each statement should fail with an insufficient-privileges error.

```sql
UPDATE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT SET EVENT_STATUS = 'COMPLETE' WHERE CALC_ID = 2001;   -- audit is append-only
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT WHERE CALC_ID = 2001;                              -- no delete
TRUNCATE TABLE SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS;                                           -- no truncate
ALTER TABLE SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS ADD COLUMN SNEAKY VARCHAR;                     -- no DDL
CREATE TABLE SBX_INT.CALC_SUPPORT.APP_MADE_THIS (ID NUMBER);                                         -- no create
```

You should see: five errors that say the role lacks a privilege or the object cannot be accessed. (Exact text is not asserted in this pack.)

Checkpoint: the app role can insert a status row. Can it run Flyway? Think before reading section 5.6.

### Step 5.2 Read-only role

```sql
USE ROLE SBX_CALC_READER;
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS;      -- works
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (CALC_ID, ADMIN_STATUS, UPDATED_BY, UPDATED_AT)
SELECT 9999, 'DRAFT', 'reader', CURRENT_TIMESTAMP();                  -- fails
```

You should see: the count works, the insert fails.

### Step 5.3 Enforced primary key

The primary key of the sandbox status table is `WORKFLOW_ID`. Use the owner role for this test so the setup is not the point.

```sql
USE ROLE SBX_CICD_PIPELINE;
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (WORKFLOW_ID, CALC_ID, ADMIN_STATUS, UPDATED_BY, UPDATED_AT)
SELECT 90001, 2002, 'DRAFT', 'pk-test', CURRENT_TIMESTAMP();
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (WORKFLOW_ID, CALC_ID, ADMIN_STATUS, UPDATED_BY, UPDATED_AT)
SELECT 90001, 2002, 'DRAFT', 'pk-test', CURRENT_TIMESTAMP();     -- same key again
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS WHERE WORKFLOW_ID = 90001;
```

You should see in **Mode H**: the second insert fails with a duplicate key error (TC-09-01) and the count is 1.

#### If you are in Mode S (no hybrid tables)

You should see: both inserts succeed and the count is **2**. A standard table keeps informational keys only. This is the point of the contrast: the real calc-admin service has to catch duplicate-key errors on create races (the live design retries a `MERGE` once as an update), and that only works because the database enforces the key.

> **VERIFIED:** Hybrid tables enforce primary key, unique, foreign key, check and not-null constraints. Standard tables enforce only `NOT NULL` (https://docs.snowflake.com/en/user-guide/tables-hybrid-limitations and the pack's contract, read 2026-11-16).

Clean the test rows:

```sql
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS WHERE WORKFLOW_ID = 90001;
```

### Step 5.4 Row-level locking with two sessions

Run in: **two Snowsight worksheets** side by side, called A and B. Each worksheet is its own session. Both use role `SBX_CICD_PIPELINE` and warehouse `SBX_CALC_WH`.

First, in both worksheets, set a short lock wait. Without it a blocked statement waits up to the account default, which can be hours.

```sql
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;
ALTER SESSION SET LOCK_TIMEOUT = 20;
```

> **TO-CONFIRM [TC-09-02]:** `ALTER SESSION SET LOCK_TIMEOUT` (seconds), and the claim that the default wait is long. The NAIC design documents use `LOCK_TIMEOUT = 30` and `SELECT ... FOR UPDATE WAIT 10`, but this pack did not read the public parameter page. How to check at lab time: Snowflake docs page for the `LOCK_TIMEOUT` parameter. Record the result in file 13.

Seed two rows (worksheet A):

```sql
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (WORKFLOW_ID, CALC_ID, ADMIN_STATUS, UPDATED_BY, UPDATED_AT)
SELECT 91001, 2001, 'DRAFT', 'lock-test', CURRENT_TIMESTAMP();
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (WORKFLOW_ID, CALC_ID, ADMIN_STATUS, UPDATED_BY, UPDATED_AT)
SELECT 91002, 2002, 'DRAFT', 'lock-test', CURRENT_TIMESTAMP();
```

Worksheet A: open a transaction and update row 91001, but do not finish it.

```sql
BEGIN;
UPDATE SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS SET ADMIN_STATUS = 'IN_REVIEW' WHERE WORKFLOW_ID = 91001;
-- leave this transaction open
```

Worksheet B: update the other row, then the same row.

```sql
UPDATE SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS SET ADMIN_STATUS = 'IN_REVIEW' WHERE WORKFLOW_ID = 91002;   -- different row
UPDATE SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS SET ADMIN_STATUS = 'APPROVED'  WHERE WORKFLOW_ID = 91001;   -- same row
```

You should see in **Mode H**: the first statement returns at once. The second waits about 20 seconds and then fails with a lock timeout error. That is row-level locking: only the row A touched is locked.

#### If you are in Mode S (no hybrid tables)

You should see: even the first statement in B waits and times out, because the update in A holds a lock on the whole standard table until it ends. Write down what you see; it is the opposite of Mode H and is the whole reason the real design uses a hybrid table for the one place where concurrent writers must not collide.

> **TO-CONFIRM [TC-09-03]:** The Mode S result (table-level wait for an update of a different row). The docs say standard tables lock at partition or table level, not row level. How to check at lab time: run the experiment and record what you see. Record the result in file 13.

Finish worksheet A: `ROLLBACK;`. Clean up (either worksheet):

```sql
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS WHERE WORKFLOW_ID IN (91001, 91002);
```

### Step 5.5 Add an index with Flyway (`V5`)

Run in: **Terminal** (repository root). Indexes on hybrid tables are Flyway's lane, because DCM cannot manage hybrid tables or their indexes.

Create two files. If they already exist from Lab 4, keep them.

`flyway/migrations/hybrid/V5__add_audit_event_type_index.sql`:

```sql
CREATE INDEX IDX_AUDIT_EVENT_TYPE ON CALC_AUDIT_EVENT (EVENT_TYPE, EVENT_AT);
```

`flyway/migrations/standard/V5__add_audit_event_type_index.sql` (a documented no-op, because a standard table has no secondary index):

```sql
-- Mode S stand-in: standard tables have no secondary indexes, so this migration only keeps the version numbers aligned with Mode H.
SELECT 1;
```

> **TO-CONFIRM [TC-09-04]:** The exact `CREATE INDEX <name> ON <table> (<cols>)` grammar for hybrid tables. The create-table page documents an inline `INDEX` clause and says indexes can be added later with `CREATE INDEX`. How to check at lab time: Snowflake docs page `CREATE INDEX`. Record the result in file 13.

Now run Flyway from your laptop as the service identity. Load your local, git-ignored settings, check, then migrate.

```bash
cd flyway
set -a; source .env.flyway; set +a      # sets FLYWAY_URL, FLYWAY_LOCATIONS and placeholders; this file is never committed
flyway info
flyway migrate
flyway info
```

You should see: before `migrate`, version 5 is `Pending`. After `migrate`, version 5 is `Success`.

Run in: **Snowsight worksheet** (Mode H):

```sql
USE ROLE SBX_CICD_PIPELINE;
SHOW INDEXES IN TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT;
```

You should see: `IDX_AUDIT_EVENT_TYPE` with status `ACTIVE` (it may show `BUILD IN PROGRESS` for a moment on a large table). The app role needs `SELECT` on the table to benefit from the index, and it has it.

> **VERIFIED:** `SHOW INDEXES IN TABLE <t>` shows index status (BUILD IN PROGRESS or ACTIVE); altering an index or adding a column to it is not supported, so you drop and re-create; a role needs SELECT on the table to use its secondary index (https://docs.snowflake.com/en/sql-reference/sql/create-hybrid-table, read 2026-11-16).

#### If you are in Mode S (no hybrid tables)

`flyway migrate` applies the no-op. `SHOW INDEXES` returns nothing. Note that your history shows version 5 as success while no index exists. A history row proves that a file ran, not that the world looks like you hoped.

### Step 5.6 Why the application must not run migrations at startup

This step is a short experiment plus a reading exercise.

Run in: **Snowsight worksheet**. Pretend the application started Flyway with its own credential, the app role. The first thing Flyway does is create its history table in the schema.

```sql
USE ROLE SBX_FN_CALC_ENGINE_NP;
CREATE TABLE SBX_INT.CALC_SUPPORT.FLYWAY_HISTORY_STARTUP_TEST (ID NUMBER);
```

You should see: the statement fails. With only DML rights the app cannot even create the history table, let alone alter a table. To make startup migrations work you would have to give the application's identity `CREATE`, `ALTER` and `DROP` rights, which destroys the least-privilege split you just verified.

Now compare the two real designs. Read `src/main/resources/application.properties` and the `application-dev.properties` file in `/Users/drbanda/NAIC/Repos/Fpr-admin/fpr-admin-service/` (read only).

| Setting | fpr-admin (PostgreSQL, Flyway at startup) | Calc Admin target (Snowflake, Flyway in pipeline) |
|---|---|---|
| Who runs Flyway | The application, when Spring Boot starts (`spring.flyway.enabled=true`) | A pipeline job under an owner identity |
| Credential | The same datasource as the app (`spring.flyway.user` from the app's own settings) | Separate owner identity; app role has DML only |
| `out-of-order` | `true` | `false` |
| `validate-on-migrate` | `false` in dev and local | `true` |
| `repair-on-migrate` | `true` in dev and local | repair is manual, with approval |
| `baseline-on-migrate` | `true` in local, `false` otherwise | `false`; baseline is an explicit one-time act |
| `clean` | not shown in properties | `cleanDisabled = true` |
| Environment data | `db/localdata`, `db/devdata`, `db/intdata`, `db/qadata` folders of `R__` data files chosen by profile | Data changes are versioned migrations with assertions (Lab 4) |
| If a migration fails | The pod fails to start | The job fails; the running application keeps working on the old schema |

Reasons from NAIC's design documents: the application credential would need DDL rights; several instances starting together would race; a failed migration would block every pod; production changes need change-request approval through protected pipelines; and Snowflake DDL cannot be rolled back. The second and third reasons are inferences in the NAIC explainer, not stated rules.

Checkpoint: write one sentence for each row of the table that says why the right-hand choice is safer.

### Step 5.7 The baseline story: INT first, QA and PROD for real

In NAIC the two live tables were made by hand in INT only. The Flyway plan for them is:

1. Write the scripts that would create the tables (sequence, status table, index, audit table with sequence default, two indexes, grants). In the NAIC plan these are `V001` to `V006`.
2. Check that the live INT objects match what those scripts would have made.
3. Only then **baseline** INT at `V006`. Flyway records "version 6 is the starting point" without running anything.
4. QA and PROD have nothing. They run `V001` to `V006` for real.

The risky part is step 2. A baseline on a wrong assumption means Flyway believes an index or a grant exists when it does not, and later migrations build on it.

Run in: **Snowsight worksheet**. Write the shape check you would run before a baseline: compare the columns of the same table in two databases. Use INT and a clone, or INT and QA if you finished Lab 6.

```sql
USE ROLE SBX_CICD_PIPELINE;
WITH a AS (
  SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT
  FROM SBX_INT.INFORMATION_SCHEMA.COLUMNS
  WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_NAME = 'CALC_AUDIT_EVENT'
), b AS (
  SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT
  FROM SBX_QA.INFORMATION_SCHEMA.COLUMNS
  WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_NAME = 'CALC_AUDIT_EVENT'
)
SELECT 'only in INT' AS side, * FROM (SELECT * FROM a MINUS SELECT * FROM b)
UNION ALL
SELECT 'only in QA',  * FROM (SELECT * FROM b MINUS SELECT * FROM a);
```

You should see: zero rows. If you get rows, the two environments differ, and a baseline of the first on the scripts of the second would have been wrong. The column default differs in text if the sequence is fully qualified in one place and not the other: read it, do not assume.

Also compare the grants and the table type. In Mode H check `is_hybrid`:

```sql
SHOW TABLES LIKE 'CALC_AUDIT_EVENT' IN SCHEMA SBX_INT.CALC_SUPPORT;
SHOW TABLES LIKE 'CALC_AUDIT_EVENT' IN SCHEMA SBX_QA.CALC_SUPPORT;
SHOW GRANTS ON TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT;
SHOW GRANTS ON TABLE SBX_QA.CALC_SUPPORT.CALC_AUDIT_EVENT;
```

Paper exercise: read the Flyway history of INT and QA and write down the difference. In the sandbox both ran every migration, so neither has a baseline row. In NAIC, INT would show a baseline row at `V006` and QA and PROD would show six executed migrations.

Where Flyway reads its own contract on this point: `baselineOnMigrate=false` normally, and a one-time baseline needs an approved decision and a runbook. Lab 2 showed the mechanics of `baseline` with the Lab 1 clone.

### Step 5.8 Event-first audit bracket, deployed through DCM

The engine procedure that activates a draft is a DCM-owned object. You will change its body so it writes the audit trail event-first. Then you will compare with the action-first gap.

#### The design

```
 event-first (target)                         action-first (today)
 1  INSERT PENDING event                      1  call the engine
 2  call the engine                           2  INSERT audit row
 3  INSERT COMPLETE event                     If step 2 fails, the change has no record.
 If the process dies after 1, a PENDING
 event with no COMPLETE remains: reconcile.
```

In ADR-0026 the third step is one permitted `UPDATE` of the same row from `PENDING` to its outcome. In the sandbox the audit table is append-only for the app role, so the miniature writes a second row instead. That choice is the heart of an open question you will meet in step 5.9.

Run in: **Terminal**. Replace the whole file `dcm/calc_engine/sources/definitions/procedures/SP_ACTIVATE_DRAFT_CALCULATION.sql`. Keep a copy of the old text so you can compare. The activation logic (retire the source calc, activate the draft) is the same as in Lab 3, written with a subquery instead of a variable. Only the audit rows are new. As in Lab 3, names are fully qualified with the template variables, so the same file works in INT, QA, PROD and clones.

```sql
DEFINE PROCEDURE {{calc_db}}.{{calc_schema}}.SP_ACTIVATE_DRAFT_CALCULATION(P_DRAFT_CALC_ID NUMBER, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE
  v_rows NUMBER DEFAULT 0;
BEGIN
  -- 1. event first: the intent is recorded before anything changes
  INSERT INTO {{calc_db}}.{{calc_schema}}.CALC_AUDIT_EVENT (CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR, EVENT_AT)
  SELECT :P_DRAFT_CALC_ID, 'ACTIVATE', 'PENDING', :P_USER, CURRENT_TIMESTAMP();

  -- 2. the engine work: retire the source calc, then activate the draft
  UPDATE {{calc_db}}.{{calc_schema}}.CALCULATIONS SET CALC_STATUS = 'INACTIVE'
  WHERE CALC_ID = (SELECT SOURCE_CALC_ID FROM {{calc_db}}.{{calc_schema}}.CALCULATIONS WHERE CALC_ID = :P_DRAFT_CALC_ID AND CALC_STATUS = 'DRAFT');
  UPDATE {{calc_db}}.{{calc_schema}}.CALCULATIONS SET CALC_STATUS = 'ACTIVE'
  WHERE CALC_ID = :P_DRAFT_CALC_ID AND CALC_STATUS = 'DRAFT';
  v_rows := SQLROWCOUNT;

  -- 3. complete: success or a recorded rejection
  IF (v_rows = 1) THEN
    INSERT INTO {{calc_db}}.{{calc_schema}}.CALC_AUDIT_EVENT (CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR, EVENT_AT)
    SELECT :P_DRAFT_CALC_ID, 'ACTIVATE', 'COMPLETE', :P_USER, CURRENT_TIMESTAMP();
    RETURN 'ACTIVATED ' || :P_DRAFT_CALC_ID;
  END IF;
  INSERT INTO {{calc_db}}.{{calc_schema}}.CALC_AUDIT_EVENT (CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR, EVENT_AT)
  SELECT :P_DRAFT_CALC_ID, 'ACTIVATE_REJECTED', 'COMPLETE', :P_USER, CURRENT_TIMESTAMP();
  RETURN 'NOT_A_DRAFT';
EXCEPTION
  WHEN OTHER THEN
    -- the PENDING event stays: an event with no outcome means "indeterminate, reconcile"
    RETURN 'ERROR: ' || SQLERRM;
END
$$;
```

> **TO-CONFIRM [TC-G03]:** Exact public `DEFINE PROCEDURE` grammar. This shape is copied from the working NAIC branch (header and `$$ ... $$` body as in `SP_ADD_CATEGORY`), and the docs say `DEFINE` behaves like `CREATE OR ALTER`. How to check at lab time: the supported-entities page links to the procedure pages. Record the result in file 13.

Plan and deploy it, as the owner (connection `sbx_owner`, role `SBX_CICD_PIPELINE`). Targets do not choose the connection or role, so both are passed explicitly.

```bash
snow dcm plan SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM --target INT --from dcm/calc_engine -c sbx_owner --role SBX_CICD_PIPELINE --save-output
```

You should see: exactly one change, an `ALTER` of the procedure `SP_ACTIVATE_DRAFT_CALCULATION`. The docs say a changed handler body is treated as a full replace, so the plan shows no line-by-line body diff.

```bash
snow dcm deploy SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM --target INT --from dcm/calc_engine -c sbx_owner --role SBX_CICD_PIPELINE --alias "lab7-event-first"
snow dcm plan   SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM --target INT --from dcm/calc_engine -c sbx_owner --role SBX_CICD_PIPELINE --save-output
```

You should see: a deployment with your alias, and then an empty re-plan (`changeset` is `[]`). Declared grants in `grants/access.sql` are re-applied by DCM, so the app role keeps `USAGE` on the procedure.

> **VERIFIED:** A changed procedure or function handler is treated as a full replace in the plan; PLAN validates creation, not runtime success (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities, read 2026-11-16).

#### If DCM is not available (Mode D-off)

Run the same text in a worksheet as the owner, with `CREATE OR REPLACE PROCEDURE` in place of `DEFINE PROCEDURE` and `SBX_INT.CALC_SUPPORT` in place of the template variables. Then run the grants again, because `CREATE OR REPLACE` drops them:

```sql
USE ROLE SBX_CICD_PIPELINE;
GRANT USAGE ON PROCEDURE SBX_INT.CALC_SUPPORT.SP_ACTIVATE_DRAFT_CALCULATION(NUMBER, VARCHAR) TO ROLE SBX_FN_CALC_ENGINE_NP;
```

This is the manual re-grant chore that DCM's declared grants remove.

#### Use it

Run in: **Snowsight worksheet**. Create a draft that points at a source calc, as the owner.

```sql
USE ROLE SBX_CICD_PIPELINE;
CALL SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION('MINI_LIFE_RATIO_DRAFT', 'lab7');   -- returns the new CALC_ID, for example 2003
UPDATE SBX_INT.CALC_SUPPORT.CALCULATIONS SET SOURCE_CALC_ID = 2001 WHERE CALC_NAME = 'MINI_LIFE_RATIO_DRAFT';
```

Now act as the app role, with the id you got:

```sql
USE ROLE SBX_FN_CALC_ENGINE_NP;
CALL SBX_INT.CALC_SUPPORT.SP_ACTIVATE_DRAFT_CALCULATION(<DRAFT_CALC_ID>, 'dheeraj');   -- ACTIVATED <id>
SELECT CALC_ID, CALC_NAME, CALC_STATUS FROM SBX_INT.CALC_SUPPORT.CALCULATIONS WHERE CALC_ID IN (2001, <DRAFT_CALC_ID>);
SELECT EVENT_ID, CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT WHERE CALC_ID = <DRAFT_CALC_ID> ORDER BY EVENT_ID;
CALL SBX_INT.CALC_SUPPORT.SP_ACTIVATE_DRAFT_CALCULATION(<DRAFT_CALC_ID>, 'dheeraj');   -- NOT_A_DRAFT
```

You should see: the call returns `ACTIVATED <id>`; the source calc `2001` is `INACTIVE` and the draft is `ACTIVE`; the audit table has a `PENDING` row followed by a `COMPLETE` row for `ACTIVATE`. The second call returns `NOT_A_DRAFT` and leaves a `PENDING` row followed by an `ACTIVATE_REJECTED` row.

Why this works with an insert-only app role: the procedure runs as its owner (`EXECUTE AS OWNER`), so the audit rows are written with the owner's rights. The app role can call the procedure but still cannot touch audit rows directly. You proved that in step 5.1.

#### The action-first gap

Run as the app role. This is what "engine first, audit second" looks like when the second step never happens.

```sql
USE ROLE SBX_FN_CALC_ENGINE_NP;
UPDATE SBX_INT.CALC_SUPPORT.CALCULATIONS SET CALC_STATUS = 'INACTIVE' WHERE CALC_NAME = 'MINI_LIFE_RATIO_DRAFT';
-- imagine the process crashes here, before the audit insert
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT WHERE EVENT_TYPE = 'DEACTIVATE';   -- 0
```

You should see: the calc changed state and no event describes it. With event-first, even a crash leaves a `PENDING` row to find.

Restore the state for later steps:

```sql
USE ROLE SBX_CICD_PIPELINE;
UPDATE SBX_INT.CALC_SUPPORT.CALCULATIONS SET CALC_STATUS = 'ACTIVE' WHERE CALC_NAME = 'MINI_LIFE_RATIO_DRAFT';
```

### Step 5.9 The honest tension: append-only versus event-first

ADR-0026's event-first design completes the `PENDING` event once with the outcome. That needs one `UPDATE` on the audit table. NAIC's current live grant gives the service `INSERT` only. Snowflake cannot narrow an `UPDATE` grant to one column or to rows that are `PENDING`. This is open question RAJA-4 in the design notes.

Three ways to resolve it, with trade-offs:

| Option | How | Cost |
|---|---|---|
| A. Grant `UPDATE` to the app role | Simple. One row per event. | The app role could now edit or falsify any audit row. Append-only is gone. |
| B. Two rows per event (what step 5.8 does) | `PENDING` row, then `COMPLETE` row. App role stays insert-only. | Queries must pair rows. Orphan `PENDING` rows need a reconciler. |
| C. Write audit through an owner's-rights procedure | App role can only call the procedure. The procedure does the insert and the one update. | The procedure must validate everything, and the owner can update any row. |

Try option A on purpose, then take it back. It shows the problem in one minute.

```sql
USE ROLE SBX_CICD_PIPELINE;
GRANT UPDATE ON TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT TO ROLE SBX_FN_CALC_ENGINE_NP;
USE ROLE SBX_FN_CALC_ENGINE_NP;
UPDATE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT SET ACTOR = 'someone else' WHERE EVENT_ID = (SELECT MIN(EVENT_ID) FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT);   -- now it works: the history was rewritten
USE ROLE SBX_CICD_PIPELINE;
REVOKE UPDATE ON TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT FROM ROLE SBX_FN_CALC_ENGINE_NP;
```

You should see: the update succeeds while the grant exists. After the revoke it fails again. Then check the actor of the first event: it now reads `someone else`. A hand-made grant also is not in any migration, so no tool reports it as drift. Flyway-owned grants are only as good as the last migration that touched them.

Fix the data you changed:

```sql
UPDATE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT SET ACTOR = CURRENT_USER() WHERE ACTOR = 'someone else';
```

> **TO-CONFIRM [TC-09-05]:** That Snowflake cannot restrict `UPDATE` to specific columns or rows through a grant. The NAIC design note says it cannot. How to check at lab time: read the `GRANT <privileges>` reference and look for column-level DML. Record the result in file 13.

### Step 5.10 Two models side by side

You now know enough to read the two designs in plain terms. Do not decide which is right: the NAIC decision between them is open (the architecture page lists "resolve the branch's two hybrid tables against ADR-0026's one hybrid lock table" as a stakeholder decision).

| Question | Live model (what you built) | ADR-0026 five-table design (Proposed, 2026-10-01) |
|---|---|---|
| Unit of review | One calculation | A calc set for one data year |
| Tables | 2 hybrid tables plus a sequence for the audit key (the sandbox adds a second sequence for the status table) | 5 tables, 2 sequences, and only one hybrid table |
| Concurrency | Last write wins, no lock held by the app; a `409` when the status is `IN_REVIEW` or `APPROVED` | One hybrid row `CALC_OPEN_REVISION` per calc set and year holds the lock and enforces "one open revision" |
| Audit | Action-first, one row after the engine call | Event-first, `PENDING` then outcome (`APPLIED`, `REJECTED`, `INDETERMINATE`) |
| Statuses | `DRAFT`, `IN_REVIEW`, `APPROVED`, `ACTIVE`, `DISCARDED` | `IN_PROGRESS`, `IN_REVIEW`, `APPROVED`, `PUBLISHED`, `DISCARDED` |
| Where | The engine's own schema | A dedicated `CALC_ADMIN_LIFECYCLE` schema per environment |
| History tables | Hybrid (gives up Fail-safe, `UNDROP`, replication, full Time Travel) | Standard tables, with 90 days of Time Travel in the design |
| Migration status | Created by hand, not in Git | Flyway target, not executed |
| Who owns it in the target split | Flyway (hybrid and lifecycle lane) | Flyway (all five, because one schema is one migration boundary) |

The reasoning behind making only one table hybrid in ADR-0026 is the cost list of hybrid tables from file 05: no `UNDROP`, no Fail-safe, no replication, Time Travel only by timestamp. Those losses hurt most on history tables, and only the open-revision claim needs an enforced key.

Exercise: list the five tables and mark each one "standard" or "hybrid" and "Flyway" or "DCM". Check your answer with the checkpoint below.

## 6. Checkpoint questions

1. Which privileges does `SBX_FN_CALC_ENGINE_NP` hold on `CALC_AUDIT_EVENT`, and which tool applied them?
2. The app role can insert a status row but cannot create a table. Why does that rule out running Flyway as the app role?
3. In Mode H, why does the second insert of `WORKFLOW_ID = 90001` fail, and why does it succeed in Mode S?
4. In the two-session test, what is the difference between Mode H and Mode S for an update on a different row?
5. What does a `PENDING` audit row with no later `COMPLETE` row tell you?
6. Why could the app role not write audit rows directly, yet the procedure could?
7. Name two reasons NAIC's target runs Flyway in a pipeline job and not at application start.
8. In the five-table design, which table is hybrid, why only that one, and which tool owns all five?

**Answers**

1. `SELECT` and `INSERT`. They were applied by Flyway inside `V2`, because the hybrid tables are Flyway-owned and DCM cannot manage them.
2. Flyway must create its history table and run DDL. With DML-only rights it cannot do either, and giving it DDL rights would remove the least-privilege split.
3. A hybrid table enforces its primary key. A standard table stores the key as information only and enforces only `NOT NULL`.
4. In Mode H the update on a different row proceeds at once (row lock). In Mode S it waits for the other transaction (coarser lock), so it can time out.
5. Something was started and its outcome is unknown. It needs reconciling against the engine state. In ADR-0026 the outcome is then recorded as `INDETERMINATE`.
6. The procedure is `EXECUTE AS OWNER`, so its inserts run with the owner's rights. The caller needs only `USAGE` on the procedure.
7. Any two of: the application credential would need DDL rights; several instances would race; a failed migration would block every instance; production changes need protected, approved pipelines; Snowflake DDL cannot be rolled back.
8. `CALC_OPEN_REVISION`. Hybrid tables lose Fail-safe, `UNDROP`, replication and full Time Travel, which matter most for history, and only the open-revision claim needs an enforced key and a row lock. Flyway owns all five, because the lifecycle schema is one migration boundary.

## 7. Break it

**Reproduce the real "sequence not visible" bug.** On 2026-11-03 the service role could not see `CALC_AUDIT_EVENT_SEQ`, so an insert that omitted `EVENT_ID` failed. The same can happen here.

Run in: **Snowsight worksheet**.

```sql
USE ROLE SBX_CICD_PIPELINE;
REVOKE USAGE ON SEQUENCE SBX_INT.CALC_SUPPORT.SEQ_AUDIT_EVENT_ID FROM ROLE SBX_FN_CALC_ENGINE_NP;

USE ROLE SBX_FN_CALC_ENGINE_NP;
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT (CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR, EVENT_AT)
SELECT 2001, 'BREAK', 'COMPLETE', CURRENT_USER(), CURRENT_TIMESTAMP();
```

- **Symptom:** the insert fails. The message says the sequence cannot be found or accessed, or the role lacks usage on it. The calling code never mentions the sequence, so the error looks unrelated to the insert.
- **Diagnosis:** `SHOW GRANTS ON SEQUENCE SBX_INT.CALC_SUPPORT.SEQ_AUDIT_EVENT_ID;` shows no `USAGE` for the app role. `DESC TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT` shows the default is `SEQ_AUDIT_EVENT_ID.NEXTVAL`.
- **Recovery, the DCM way (Mode D):** the sequence grant is declared in `grants/access.sql`, so a plan sees the missing grant. Run:

```bash
snow dcm plan SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM --target INT --from dcm/calc_engine -c sbx_owner --role SBX_CICD_PIPELINE --save-output
snow dcm deploy SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM --target INT --from dcm/calc_engine -c sbx_owner --role SBX_CICD_PIPELINE --alias "lab7-regrant"
```

You should see: the plan lists a `GRANT` change for the sequence, deploy restores it, and the insert works again.

- **Recovery, Mode D-off:** `GRANT USAGE ON SEQUENCE SBX_INT.CALC_SUPPORT.SEQ_AUDIT_EVENT_ID TO ROLE SBX_FN_CALC_ENGINE_NP;` by hand.
- **Lesson:** a grant on a sequence is a dependency of a table default. A declared grant is both documentation and a repair tool. The same grant for a table owned by Flyway would not be repaired by DCM, only by a new migration.

> **TO-CONFIRM [TC-09-06]:** That a plan reports a hand-revoked declared grant as a change. The docs say a plan against live state shows changes made outside DCM. How to check at lab time: run the commands above. Record the result in file 13.

Clean up the test row:

```sql
USE ROLE SBX_CICD_PIPELINE;
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT WHERE EVENT_TYPE = 'BREAK';
```

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| Trial accounts have no hybrid tables | Mode S contrast in 5.3 and 5.4 | Stand-ins plus read-only expected results; optional upgrade | VERIFIED (docs); exact error TC-G01 |
| Standard tables do not enforce primary keys | 5.3 in Mode S | Hybrid table, or application checks | VERIFIED |
| DCM cannot manage hybrid tables or their indexes | 5.5 | Flyway migration `V5` | VERIFIED (not in the supported list) |
| A grant cannot be limited to one column or row | 5.9 option A | Procedure with owner's rights, two-row audit, or accept `UPDATE` | TO-CONFIRM (TC-09-05) |
| History row proves a file ran, not what the world looks like | 5.5 in Mode S | Shape check before baseline, `validate` plus fingerprints | VERIFIED (design) |
| Hand-made grants are not reported by Flyway | 5.9 | Declared grants in DCM for DCM-owned objects, review of Flyway grants | VERIFIED (design) |
| Flyway has no preview and no drift check in Community | 5.7 | Shape query, DCM plan for DCM-owned objects | VERIFIED (Community feature list) |
| Hybrid tables lose `UNDROP` and Fail-safe | 5.10 | Standard tables for history, backups, Time Travel by timestamp | VERIFIED (docs) |

## 9. What you learned

- An app role with DML only can do its job and nothing more, and you proved the "nothing more" with failing statements.
- A hybrid table enforces its key and locks by row. A standard stand-in does neither, and the difference is visible in two worksheets.
- Migrations belong to a pipeline with an owner identity. Running them at application start needs rights the app should never have.
- A baseline is safe only after you compare the live shape with the scripts.
- Event-first auditing turns a silent gap into a visible `PENDING` row, and it forces a design choice about who may update an audit table.

## 10. Cleanup

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS WHERE UPDATED_BY IN ('pk-test', 'lock-test');
DROP TABLE IF EXISTS SBX_INT.CALC_SUPPORT.APP_MADE_THIS;
```

Keep: `V5` applied, the v2 procedure, the audit rows. Lab 8 and the capstone use them. Nothing in this lab costs more than seconds of X-Small compute.

## 11. State left behind

App role grants verified; `V5` applied (Mode H) or a documented no-op (Mode S); `SP_ACTIVATE_DRAFT_CALCULATION` v2 deployed through DCM (alias `lab7-event-first`); audit rows exist in `CALC_AUDIT_EVENT`; the draft calculation `MINI_LIFE_RATIO_DRAFT` exists and is `ACTIVE`; source calc `2001` is `INACTIVE`.

> Heads-up for Lab 8 and the capstone: calc `2001` is now `INACTIVE`. If you want it `ACTIVE` again for smoke tests, run as the owner: `UPDATE SBX_INT.CALC_SUPPORT.CALCULATIONS SET CALC_STATUS = 'ACTIVE' WHERE CALC_ID = 2001;`.

## 12. Open items for file 13

TC-G01, TC-G03, TC-08-08, TC-09-01, TC-09-02, TC-09-03, TC-09-04, TC-09-05, TC-09-06.

## 13. Next

Go on to [Lab 8: failure drills and limitations](10-lab8-failure-drills-and-limitations.md).
