# Lab 5: zero-copy clones as ticket environments (file 07)

> **Lab no:** 5 | **File no:** 07 | **Time estimate:** 2.5 h | **Prerequisites:** file 06 (Lab 4 state: Flyway history at version 4, hybrid pair or stand-ins, DCM re-plan empty) | **Modes supported:** H or S, D (a Mode D-off path replaces the DCM deploy into the clone) | **Leaves behind:** procedures `CREATE_CALC_DEV_CLONE` and `DROP_CALC_DEV_CLONE` deployed and granted, `scripts/refresh_dev_clone.sh`, `scripts/assert_zero_int_writes.sql`, `scripts/smoke_clone.sql`, clone `SBX_DEV_LAB5A` dropped, the "INT baseline" row counts written down

## 1. Goal

By the end of this lab you can:

1. Create a database clone, explain why it costs almost nothing, and say what `DATA_RETENTION_TIME_IN_DAYS = 0` removes.
2. Show what a schema clone does with hybrid tables, and how `IGNORE HYBRID TABLES` and an empty re-creation fix it.
3. Explain and demonstrate the Flyway history trap in a clone.
4. Write a guarded pair of stored procedures that create and drop a ticket clone from a validated ticket id only.
5. Deploy a branch's DCM definitions into the clone so the clone writes only to itself, and prove that nothing was written to INT.
6. Clean up completely, and know what a clone leaves behind if you forget.

## 2. Why this matters in calc-engine and calc-admin

> **Real world.** In NAIC INT every developer used to share one set of engine tables, and the old way to test a change was to paste it into INT. The `feature/dcm-poc` branch replaced that with one clone per ticket: `CALC_DEV_<ticket>`. The procedures `COMMON_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE` and `DROP_CALC_DEV_CLONE` are `EXECUTE AS OWNER` (owner `CICD_PIPELINE`), accept only a ticket id of 1 to 40 letters or digits, and build the database name themselves. The script `scripts/refresh_dev_clone.sh <ticket>` drops, recreates, creates a DCM project in the clone, plans, deploys and re-plans. The proof of isolation was a real calculation group run in a clone: 267,427 result rows and 93 `EXECUTIONS` rows in the clone, 0 in INT, and 27 DML statements in query history, all against the clone.
>
> Two NAIC facts drive this lab. First, `CREATE SCHEMA ... CLONE` fails when the schema holds hybrid tables (error 392105 in the POC), so the procedure clones with `IGNORE HYBRID TABLES` and recreates `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT` empty from `GET_DDL`. Second, a cloned Flyway history table "can falsely say the missing hybrid migration already ran", so the target design creates the lifecycle schema and its history fresh in each review clone.
>
> A working note dated 2026-11-11 recorded pausing DCM. The team has since confirmed that both DCM and Flyway are being implemented. Clones are where the two tools meet developers: Flyway prepares the hybrid objects, DCM deploys the branch's definitions into the clone. The architecture's roadmap calls the pieces "clone bootstrap" (phase 3) and "guarded review clone" (pipeline).
>
> **A rule for this lab:** you write the miniature procedures and script yourself from the descriptions here. Do not copy NAIC files into your sandbox repository, which is public.

## 3. Prerequisites and state check

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;
SELECT COUNT(*) AS RESULT_ROWS FROM SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE;
SELECT COUNT(*) AS EXEC_ROWS   FROM SBX_INT.CALC_SUPPORT.EXECUTIONS;
SELECT TABLE_NAME, TABLE_OWNER FROM SBX_INT.INFORMATION_SCHEMA.TABLES
 WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_OWNER <> 'SBX_CICD_PIPELINE';
SHOW SCHEMAS LIKE 'CALC_DEV_SUPPORT' IN DATABASE SBX_INT;
```

You should see: two counts (probably `0` and `0`; write them down as your **INT baseline**), an empty result for the owner check, and the schema `CALC_DEV_SUPPORT`.

Why the owner check: a clone keeps the source objects' owners. The ownership hand-off in this lab only works if every source object is owned by `SBX_CICD_PIPELINE`. NAIC met this exactly: objects made by `SYSADMIN` or by another role broke clones until everything was moved to `CICD_PIPELINE`. If the query returns rows, change the owner (`GRANT OWNERSHIP ON TABLE ... TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS`) or recreate the object.

Run in: **Terminal**: `snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH` followed by `python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json`. You should see an empty changeset. If not, finish Lab 4 first.

## 4. Concepts

**Zero-copy clone.** `CREATE ... CLONE` makes a new object that shares the source's storage. Only metadata is copied at first, so it is fast and costs almost nothing. Storage is added only when the clone or the source changes data afterwards. A clone is independent: writes to it never touch the source.

**Time Travel and retention.** Snowflake keeps old versions of changed data for a number of days (`DATA_RETENTION_TIME_IN_DAYS`), so you can query the past or `UNDROP`. Setting it to 0 turns Time Travel off for that object. For a throwaway clone this saves storage, and it means a dropped clone cannot be restored. NAIC's clones use 0, and the refresh script warns that results, drafts and manual metadata edits in the old clone are lost.

**Hybrid tables and clones.** A single hybrid table cannot be cloned. A schema that contains hybrid tables cannot be cloned unless you add `IGNORE HYBRID TABLES`, which skips them. A database-level clone is documented as supported and copies hybrid data physically, which costs compute in proportion to the data. NAIC chose to skip them and recreate them empty, because a developer clone should not carry live workflow rows.

**Ticket environment.** A short-lived database named from a ticket id (here `SBX_DEV_<TICKET>`) in which a developer or a pipeline job can run a change end to end without touching INT.

**`EXECUTE AS OWNER`.** A stored procedure normally runs with the caller's rights (caller's rights). With `EXECUTE AS OWNER` it runs with the rights of the role that owns the procedure. That lets a developer role, which cannot create databases, ask a controlled procedure to do it. The procedure is the gate. The security rule: the only input from the caller is a ticket id that the procedure validates against a fixed pattern, and the procedure builds the database name itself. A caller-supplied database name would let someone drop `SBX_INT`.

**Redirects.** The engine stores SQL text that names INT objects. In a clone those names must point to the clone. The view `ENV_CONFIG` holds a map of "from" and "to" prefixes and the function `FN_APPLY_REDIRECTS` rewrites text with it just before it is executed (you defined both in Lab 3).

```
 SBX_INT (source)                              SBX_DEV_<TICKET> (clone, retention 0)
  CALC_SUPPORT  --- clone, IGNORE HYBRID --->   CALC_SUPPORT   (hybrid tables recreated empty)
  (SBX_INT_FIN) FIN_CALCULATION --- clone ---> FIN_CALCULATION
                                                owners handed to SBX_CALC_DEV
                                                own DCM project object (CLONE target)
                                                ENV_CONFIG redirects SBX_INT.* to the clone
```

> **VERIFIED:** hybrid tables can be cloned only at database level; `CREATE HYBRID TABLE ... CLONE` fails with error 391411; a schema clone of a schema with hybrid tables fails unless `IGNORE HYBRID TABLES` is appended; the clause also works on database clones; point-in-time clones support TIMESTAMP and OFFSET but not STATEMENT (https://docs.snowflake.com/en/user-guide/tables-hybrid-clone, read 2026-11-16).

> **VERIFIED:** hybrid tables are not supported in trial accounts, and UNDROP cannot restore hybrid tables (https://docs.snowflake.com/en/user-guide/tables-hybrid-limitations, read 2026-11-16).

## 5. Step-by-step

### Part A: clone by hand and meet every trap (about 50 min)

You first do by hand what the procedures will automate, so that each line of the procedures has a reason you have seen.

### Step 5.1 Make an empty clone target and try a plain schema clone

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`; this role may create databases, see the bootstrap):

```sql
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;
CREATE DATABASE SBX_DEV_LAB5M DATA_RETENTION_TIME_IN_DAYS = 0 COMMENT = 'Lab 5 manual clone. Safe to drop.';
CREATE SCHEMA SBX_DEV_LAB5M.CALC_SUPPORT CLONE SBX_INT.CALC_SUPPORT;
```

You should see: in Mode H an error on the last statement because the source schema contains hybrid tables (the NAIC POC saw error 392105; the docs say the clone fails unless `IGNORE HYBRID TABLES` is used). In Mode S the clone simply succeeds, because the stand-ins are standard tables.

> **TO-CONFIRM [TC-07-01]:** the exact error number and text of a schema clone that contains hybrid tables. The docs name error 391411 for a single hybrid-table clone; NAIC recorded 392105 for the schema clone. How to check at lab time: read your own error and paste it into file 13.

#### If you are in Mode S (no hybrid tables)

This experiment is read-only for you. The docs say a schema clone fails when it contains hybrid tables unless `IGNORE HYBRID TABLES` is added. You cannot reproduce that in a trial. Read the transcript above and continue; Step 5.2 tests whether the clause is accepted when there are no hybrid tables.

### Step 5.2 Clone with `IGNORE HYBRID TABLES` and inspect the result

Run in: **Snowsight worksheet**:

```sql
CREATE OR REPLACE SCHEMA SBX_DEV_LAB5M.CALC_SUPPORT CLONE SBX_INT.CALC_SUPPORT IGNORE HYBRID TABLES DATA_RETENTION_TIME_IN_DAYS = 0;
SHOW TABLES IN SCHEMA SBX_DEV_LAB5M.CALC_SUPPORT;
SHOW SEQUENCES LIKE 'SEQ_CALC_ID' IN SCHEMA SBX_DEV_LAB5M.CALC_SUPPORT;
SHOW SEQUENCES LIKE 'SEQ_CALC_ID' IN SCHEMA SBX_INT.CALC_SUPPORT;
```

You should see: in Mode H, success, and a table list without `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT` (they were skipped). The Flyway history table, `CALCULATIONS`, `CALC_CATEGORIES` and the other standard tables are present, owned by `SBX_CICD_PIPELINE` (a clone keeps the source owners). The two `SHOW SEQUENCES` rows have the same next value: the clone's sequences continue from INT's current values.

> **VERIFIED:** `CREATE OR REPLACE SCHEMA clone_ht_schema CLONE ht_schema IGNORE HYBRID TABLES;` is the documented form (https://docs.snowflake.com/en/user-guide/tables-hybrid-clone, read 2026-11-16). The NAIC POC adds that the clause "must come right after the CLONE source", and the example above follows that order.

> **TO-CONFIRM [TC-G29]:** in Mode S, whether `IGNORE HYBRID TABLES` is accepted when the schema has no hybrid tables. How to check at lab time: the statement above. If it is rejected, the procedure in Step 5.7 needs a Mode S variant without the clause. Record the result in file 13.

Now prove independence. Run in: **Snowsight worksheet**:

```sql
INSERT INTO SBX_DEV_LAB5M.CALC_SUPPORT.CALCULATIONS (CALC_NAME, CALC_STATUS) VALUES ('CLONE_ONLY_ROW', 'DRAFT');
SELECT COUNT(*) FROM SBX_DEV_LAB5M.CALC_SUPPORT.CALCULATIONS WHERE CALC_NAME = 'CLONE_ONLY_ROW';
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALCULATIONS WHERE CALC_NAME = 'CLONE_ONLY_ROW';
SHOW DCM PROJECTS IN SCHEMA SBX_DEV_LAB5M.CALC_SUPPORT;  -- TO-CONFIRM TC-07-02
```

You should see: `1` in the clone and `0` in INT. The last statement probably lists a cloned `CALC_ENGINE_DCM` project object. That is why the procedure drops it: the INT project's memory must not survive in a clone, or a plan there would be measured against INT's snapshot.

> **TO-CONFIRM [TC-07-02]:** whether a schema clone copies the DCM project object, and the exact `SHOW DCM PROJECTS ... IN SCHEMA` form. The NAIC procedure contains `DROP DCM PROJECT IF EXISTS` for the clone, which implies the object is copied. How to check at lab time: run the statement; if the syntax is rejected, use `SHOW DCM PROJECTS;` and look for the clone's database. Record the result in file 13.

### Step 5.3 The Flyway history trap

Run in: **Terminal** (`flyway/` folder, environment loaded as in Lab 4). Point Flyway at the clone by changing only the database in the URL:

```
cd flyway
set -a; source .env.flyway; set +a
export FLYWAY_URL="${FLYWAY_URL/db=SBX_INT/db=SBX_DEV_LAB5M}"
export FLYWAY_PLACEHOLDERS_CALC_DB=SBX_DEV_LAB5M
flyway info
flyway migrate
cd ..
```

You should see: `info` lists V1 to V4 as `Success`, and `migrate` says the schema is up to date. Flyway believes the hybrid tables exist because the cloned history table says V1 and V2 ran. They do not exist in the clone.

#### If you are in Mode S (no hybrid tables)

Simulate what `IGNORE HYBRID TABLES` does by dropping one stand-in from the clone first. Run in: **Snowsight worksheet**: `DROP TABLE SBX_DEV_LAB5M.CALC_SUPPORT.CALC_AUDIT_EVENT;` and then run the Terminal commands above. The trap is the same: history says it ran, the object is missing.

Run in: **Snowsight worksheet**: now show the consequence.

```sql
INSERT INTO SBX_DEV_LAB5M.CALC_SUPPORT.CALC_AUDIT_EVENT (CALC_ID, EVENT_STATUS) VALUES (2001, 'PENDING');
```

You should see: an error that the table does not exist. A developer would meet this as a failing feature in a clone while `flyway info` is green. NAIC's architecture states it as "A cloned history table can falsely say the missing hybrid migration already ran."

Your options for the fix:

| Option | How | Cost | Chosen here |
|---|---|---|---|
| Recreate the missing tables from the source DDL, keep the cloned history | `GET_DDL`, replace the INT prefix, run | History and objects stay consistent; the DDL comes from INT's live objects, not from Git | **Yes** (matches the NAIC clone procedure) |
| Drop the cloned history and run `flyway migrate` fresh in the clone | history table removed, all migrations replay | Tests the migrations themselves (the NAIC target design), but replays every data change | Described only |
| `flyway repair` or `baseline` | not applicable | `repair` only fixes checksums and failed rows; it cannot recreate objects | No |

Do the first fix by hand (one `GET_DDL` call per table; a computed name fails with error 1046):

```sql
SELECT GET_DDL('TABLE', 'SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT', TRUE);
```

Copy the result, replace `SBX_INT.` with `SBX_DEV_LAB5M.` everywhere (this also repoints the sequence default to the clone's sequence), and run it. Repeat for `CALC_WORKFLOW_STATUS` if it is missing. Then repeat the failing insert, which now succeeds, and delete the row.

You should see: the insert succeeds. Check the default: `SELECT GET_DDL('TABLE', 'SBX_DEV_LAB5M.CALC_SUPPORT.CALC_AUDIT_EVENT', TRUE);` shows `SBX_DEV_LAB5M....SEQ_AUDIT_EVENT_ID.NEXTVAL`, not the INT sequence.

> **TO-CONFIRM [TC-G15]:** how a database-level clone behaves when hybrid tables exist and retention is 0. See Step 5.4.

### Step 5.4 Optional: a database-level clone with hybrid tables (Mode H only)

Run in: **Snowsight worksheet**:

```sql
CREATE DATABASE SBX_DEV_LAB5FULL CLONE SBX_INT DATA_RETENTION_TIME_IN_DAYS = 0;
SELECT TABLE_SCHEMA, TABLE_NAME, IS_HYBRID FROM SBX_DEV_LAB5FULL.INFORMATION_SCHEMA.TABLES WHERE IS_HYBRID = 'YES';
DROP DATABASE SBX_DEV_LAB5FULL;
```

You should see: the docs say a database clone copies hybrid data physically, so the hybrid tables should appear in the clone. Write down whether they did, and whether retention 0 caused any error. Drop it at once; it also copied every other schema in `SBX_INT`.

Why NAIC still skips hybrid tables: a physical copy costs compute in proportion to the data and would carry live workflow rows into a developer clone.

> **TO-CONFIRM [TC-G15]:** the observed result. Record it in file 13.

### Step 5.5 Retention 0 means no way back

Run in: **Snowsight worksheet**:

```sql
DROP DATABASE SBX_DEV_LAB5M;
UNDROP DATABASE SBX_DEV_LAB5M;
```

You should see: the second statement fails (or finds nothing to restore), because retention was 0. If you want to see the contrast, create a throwaway database with the default retention, drop it, and undrop it. Write down which behaviour you saw and the exact error. In a real clone refresh this is why the script says "results, drafts and manual metadata edits in the old clone are lost".

> **TO-CONFIRM [TC-07-03]:** the exact UNDROP error text with retention 0. Record it in file 13. Hybrid tables cannot be undropped at all, so retention does not matter for them.

### Part B: the guarded procedures (about 50 min)

### Step 5.6 Why a developer cannot just create a clone, and what the procedure is for

Run in: **Snowsight worksheet** as the developer role:

```sql
USE ROLE SBX_CALC_DEV;
CREATE DATABASE SBX_DEV_NOPE;
```

You should see: an insufficient-privileges error. The developer role has no `CREATE DATABASE` on the account (only the pipeline role has it, as in NAIC where only the proc owner can create databases). The procedure is the controlled door: the developer gets `USAGE` on the procedure, not the privilege behind it.

> **TO-CONFIRM [TC-07-04]:** the exact error text. Record it in file 13.

### Step 5.7 Write and deploy the clone procedures

Run in: **Terminal**. Create `sql/clone_support/calc_dev_clone_procs.sql`. The procedures follow the structure of the NAIC procedures: validate the ticket, refuse an existing database, create it with retention 0, clone both schemas, drop the cloned DCM project, recreate hybrid tables empty, hand ownership to the developer role, return a status object, and on any error drop the half-built database only if this call created it.

```sql
-- Deploy with: snow sql -c sbx_owner --role SBX_CICD_PIPELINE -f sql/clone_support/calc_dev_clone_procs.sql
-- Clone-support objects are not owned by DCM or Flyway in the sandbox (see file 06, decision table).
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;

CREATE OR REPLACE PROCEDURE SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE(P_TICKET VARCHAR)
RETURNS VARIANT
LANGUAGE SQL
EXECUTE AS OWNER
AS
$$
DECLARE
    e_bad_ticket EXCEPTION (-20001, 'Ticket must be 1-40 letters or digits');
    e_exists     EXCEPTION (-20002, 'Clone database already exists; drop it first or use a different ticket');
    v_db      VARCHAR;
    v_info    VARCHAR;
    v_q       VARCHAR;
    v_sql     VARCHAR;
    v_ddl     VARCHAR;
    v_n       NUMBER;
    v_created BOOLEAN DEFAULT FALSE;
    v_steps   ARRAY DEFAULT ARRAY_CONSTRUCT();
    v_res     RESULTSET;
BEGIN
    -- 1. validate: the only caller input is a ticket id with a fixed shape
    IF (:P_TICKET IS NULL OR NOT REGEXP_LIKE(:P_TICKET, '^[A-Za-z0-9]{1,40}$')) THEN
        RAISE e_bad_ticket;
    END IF;
    v_db := 'SBX_DEV_' || UPPER(:P_TICKET);

    -- 2. refuse to touch an existing database (never drop what we did not create)
    EXECUTE IMMEDIATE 'SHOW DATABASES LIKE ''' || :v_db || '''';
    SELECT COUNT(*) INTO :v_n FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) WHERE "name" = :v_db;
    IF (:v_n > 0) THEN
        RAISE e_exists;
    END IF;

    -- 3. create the database with retention 0, then clone both schemas
    EXECUTE IMMEDIATE 'CREATE DATABASE ' || :v_db || ' DATA_RETENTION_TIME_IN_DAYS = 0 COMMENT = ''Ticket clone of SBX_INT. Safe to drop.''';
    v_created := TRUE;
    v_steps := ARRAY_APPEND(:v_steps, 'database created');
    EXECUTE IMMEDIATE 'DROP SCHEMA IF EXISTS ' || :v_db || '.PUBLIC';
    EXECUTE IMMEDIATE 'CREATE SCHEMA ' || :v_db || '.CALC_SUPPORT CLONE SBX_INT.CALC_SUPPORT IGNORE HYBRID TABLES DATA_RETENTION_TIME_IN_DAYS = 0';
    EXECUTE IMMEDIATE 'CREATE SCHEMA ' || :v_db || '.FIN_CALCULATION CLONE SBX_INT_FIN.FIN_CALCULATION DATA_RETENTION_TIME_IN_DAYS = 0';
    v_steps := ARRAY_APPEND(:v_steps, 'schemas cloned');

    -- 4. the cloned INT project must not live on in the clone
    EXECUTE IMMEDIATE 'DROP DCM PROJECT IF EXISTS ' || :v_db || '.CALC_SUPPORT.CALC_ENGINE_DCM';

    -- 5. recreate hybrid tables empty (GET_DDL needs a constant name, so one call per table)
    SELECT COUNT(*) INTO :v_n FROM SBX_INT.INFORMATION_SCHEMA.TABLES
     WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_NAME = 'CALC_AUDIT_EVENT' AND IS_HYBRID = 'YES';
    IF (:v_n > 0) THEN
        SELECT GET_DDL('TABLE', 'SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT', TRUE) INTO :v_ddl;
        v_ddl := REPLACE(:v_ddl, 'SBX_INT.', :v_db || '.');
        EXECUTE IMMEDIATE :v_ddl;
        v_steps := ARRAY_APPEND(:v_steps, 'CALC_AUDIT_EVENT recreated empty');
    END IF;
    SELECT COUNT(*) INTO :v_n FROM SBX_INT.INFORMATION_SCHEMA.TABLES
     WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_NAME = 'CALC_WORKFLOW_STATUS' AND IS_HYBRID = 'YES';
    IF (:v_n > 0) THEN
        SELECT GET_DDL('TABLE', 'SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS', TRUE) INTO :v_ddl;
        v_ddl := REPLACE(:v_ddl, 'SBX_INT.', :v_db || '.');
        EXECUTE IMMEDIATE :v_ddl;
        v_steps := ARRAY_APPEND(:v_steps, 'CALC_WORKFLOW_STATUS recreated empty');
    END IF;

    -- 6. hand ownership of every object to the developer role, then the schemas, then database usage
    v_info := :v_db || '.INFORMATION_SCHEMA.';
    v_q := 'SELECT ''TABLE'' AS K, TABLE_SCHEMA AS S, TABLE_NAME AS N, '''' AS A FROM ' || :v_info || 'TABLES'
        || ' WHERE TABLE_SCHEMA <> ''INFORMATION_SCHEMA'' AND TABLE_TYPE = ''BASE TABLE'''
        || ' UNION ALL SELECT ''VIEW'', TABLE_SCHEMA, TABLE_NAME, '''' FROM ' || :v_info || 'VIEWS'
        || ' WHERE TABLE_SCHEMA <> ''INFORMATION_SCHEMA'''
        || ' UNION ALL SELECT ''SEQUENCE'', SEQUENCE_SCHEMA, SEQUENCE_NAME, '''' FROM ' || :v_info || 'SEQUENCES'
        || ' WHERE SEQUENCE_SCHEMA <> ''INFORMATION_SCHEMA'''
        || ' UNION ALL SELECT ''FUNCTION'', FUNCTION_SCHEMA, FUNCTION_NAME, REGEXP_REPLACE(ARGUMENT_SIGNATURE, ''[A-Za-z0-9_]+ '', '''') FROM ' || :v_info || 'FUNCTIONS'
        || ' WHERE FUNCTION_SCHEMA <> ''INFORMATION_SCHEMA'''
        || ' UNION ALL SELECT ''PROCEDURE'', PROCEDURE_SCHEMA, PROCEDURE_NAME, REGEXP_REPLACE(ARGUMENT_SIGNATURE, ''[A-Za-z0-9_]+ '', '''') FROM ' || :v_info || 'PROCEDURES'
        || ' WHERE PROCEDURE_SCHEMA <> ''INFORMATION_SCHEMA''';
    v_res := (EXECUTE IMMEDIATE :v_q);
    FOR rec IN v_res DO
        v_sql := 'GRANT OWNERSHIP ON ' || rec.K || ' ' || :v_db || '.' || rec.S || '.' || rec.N || rec.A
              || ' TO ROLE SBX_CALC_DEV REVOKE CURRENT GRANTS';
        EXECUTE IMMEDIATE :v_sql;
    END FOR;
    EXECUTE IMMEDIATE 'GRANT OWNERSHIP ON SCHEMA ' || :v_db || '.CALC_SUPPORT TO ROLE SBX_CALC_DEV REVOKE CURRENT GRANTS';
    EXECUTE IMMEDIATE 'GRANT OWNERSHIP ON SCHEMA ' || :v_db || '.FIN_CALCULATION TO ROLE SBX_CALC_DEV REVOKE CURRENT GRANTS';
    EXECUTE IMMEDIATE 'GRANT USAGE ON DATABASE ' || :v_db || ' TO ROLE SBX_CALC_DEV';
    v_steps := ARRAY_APPEND(:v_steps, 'ownership handed to SBX_CALC_DEV');

    RETURN OBJECT_CONSTRUCT('status', 'SUCCESS', 'database', :v_db, 'steps', :v_steps,
                            'next', 'snow dcm create + plan + deploy with the CLONE target');
EXCEPTION
    WHEN OTHER THEN
        IF (:v_created) THEN
            EXECUTE IMMEDIATE 'DROP DATABASE IF EXISTS ' || :v_db;
        END IF;
        RETURN OBJECT_CONSTRUCT('status', 'ERROR', 'sqlcode', SQLCODE, 'sqlerrm', SQLERRM, 'cleaned_up', :v_created);
END;
$$;
```

Read it with these questions. Where is the caller's text used? Only in the regex test and in `UPPER(:P_TICKET)` after it passed. Why does the existence check come before `CREATE DATABASE`? So that the error path can never drop a database somebody else made. Why one `GET_DDL` call per table? A computed name fails with error 1046. Why `REVOKE CURRENT GRANTS` on the ownership transfer? So the clone's objects carry only the developer role's grants, not the pipeline's. Why does the pipeline role need `SBX_CALC_DEV` granted to it (bootstrap hierarchy)? Since the 2026_07 bundle (BCR-2395) ownership can only be transferred to a role that is active in the session or by a role with `MANAGE GRANTS`; inherited roles count as available inside an owner's-rights procedure, and `MANAGE GRANTS` was judged too broad.

> **TO-CONFIRM [TC-07-05]:** two syntax points in this procedure that were written from the Snowflake Scripting language without a public page being checked: (a) `FOR rec IN v_res DO ... END FOR;` over a `RESULTSET` variable. Fallback: `LET c CURSOR FOR v_res;` followed by `FOR rec IN c DO`. (b) the `REGEXP_REPLACE` that turns `(P_X VARCHAR, P_Y NUMBER)` into `(VARCHAR, NUMBER)` for `GRANT OWNERSHIP ON PROCEDURE`. How to check at lab time: if the hand-off step fails, read `sqlerrm` in the returned object; if the types come out wrong, query `INFORMATION_SCHEMA.PROCEDURES` and look at `ARGUMENT_SIGNATURE`. The NAIC procedure takes the same job from `SHOW PROCEDURES` output instead; its tested form is in the POC branch if you ever need it for reference. Record what you changed in file 13.

> **TO-CONFIRM [TC-G29]:** `IGNORE HYBRID TABLES` in step 3 must be accepted when the schema has no hybrid tables (Mode S). If Step 5.2 showed it is rejected, remove the clause in the Mode S copy of this file.

Append the drop procedure and the grants to the same file. The database prefix is fixed in the procedure, so only `SBX_DEV_` databases can ever be dropped. The ticket pattern forbids underscores, so no ticket can ever name the placeholder `SBX_DEV_SET_ON_CLI`.

```sql
CREATE OR REPLACE PROCEDURE SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE(P_TICKET VARCHAR)
RETURNS VARIANT
LANGUAGE SQL
EXECUTE AS OWNER
AS
$$
DECLARE
    e_bad_ticket EXCEPTION (-20001, 'Ticket must be 1-40 letters or digits');
    v_db VARCHAR;
    v_n  NUMBER;
BEGIN
    IF (:P_TICKET IS NULL OR NOT REGEXP_LIKE(:P_TICKET, '^[A-Za-z0-9]{1,40}$')) THEN
        RAISE e_bad_ticket;
    END IF;
    v_db := 'SBX_DEV_' || UPPER(:P_TICKET);
    EXECUTE IMMEDIATE 'SHOW DATABASES LIKE ''' || :v_db || '''';
    SELECT COUNT(*) INTO :v_n FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) WHERE "name" = :v_db;
    IF (:v_n = 0) THEN
        RETURN OBJECT_CONSTRUCT('status', 'NOT_FOUND', 'database', :v_db);
    END IF;
    EXECUTE IMMEDIATE 'DROP DATABASE ' || :v_db;
    RETURN OBJECT_CONSTRUCT('status', 'DROPPED', 'database', :v_db);
EXCEPTION
    WHEN OTHER THEN
        RETURN OBJECT_CONSTRUCT('status', 'ERROR', 'sqlcode', SQLCODE, 'sqlerrm', SQLERRM);
END;
$$;

GRANT USAGE ON PROCEDURE SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE(VARCHAR) TO ROLE SBX_CALC_DEV;
GRANT USAGE ON PROCEDURE SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE(VARCHAR) TO ROLE SBX_CALC_DEV;
```

The grants sit in the file because `CREATE OR REPLACE` removes grants every time (the lesson from Lab 1). NAIC's file does the same.

Run in: **Terminal**:

```
snow sql -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH -f sql/clone_support/calc_dev_clone_procs.sql
```

You should see: a "successfully created" line for each procedure and two grant results. This is plain SQL deployment with `snow sql`, which is also how NAIC deployed procedures from a laptop before DCM.

### Step 5.8 Test the gate with harmless bad input

Run in: **Snowsight worksheet**, as the developer role:

```sql
USE ROLE SBX_CALC_DEV;
USE WAREHOUSE SBX_CALC_WH;
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('SBX_INT.X');
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('NOPE; SELECT 1');
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('');
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE(NULL);
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('NOSUCHTICKET');
```

Also call it with a 41-letter ticket (type 41 letters `A` yourself).

You should see: the four invalid inputs and the 41-letter one return `status ERROR` with `sqlcode -20001` and the message "Ticket must be 1-40 letters or digits", before any SQL was built, and `NOSUCHTICKET` returns `NOT_FOUND`. Never test with a real database name such as `SBX_INT`. The NAIC team recorded the same decision, and used these same kinds of payloads.

### Step 5.9 Create a ticket clone, read the result

Run in: **Snowsight worksheet**, as the developer role:

```sql
CALL SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE('LAB5A');
```

You should see: a `status` of `SUCCESS`, `database` `SBX_DEV_LAB5A` and the list of steps. If you see `ERROR`, read `sqlerrm`: it is the message of the statement that failed, and `cleaned_up` tells you the half-built database was removed. Fix the cause and call again.

Verify the result. Run in: **Snowsight worksheet**:

```sql
SHOW SCHEMAS IN DATABASE SBX_DEV_LAB5A;
SHOW TABLES IN SCHEMA SBX_DEV_LAB5A.CALC_SUPPORT;
SELECT TABLE_NAME, TABLE_OWNER, IS_HYBRID FROM SBX_DEV_LAB5A.INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'CALC_SUPPORT';
SELECT COUNT(*) FROM SBX_DEV_LAB5A.CALC_SUPPORT.CALC_AUDIT_EVENT;
SHOW PARAMETERS LIKE 'DATA_RETENTION_TIME_IN_DAYS' IN DATABASE SBX_DEV_LAB5A;
```

You should see: schemas `CALC_SUPPORT` and `FIN_CALCULATION` (no `PUBLIC`), every table owned by `SBX_CALC_DEV`, the hybrid pair back and empty (Mode H) or the stand-ins (Mode S), and retention `0`.

> **TO-CONFIRM [TC-G15]:** record whether the recreated hybrid tables behave exactly like INT's: `SHOW INDEXES IN TABLE SBX_DEV_LAB5A.CALC_SUPPORT.CALC_AUDIT_EVENT;` should list the secondary index.

Run in: **Snowsight worksheet**: confirm that a second call with the same ticket is refused and does not damage the first clone.

```sql
CALL SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE('lab5a');
SELECT COUNT(*) FROM SBX_DEV_LAB5A.CALC_SUPPORT.CALCULATIONS;
```

You should see: `ERROR` with `sqlcode -20002` (the ticket is upper-cased, so it names the same database), `cleaned_up` false, and the clone still queryable.

### Part C: put a branch's definitions into the clone and prove isolation (about 50 min)

### Step 5.10 Deploy the DCM definitions into the clone

The clone has no DCM project (the procedure dropped the cloned one). Create one inside the clone, plan with the CLONE target, and deploy. The CLONE target's project owner is `SBX_CALC_DEV`, which now owns every object in the clone, so adoption works.

Run in: **Terminal** (repository root):

```
snow dcm create SBX_DEV_LAB5A.CALC_SUPPORT.CALC_ENGINE_DCM --target CLONE --from dcm/calc_engine -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH
snow dcm plan SBX_DEV_LAB5A.CALC_SUPPORT.CALC_ENGINE_DCM --target CLONE -D "calc_db='SBX_DEV_LAB5A'" -D "fin_db='SBX_DEV_LAB5A'" --from dcm/calc_engine --save-output -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

Why the explicit project name and the two `-D` values: the manifest is not rendered, so its CLONE `project_name` is the placeholder `SBX_DEV_SET_ON_CLI....`. That database does not exist, so forgetting the project name fails instead of touching INT. In the same way, forgetting `-D` leaves `calc_db` at `SBX_DEV_SET_ON_CLI` and the plan fails. The quotes inside the `-D` value are required (error 1510 in NAIC without them).

You should see: no DROP. The entries are mostly adoption, plus real changes: the view `ENV_CONFIG` (the clone variant has redirects), and the GRANT entries from `access.sql` (the ownership hand-off used `REVOKE CURRENT GRANTS`, so the app-role grants have to be declared again). In the NAIC POC the first clone plan was 70 entities, 0 create, 70 alter, 0 drop, with 69 adoption-only and exactly one real change (`ENV_CONFIG`).

> **TO-CONFIRM [TC-07-06]:** the exact count and kinds of entries in your first clone plan. Record them in file 13. Any DROP, or any change to a table, is a stop sign: read it before deploying.

Deploy and re-plan:

```
snow dcm deploy SBX_DEV_LAB5A.CALC_SUPPORT.CALC_ENGINE_DCM --target CLONE -D "calc_db='SBX_DEV_LAB5A'" -D "fin_db='SBX_DEV_LAB5A'" --from dcm/calc_engine --alias lab5a-initial -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH
snow dcm plan SBX_DEV_LAB5A.CALC_SUPPORT.CALC_ENGINE_DCM --target CLONE -D "calc_db='SBX_DEV_LAB5A'" -D "fin_db='SBX_DEV_LAB5A'" --from dcm/calc_engine --save-output -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: `EMPTY changeset`. Check that the INT project was not touched: `snow dcm plan --target INT ...` as in Lab 4 is still empty.

#### If DCM is not available (Mode D-off)

Replace the project work with one worksheet statement as `SBX_CALC_DEV`: `CREATE OR REPLACE VIEW SBX_DEV_LAB5A.CALC_SUPPORT.ENV_CONFIG (CONFIG_KEY, CONFIG_VALUE) AS ...` using the clone variant of the three rows (set `HOME_SCHEMA` and `RESULT_SCHEMA` to the clone, and `SCHEMA_REDIRECTS` to the two-entry map). It is the only real difference. Re-create the app-role grants by hand.

### Step 5.11 See the redirects

Run in: **Snowsight worksheet**:

```sql
SELECT CONFIG_KEY, CONFIG_VALUE FROM SBX_INT.CALC_SUPPORT.ENV_CONFIG;
SELECT CONFIG_KEY, CONFIG_VALUE FROM SBX_DEV_LAB5A.CALC_SUPPORT.ENV_CONFIG;
SELECT SBX_DEV_LAB5A.CALC_SUPPORT.FN_APPLY_REDIRECTS(
  'INSERT INTO SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE SELECT 1',
  (SELECT CONFIG_VALUE::ARRAY FROM SBX_DEV_LAB5A.CALC_SUPPORT.ENV_CONFIG WHERE CONFIG_KEY = 'SCHEMA_REDIRECTS'));
```

You should see: in INT the `SCHEMA_REDIRECTS` value is `[]`; in the clone it has two `from`/`to` entries; and the function turns `SBX_INT_FIN.FIN_CALCULATION.` into `SBX_DEV_LAB5A.FIN_CALCULATION.` inside the text.

This is the mechanism that makes a clone write to itself. The engine's generated SQL names INT objects, and the redirect rewrites them at run time. The CLONE target derives the map from `calc_db` through `clone_redirects: true`, so nobody can forget to pass it; NAIC rejected a design where the JSON map was passed on the command line because forgetting it silently left redirects off.

### Step 5.12 Prove zero writes to the source

Run in: **Terminal**. Create `scripts/assert_zero_int_writes.sql`. It compares the INT row counts you recorded as the baseline with the current counts and raises an error when they differ:

```sql
-- Usage: snow sql -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH -f scripts/assert_zero_int_writes.sql -D before_results=<n> -D before_exec=<n>
EXECUTE IMMEDIATE $$
DECLARE
    e_leak EXCEPTION (-20020, 'INT was written to during the clone test');
    v_results NUMBER;
    v_exec    NUMBER;
BEGIN
    SELECT COUNT(*) INTO :v_results FROM SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE;
    SELECT COUNT(*) INTO :v_exec FROM SBX_INT.CALC_SUPPORT.EXECUTIONS;
    IF (:v_results <> <% before_results %> OR :v_exec <> <% before_exec %>) THEN
        RAISE e_leak;
    END IF;
    RETURN 'PASS: INT row counts unchanged';
END;
$$;
```

Create `scripts/smoke_clone.sql`. It refuses to run against anything that is not an `SBX_DEV_` database, registers a draft calculation in the clone and runs the miniature compute procedure there. The name is deliberately not `SP_RUN_...`:

```sql
-- Usage: snow sql -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH -f scripts/smoke_clone.sql -D clone_db=SBX_DEV_<TICKET>
EXECUTE IMMEDIATE $$
DECLARE
    e_wrong_target EXCEPTION (-20030, 'Smoke tests may only run in an SBX_DEV_ clone');
    v_id VARCHAR;
BEGIN
    IF (NOT STARTSWITH('<% clone_db %>', 'SBX_DEV_')) THEN
        RAISE e_wrong_target;
    END IF;
    CALL <% clone_db %>.CALC_SUPPORT.SP_REGISTER_CALCULATION('SMOKE_CALC', CURRENT_USER()) INTO :v_id;
    CALL <% clone_db %>.CALC_SUPPORT.SP_COMPUTE_MINI_RESULT(:v_id::NUMBER, NULL);
    RETURN 'smoke ok for calc ' || :v_id;
END;
$$;
```

> **TO-CONFIRM [TC-07-07]:** that `snow sql` accepts `-D name=value` with the `<% name %>` template form inside the file. How to check at lab time: `snow sql --help` and one run. Fallback: edit the numbers and the clone name into a temporary copy of the file by hand. Record the result in file 13.

Run the two scripts. Run in: **Terminal**:

```
snow sql -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH -f scripts/smoke_clone.sql -D clone_db=SBX_DEV_LAB5A
snow sql -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH -f scripts/assert_zero_int_writes.sql -D before_results=<RESULT_ROWS> -D before_exec=<EXEC_ROWS>
```

Replace the two placeholders with your INT baseline from Section 3.

You should see: `smoke ok for calc <id>` and then `PASS: INT row counts unchanged`.

Run in: **Snowsight worksheet**: check the clone side, and the history.

```sql
SELECT COUNT(*) FROM SBX_DEV_LAB5A.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE;
SELECT COUNT(*) FROM SBX_DEV_LAB5A.CALC_SUPPORT.EXECUTIONS;
SELECT QUERY_TYPE, LEFT(QUERY_TEXT, 120) AS QUERY_START
  FROM TABLE(SBX_INT.INFORMATION_SCHEMA.QUERY_HISTORY(RESULT_LIMIT => 200))
 WHERE QUERY_TYPE IN ('INSERT', 'UPDATE', 'DELETE', 'MERGE')
   AND START_TIME > DATEADD(minute, -30, CURRENT_TIMESTAMP())
 ORDER BY START_TIME;
```

You should see: `1` result row and `1` execution row in the clone, and a list of DML statements whose text names the clone and not `SBX_INT_FIN`. (`SBX_INT_FIN.FIN_CALCULATION...` appears only inside the string that the procedure rewrote; read the statements that actually ran.) This is the sandbox version of the NAIC proof: counts in the clone, none in INT, and every DML statement against the clone.

Limits of the proof, to write in your notes: a row count cannot detect an `UPDATE` or `DELETE` in INT that leaves the count unchanged, and the query-history function only returns what your role can see. The strongest protection is not the check but the grants: `SBX_CALC_DEV` has only `SELECT` on the INT result table, so a write that escaped redirection would fail outright. The Break it section shows this.

### Step 5.13 Automate it: `scripts/refresh_dev_clone.sh`

Run in: **Terminal**. Create `scripts/refresh_dev_clone.sh` (the same five steps as the NAIC script: validate the ticket, confirm, drop, create, create the project, plan, deploy, re-plan):

```bash
#!/usr/bin/env bash
# Usage: scripts/refresh_dev_clone.sh <ticket> [--yes] [--deploy-only]
# Drops and recreates SBX_DEV_<TICKET>, deploys dcm/calc_engine into it, and requires an empty re-plan.
# --deploy-only keeps the clone and its data and only plans and deploys.
set -euo pipefail

TICKET="${1:-}"
shift || true
ASSUME_YES=false
DEPLOY_ONLY=false
for arg in "$@"; do
  case "$arg" in
    --yes) ASSUME_YES=true ;;
    --deploy-only) DEPLOY_ONLY=true ;;
    *) echo "Unknown option: $arg" >&2; exit 2 ;;
  esac
done
if [[ ! "$TICKET" =~ ^[A-Za-z0-9]{1,40}$ ]]; then
  echo "Ticket must be 1-40 letters or digits" >&2
  exit 2
fi
TICKET="$(echo "$TICKET" | tr '[:lower:]' '[:upper:]')"

SNOW="${SNOW_BIN:-snow}"
CONN="${SNOW_CONNECTION:-sbx_dev}"
ROLE="${CALC_DEV_ROLE:-SBX_CALC_DEV}"
WH="${CALC_WAREHOUSE:-SBX_CALC_WH}"
DCM_MODE="${DCM_MODE:-on}"
PLAN_FILE="${PLAN_FILE:-plan_result.json}"   # TO-CONFIRM TC-G04: real file name
SRC="dcm/calc_engine"
CLONE_DB="SBX_DEV_${TICKET}"
PROJECT="${CLONE_DB}.CALC_SUPPORT.CALC_ENGINE_DCM"
COMMON=(-c "$CONN" --role "$ROLE" --warehouse "$WH")
VARS=(-D "calc_db='${CLONE_DB}'" -D "fin_db='${CLONE_DB}'")

confirm() {
  if $ASSUME_YES; then return 0; fi
  read -r -p "$1 [y/N] " answer
  [[ "$answer" == "y" || "$answer" == "Y" ]]
}
call_proc() {
  "$SNOW" sql "${COMMON[@]}" --format json -q "CALL SBX_INT.CALC_DEV_SUPPORT.$1('${TICKET}')"
}
run_plan() {
  "$SNOW" dcm plan "$PROJECT" --target CLONE "${VARS[@]}" --from "$SRC" --save-output "${COMMON[@]}"
}
plan_is_empty() {
  python3 - "$SRC/out/$PLAN_FILE" <<'PY'
import json, sys
plan = json.load(open(sys.argv[1]))
sys.exit(0 if not plan.get("changeset") else 1)
PY
}

echo "Target clone: $CLONE_DB"
if ! $DEPLOY_ONLY; then
  echo "WARNING: results, drafts and manual edits in $CLONE_DB will be lost. Retention is 0, so there is no UNDROP."
  confirm "Drop and recreate $CLONE_DB?" || { echo "Cancelled"; exit 1; }
  out="$(call_proc DROP_CALC_DEV_CLONE)"; echo "$out"
  if echo "$out" | grep -Eq 'status[^A-Za-z]+ERROR'; then echo "Drop failed" >&2; exit 1; fi
  out="$(call_proc CREATE_CALC_DEV_CLONE)"; echo "$out"
  if echo "$out" | grep -Eq 'status[^A-Za-z]+ERROR'; then echo "Create failed" >&2; exit 1; fi
  if [[ "$DCM_MODE" == "off" ]]; then echo "DCM_MODE=off: stopping after clone creation"; exit 0; fi
  "$SNOW" dcm create "$PROJECT" --target CLONE --from "$SRC" "${COMMON[@]}"
fi

run_plan
if plan_is_empty; then echo "No changes: the clone matches the definitions"; exit 0; fi
confirm "Deploy these changes to $CLONE_DB?" || { echo "Cancelled"; exit 1; }
SHA="$(git rev-parse --short HEAD 2>/dev/null || echo nogit)"
"$SNOW" dcm deploy "$PROJECT" --target CLONE "${VARS[@]}" --from "$SRC" --alias "refresh-${TICKET}-${SHA}" "${COMMON[@]}"
run_plan
plan_is_empty || { echo "Re-plan is not empty: investigate before using this clone" >&2; exit 1; }
echo "Clone $CLONE_DB is in sync."
```

The script uses `git rev-parse` only to read a short commit id for the deployment alias. It makes no change to the repository.

Run in: **Terminal**: `chmod +x scripts/refresh_dev_clone.sh`, then `scripts/refresh_dev_clone.sh LAB5A --deploy-only` first. You should see the plan report "No changes". Then run the full refresh: `scripts/refresh_dev_clone.sh LAB5A`, answer `y` twice. You should see the drop, the create, the project creation, a non-empty first plan, a deploy with alias `refresh-LAB5A-<sha>`, and `Clone SBX_DEV_LAB5A is in sync.`

Timing for orientation: in NAIC a full refresh took about 5 minutes, mostly because of the size of the real engine; yours takes a minute or two.

### Step 5.14 Clean up the ticket clone

Run in: **Snowsight worksheet**, as the developer role:

```sql
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('LAB5A');
SHOW DATABASES LIKE 'SBX_DEV_%';
```

You should see: `DROPPED`, and no `SBX_DEV_` database in the list (the manual ones from Steps 5.1 to 5.5 are already gone).

## 6. Checkpoint questions

1. Why is a clone almost free, and when does it start to cost storage?
2. What does `DATA_RETENTION_TIME_IN_DAYS = 0` take away from a clone?
3. Why is a developer-supplied database name unsafe in a procedure that drops databases, and how does the ticket pattern prevent it?
4. Why does `EXECUTE AS OWNER` let a developer create a database that the developer role cannot create directly?
5. What is the Flyway history trap, and which fix does this lab use?
6. Why must the cloned DCM project object be dropped inside the clone?
7. Which three things make it hard for a clone run to write to INT by accident?
8. What can the row-count check not detect?

<details>
<summary>Answers</summary>

1. It shares storage with the source and copies only metadata. It costs storage once data in either side changes.
2. Time Travel and `UNDROP`. A dropped clone cannot be restored.
3. A caller-supplied name could be `SBX_INT`. The pattern allows only letters and digits and the procedure adds the fixed `SBX_DEV_` prefix itself, so only `SBX_DEV_` databases can be named.
4. The procedure runs with the rights of its owner (`SBX_CICD_PIPELINE`), which can create databases. The developer role only holds `USAGE` on the procedure.
5. The cloned history says the hybrid migrations ran, but the tables were skipped. The lab recreates the tables from INT's `GET_DDL` and keeps the history.
6. It would carry INT's snapshot of managed objects, so plans in the clone would be measured against the wrong state.
7. The redirect map derived from `calc_db`, the safety placeholder database in the CLONE target, and grants that give the developer role only `SELECT` on INT's tables.
8. An update or delete that leaves the row count unchanged, and writes to other tables.

</details>

## 7. Break it

Scenario: the redirects are forgotten, so generated SQL still names INT.

Run in: **Snowsight worksheet** (create a clone first: `CALL SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE('LAB5B');` and deploy the definitions as in Step 5.10, using `-D "calc_db='SBX_DEV_LAB5B'" -D "fin_db='SBX_DEV_LAB5B'"` and the project `SBX_DEV_LAB5B.CALC_SUPPORT.CALC_ENGINE_DCM`). Then as `SBX_CALC_DEV` replace the redirect map in the clone by hand:

```sql
CREATE OR REPLACE VIEW SBX_DEV_LAB5B.CALC_SUPPORT.ENV_CONFIG (CONFIG_KEY, CONFIG_VALUE) AS
SELECT 'SCHEMA_REDIRECTS', PARSE_JSON('[]');
CALL SBX_DEV_LAB5B.CALC_SUPPORT.SP_COMPUTE_MINI_RESULT(2001, NULL);
```

Symptom: the call fails with an insufficient-privileges error on the INT result table. Diagnosis: with no redirect the procedure tried to insert into `SBX_INT_FIN.FIN_CALCULATION...`, and the developer role has only `SELECT` there. That grant is the last line of defence. Run the INT baseline check from Step 5.12: still unchanged. Recovery: a DCM plan against the clone shows the drift on `ENV_CONFIG`; deploy to restore it and confirm an empty re-plan. Lesson: in NAIC the guard `SP_GET_ENV_REDIRECTS` checks only `HOME_SCHEMA`, not the map, which is why the clone target derives the map from `calc_db` and why grants matter. Then drop `LAB5B` with the drop procedure.

> **TO-CONFIRM [TC-07-08]:** the exact error text and whether `SP_COMPUTE_MINI_RESULT` fails at the dynamic insert (expected) or earlier. Record it in file 13.

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| Schema clone fails with hybrid tables unless `IGNORE HYBRID TABLES` | Steps 5.1, 5.2 | The clause plus empty recreation | VERIFIED (docs); error number TO-CONFIRM TC-07-01 |
| Hybrid tables cannot be single-table cloned or undropped | Step 5.5 | Database-level clone, or recreate | VERIFIED (docs) |
| Clone history says hybrid migrations ran | Step 5.3 | Recreate from `GET_DDL`, or fresh history | observed |
| Clones keep the source owners, so mixed ownership breaks the hand-off | Section 3 owner check | One owner role for all source objects | observed in NAIC |
| Retention 0 means a dropped clone is gone | Step 5.5 | Intentional; refresh scripts warn | TO-CONFIRM TC-07-03 |
| `GET_DDL` needs a constant object name | Step 5.7 | One call per table | observed in NAIC (error 1046) |
| A row count cannot prove "no writes" | Step 5.12 | Grants, query history | observed |
| Physical copy of hybrid data in a database clone costs compute | Step 5.4 | Skip hybrid tables | VERIFIED (docs); behaviour TO-CONFIRM TC-G15 |
| Scripting syntax in the procedures was not checked against public pages | Step 5.7 | Read `sqlerrm`, fallbacks in the callout | TO-CONFIRM TC-07-05 |

## 9. What you learned

- A zero-copy clone gives an isolated copy for almost no cost, and retention 0 removes the safety net.
- Hybrid tables need `IGNORE HYBRID TABLES` in a schema clone and are recreated empty from `GET_DDL`.
- A cloned Flyway history can claim work that the clone does not contain.
- A guarded `EXECUTE AS OWNER` procedure that accepts only a validated ticket is how a developer role gets a capability it does not hold.
- Redirects plus DCM's CLONE target make a clone write to itself, and grants are the last defence.

## 10. Cleanup

Run in: **Snowsight worksheet**:

```sql
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('LAB5A');
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('LAB5B');
DROP DATABASE IF EXISTS SBX_DEV_LAB5M;
DROP DATABASE IF EXISTS SBX_DEV_LAB5FULL;
SHOW DATABASES LIKE 'SBX_DEV_%';
```

Run in: **Terminal**: `unset FLYWAY_URL FLYWAY_PLACEHOLDERS_CALC_DB` (so the next lab does not point Flyway at a dropped clone), then reload `.env.flyway` when needed. Delete `dcm/calc_engine/out`.

Keep: both procedures and their grants, the three scripts, and the INT row counts. Cost note: clone storage is near zero until data diverges, and the warehouse runs only while statements run. Leftover clones are the usual cost leak; Lab 6 adds a scheduled sweeper for pull-request clones.

Learner action (git): commit `sql/clone_support/`, `scripts/refresh_dev_clone.sh`, `scripts/assert_zero_int_writes.sql` and `scripts/smoke_clone.sql` on a branch of your choice.

## 11. State left behind

- Procedures `SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE` and `DROP_CALC_DEV_CLONE` exist and `SBX_CALC_DEV` holds `USAGE` on both.
- `scripts/refresh_dev_clone.sh`, `scripts/assert_zero_int_writes.sql`, `scripts/smoke_clone.sql` exist. `scripts/plan_summary.py` from Lab 3 is used by them.
- Clone `SBX_DEV_LAB5A` (and the manual and Break it clones) are dropped.
- Your INT baseline row counts are written down: `SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE` and `SBX_INT.CALC_SUPPORT.EXECUTIONS`.

## 12. Open items for file 13

TC-G04, TC-G15, TC-G29, TC-07-01 (error text of a schema clone with hybrid tables), TC-07-02 (`SHOW DCM PROJECTS` form and whether a clone copies the project), TC-07-03 (UNDROP error with retention 0), TC-07-04 (developer `CREATE DATABASE` error), TC-07-05 (Scripting syntax: `FOR` over `RESULTSET`, argument-signature regex), TC-07-06 (first clone plan composition), TC-07-07 (`snow sql -D` templating), TC-07-08 (redirect-forgotten error). New names introduced here for the contract: none beyond the contract's scripts (`assert_zero_int_writes.sql` is written in this lab; `smoke_clone.sql` too).

## 13. Next

Continue with [08-lab6-cicd-github-actions.md](08-lab6-cicd-github-actions.md).
