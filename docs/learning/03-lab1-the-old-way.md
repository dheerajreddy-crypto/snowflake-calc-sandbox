# Lab 1 (file 03): the old way, a calc engine built by hand

> Lab no: 1 | File no: 03 | Time: 1 to 1.5 hours | Prerequisites: file 02 finished | Modes supported: H/S, D (unchanged in every mode) | Leaves behind: hand-made miniature engine in `SBX_INT.CALC_SUPPORT`, `sql/lab1/*` files, a written pain-point list.

## 1. Goal

Build a miniature calc engine the way NAIC did it before tooling: run SQL by hand, overwrite a procedure with `CREATE OR REPLACE`, edit a table in the UI, and then **feel the problems**: silent overwrite, drift between Git and Snowflake, no audit trail, no safe place to test. You finish with a written problem statement. Labs 2 to 11 solve it.

## 2. Why this matters in calc-engine and calc-admin

> **Real world.** The real engine was deployed by editing `sql/procedures.sql` (about 6,700 lines, 31 `CREATE OR REPLACE PROCEDURE` statements), pasting it into Snowsight and running it in INT. Evidence of drift found later: live objects with no DDL in Git (`SP_PROMOTE_METADATA`, `SP_TOP_STATES`, `FIN_CALCULATION_RESULT_DRAFT`, `DYNAMIC_FUNCTION_CONFIGS`, `PROMOTION_LOG`); `main` behind INT (no `FN_APPLY_REDIRECTS`, `SP_GET_ENV_REDIRECTS`, `ENV_CONFIG`); two Calc Admin hybrid tables, `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT`, created by hand on 2026-11-03 with no DDL in `sql/setup.sql`; mixed object owners (`ACCOUNTADMIN`, `SYSADMIN`, `CICD_PIPELINE`) that later broke clones. The deploy audit lived in GitLab, not in Snowflake. See file 01.

The sandbox objects are the same shapes in miniature: `CALCULATIONS`, `FORMULAS`, `FORMULA_CATEGORIES`, `SEQ_CALC_ID`, `SP_REGISTER_CALCULATION` (the real one is `SP_REGISTER_CALCULATIONS`), `SP_ADD_CATEGORY`.

## 3. Prerequisites and state check

Run in: **Snowsight worksheet** (or Terminal with `snow sql -c sbx_owner -q "<statement>"`).

```sql
USE ROLE SBX_CICD_PIPELINE;
SHOW SCHEMAS IN DATABASE SBX_INT;
SHOW GRANTS ON SCHEMA SBX_INT.CALC_SUPPORT;
```

You should see: schema `CALC_SUPPORT` owned by `SBX_CICD_PIPELINE`. If the schema is missing or owned by someone else, the bootstrap in file 02 did not finish; rerun parts 2 and 4.

## 4. Concepts

- **Imperative SQL.** You tell Snowflake each action: create this, alter that. There is no memory of intent. Running the same file twice can create duplicates, fail, or change nothing, depending on how the statements were written.
- **`CREATE OR REPLACE`.** Drops and recreates an object in one statement. Safe to repeat, but it silently destroys whatever was live, including a newer version, and (for tables) the data. Grants on a replaced object must be reapplied.
- **`CREATE ... IF NOT EXISTS`.** Creates only if the name is free. It never updates an existing object, so it hides differences.
- **Drift.** The live objects differ from the files in Git. It happens through hand edits, edits by someone else, and files that were never updated.
- **`INFORMATION_SCHEMA`.** Read-only views Snowflake provides in each database that describe its objects (tables, columns, procedures). You use it to detect drift.
- **Owner's rights procedure (`EXECUTE AS OWNER`).** The procedure runs with the privileges of the role that owns it, not the caller. The real engine uses this widely.
- **Sequence.** An object that hands out increasing numbers (`SEQ_CALC_ID.NEXTVAL`).

```text
   Git file (what we think exists)        Snowflake (what exists)
   -------------------------------        -------------------------
   procedures_v1.sql  ---- paste ------>  SP_REGISTER_CALCULATION v1
                                           ^
   (nobody notices)       Dev B pastes ----+  SP_REGISTER_CALCULATION v-B  (overwrites)
                          hand-edit   ----->  CALCULATIONS + NOTES column (not in Git)
```

> **TO-CONFIRM [TC-03-01]:** the SQL in this lab is core Snowflake syntax (`CREATE SEQUENCE`, `CREATE TABLE ... DEFAULT <seq>.NEXTVAL`, Snowflake Scripting procedures with `SELECT ... INTO :var`, `INFORMATION_SCHEMA` views). It was not re-read against the docs on 2026-11-16. How to check: run it; any error names the line. If `SELECT <seq>.NEXTVAL INTO :v_id` is rejected, use `v_id := (SELECT <seq>.NEXTVAL);`. Record the working form in file 13.

## 5. Step-by-step

### Step 5.1 Create the table and sequence files

Run in: **Terminal**, inside your repository folder. Create `sql/lab1/tables_v1.sql`:

```sql
-- sql/lab1/tables_v1.sql   (Lab 1, version 1 of the "Git file")
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;

CREATE SEQUENCE IF NOT EXISTS SBX_INT.CALC_SUPPORT.SEQ_CALC_ID    START = 2001 INCREMENT = 1 NOORDER;
CREATE SEQUENCE IF NOT EXISTS SBX_INT.CALC_SUPPORT.SEQ_FORMULA_ID START = 1    INCREMENT = 1 NOORDER;

CREATE TABLE IF NOT EXISTS SBX_INT.CALC_SUPPORT.CALCULATIONS (
  CALC_ID         NUMBER(38,0) NOT NULL DEFAULT SBX_INT.CALC_SUPPORT.SEQ_CALC_ID.NEXTVAL,
  CALC_NAME       VARCHAR(200) NOT NULL,
  CALC_STATUS     VARCHAR(10)  DEFAULT 'ACTIVE',     -- DRAFT, ACTIVE, INACTIVE
  SOURCE_CALC_ID  NUMBER(38,0),
  CREATED_BY      VARCHAR(255),
  CREATED_AT      TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (CALC_ID)
);

CREATE TABLE IF NOT EXISTS SBX_INT.CALC_SUPPORT.FORMULAS (
  FORMULA_ID    NUMBER(38,0) NOT NULL DEFAULT SBX_INT.CALC_SUPPORT.SEQ_FORMULA_ID.NEXTVAL,
  CALC_ID       NUMBER(38,0) NOT NULL,
  FORMULA_TEXT  VARCHAR(4000),
  PRIMARY KEY (FORMULA_ID)
);

CREATE TABLE IF NOT EXISTS SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES (
  CATEGORY_ID    NUMBER(38,0) NOT NULL AUTOINCREMENT START 1 INCREMENT 1 NOORDER,
  CATEGORY_NAME  VARCHAR(100) NOT NULL,
  CREATED_BY     VARCHAR(255),
  PRIMARY KEY (CATEGORY_ID)
);
```

You should see: a new file. Nothing has run yet.

> **Note for later labs.** The real `SEQ_CALC_ID` starts at 2001 (as in the branch definition). The pack adds `SEQ_FORMULA_ID` here because `FORMULAS` uses it. Lab 3 adopts both.

### Step 5.2 Create the procedure file

Run in: **Terminal**. Create `sql/lab1/procedures_v1.sql`:

```sql
-- sql/lab1/procedures_v1.sql   (Lab 1, version 1 of the "Git file")
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;

CREATE OR REPLACE PROCEDURE SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION(P_CALC_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS OWNER
AS
$$
DECLARE
    v_id NUMBER;
BEGIN
    SELECT SBX_INT.CALC_SUPPORT.SEQ_CALC_ID.NEXTVAL INTO :v_id;
    INSERT INTO SBX_INT.CALC_SUPPORT.CALCULATIONS (CALC_ID, CALC_NAME, CALC_STATUS, CREATED_BY)
    VALUES (:v_id, :P_CALC_NAME, 'DRAFT', :P_USER);
    RETURN TO_VARCHAR(:v_id);
END;
$$;

CREATE OR REPLACE PROCEDURE SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(P_CATEGORY_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS OWNER
AS
$$
BEGIN
    INSERT INTO SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES (CATEGORY_NAME, CREATED_BY)
    SELECT :P_CATEGORY_NAME, :P_USER
    WHERE NOT EXISTS (SELECT 1 FROM SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES WHERE CATEGORY_NAME = :P_CATEGORY_NAME);
    RETURN 'OK';
END;
$$;
```

You should see: a new file. Notice that the bodies use **fully qualified INT names** (`SBX_INT.CALC_SUPPORT....`). The real engine procedures do the same, and that is exactly why a clone needs a redirect mechanism (Lab 5).

`SP_ADD_CATEGORY` is idempotent: calling it twice with the same name inserts one row. `SP_REGISTER_CALCULATION` is not: each call creates a new draft.

### Step 5.3 Create the seed and drift files

Run in: **Terminal**. Create `sql/lab1/seed_v1.sql`:

```sql
-- sql/lab1/seed_v1.sql   run ONCE; it is not idempotent for the two CALL statements
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;

CALL SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION('MINI_LIFE_RATIO',     'LAB1');
CALL SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION('MINI_PROPERTY_TOTAL', 'LAB1');

UPDATE SBX_INT.CALC_SUPPORT.CALCULATIONS
   SET CALC_STATUS = 'ACTIVE'
 WHERE CALC_NAME IN ('MINI_LIFE_RATIO', 'MINI_PROPERTY_TOTAL') AND CALC_STATUS <> 'ACTIVE';

CALL SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY('RATIO', 'LAB1');
CALL SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY('TOTAL', 'LAB1');

INSERT INTO SBX_INT.CALC_SUPPORT.FORMULAS (CALC_ID, FORMULA_TEXT)
SELECT CALC_ID,
       CASE CALC_NAME WHEN 'MINI_LIFE_RATIO' THEN 'NUMERATOR / DENOMINATOR' ELSE 'SUM(AMOUNT)' END
  FROM SBX_INT.CALC_SUPPORT.CALCULATIONS
 WHERE CALC_NAME IN ('MINI_LIFE_RATIO', 'MINI_PROPERTY_TOTAL');
```

And `sql/lab1/drift_detective.sql`:

```sql
-- sql/lab1/drift_detective.sql   compares live columns with the list the Git file creates
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;

WITH expected (TABLE_NAME, COLUMN_NAME) AS (
  SELECT column1, column2 FROM VALUES
    ('CALCULATIONS','CALC_ID'), ('CALCULATIONS','CALC_NAME'), ('CALCULATIONS','CALC_STATUS'),
    ('CALCULATIONS','SOURCE_CALC_ID'), ('CALCULATIONS','CREATED_BY'), ('CALCULATIONS','CREATED_AT'),
    ('FORMULAS','FORMULA_ID'), ('FORMULAS','CALC_ID'), ('FORMULAS','FORMULA_TEXT'),
    ('FORMULA_CATEGORIES','CATEGORY_ID'), ('FORMULA_CATEGORIES','CATEGORY_NAME'), ('FORMULA_CATEGORIES','CREATED_BY')
),
live AS (
  SELECT TABLE_NAME, COLUMN_NAME
    FROM SBX_INT.INFORMATION_SCHEMA.COLUMNS
   WHERE TABLE_SCHEMA = 'CALC_SUPPORT'
)
SELECT 'IN_LIVE_NOT_IN_FILE' AS DRIFT, l.TABLE_NAME, l.COLUMN_NAME
  FROM live l LEFT JOIN expected e ON e.TABLE_NAME = l.TABLE_NAME AND e.COLUMN_NAME = l.COLUMN_NAME
 WHERE e.COLUMN_NAME IS NULL
UNION ALL
SELECT 'IN_FILE_NOT_IN_LIVE', e.TABLE_NAME, e.COLUMN_NAME
  FROM expected e LEFT JOIN live l ON e.TABLE_NAME = l.TABLE_NAME AND e.COLUMN_NAME = l.COLUMN_NAME
 WHERE l.COLUMN_NAME IS NULL
ORDER BY 1, 2, 3;

-- Procedures: the definition text and a fingerprint of it.
-- Note the fingerprint after Step 5.5 and compare it after Step 5.7.
SELECT PROCEDURE_NAME, LAST_ALTERED, SHA2(PROCEDURE_DEFINITION) AS DEFINITION_FINGERPRINT
  FROM SBX_INT.INFORMATION_SCHEMA.PROCEDURES
 WHERE PROCEDURE_SCHEMA = 'CALC_SUPPORT'
 ORDER BY PROCEDURE_NAME;

-- Tables and other objects the file does not know about
SELECT TABLE_NAME, TABLE_TYPE, CREATED, LAST_ALTERED
  FROM SBX_INT.INFORMATION_SCHEMA.TABLES
 WHERE TABLE_SCHEMA = 'CALC_SUPPORT'
 ORDER BY TABLE_NAME;
```

### Step 5.4 Run the files

Run in: **Terminal** (activate your venv first).

```bash
snow sql -c sbx_owner -f sql/lab1/tables_v1.sql
snow sql -c sbx_owner -f sql/lab1/procedures_v1.sql
snow sql -c sbx_owner -f sql/lab1/seed_v1.sql
```

You should see: no errors; the two `CALL` statements return `2001` and `2002` (the sequence starts at 2001, so these are the first two ids; if you ran the seed twice you will see larger ids).

### Step 5.5 Look at the state and record a baseline

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`.

```sql
SELECT * FROM SBX_INT.CALC_SUPPORT.CALCULATIONS ORDER BY CALC_ID;
SELECT * FROM SBX_INT.CALC_SUPPORT.FORMULAS;
SELECT * FROM SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES;
```

You should see: two calculations, `MINI_LIFE_RATIO` (2001) and `MINI_PROPERTY_TOTAL` (2002), both `ACTIVE`; two formulas; two categories, `RATIO` and `TOTAL`.

Now run the drift detective file. You should see: the first query returns **no rows** (live equals file), the second lists both procedures with fingerprints. **Write down the fingerprint of `SP_REGISTER_CALCULATION`.** This is your stand-in for "what Git says".

Commit the four `sql/lab1/*` files yourself when you are ready (`git add`, `git commit`). Nothing in this lab needs a push.

### Step 5.6 A second developer overwrites the procedure

Developer B pastes a different version into a worksheet. In the sandbox the same role plays developer B.

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`.

```sql
CREATE OR REPLACE PROCEDURE SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION(P_CALC_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS OWNER
AS
$$
DECLARE
    v_id NUMBER;
BEGIN
    SELECT SBX_INT.CALC_SUPPORT.SEQ_CALC_ID.NEXTVAL INTO :v_id;
    INSERT INTO SBX_INT.CALC_SUPPORT.CALCULATIONS (CALC_ID, CALC_NAME, CALC_STATUS, CREATED_BY)
    VALUES (:v_id, UPPER(:P_CALC_NAME), 'ACTIVE', :P_USER);   -- Dev B "improvement": upper-case names, skip the draft state
    RETURN TO_VARCHAR(:v_id);
END;
$$;
```

You should see: `Function SP_REGISTER_CALCULATION successfully created.` (the message wording may differ). **No warning appeared** that you replaced a version someone else owned. Run the drift detective again. You should see: a different `DEFINITION_FINGERPRINT` and a newer `LAST_ALTERED` for `SP_REGISTER_CALCULATION`. Nothing in Git changed, so `git status` shows a clean tree.

Restore the Git version by re-running `procedures_v1.sql`:

```bash
snow sql -c sbx_owner -f sql/lab1/procedures_v1.sql
```

You should see: the fingerprint equals the one you wrote down. The overwrite and restore worked only because **you knew** the file was the truth.

### Step 5.7 Edit a table by hand in the UI

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`. This is what a colleague does at 5 pm to "just add a column".

```sql
ALTER TABLE SBX_INT.CALC_SUPPORT.CALCULATIONS ADD COLUMN NOTES VARCHAR(500);
UPDATE SBX_INT.CALC_SUPPORT.CALCULATIONS SET NOTES = 'checked by hand' WHERE CALC_ID = 2001;
```

Run the drift detective. You should see: the first query returns one row, `IN_LIVE_NOT_IN_FILE | CALCULATIONS | NOTES`. Re-running `tables_v1.sql` does **not** fix it: `CREATE TABLE IF NOT EXISTS` sees the table and does nothing, so it hides the drift.

### Step 5.8 No audit trail

Run in: **Snowsight worksheet**.

```sql
SELECT QUERY_TEXT, USER_NAME, ROLE_NAME, START_TIME
  FROM TABLE(SBX_INT.INFORMATION_SCHEMA.QUERY_HISTORY(RESULT_LIMIT => 200))
 WHERE QUERY_TEXT ILIKE '%CREATE OR REPLACE PROCEDURE%'
 ORDER BY START_TIME DESC;
```

You should see: your own statements, with your user name for both "developers" because you were one person. In a team on a shared service user, you would see the same name for everyone. Query history is short-lived and not a deployment record: it does not say which Git commit or ticket the change belongs to.

> **TO-CONFIRM [TC-03-02]:** how far back `INFORMATION_SCHEMA.QUERY_HISTORY` reaches and which columns it shows in your edition. How to check: the output above and the Snowflake docs page for `QUERY_HISTORY`. Record in file 13.

### Step 5.9 No safe test environment

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`.

```sql
CALL SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION('TRY_NEW_IDEA', 'LAB1');
SELECT * FROM SBX_INT.CALC_SUPPORT.CALCULATIONS ORDER BY CALC_ID;
```

You should see: a third row `TRY_NEW_IDEA` as `DRAFT`. To test a procedure you had to write to the "real" table. In NAIC the equivalent test would write to INT, which other people use. Remove the test row by hand:

```sql
DELETE FROM SBX_INT.CALC_SUPPORT.CALCULATIONS WHERE CALC_NAME = 'TRY_NEW_IDEA';
```

Notice that the sequence does not go back: the next id is 2004 or higher. A hand cleanup leaves traces.

### Step 5.10 Write your pain points and the problem statement

Run in: **your editor**. Create `docs/learning/my-notes/lab1-pain-points.md` (this file is yours; the pack does not contain it). List at least:

1. A silent overwrite (Step 5.6) and how you noticed it.
2. Drift in a table (Step 5.7) and that `IF NOT EXISTS` hid it.
3. No record of who changed what and why (Step 5.8).
4. No safe place to test (Step 5.9).
5. No preview: you cannot ask "what would this file change?" before running it.
6. One more of your own.

Then complete this sentence in your own words: "We need a way to ______ so that ______."

A good answer: "We need a way to declare the intended state of the database objects in files, preview the difference, and apply it from a pipeline, so that the live objects always match Git, every change has an author and a review, and developers test in a copy and not in the shared environment."

## 6. Checkpoint questions

1. Why did re-running `tables_v1.sql` not remove the `NOTES` column or report it?
2. What is the difference between `CREATE OR REPLACE` and `CREATE ... IF NOT EXISTS`, and what risk does each have?
3. How did the drift detective know the procedure changed? What would it miss (give one example)?
4. Why did the two "developers" look like one user in the query history?
5. Why is `SP_ADD_CATEGORY` safe to run twice but `SP_REGISTER_CALCULATION` is not?
6. In the real engine, which two problems did hand-pasting `procedures.sql` into INT cause (use file 01)?
7. The procedure bodies hold `SBX_INT...` names. What goes wrong if you clone the schema and call the procedure in the clone?

Answers:

1. `IF NOT EXISTS` skips an existing table, so it never compares columns.
2. `OR REPLACE` always overwrites (destroys newer work and, for tables, data and grants). `IF NOT EXISTS` never updates (hides differences).
3. Through a changed `SHA2` of `PROCEDURE_DEFINITION` and `LAST_ALTERED`. It misses grants, owners, comments and any property it does not query.
4. Both used the same role and login. Snowflake logs the user, not the person or ticket behind it.
5. The insert in `SP_ADD_CATEGORY` has a `NOT EXISTS` guard; `SP_REGISTER_CALCULATION` takes a new sequence value each call.
6. Any two of: silent revert of newer code (`main` behind INT), live objects without DDL in Git, mixed owners that broke clones, no deploy audit, hot fixes overwritten.
7. The call in the clone still writes to `SBX_INT` because the name is fully qualified. Lab 5 shows redirects.

## 7. Break it

**Exercise: a developer without rights tries to deploy.** Run in **Snowsight worksheet**, role `SBX_CALC_DEV`:

```sql
USE ROLE SBX_CALC_DEV;
CREATE OR REPLACE PROCEDURE SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(P_CATEGORY_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL AS $$ BEGIN RETURN 'HACK'; END; $$;
```

- **Symptom:** an error about insufficient privileges (the exact text is not recorded in this pack; copy it into your notes).
- **Diagnosis:** `SBX_CALC_DEV` has `USAGE` on the schema but does not own it and has no `CREATE PROCEDURE` right there. Check with `SHOW GRANTS ON SCHEMA SBX_INT.CALC_SUPPORT;`.
- **Recovery:** nothing to recover, nothing changed. Switch back with `USE ROLE SBX_CICD_PIPELINE;`.
- **Lesson:** ownership is what protects objects. The real clones broke when objects had different owners, which is why every INT object ended up owned by `CICD_PIPELINE`. Without a tool, ownership is the only guard.

## 8. Limitations seen in this lab

| Limitation of "no tooling" | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| Silent overwrite by `CREATE OR REPLACE` | Step 5.6 | Pipeline-only deploy with review (Lab 6), DCM plan (Lab 3) | VERIFIED by running |
| Drift between Git and live | Step 5.7 | DCM plan as a drift detector (Lab 3), ownership rules | VERIFIED by running |
| `IF NOT EXISTS` hides differences | Step 5.7 | Declarative diff (DCM) or numbered migrations (Flyway) | VERIFIED by running |
| No audit trail with author, ticket, commit | Step 5.8 | Git history plus pipeline logs; Flyway history; DCM deployment history | VERIFIED by running |
| No safe test environment | Step 5.9 | Zero-copy clones (Lab 5) | VERIFIED by running |
| No preview of a change | whole lab | DCM `plan` (Lab 3); Flyway `info` shows only pending files | VERIFIED by running |

## 9. What you learned

- Hand-run SQL has no memory: `OR REPLACE` overwrites, `IF NOT EXISTS` ignores differences.
- Drift is easy to create and invisible unless you go looking with queries.
- A shared login destroys the audit trail.
- Testing in the real schema leaves traces (rows, sequence gaps).
- The real engine had every one of these problems, and the tooling in the next labs exists to remove them.

## 10. Cleanup

Keep every object: Lab 3 adopts them and Lab 2 clones them. Remove the hand-added column so the live shape equals the Git files again (Lab 2's baseline and Lab 3's adoption assume that shape; Lab 3 recreates drift on purpose):

```sql
ALTER TABLE SBX_INT.CALC_SUPPORT.CALCULATIONS DROP COLUMN NOTES;
```

Run the drift detective once more. You should see: no rows from the first query. Then run `ALTER WAREHOUSE SBX_CALC_WH SUSPEND;` at the end of your session.

## 11. State left behind

`SBX_INT.CALC_SUPPORT` holds, hand-made: `SEQ_CALC_ID`, `SEQ_FORMULA_ID`, `CALCULATIONS`, `FORMULAS`, `FORMULA_CATEGORIES`, `SP_REGISTER_CALCULATION` (v1, overwritten once and restored), `SP_ADD_CATEGORY`, with seed rows (calc ids 2001 and 2002, two categories, two formulas). `sql/lab1/*` is created (committed by you). Pain-point list written. Objects are owned by `SBX_CICD_PIPELINE`.

## 12. Open items for file 13

TC-03-01, TC-03-02.

## 13. Next

Go to [04-lab2-flyway-fundamentals.md](04-lab2-flyway-fundamentals.md).
