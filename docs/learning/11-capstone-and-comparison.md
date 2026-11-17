# Capstone and comparison (file 11)

> **Lab no:** capstone | **File no:** 11 | **Time:** 3 h | **Prerequisites:** file 10 (Lab 8) | **Modes supported:** H or S, D (way 3 and way 4 use standard stand-ins in Mode S; in Mode D-off the DCM parts are shown as expected output and replaced by hand-run statements) | **Leaves behind:** schemas `CAP_HAND`, `CAP_FLYWAY`, `CAP_DCM` and clone `SBX_DEV_CAP4` created and dropped by you; the quiz answered; the teach-back done.

This file adapts the lab template: step-by-step is split into four "ways", and there is a comparison, a decision guide, a quiz and a teach-back.

## 1. Goal

Do the same realistic change request four times, so you feel what each way of working costs and protects:

1. hand-run SQL;
2. Flyway only;
3. DCM only;
4. DCM plus Flyway plus the pipeline.

Then compare them with a matrix, say what you would choose for the calc engine and why, answer a quiz that covers files 03 to 10, and teach the whole picture back.

**The change request** (one ticket, five parts):

| Part | Change |
|---|---|
| a | Add a column `CALC_DESCRIPTION VARCHAR(500)` to `CALCULATIONS`. |
| b | Change the body of `SP_REGISTER_CALCULATION`: trim the name and fill the new column. |
| c | Rename the table `FORMULAS` to `CALC_FORMULAS`. |
| d | Add a data fix: give every existing calculation a description (`'Legacy calculation ' \|\| CALC_NAME`). |
| e | Add a new index `IDX_AUDIT_ACTOR` on the audit table `CALC_AUDIT_EVENT (ACTOR, EVENT_AT)`. |

Part e needs a hybrid table. In Mode S it is documented as unavailable and you write what you would have run.

## 2. Why this matters in calc-engine and calc-admin

> **Real world (NAIC).** The change request is a miniature of normal calc-engine work. Part a and b are engine content: a standard table and a procedure, which the target design gives to DCM. Part c is a rename, which the architecture page calls out explicitly: DCM can read a rename as "remove and add", so a coordinated protocol with an explicit Flyway rename comes first. Part d is a data change, which belongs to Flyway ("ordered, checksum-protected, idempotent where replayable, row-count asserted"). Part e is a hybrid index on a Calc Admin lifecycle table, which DCM cannot define and Flyway owns.
>
> The architecture decision is that both tools are implemented, each with its own lane, with Terraform for platform objects and a pipeline around them. The DCM project for INT is being re-adopted in roadmap phase 2 ("DCM core re-adoption in INT": an adoption-only plan, unchanged data and grants, an empty re-plan). Flyway boundaries and clone bootstrap are phase 3. A minimal merged-result plan plus a manual INT pipeline is phase 4. This capstone lets you compare the whole design with the simpler alternatives before you form an opinion.

## 3. Prerequisites and state check

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`).

```sql
SHOW SCHEMAS LIKE 'CAP_%' IN DATABASE SBX_INT;           -- CAP_HAND, CAP_FLYWAY, CAP_DCM
SHOW DATABASES LIKE 'SBX_DEV_%';                         -- 0 rows
SELECT "version", "success" FROM SBX_INT.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT ORDER BY "installed_rank";   -- 1 to 5, success
```

If a `CAP_*` schema is missing, create it as `SYSADMIN` and hand it over:

```sql
USE ROLE SYSADMIN;
CREATE SCHEMA IF NOT EXISTS SBX_INT.CAP_HAND;     -- repeat for CAP_FLYWAY and CAP_DCM
GRANT OWNERSHIP ON SCHEMA SBX_INT.CAP_HAND TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
```

Keep a notes table to fill after each way: time taken in minutes, number of manual steps, mistakes made, what evidence exists afterwards, how you would undo it.

## 4. Concepts

Everything here was introduced in earlier labs. The capstone adds two ideas:

**Ownership by change type.** The question is not "which tool is better" but "which tool owns this kind of change". Each of the five parts above has an owner in the full design. You will write that table yourself after way 4.

**Evidence.** After a change, what can you show? A change you cannot show later is a change you cannot audit. Each way leaves a different amount of evidence.

## 5. Step-by-step

### Step 5.0 The common starting state

Run in: **Terminal**. Create the file `sql/capstone/capstone_seed.sql`. It builds the same small engine in whatever schema you select with `USE SCHEMA`. It is the "before" picture for ways 1 to 3. Way 4 starts from your real repository state instead.

```sql
-- Run after:  USE ROLE SBX_CICD_PIPELINE;  USE WAREHOUSE SBX_CALC_WH;  USE SCHEMA SBX_INT.<CAP_HAND | CAP_FLYWAY | CAP_DCM>;
CREATE SEQUENCE IF NOT EXISTS SEQ_CALC_ID START = 3001 INCREMENT = 1;

CREATE TABLE IF NOT EXISTS CALCULATIONS (
  CALC_ID NUMBER(38,0) NOT NULL DEFAULT SEQ_CALC_ID.NEXTVAL,
  CALC_NAME VARCHAR(255) NOT NULL,
  CALC_STATUS VARCHAR(10) DEFAULT 'ACTIVE',
  CREATED_BY VARCHAR(255),
  PRIMARY KEY (CALC_ID)
);

CREATE TABLE IF NOT EXISTS FORMULAS (
  FORMULA_ID NUMBER(38,0) NOT NULL AUTOINCREMENT,
  CALC_ID NUMBER(38,0),
  FORMULA_TEXT VARCHAR(4000),
  PRIMARY KEY (FORMULA_ID)
);

CREATE OR REPLACE PROCEDURE SP_REGISTER_CALCULATION(P_CALC_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE v_id NUMBER;
BEGIN
  SELECT SEQ_CALC_ID.NEXTVAL INTO :v_id;
  INSERT INTO CALCULATIONS (CALC_ID, CALC_NAME, CALC_STATUS, CREATED_BY)
  SELECT :v_id, :P_CALC_NAME, 'DRAFT', :P_USER;
  RETURN TO_VARCHAR(:v_id);
END
$$;

INSERT INTO CALCULATIONS (CALC_NAME, CALC_STATUS, CREATED_BY)
VALUES ('CAP_LIFE_RATIO', 'ACTIVE', 'seed'), ('CAP_PROPERTY_TOTAL', 'ACTIVE', 'seed'), ('CAP_HEALTH_RATIO', 'ACTIVE', 'seed');
INSERT INTO FORMULAS (CALC_ID, FORMULA_TEXT)
SELECT CALC_ID, 'SUM(' || CALC_NAME || ')' FROM CALCULATIONS;

-- Mode H (hybrid tables): run this statement
CREATE HYBRID TABLE IF NOT EXISTS CALC_AUDIT_EVENT (
  EVENT_ID NUMBER(38,0) NOT NULL AUTOINCREMENT,
  CALC_ID NUMBER(38,0), EVENT_TYPE VARCHAR(40), EVENT_STATUS VARCHAR(10),
  ACTOR VARCHAR(255), EVENT_AT TIMESTAMP_NTZ,
  PRIMARY KEY (EVENT_ID)
);
-- Mode S (no hybrid): run this one instead (a standard stand-in, key not enforced)
-- CREATE TABLE IF NOT EXISTS CALC_AUDIT_EVENT ( ...same columns... );

INSERT INTO CALC_AUDIT_EVENT (CALC_ID, EVENT_TYPE, EVENT_STATUS, ACTOR, EVENT_AT)
SELECT CALC_ID, 'CREATE', 'COMPLETE', 'seed', CURRENT_TIMESTAMP() FROM CALCULATIONS;
```

> **TO-CONFIRM [TC-11-01]:** That `AUTOINCREMENT` in a column list with `NOT NULL` and an out-of-line `PRIMARY KEY` is accepted on a hybrid table in the order written above. The create-table grammar allows both. How to check at lab time: run the seed in `CAP_HAND`. If the hybrid statement fails, copy the error into file 13 and switch that schema to the Mode S statement. Record the result in file 13.

Run the seed in `CAP_HAND` now (worksheet, set `USE SCHEMA SBX_INT.CAP_HAND` first). You should see 3 rows in `CALCULATIONS` (ids 3001 to 3003), 3 formulas, and 3 audit events. You will run it in the other two schemas just before ways 2 and 3.

Mode S: also remember that `PRIMARY KEY` on the standard stand-in is not enforced.

### Step 5.1 Way 1: hand-run SQL (about 20 minutes)

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`, `USE SCHEMA SBX_INT.CAP_HAND`. Start a stopwatch.

```sql
-- a. add the column
ALTER TABLE CALCULATIONS ADD COLUMN CALC_DESCRIPTION VARCHAR(500);

-- b. change the procedure (silent overwrite of the old version)
CREATE OR REPLACE PROCEDURE SP_REGISTER_CALCULATION(P_CALC_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE v_id NUMBER;
BEGIN
  SELECT SEQ_CALC_ID.NEXTVAL INTO :v_id;
  INSERT INTO CALCULATIONS (CALC_ID, CALC_NAME, CALC_STATUS, CREATED_BY, CALC_DESCRIPTION)
  SELECT :v_id, TRIM(:P_CALC_NAME), 'DRAFT', :P_USER, 'Registered via SP_REGISTER_CALCULATION';
  RETURN TO_VARCHAR(:v_id);
END
$$;

-- c. rename
ALTER TABLE FORMULAS RENAME TO CALC_FORMULAS;

-- d. data fix: nothing counts the rows for you
UPDATE CALCULATIONS SET CALC_DESCRIPTION = 'Legacy calculation ' || CALC_NAME WHERE CALC_DESCRIPTION IS NULL;

-- e. index (Mode H only)
CREATE INDEX IDX_AUDIT_ACTOR ON CALC_AUDIT_EVENT (ACTOR, EVENT_AT);
```

You should see: every statement succeeds, the update reports 3 rows.

Now check the result and the evidence:

```sql
SELECT CALC_ID, CALC_NAME, CALC_DESCRIPTION FROM CALCULATIONS ORDER BY CALC_ID;
SHOW INDEXES IN TABLE CALC_AUDIT_EVENT;                  -- Mode H
SELECT QUERY_TEXT, START_TIME FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_USER(RESULT_LIMIT => 20)) ORDER BY START_TIME DESC;
```

Write in your notes: how long it took, what proof you can show tomorrow that this happened (only query history, which is not designed as an audit trail and has limited retention), whether anyone can tell a hand edit from an official change, and how you would undo the rename and the data fix.

Mistake to try in 30 seconds: switch to `USE SCHEMA SBX_INT.CAP_FLYWAY`, run statement b again, and notice that nothing stops you. If that schema already has its own procedure, you just overwrote it. If it has none, you just created a stray copy. Neither is reported anywhere. That is what "no tool" means. Switch back and drop the stray copy: `DROP PROCEDURE IF EXISTS SBX_INT.CAP_FLYWAY.SP_REGISTER_CALCULATION(VARCHAR, VARCHAR);` before way 2 (way 2 seeds the schema itself).

### Step 5.2 Way 2: Flyway only (about 40 minutes)

Run in: **Snowsight worksheet**: `USE SCHEMA SBX_INT.CAP_FLYWAY;` then run `sql/capstone/capstone_seed.sql`. This is the schema Flyway will adopt, as in the Lab 2 baseline.

Run in: **Terminal**. Create a small separate Flyway project. (New name, a helper of this capstone: `flyway/capstone_flyway/`.)

`flyway/capstone_flyway/flyway.toml`:

```toml
[environments.target]
url = "jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE"
user = "SBX_SVC_CICD"

[flyway]
environment = "target"
locations = ["filesystem:migrations"]
defaultSchema = "CAP_FLYWAY"
schemas = ["CAP_FLYWAY"]
table = "FLYWAY_HISTORY_CAP_FLYWAY"
baselineVersion = "1"
cleanDisabled = true
validateOnMigrate = true
outOfOrder = false
```

Migrations in `flyway/capstone_flyway/migrations/`:

`V2__add_calc_description.sql`

```sql
ALTER TABLE CALCULATIONS ADD COLUMN CALC_DESCRIPTION VARCHAR(500);
```

`V3__rename_formulas_to_calc_formulas.sql`

```sql
ALTER TABLE FORMULAS RENAME TO CALC_FORMULAS;
```

`V4__backfill_calc_description.sql` (data fix with an assertion: the number of rows updated must equal the number counted as pending, and none may remain):

```sql
EXECUTE IMMEDIATE $$
DECLARE
  pending NUMBER; updated NUMBER; remaining NUMBER;
  e_assert EXCEPTION (-20020, 'data fix assertion failed');
BEGIN
  BEGIN TRANSACTION;
  SELECT COUNT(*) INTO :pending FROM CALCULATIONS WHERE CALC_DESCRIPTION IS NULL;
  UPDATE CALCULATIONS SET CALC_DESCRIPTION = 'Legacy calculation ' || CALC_NAME WHERE CALC_DESCRIPTION IS NULL;
  updated := SQLROWCOUNT;
  SELECT COUNT(*) INTO :remaining FROM CALCULATIONS WHERE CALC_DESCRIPTION IS NULL;
  IF (updated <> pending OR remaining <> 0) THEN
    ROLLBACK;
    RAISE e_assert;
  END IF;
  COMMIT;
  RETURN 'rows updated: ' || updated;
END;
$$;
```

> **TO-CONFIRM [TC-G08]:** Whether Flyway 13.12.2 parses a `$$ ... $$` Snowflake Scripting block that contains semicolons as one statement. How to check at lab time: run `flyway migrate` and read the error, if any. Fallback from the contract: two plain statements (the `UPDATE`, then a `SELECT` that checks `COUNT(*)`). Record the result in file 13.

`V5__add_audit_actor_index.sql` (Mode H):

```sql
CREATE INDEX IDX_AUDIT_ACTOR ON CALC_AUDIT_EVENT (ACTOR, EVENT_AT);
```

In Mode S write a no-op instead: `SELECT 1;` with a comment that the stand-in has no secondary index.

`R__sp_register_calculation.sql` (a repeatable migration: the whole procedure is rewritten and re-applied when its text changes):

```sql
CREATE OR REPLACE PROCEDURE SP_REGISTER_CALCULATION(P_CALC_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE v_id NUMBER;
BEGIN
  SELECT SEQ_CALC_ID.NEXTVAL INTO :v_id;
  INSERT INTO CALCULATIONS (CALC_ID, CALC_NAME, CALC_STATUS, CREATED_BY, CALC_DESCRIPTION)
  SELECT :v_id, TRIM(:P_CALC_NAME), 'DRAFT', :P_USER, 'Registered via SP_REGISTER_CALCULATION';
  RETURN TO_VARCHAR(:v_id);
END
$$;
```

Run in: **Terminal**:

```bash
cd flyway/capstone_flyway
set -a; source ../.env.flyway; set +a
export FLYWAY_LOCATIONS="filesystem:migrations"            # the shared env file points elsewhere
flyway baseline -baselineVersion=1 -baselineDescription="capstone seed state"
flyway info
flyway migrate
flyway info
```

You should see: `baseline` writes version 1 as the baseline. `info` lists versions 2 to 5 and the repeatable as pending. `migrate` applies them in order, repeatables last. The second `info` shows all `Success`.

Check the data:

```sql
USE SCHEMA SBX_INT.CAP_FLYWAY;
SELECT CALC_ID, CALC_NAME, CALC_DESCRIPTION FROM CALCULATIONS ORDER BY CALC_ID;
SELECT * FROM FLYWAY_HISTORY_CAP_FLYWAY ORDER BY 1;
```

Now test the limits you learned in Lab 2:

1. **Preview.** Before `migrate`, was there any way to see what the five files would change in the database, without running them? (No dry run in Community.)
2. **Drift.** In a worksheet run `ALTER TABLE CALC_FORMULAS ADD COLUMN SNEAKY VARCHAR;` then `flyway validate`. It passes. Remove the column again.
3. **Rollback.** Undo version 3. Community has no undo. You would write a new migration that renames back.

Write in your notes: time, what evidence remains (the history table: version, description, user, time, checksum, success), what you could not see before running, and how the procedure change was handled (the whole file is replaced, there is no per-line diff).

### Step 5.3 Way 3: DCM only (about 50 minutes)

Mode D only. In Mode D-off, read this section and the "You should see" blocks, then do the same changes by hand as in way 1 and compare in your notes.

Run in: **Snowsight worksheet**: `USE SCHEMA SBX_INT.CAP_DCM;` then run `sql/capstone/capstone_seed.sql`.

Run in: **Terminal**. Create a separate DCM project folder (new names, helpers of this capstone): `dcm/capstone/` with a manifest and definitions. Grant the right to create a project object first:

```sql
-- Snowsight, role SBX_CICD_PIPELINE (owner of the schema; the explicit grant is only to make the privilege visible)
GRANT CREATE DCM PROJECT ON SCHEMA SBX_INT.CAP_DCM TO ROLE SBX_CICD_PIPELINE;
```

`dcm/capstone/manifest.yml`:

```yaml
manifest_version: 2
type: DCM_PROJECT
default_target: 'CAP'
targets:
  CAP:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_INT.CAP_DCM.CAP_DCM_PROJECT'
    project_owner: SBX_CICD_PIPELINE
```

`dcm/capstone/sources/definitions/SEQ_CALC_ID.sql`:

```sql
DEFINE SEQUENCE SBX_INT.CAP_DCM.SEQ_CALC_ID start with 3001 increment by 1 noorder;
```

`dcm/capstone/sources/definitions/CALCULATIONS.sql`:

```sql
DEFINE TABLE SBX_INT.CAP_DCM.CALCULATIONS (
  CALC_ID NUMBER(38,0) NOT NULL DEFAULT SBX_INT.CAP_DCM.SEQ_CALC_ID.NEXTVAL,
  CALC_NAME VARCHAR(255) NOT NULL,
  CALC_STATUS VARCHAR(10) DEFAULT 'ACTIVE',
  CREATED_BY VARCHAR(255),
  primary key (CALC_ID)
);
```

`dcm/capstone/sources/definitions/FORMULAS.sql`:

```sql
DEFINE TABLE SBX_INT.CAP_DCM.FORMULAS (
  FORMULA_ID NUMBER(38,0) NOT NULL autoincrement start 1 increment 1 noorder,
  CALC_ID NUMBER(38,0),
  FORMULA_TEXT VARCHAR(4000),
  primary key (FORMULA_ID)
);
```

`dcm/capstone/sources/definitions/SP_REGISTER_CALCULATION.sql` (the v1 body from the seed, in `DEFINE` form):

```sql
DEFINE PROCEDURE SBX_INT.CAP_DCM.SP_REGISTER_CALCULATION(P_CALC_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE v_id NUMBER;
BEGIN
  SELECT SEQ_CALC_ID.NEXTVAL INTO :v_id;
  INSERT INTO CALCULATIONS (CALC_ID, CALC_NAME, CALC_STATUS, CREATED_BY)
  SELECT :v_id, :P_CALC_NAME, 'DRAFT', :P_USER;
  RETURN TO_VARCHAR(:v_id);
END
$$;
```

Not defined, on purpose: `CALC_AUDIT_EVENT`. DCM does not list hybrid tables as supported.

> **TO-CONFIRM [TC-G03]:** Exact `DEFINE SEQUENCE`, `DEFINE TABLE` and `DEFINE PROCEDURE` grammar. The shapes above come from the working NAIC branch and the public `DEFINE TABLE` example. How to check at lab time: the supported-entities page and `snow init --template dcm_project`. Record the result in file 13.

Create the project, adopt, and read the plan, as in Lab 3:

```bash
snow dcm create SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE
snow dcm plan   SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --save-output
```

You should see: a plan of adoption alters for the four objects. If a table definition differs from the live table in a detail, the plan shows an `ALTER` that is not just adoption. Read it and adjust the definition until the plan matches what Lab 3 taught you to expect. Then deploy with an alias and re-plan until empty:

```bash
snow dcm deploy SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --alias "cap-adopt"
snow dcm plan   SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --save-output
```

Now apply the change request.

**a and b.** Edit `CALCULATIONS.sql`: add `CALC_DESCRIPTION VARCHAR(500),` as the **last** column before the `primary key` line (reordering columns is not supported). Edit `SP_REGISTER_CALCULATION.sql`: use `TRIM(:P_CALC_NAME)` and add the description column and value, as in way 1.

**c, the unsafe way first.** Delete `FORMULAS.sql`. Create `CALC_FORMULAS.sql` with the same columns and the new name. Plan:

```bash
snow dcm plan SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --save-output
python3 scripts/check_plan_for_drops.py dcm/capstone/out/plan*.json
```

You should see: a `DROP` of `FORMULAS` and a `CREATE` of `CALC_FORMULAS`, and `check_plan_for_drops.py` exits 1. Deploying this would lose the three formula rows.

> **TO-CONFIRM [TC-G06]:** That DCM reads a renamed definition as drop plus create. The docs do not describe renames at all, and the architecture page calls it the "unsafe remove/add interpretation". How to check at lab time: the plan output above. Record the result in file 13.

**c, the way that works.** DCM alone has no rename. Do it by hand first, outside DCM, then plan again:

```sql
-- Snowsight, role SBX_CICD_PIPELINE
ALTER TABLE SBX_INT.CAP_DCM.FORMULAS RENAME TO CALC_FORMULAS;
```

```bash
snow dcm plan SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --save-output
python3 scripts/check_plan_for_drops.py dcm/capstone/out/plan*.json
```

You should see: no drop of the data-bearing table. Read what the plan now says about `FORMULAS` and `CALC_FORMULAS`. Do not predict it: this is the question the experiment answers (the old managed entity no longer exists, and the new table has a definition). Write down exactly what it listed.

> **TO-CONFIRM [TC-11-02]:** What the plan lists after a hand-run rename (the old entity disappears, the new table is adopted). How to check at lab time: read the plan. If it still contains a `DROP` of a data-bearing table, stop and do not deploy. Record the result in file 13.

Deploy only after the plan has no drop of a table with data:

```bash
snow dcm deploy SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --alias "cap-change"
snow dcm plan   SBX_INT.CAP_DCM.CAP_DCM_PROJECT --target CAP --from dcm/capstone -c sbx_owner --role SBX_CICD_PIPELINE --save-output
```

You should see: one deployment in `SHOW DEPLOYMENTS IN DCM PROJECT SBX_INT.CAP_DCM.CAP_DCM_PROJECT`, and an empty re-plan.

**d and e, outside DCM.** DCM cannot run row changes and cannot define a hybrid index. Do both by hand:

```sql
USE SCHEMA SBX_INT.CAP_DCM;
UPDATE CALCULATIONS SET CALC_DESCRIPTION = 'Legacy calculation ' || CALC_NAME WHERE CALC_DESCRIPTION IS NULL;
CREATE INDEX IDX_AUDIT_ACTOR ON CALC_AUDIT_EVENT (ACTOR, EVENT_AT);     -- Mode H
```

Write in your notes: time; what DCM gave you (a preview plan, an empty re-plan, deployment history with an alias); what it could not do (rename safely, data, hybrid index, so two of the five parts left no record in any tool); and the rollback story (`FROM DEPLOYMENT` can re-deploy older definitions, but not data).

> **VERIFIED:** DCM supports tables, sequences, procedures and grants; hybrid tables, users, integrations and resource monitors are not in the supported-entities list; `FROM DEPLOYMENT` re-plans or re-deploys a previous deployment's rendered definitions (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities and .../dcm-projects-use, read 2026-11-16).

### Step 5.4 Way 4: DCM, Flyway and the pipeline (about 70 minutes)

This is the design NAIC is building. You work on a private clone of INT, in a branch, with each part owned by one tool.

#### 5.4.1 The ownership map for this change

Fill the "tool" column before you start, then compare with the answer table in step 5.6.

| Part | Change | Tool that owns it |
|---|---|---|
| a | add column to `CALCULATIONS` | ? |
| b | change `SP_REGISTER_CALCULATION` body | ? |
| c | rename `FORMULAS` to `CALC_FORMULAS` | ? |
| d | backfill descriptions | ? |
| e | index `IDX_AUDIT_ACTOR` on the hybrid `CALC_AUDIT_EVENT` | ? |

#### 5.4.2 Make the clone and the branch

Run in: **Terminal**:

```bash
scripts/refresh_dev_clone.sh CAP4 --yes
git switch -c feature/CAP4
```

You run git yourself. You should see the clone `SBX_DEV_CAP4` with an empty re-plan at the end of the script.

#### 5.4.3 Edit the DCM definitions (parts a, b, c)

Run in: **Terminal** (editor). Paths are in `dcm/calc_engine/sources/definitions/`.

1. `tables/CALCULATIONS.sql`: add `CALC_DESCRIPTION VARCHAR(500),` as the **last** column, before the `primary key` line.
2. `procedures/SP_REGISTER_CALCULATION.sql`: change the name expression to `TRIM(:P_CALC_NAME)` and add the `CALC_DESCRIPTION` column and its value `'Registered via SP_REGISTER_CALCULATION'` to the insert. Keep the rest of the body as Lab 3 left it.
3. `tables/FORMULAS.sql`: rename the file to `CALC_FORMULAS.sql` and change the name inside the `DEFINE TABLE` line.
4. `grants/access.sql`: replace `FORMULAS` with `CALC_FORMULAS` in the grants.

(In Mode D-off skip this step and keep a list of what you would change.)

#### 5.4.4 Write the Flyway migrations (parts c, d, e)

Run in: **Terminal**. Files in `flyway/migrations/`:

`common/V6__rename_formulas_to_calc_formulas.sql`

```sql
ALTER TABLE IF EXISTS FORMULAS RENAME TO CALC_FORMULAS;
```

The `IF EXISTS` form makes the file safe on a brand-new environment, where DCM has already created `CALC_FORMULAS` and `FORMULAS` never existed. That is the lesson of TC-08-04 in Lab 6.

> **TO-CONFIRM [TC-11-03]:** That `ALTER TABLE IF EXISTS ... RENAME TO` is a no-op when the table is missing. The syntax is standard Snowflake `ALTER TABLE`. How to check at lab time: run it in a worksheet against a name that does not exist. Record the result in file 13.

`common/V7__backfill_calc_description.sql` (this one writes the ledger row the pack's pattern asks for, and asserts against the **pending count**, so it also works in an environment with different rows. Compare with `V3` of Lab 4, which used a constant `EXPECTED_ROWS`):

```sql
EXECUTE IMMEDIATE $$
DECLARE
  pending NUMBER; updated NUMBER; remaining NUMBER;
  e_assert EXCEPTION (-20021, 'V7 assertion failed');
BEGIN
  BEGIN TRANSACTION;
  SELECT COUNT(*) INTO :pending FROM CALCULATIONS WHERE CALC_DESCRIPTION IS NULL;
  UPDATE CALCULATIONS SET CALC_DESCRIPTION = 'Legacy calculation ' || CALC_NAME WHERE CALC_DESCRIPTION IS NULL;
  updated := SQLROWCOUNT;
  SELECT COUNT(*) INTO :remaining FROM CALCULATIONS WHERE CALC_DESCRIPTION IS NULL;
  IF (updated <> pending OR remaining <> 0) THEN
    ROLLBACK;
    RAISE e_assert;
  END IF;
  INSERT INTO DATA_CHANGE_LOG (CHANGE_VERSION, DESCRIPTION, EXPECTED_ROWS, ROWS_AFFECTED)
  SELECT '7', 'backfill CALC_DESCRIPTION', :pending, :updated;
  COMMIT;
  RETURN 'rows updated: ' || updated;
END;
$$;
```

`hybrid/V8__add_audit_actor_index.sql`

```sql
CREATE INDEX IDX_AUDIT_ACTOR ON CALC_AUDIT_EVENT (ACTOR, EVENT_AT);
```

`standard/V8__add_audit_actor_index.sql` (Mode S, a documented no-op)

```sql
-- Mode S stand-in: no secondary indexes on standard tables. Keeps version numbers aligned with Mode H.
SELECT 1;
```

Tell the pipeline that version 6 is a prerequisite of the DCM step. Put the number in the file `flyway/PRE_DCM_TARGET`:

```
6
```

#### 5.4.5 Run it by hand on the clone, in the right order

Run in: **Terminal**. Use the helpers from Lab 8 (`dplan`, `ddeploy`, `flyclone`), but point them at the new clone first:

```bash
export CLONE_DB=SBX_DEV_CAP4
export PROJECT="$CLONE_DB.CALC_SUPPORT.CALC_ENGINE_DCM"
```

**Step A. See the unsafe plan first** (protocol step 2: plan against the pre-migration state):

```bash
dplan
python3 scripts/check_plan_for_drops.py dcm/calc_engine/out/plan*.json; echo "exit: $?"
```

You should see: a `DROP` of `FORMULAS` and a `CREATE` of `CALC_FORMULAS`. The drop check exits 1. This is the unsafe interpretation. Do not deploy.

**Step B. Prerequisite migration: the explicit rename** (Flyway first):

```bash
flyclone info
flyclone migrate -target=6
```

You should see: version 6 applied, versions 7 and 8 still pending. (`-target` is TC-08-06.) The table was renamed with its rows, its grants and its ownership unchanged.

**Step C. Plan against the final definitions.**

```bash
dplan
python3 scripts/check_plan_for_drops.py dcm/calc_engine/out/plan*.json; echo "exit: $?"
```

You should see: no `DROP` of a data-bearing table, and exit 0. Read every entry: the new column, the procedure body, and the adoption of the renamed table. Any unexpected create or drop is a reason to stop.

**Step D. Deploy the definitions.**

```bash
ddeploy --alias "cap4-defs"
```

**Step E. Remaining migrations: data fix and index** (they need the new column and the hybrid table):

```bash
flyclone migrate
flyclone info
```

You should see: versions 7 and 8 applied.

**Step F. Empty re-plan and smoke test.**

```bash
dplan
python3 scripts/assert_plan_empty.py dcm/calc_engine/out/plan*.json
```

Run in: **Snowsight worksheet** (role `SBX_CALC_DEV`):

```sql
CALL SBX_DEV_CAP4.CALC_SUPPORT.SP_REGISTER_CALCULATION('  CAP4_SMOKE  ', 'dheeraj');
SELECT CALC_ID, CALC_NAME, CALC_DESCRIPTION FROM SBX_DEV_CAP4.CALC_SUPPORT.CALCULATIONS ORDER BY CALC_ID DESC LIMIT 5;
SELECT * FROM SBX_DEV_CAP4.CALC_SUPPORT.DATA_CHANGE_LOG ORDER BY CHANGE_ID DESC LIMIT 1;
SHOW INDEXES IN TABLE SBX_DEV_CAP4.CALC_SUPPORT.CALC_AUDIT_EVENT;      -- Mode H
```

You should see: the new calculation has the trimmed name `CAP4_SMOKE` and the description from the procedure; all older rows have `Legacy calculation ...`; the ledger shows version 7 with `EXPECTED_ROWS` equal to `ROWS_AFFECTED`; the hybrid index `IDX_AUDIT_ACTOR` is `ACTIVE`. The re-plan is empty.

Zero writes to INT: run the two count queries from Lab 5 or Lab 8 (drill 7) and compare with your baseline.

> **Design reference (NAIC, not a public doc):** The architecture page `calc-engine-complete-architecture.html`, section 7, defines a six-step coordinated rename protocol: review the explicit rename and the final definition in one change; full plan against the pre-migration state and verify the unsafe remove-and-add interpretation; capture backups, consumers, owners, grants, row counts and fingerprints; apply the explicit rename through Flyway without transferring ownership; new full plan against the final definition and reject unexpected creates or drops; deploy DCM, require an empty re-plan, and run contract and parity tests. Steps A to F above are the sandbox version of it (step 3, the capture, is the part you do by reading counts and grants before step B).

#### 5.4.6 Let the pipeline do the same

Run in: **Terminal**:

```bash
git add dcm flyway
git commit -m "feat: capstone change request on clone"
git push -u origin feature/CAP4
```

You run these git commands yourself. Open a pull request in **GitHub UI**. Do not merge it.

You should see: `validate` is green, `pr-clone-check` runs the same steps you just ran by hand (prerequisite migrations because of `PRE_DCM_TARGET`, plan, drop check, deploy, migrate, empty re-plan, smoke, zero-writes), and the `cleanup` job drops the pull request clone.

Close the pull request without merging. Merging would change `SBX_INT` and move your Lab 7 end state. If you want to see the INT deploy, you can merge, approve `int`, and later fix forward, but nothing after this file needs it.

If the pipeline fails in the rename or data fix on a brand-new environment, that is the TC-08-04 finding from Lab 6, and the tolerant `IF EXISTS` and pending-count forms above are the fix.

#### If you are in Mode S (no hybrid tables)

Use the `standard/V8` no-op. Part e is documented as unavailable: write one sentence on what you would have lost (an index that speeds lookups by actor) and what the standard table does instead.

#### If DCM is not available (Mode D-off)

Apply parts a and b by hand with `ALTER TABLE ... ADD COLUMN` and `CREATE OR REPLACE PROCEDURE`, then re-run the grants. Keep Flyway steps B and E. Note in your table that re-grants and the drop check are manual chores now.

### Step 5.5 Fill in the notes table

| Item | Way 1 hand-run | Way 2 Flyway | Way 3 DCM | Way 4 all + pipeline |
|---|---|---|---|---|
| Minutes spent | | | | |
| Manual steps | | | | |
| Mistakes or surprises | | | | |
| Could you preview before changing? | | | | |
| Proof you can show tomorrow | | | | |
| Could you detect a hand edit? | | | | |
| How would you roll back? | | | | |

### Step 5.6 Ownership answer, comparison matrix and decision guide

#### The answer to the ownership map

| Part | Change | Owner in the full design | Why |
|---|---|---|---|
| a | add column to `CALCULATIONS` | DCM | A standard table is a supported entity. The definition is the desired end state, and the plan previews the alter. |
| b | procedure body | DCM | Procedures are supported. One definition per object, per-object diff, grants re-applied. |
| c | rename a table | Flyway first, then the DCM definition | DCM reads a rename as drop plus create (TC-G06). The explicit rename keeps the rows. |
| d | data fix | Flyway | DCM does not handle row data. Flyway gives ordering, a checksum and an assertion. |
| e | hybrid index | Flyway | Hybrid tables and indexes are not in DCM's supported list. |

The ordering rules from file 06: a normal change runs DCM plan, review, DCM deploy, Flyway migrate, then a DCM re-plan that must be empty. A rename runs Flyway first, then the DCM definition, then an empty re-plan.

#### Comparison matrix

Fill the "your measure" notes from step 5.5. The text in the cells is what the labs showed.

| Dimension | 1 Hand-run SQL | 2 Flyway only | 3 DCM only | 4 DCM + Flyway + pipeline |
|---|---|---|---|---|
| Effort for this change | Lowest. Five statements, no setup. | Setup of project and files, then low per change. | Setup of manifest and definitions, then low per change; three parts need hand work. | Highest setup (clone, workflows, secrets). Each change then runs the same way. |
| Safety against mistakes | None. Wrong schema or order is silent. | Ordered and checksummed, but no preview and a half-failed file stays half-applied. | Plan shows creates, alters and drops. A deploy drops what is undefined. | Plan plus drop check plus clone test plus empty re-plan plus zero-write check. |
| Drift detection | Hand query only | None in Community | Full plan | DCM objects only; Flyway-owned objects have none |
| Audit and evidence | Query history only | History table: version, user, time, checksum | Deployment history with alias | Both plus Git, pull request, pipeline logs and approvals |
| Rollback | None | None (fix forward) | Re-deploy earlier definitions, not data | Same, plus clones to rehearse and Time Travel for data. No universal rollback. |
| Preview | None | None (no dry run in Community) | Yes | Yes, in the pull request |
| Team collaboration | Overwrites each other | Version clashes at merge | One project per developer or ticket | Clone and project per ticket, reviewed in pull requests |
| Cost | Nothing extra | Docker image, tiny compute | Tiny compute | Clone storage until data diverges, runner minutes, a warehouse for a few minutes per run |
| Learning curve | Lowest | Low | Medium (templating, plan reading, ownership) | Highest (all of it plus CI, secrets, identity) |
| Handles all five parts? | Yes, with no protection | Mostly; procedure is a whole-file replace | Parts a and b only (c is unsafe, d and e impossible) | Yes, each by its owner |

> **VERIFIED:** Flyway Community has no undo, no dry run and no drift check; DCM plan compares definitions with live state and DEPLOY drops what is no longer defined; hybrid tables are not in the DCM supported list (sources as in files 04 and 05, read 2026-11-16).

#### Decision guide: what would you choose for the calc engine, and why

The calc engine has these properties, in the order that matters:

1. **Most of the code is procedures and standard tables** (34 procedures, 5 functions, 24 tables in INT at the last count). The old way pasted a 6,700-line file by hand and silently overwrote newer versions. Per-object definitions with a plan are what that problem needs. That is DCM's lane.
2. **Some objects are hybrid tables** (Calc Admin lifecycle) and some changes are renames or row changes. Those cannot be DCM. That is Flyway's lane.
3. **Clones are the test environment**, and they need a procedure that is deployed outside both tools (`CREATE_CALC_DEV_CLONE`), plus careful handling of the hybrid history trap.
4. **Environments are many and approvals matter** (INT, QA, PROD with change requests). That needs a pipeline with protected environments and identities that do not travel with developers.

So the choice is way 4, with explicit ownership. The simpler ways each fit a narrower case:

| Way | Fits when | Does not fit the calc engine because |
|---|---|---|
| 1 Hand-run | A throwaway playground or a one-off investigation | There is no audit and no protection from the overwrite problem that started this whole effort. |
| 2 Flyway only | The platform is mostly tables and data, or the project is on a platform where Flyway is the standard (EDP's `FIN_CALCULATION`, fpr-admin on PostgreSQL) | The engine is procedure-heavy. A repeatable file replaces a whole procedure and gives no per-object diff, and there is no drift detection or plan. |
| 3 DCM only | A project with only supported objects and no data or hybrid tables | It cannot own the hybrid pair, the data changes or an explicit rename. |

What way 4 costs and risks, so you can argue the other side:

- More moving parts: two tools, a manifest, a pipeline, secrets or federation, clones. Every extra part can fail (Lab 8).
- Ownership discipline is manual: one tool per object. No tool enforces it today. The NAIC design proposes an ownership file and a check, and the branch does not have one yet.
- Flyway-owned objects have no drift detector in Community.
- The clone procedure and the Flyway-history trap need real attention.
- DEPLOY can do more than the reviewed plan. Pipeline checks reduce that risk, they do not remove it.
- Learning curve and review load for a small team.

NAIC's chosen path keeps both tools: DCM for supported engine objects (INT re-adoption is roadmap phase 2), Flyway for hybrid tables, renames, data changes, lifecycle schema and clone support (phase 3), Terraform for platform objects, and GitLab pipelines around all of it (phase 4 starts with a minimal merged-result plan and a manual INT pipeline). Your capstone is that design in miniature.

> **Design reference (NAIC):** Roadmap phases and the ownership split come from `calc-engine-complete-architecture.html` and `calc-engine-pipeline.html`. The real GitLab pipeline does not exist yet. This pack's sandbox versions are for learning, not a copy of any NAIC implementation.

### Step 5.7 Teach-back (30 minutes)

Run in: **a text file or a voice recording**, no tools. Explain the following in your own words, as if to a teammate who knows SQL but has never seen DCM or Flyway. Aim for about five minutes.

1. Why do we need these tools at all? Describe the old way and one concrete way it failed.
2. What does DCM do, in one sentence, and what is the single most dangerous thing it does?
3. What does Flyway do, in one sentence, and why can it not roll back on Snowflake?
4. Which tool owns which object, and what happens when two tools own the same object?
5. Walk through a rename change step by step and say why the order matters.
6. What is a clone for, what does `IGNORE HYBRID TABLES` have to do with it, and what is the history trap?
7. Why is the application's role not allowed to run migrations?
8. Name three things the pipeline checks before a change reaches INT.
9. What can still go wrong with all of it in place?

Self-check: if you needed to look something up for more than two items, redo the matching lab step. Write the lookups in your notes.

## 6. Checkpoint questions: the quiz

Thirty questions cover files 03 to 10. Answers follow. Write your answers first.

**Lab 1: the old way (file 03)**

1. What does `CREATE OR REPLACE PROCEDURE` do when a second developer runs a different version of the same procedure?
2. Name three pain points of the old way that you wrote down in Lab 1.
3. What does the drift detective compare, and what can it not do?

**Lab 2: Flyway fundamentals (file 04)**

4. What is stored in the Flyway history table, and what does the checksum protect?
5. What is the difference between a versioned (`V`) and a repeatable (`R__`) migration?
6. Why does a migration that fails on its second statement leave the first statement applied on Snowflake?
7. What does `outOfOrder = false` do to a migration numbered below the latest applied version?
8. What does `baseline` do, and when is it safe?
9. Why is `cleanDisabled` true by default, and why is that a good thing?

**Lab 3: DCM fundamentals (file 05)**

10. What does `DEFINE` behave like, and what does that mean for a table definition?
11. What is the empty re-plan, and what does it prove?
12. What happens to a managed object whose definition you delete, then deploy?
13. Why does DEPLOY not simply replay the plan you reviewed?
14. Name three kinds of objects DCM does not manage.
15. What does `DROP DCM PROJECT` do to the objects and to the deployment history?
16. Which privilege lets a role run a plan but not deploy?

**Lab 4: both tools together (file 06)**

17. Which tool owns hybrid tables, and why not the other?
18. A table is renamed. Which comes first, the Flyway rename or the DCM definition change, and why?
19. What goes wrong when two tools own the same object?
20. What are the parts of the data-change pattern?

**Lab 5: clones (file 07)**

21. Why does a schema clone containing hybrid tables need `IGNORE HYBRID TABLES`?
22. What is the Flyway-history trap in a clone, and how does the pack fix it?
23. Why is the clone procedure `EXECUTE AS OWNER`, and why does it never take a database name from the caller?
24. What does `DATA_RETENTION_TIME_IN_DAYS = 0` mean for a clone?

**Lab 6: CI/CD (file 08)**

25. Which GitHub feature replaces GitLab's `resource_group`, and why is `cancel-in-progress` false for deploys?
26. Why does the pull request job use an environment with no reviewer, and the INT job one with a reviewer?
27. What is the safer way to authenticate than a stored private key, and what still needs a key in the sandbox?

**Lab 7: calc-admin lifecycle (file 09)**

28. Why can the app role insert audit rows but not update or delete them?
29. What does event-first auditing leave behind when the process dies mid-way?
30. Why does NAIC run Flyway in a pipeline and not at application start?

**Lab 8: failure drills (file 10)**

31. A migration failed half-way. List the three recovery actions in order.
32. Why does a `--delta` plan not find drift?
33. A CI login fails with 390144. What do you do first?

### Answers

1. The last one to run wins, silently. Nothing warns that a newer version was overwritten.
2. Any three of: no history of who changed what; hand edits and the Git file disagree (drift); no safe test environment; a newer version overwritten; objects owned by different roles; grants lost after `CREATE OR REPLACE`; no preview.
3. It compares what is in `INFORMATION_SCHEMA` with what the SQL file says. It can find a difference. It cannot fix it, and it only knows what you queried.
4. One row per migration: version, description, type, checksum, who applied it, when, and success. The checksum detects an applied file that was edited later.
5. A versioned migration runs once in version order and is never edited after it ran. A repeatable runs after the versioned ones and again whenever its checksum changes.
6. Snowflake commits DDL immediately, so Flyway cannot wrap the file in a rollback.
7. Flyway ignores it (and `validate` may complain) instead of applying it late.
8. It records "the database is already at version N" without running anything. It is safe only after you verified the live shape equals what the scripts up to N would produce.
9. `clean` drops every object in the configured schemas. Disabled by default, it cannot be run by accident in a shared environment.
10. `CREATE OR ALTER`. The table definition is the desired end state, and a deploy alters the live table to match it, within the limits of `CREATE OR ALTER` (for example, columns cannot be reordered).
11. A plan whose change set is empty. It proves the live objects match the definitions at that moment.
12. DEPLOY drops it, with its rows.
13. DEPLOY recomputes a fresh change set against the live state. The live state may have changed since the plan.
14. Any three of: hybrid tables, users, integrations, resource monitors, row data, application roles.
15. Objects stay in place, now unmanaged. The deployment history artifacts are deleted.
16. `PLAN` (on the project object). Deploy needs `OWNERSHIP`.
17. Flyway. Hybrid tables and indexes are not in DCM's supported list.
18. The Flyway rename comes first. DCM would otherwise read the new name as drop plus create and lose the rows.
19. Each tool tries to make the object match its own idea: one drops what the other created, plans show noise, and results depend on the order of runs.
20. Count the rows to change, run an idempotent statement, assert the updated count equals the expected count, write one ledger row, and never edit the file after it ran.
21. Hybrid tables cannot be cloned at schema level. Without the clause the clone fails. Database-level clones can copy them physically, at a cost that grows with data size.
22. The cloned history says the hybrid migrations already ran, but the hybrid tables were not cloned. The pack recreates the hybrid tables empty from `GET_DDL`, so history and objects agree.
23. The developer role cannot create databases or take ownership. The procedure runs with the owner's rights, and the only caller input is a validated ticket, so a caller cannot make it touch a database it should not.
24. No Time Travel, no `UNDROP`, and lower storage cost. A dropped table in the clone cannot be restored.
25. `concurrency: group`. Cancelling a running deploy can leave half of the DDL applied, and Snowflake DDL commits as it goes.
26. The pull request job must run without a human click and only touches a throwaway clone. The INT deployment changes a shared environment, so a person must approve it.
27. OIDC federation (workload identity), which uses a short-lived token. Flyway still needs a key file, because the Flyway driver page does not list workload identity.
28. The grants give it `SELECT` and `INSERT` on the audit table only. The procedure writes through the owner's rights.
29. A `PENDING` event with no outcome. It marks the call as indeterminate and in need of reconciling.
30. The application credential would need DDL rights, parallel instances would race, a failed migration would block the application, production changes need approved pipelines, and Snowflake DDL cannot be rolled back.
31. Drop the leftover objects the failed file created; run `repair`; fix the never-applied file and migrate again.
32. It only evaluates changed definitions and what depends on them, and you changed no definition.
33. Get the UUID from the error and ask an administrator for the detailed code (`SYSTEM$GET_LOGIN_FAILURE_DETAILS`). Then compare fingerprints, clock, account and login name.

Scoring: 28 or more right means you can explain the design. Fewer than 20 means redo the labs where you missed questions: the question numbers are grouped by lab.

## 7. Break it

Do way 4 in the wrong order, on purpose, in the clone only.

1. Refresh the clone: `scripts/refresh_dev_clone.sh CAP4 --yes`. Keep the definition edits, delete migration V6 from your working copy (or move it away temporarily).
2. Run `dplan` and see the drop of `FORMULAS`.
3. Run `ddeploy` anyway, ignoring the drop check.
4. Symptom: the deploy succeeds. `SELECT COUNT(*) FROM SBX_DEV_CAP4.CALC_SUPPORT.CALC_FORMULAS;` returns 0, and `FORMULAS` does not exist.
5. Diagnosis: the three seed formula rows were in `FORMULAS`. DCM dropped that table and created an empty `CALC_FORMULAS`. In the clone with retention 0 there is no `UNDROP`.
6. Recovery: refresh the clone, restore V6, do it in the right order. In a real environment the recovery would be a restore from Time Travel or a backup, which is why the protocol puts the explicit rename first.

Lesson: all the safety was a plan you could read. The tool did what the definitions said.

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| DCM cannot rename safely | Way 3 step c | Flyway rename first, then definition | TO-CONFIRM (TC-G06, TC-11-02) |
| DCM cannot do data changes or hybrid indexes | Way 3 steps d and e | Flyway | VERIFIED (supported-entities list) |
| Flyway Community has no preview, drift or rollback | Way 2 | DCM plan for DCM objects, fix forward | VERIFIED |
| Hand-run SQL leaves no audit | Way 1 | Any tool plus Git | VERIFIED (observed) |
| Migrations may not replay on new environments | Way 4 and file 08 | Tolerant forms (`IF EXISTS`, pending-count assertions) | TO-CONFIRM (TC-08-04, TC-11-03) |
| Hybrid index unavailable in a trial | Mode S | Documented no-op | VERIFIED (docs: trials have no hybrid tables) |
| No tool enforces "one owner per object" | Way 4 map | A reviewed ownership table, later a check | VERIFIED (design) |
| Parsing `$$` blocks in Flyway | Ways 2 and 4 | Two plain statements | TO-CONFIRM (TC-G08) |

## 9. What you learned

- The same change costs a few minutes by hand and more setup in the full design, and the extra setup buys preview, order, evidence and a safe test.
- Each kind of change has an owner: columns and procedures belong to DCM, renames, data and hybrid objects belong to Flyway.
- A rename is the clearest case: DCM alone loses data, Flyway alone has no preview, together they work in the right order.
- Every way works until the day it does not. The labs showed the day.
- You can state the trade-offs of the NAIC design and argue both sides.

## 10. Cleanup

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('CAP4');
EXECUTE DCM PROJECT SBX_INT.CAP_DCM.CAP_DCM_PROJECT PURGE;       -- non-production only: drops everything the project manages
DROP DCM PROJECT IF EXISTS SBX_INT.CAP_DCM.CAP_DCM_PROJECT;
DROP SCHEMA IF EXISTS SBX_INT.CAP_HAND;
DROP SCHEMA IF EXISTS SBX_INT.CAP_FLYWAY;
DROP SCHEMA IF EXISTS SBX_INT.CAP_DCM;
```

> **VERIFIED:** `EXECUTE DCM PROJECT <p> PURGE` drops everything managed (non-production only), and `DROP DCM PROJECT` removes the project and its deployment history (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage, read 2026-11-16). Skip the two DCM lines if you used Mode D-off.

Also: close any capstone pull request without merging; delete local branches `feature/CAP4` and `feature/LAB6B`, `feature/LAB8A` if you like (you run git). Remove the folders `flyway/capstone_flyway/` and `dcm/capstone/` if you do not want them in the repository, and empty `flyway/PRE_DCM_TARGET`.

Drop the schemas last, because they were created by the bootstrap. To redo the capstone, create them again (see section 3).

Keep: all pack files, the workflows, `SBX_INT`, `SBX_QA`, `SBX_PROD` and the roles.

When you are done learning for a while, suspend cost: `ALTER WAREHOUSE SBX_CALC_WH SUSPEND;` as `SBX_CICD_PIPELINE` or `ACCOUNTADMIN`, and check the trial balance in Snowsight. The trial lasts 30 days or until the free credits (about $400 of free usage) are used, whichever comes first.

## 11. State left behind

Schemas `CAP_HAND`, `CAP_FLYWAY`, `CAP_DCM` and clone `SBX_DEV_CAP4` created and dropped; `flyway/PRE_DCM_TARGET` empty; the quiz answered; the teach-back done; the notes table filled.

## 12. Open items for file 13

TC-G03, TC-G06, TC-G08, TC-08-04, TC-08-06, TC-11-01, TC-11-02, TC-11-03.

New names introduced by this file: folders `flyway/capstone_flyway/`, `dcm/capstone/`; DCM project `SBX_INT.CAP_DCM.CAP_DCM_PROJECT`; Flyway history table `FLYWAY_HISTORY_CAP_FLYWAY`; migrations `V6__rename_formulas_to_calc_formulas.sql`, `V7__backfill_calc_description.sql`, `V8__add_audit_actor_index.sql` (hybrid and standard) in the main project for way 4.

## 13. Next

Use [the glossary and reference](12-glossary-and-reference.md) any time. When you finish, fill in [the open questions and verification log](13-open-questions-and-verification-log.md).
