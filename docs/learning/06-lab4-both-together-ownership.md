# Lab 4: DCM and Flyway together, one owner per object (file 06)

> **Lab no:** 4 | **File no:** 06 | **Time estimate:** 2.5 to 3 h | **Prerequisites:** file 04 (Flyway fundamentals, Flyway installed) and file 05 (DCM project adopted, empty re-plan) | **Modes supported:** H or S (standard stand-ins), D (DCM parts become read-along in Mode D-off) | **Leaves behind:** Flyway history `FLYWAY_HISTORY_CALC_SUPPORT` at version 4, hybrid pair (Mode H) or stand-in tables (Mode S), a `DATA_CHANGE_LOG` ledger row for V3, `FORMULA_CATEGORIES` renamed to `CALC_CATEGORIES` with the DCM definition renamed and an empty re-plan, conflict experiment cleaned up, a written decision table

## 1. Goal

By the end of this lab you can:

1. Split a database between DCM and Flyway so that every object has exactly one owner tool.
2. Show that DCM and Flyway coexist in one schema without touching each other's objects.
3. Run an idempotent data fix with a row-count assertion as a Flyway migration.
4. Rename a table safely with the coordinated protocol, after first reading the unsafe DCM plan.
5. Cause a two-owner conflict on purpose, watch the tools fight, and explain the cure.
6. Say which tool to use for any given change, and in which order they run.

## 2. Why this matters in calc-engine and calc-admin

> **Real world.** The NAIC target architecture states the rule in one sentence: every runtime responsibility, database object, migration, identity and artifact has one named owner, and no object has overlapping DCM and Flyway ownership. The split is: DCM owns the supported engine objects (standard tables, procedures, functions, views, sequences, approved grants, ordinary tasks). Flyway owns the Calc Admin lifecycle schema with its hybrid tables, explicit renames and type transitions, data changes, clone-support objects and the existing `FIN_CALCULATION` migrations. Terraform owns accounts, databases, schemas, roles, warehouses and integrations.
>
> Why two tools? Because DCM cannot manage hybrid tables (its `DEFINE TABLE` has no `HYBRID` or `INDEX`; the POC saw `unexpected 'HYBRID'`), cannot run data changes, and reads a rename as drop plus create. The two live Calc Admin tables `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT` were created by hand in INT on 2026-11-03 and exist in no Git file, so they are the first things Flyway must adopt. The engine's hand-built `data_changes/` runner with its `DATA_CHANGE_LOG` ledger is Flyway's idea reimplemented by hand; the target moves it into the Flyway lane. This lab builds those pieces in miniature.
>
> A working note dated 2026-11-11 recorded pausing DCM; the team has since confirmed that both tools are being implemented, so this lab teaches them as a pair, not as alternatives.

## 3. Prerequisites and state check

Run in: **Terminal** (repository root).

```
flyway -v
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: a Flyway version (the pack pins 13.12.2) and `EMPTY changeset`. If the plan is not empty, finish Lab 3 first. If `flyway -v` fails, install it as in file 02.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SELECT COUNT(*) AS CATEGORIES_N, HASH_AGG(*) AS CATEGORIES_HASH FROM SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES;
SHOW SEQUENCES LIKE 'SEQ_%' IN SCHEMA SBX_INT.CALC_SUPPORT;
SHOW TABLES LIKE 'CALC_%' IN SCHEMA SBX_INT.CALC_SUPPORT;
```

You should see: the two seed categories, sequences `SEQ_WORKFLOW_ID` and `SEQ_AUDIT_EVENT_ID` (created by Lab 3, because the hybrid tables use them as defaults), and no `CALC_WORKFLOW_STATUS` or `CALC_AUDIT_EVENT` yet. Write down `CATEGORIES_N` and `CATEGORIES_HASH` for Step 4.9.

Decide your mode from file 02: `HYBRID_MODE = H` (hybrid tables work in your account) or `S` (trial, no hybrid tables). In this lab, Mode S differs in three places, each marked with `#### If you are in Mode S (no hybrid tables)`: the migration folder, the missing indexes, and the primary-key demo.

> **VERIFIED:** hybrid tables are not supported in trial accounts (https://docs.snowflake.com/en/user-guide/tables-hybrid-limitations and https://docs.snowflake.com/en/user-guide/admin-trial-account, read 2026-11-16).

## 4. Concepts

**Owner tool.** The one tool whose files are allowed to create, alter and drop an object. Everyone else may read it, grant on it, or depend on it, but never define it.

**Why one owner.** Each tool believes it knows the desired state. DCM compares definitions with a snapshot. Flyway compares a history table with files. Neither looks at the other. If both write the same object, one overwrites the other and neither notices until data is lost (Step 4.11).

```
 Terraform / bootstrap : databases, schemas, roles, warehouses, resource monitors, users
 DCM                   : sequences, standard tables, views, functions, procedures, declared grants
 Flyway                : hybrid tables + indexes + their grants, data changes, explicit renames
 snow sql -f           : clone-support procedures (sandbox choice, see Lab 5)

 same schema SBX_INT.CALC_SUPPORT, different objects, different histories:
   DCM project object CALC_ENGINE_DCM        (what DCM defined)
   table FLYWAY_HISTORY_CALC_SUPPORT         (what Flyway ran)
```

**Ordering.** In a normal change, DCM goes first and Flyway second, because the Flyway hybrid tables use sequences DCM creates. Then a fresh DCM re-plan must be empty. In a rename change the order flips: Flyway renames first, then DCM updates the definition (Step 4.9).

**Idempotent data fix.** A change that is safe to run twice: the second run finds nothing to do. It is guarded by a row-count assertion so it fails loudly when the data is not what you expected, instead of updating the wrong rows.

**Hybrid table (Mode H).** A Snowflake table type with a mandatory enforced primary key, row-level locking and secondary indexes, designed for many small reads and writes. Lab 7 goes deeper. Here you only need: it cannot be defined by DCM, and in Mode S you replace it with a standard table that has the same columns.

**Forward-only.** An applied Flyway migration is never edited. A mistake is fixed by the next version.

> **VERIFIED:** Flyway Community has no undo command and no dry run; Snowflake commits DDL immediately, so a half-failed migration leaves earlier statements applied (https://documentation.red-gate.com/fd/feature-summary-165740620.html and https://documentation.red-gate.com/flyway/reference/database-driver-reference/snowflake, read 2026-11-16).

## 5. Step-by-step

### Part A: Flyway takes the hybrid pair (about 50 min)

### Step 4.1 Configure the main Flyway project

Run in: **Terminal**. Create `flyway/flyway.toml` (committed, no secrets). It is the file from the contract; Lab 2 used a separate scratch project, this one is the real one.

```toml
[environments.target]
url = "jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE"
user = "SBX_SVC_CICD"
# private key and url overrides come from environment variables / CLI parameters, never from this file

[flyway]
environment = "target"
locations = ["filesystem:migrations/common", "filesystem:migrations/hybrid"]
defaultSchema = "CALC_SUPPORT"
schemas = ["CALC_SUPPORT"]
table = "FLYWAY_HISTORY_CALC_SUPPORT"
validateOnMigrate = true
outOfOrder = false
baselineOnMigrate = false
cleanDisabled = true
createSchemas = false

[flyway.placeholders]
calc_db = "SBX_INT"
calc_schema = "CALC_SUPPORT"
app_role = "SBX_FN_CALC_ENGINE_NP"
reader_role = "SBX_CALC_READER"
```

Every setting in the `[flyway]` block mirrors the NAIC target contract: validate on migrate, no out-of-order, no automatic baseline, `clean` disabled, no schema creation (the platform lane creates schemas), and a dedicated history table per migration boundary and environment.

Create `flyway/.env.flyway.example` (committed) and copy it to `flyway/.env.flyway` (gitignored). Edit only the copy:

```
FLYWAY_URL=jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE&authenticator=snowflake_jwt&private_key_file=<ABSOLUTE_PATH_TO_HOME>/.snowflake-sandbox/keys/sbx_svc_cicd_rsa_key.p8
FLYWAY_LOCATIONS=filesystem:migrations/common,filesystem:migrations/hybrid
FLYWAY_PLACEHOLDERS_CALC_DB=SBX_INT
FLYWAY_PLACEHOLDERS_CALC_SCHEMA=CALC_SUPPORT
```

The file holds only a path to the key, never the key and never the passphrase.

#### If you are in Mode S (no hybrid tables)

Set `FLYWAY_LOCATIONS=filesystem:migrations/common,filesystem:migrations/standard` in your `flyway/.env.flyway`. Everything else is identical. Do not switch modes later without a fresh history, because the checksums of V1, V2 and V5 differ between the two folders.

Run in: **Terminal**. Load the settings and type the passphrase without echo (never write it in a file or in chat):

```
cd flyway
set -a; source .env.flyway; set +a
read -rs PRIVATE_KEY_PASSPHRASE; export PRIVATE_KEY_PASSPHRASE
flyway info
```

You should see: a connection to Snowflake and an empty list of migrations (the folders are still empty, and the history table does not exist yet; `migrate` creates it).

> **VERIFIED:** the Snowflake JDBC URL needs `db` and `warehouse`; key-pair login uses `authenticator=snowflake_jwt&private_key_file=<absolute path>`; `flyway.toml` is the default config format and cannot be mixed with `flyway.conf`; environment variables follow `FLYWAY_<UPPER_SNAKE>` and override files (https://documentation.red-gate.com/flyway/reference/database-driver-reference/snowflake and https://www.red-gate.com/hub/product-learning/flyway/using-toml-configuration-files-with-flyway, read 2026-11-16).

> **TO-CONFIRM [TC-G21]:** whether an encrypted key needs `JAVA_OPTS='-Dnet.snowflake.jdbc.useBundledBouncyCastleForPrivateKeyDecryption=true'` on Flyway 13.12.2 (the driver page mentions it for Linux). How to check at lab time: if `flyway info` fails with a key decryption error, export that variable and retry. Record the result in file 13.

> **TO-CONFIRM [TC-06-01]:** that `FLYWAY_URL` overrides the `url` of the `target` environment in the TOML file, and that `FLYWAY_LOCATIONS` accepts a comma-separated list. How to check at lab time: `flyway info` shows the connected database and the locations it scanned. Fallback: pass `-url="..."` and `-locations="..."` on the command line. Record the result in file 13.

### Step 4.2 Write V1 and V2 (the hybrid pair)

Run in: **Terminal**. Create the folders and files (you are still in `flyway/`):

```
mkdir -p migrations/common migrations/hybrid migrations/standard
```

`migrations/hybrid/V1__create_calc_workflow_status.sql`:

```sql
CREATE HYBRID TABLE ${calc_db}.${calc_schema}.CALC_WORKFLOW_STATUS (
    WORKFLOW_ID NUMBER(38,0) DEFAULT ${calc_db}.${calc_schema}.SEQ_WORKFLOW_ID.NEXTVAL NOT NULL,
    CALC_ID NUMBER(38,0) NOT NULL,
    ADMIN_STATUS VARCHAR(16) NOT NULL,
    UPDATED_BY VARCHAR(128),
    UPDATED_AT TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_CALC_WORKFLOW_STATUS PRIMARY KEY (WORKFLOW_ID),
    INDEX IDX_WORKFLOW_CALC_ID (CALC_ID)
);

GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE ${calc_db}.${calc_schema}.CALC_WORKFLOW_STATUS TO ROLE ${app_role};
GRANT SELECT ON TABLE ${calc_db}.${calc_schema}.CALC_WORKFLOW_STATUS TO ROLE ${reader_role};
```

`migrations/hybrid/V2__create_calc_audit_event.sql` (append-only for the app role: INSERT and SELECT only):

```sql
CREATE HYBRID TABLE ${calc_db}.${calc_schema}.CALC_AUDIT_EVENT (
    EVENT_ID NUMBER(38,0) DEFAULT ${calc_db}.${calc_schema}.SEQ_AUDIT_EVENT_ID.NEXTVAL NOT NULL,
    CALC_ID NUMBER(38,0) NOT NULL,
    EVENT_TYPE VARCHAR(32),
    EVENT_STATUS VARCHAR(16) NOT NULL,
    ACTOR VARCHAR(128),
    EVENT_AT TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_CALC_AUDIT_EVENT PRIMARY KEY (EVENT_ID),
    INDEX IDX_AUDIT_CALC_ID (CALC_ID)
);

GRANT SELECT, INSERT ON TABLE ${calc_db}.${calc_schema}.CALC_AUDIT_EVENT TO ROLE ${app_role};
GRANT SELECT ON TABLE ${calc_db}.${calc_schema}.CALC_AUDIT_EVENT TO ROLE ${reader_role};
```

`EVENT_STATUS` takes the values `PENDING` or `COMPLETE`. The grants are in the migration on purpose: the hybrid tables are Flyway's, so their grants are Flyway's too. DCM's `access.sql` does not mention them.

#### If you are in Mode S (no hybrid tables)

Create the same two file names in `migrations/standard/`. Change `CREATE HYBRID TABLE` to `CREATE TABLE` and delete the `INDEX ...` lines (and the comma that precedes them). Keep every column, the primary key constraint and the grants. A standard table accepts the `PRIMARY KEY` clause but does not enforce it; Step 4.5 shows what that means.

> **VERIFIED:** `CREATE HYBRID TABLE` syntax with a mandatory primary key, `DEFAULT <sequence>.NEXTVAL`, and an out-of-line `INDEX <name> (<cols>)` clause; the primary key and the other constraints are enforced (https://docs.snowflake.com/en/sql-reference/sql/create-hybrid-table and https://docs.snowflake.com/en/user-guide/tables-hybrid-limitations, read 2026-11-16).

> **TO-CONFIRM [TC-G14]:** `CREATE HYBRID TABLE` needs the schema privilege `CREATE HYBRID TABLE` since bundle 2026_02. Because `SBX_CICD_PIPELINE` owns `CALC_SUPPORT`, the migration should work without an explicit grant. If it fails with an insufficient-privileges error, run in a worksheet as `SECURITYADMIN` or `ACCOUNTADMIN`: `GRANT CREATE HYBRID TABLE ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CICD_PIPELINE;` and record the exact error in file 13.

### Step 4.3 Migrate and read the result

Run in: **Terminal** (still in `flyway/` with the environment loaded):

```
flyway validate
flyway migrate
flyway info
```

You should see: `validate` succeeds (there is nothing applied yet), `migrate` reports that it created the history table and applied versions 1 and 2, and `info` lists both with state `Success`.

> **TO-CONFIRM [TC-G08]:** none of these two files contains `$$` blocks, so parsing is not at risk here. The data-fix migration in Step 4.6 does contain them; see there.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SELECT * FROM SBX_INT.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT;
SELECT TABLE_NAME, IS_HYBRID FROM SBX_INT.INFORMATION_SCHEMA.TABLES
 WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_NAME LIKE 'CALC\_%' ORDER BY 1;
SHOW INDEXES IN TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT;
SHOW GRANTS ON TABLE SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT;
```

You should see: two history rows (version 1 and 2, success), `IS_HYBRID = YES` for both tables in Mode H (`NO` in Mode S), in Mode H an index list that includes your secondary index with a status of `ACTIVE` (or `BUILD IN PROGRESS` for a moment), and grants showing only `SELECT` and `INSERT` for the app role.

### Step 4.4 Prove the two tools coexist

Run in: **Terminal** (repository root, so `cd ..` first):

```
cd ..
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

Run in: **Snowsight worksheet**:

```sql
SHOW ENTITIES IN DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;
```

You should see: the DCM plan is still `EMPTY changeset`, and the entity list does not contain `CALC_WORKFLOW_STATUS`, `CALC_AUDIT_EVENT` or `FLYWAY_HISTORY_CALC_SUPPORT`. DCM only drops objects it manages. Objects it never defined are invisible to it, which is exactly why a Flyway-owned table can live next to DCM-owned tables in one schema.

This is the first half of the ownership story. The second half (Step 4.11) shows what happens when both tools claim one object.

### Step 4.5 What the hybrid pair gives you: an enforced primary key and a sequence default

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (CALC_ID, ADMIN_STATUS) VALUES (2001, 'DRAFT');
SELECT WORKFLOW_ID, CALC_ID, ADMIN_STATUS FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS;
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (WORKFLOW_ID, CALC_ID, ADMIN_STATUS) VALUES (900, 2001, 'DRAFT');
INSERT INTO SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS (WORKFLOW_ID, CALC_ID, ADMIN_STATUS) VALUES (900, 2002, 'DRAFT');
```

You should see: the first insert gets a `WORKFLOW_ID` from `SEQ_WORKFLOW_ID` without you supplying it (the sequence is DCM's object, used by Flyway's table). In Mode H the last statement fails with a duplicate key error on the primary key. In Mode S it succeeds, and two rows now share `WORKFLOW_ID` 900.

> **TO-CONFIRM [TC-06-02]:** the exact duplicate-key error text for a hybrid table. The docs say the primary key is enforced; the message text was not captured. How to check at lab time: read your own error and paste it into file 13.

Why this matters: in the target Calc Admin design the only job of the hybrid table is to hold an enforced key and a row lock, "one open revision per calc set and year". A standard table can only hold that key as documentation. Lab 7 builds on this.

Clean the demo rows:

```sql
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS;
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS;
```

You should see: `0`. The tables must be empty when you leave this step (the NAIC target requires an "empty state" check after creating a hybrid table in a clone).

### Part B: a data change as a migration (about 30 min)

### Step 4.6 Write V3, the idempotent data fix

Scenario: some old calculation rows have no status. The business rule says the default is `ACTIVE`. The fix must be repeatable, must touch exactly the expected rows, and must leave a ledger row. This is the Flyway version of the NAIC data change `V202611110955__sync_legacy_target_db_schema.sql`, which used unqualified table names, a schema guard, a precondition check, one transaction, a count of pending rows, the update, a count of remaining rows, an assertion and a commit.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`): create two "legacy" rows. Rows are data, so DCM does not care.

```sql
INSERT INTO SBX_INT.CALC_SUPPORT.CALCULATIONS (CALC_NAME, CALC_STATUS) VALUES ('LEGACY_A', NULL), ('LEGACY_B', NULL);
SELECT COUNT(*) AS NULL_STATUS_ROWS FROM SBX_INT.CALC_SUPPORT.CALCULATIONS WHERE CALC_STATUS IS NULL;
```

You should see: `2`. The migration will hard-code that number as `expected_rows`.

Run in: **Terminal**. Create `flyway/migrations/common/V3__data_fix_default_calc_status.sql`:

```sql
EXECUTE IMMEDIATE $$
DECLARE
    expected_rows NUMBER DEFAULT 2;
    pending_rows NUMBER;
    changed_rows NUMBER;
    e_unexpected EXCEPTION (-20010, 'V3 assertion failed: unexpected number of rows');
BEGIN
    SELECT COUNT(*) INTO :pending_rows
      FROM ${calc_db}.${calc_schema}.CALCULATIONS WHERE CALC_STATUS IS NULL;
    IF (:pending_rows = 0) THEN
        RETURN 'nothing to do';
    END IF;
    IF (:pending_rows <> :expected_rows) THEN
        RAISE e_unexpected;
    END IF;
    BEGIN TRANSACTION;
    UPDATE ${calc_db}.${calc_schema}.CALCULATIONS SET CALC_STATUS = 'ACTIVE' WHERE CALC_STATUS IS NULL;
    changed_rows := SQLROWCOUNT;
    IF (:changed_rows <> :expected_rows) THEN
        ROLLBACK;
        RAISE e_unexpected;
    END IF;
    INSERT INTO ${calc_db}.${calc_schema}.DATA_CHANGE_LOG (CHANGE_VERSION, DESCRIPTION, EXPECTED_ROWS, ROWS_AFFECTED)
    VALUES ('3', 'default calc status to ACTIVE', :expected_rows, :changed_rows);
    COMMIT;
    RETURN 'updated ' || :changed_rows;
END;
$$;
```

Read it as five rules, which every data-change migration follows: (1) count the rows the change should touch; (2) stop if the count is not what you expected; (3) the change statement is safe to repeat (it only touches rows still needing the change); (4) assert the affected rows equal the expectation, and roll back otherwise; (5) write one ledger row. After it is applied, you never edit it (fix forward).

Two honest weaknesses to write in your notes: the expected row count is a constant that differs per environment (NAIC INT had 689 rows on its first run, other environments will differ), and the ledger row is written only when something changed.

> **TO-CONFIRM [TC-G08]:** that Flyway's parser accepts an `EXECUTE IMMEDIATE $$ ... $$` block with semicolons inside. NAIC's engine is procedure-heavy, so this matters. How to check at lab time: Step 4.7. Fallback if the migration fails to parse: keep only the plain `UPDATE` and the ledger `INSERT` with the constants in the file, and run the count check by hand before migrating. You lose the in-file assertion; note that cost. Record the result in file 13.

### Step 4.7 Migrate V3, then prove it is idempotent

Run in: **Terminal** (in `flyway/`, environment loaded):

```
cd flyway
flyway info
flyway migrate
flyway info
cd ..
```

You should see: before `migrate`, V3 is `Pending`; after, it is `Success`.

Run in: **Snowsight worksheet**:

```sql
SELECT CALC_NAME, CALC_STATUS FROM SBX_INT.CALC_SUPPORT.CALCULATIONS WHERE CALC_NAME LIKE 'LEGACY%';
SELECT * FROM SBX_INT.CALC_SUPPORT.DATA_CHANGE_LOG;
```

You should see: both legacy rows now `ACTIVE`, and exactly one ledger row with `CHANGE_VERSION = 3`, `EXPECTED_ROWS = 2`, `ROWS_AFFECTED = 2`.

Now prove the "safe to repeat" claim without cheating Flyway. Run the same block by hand in a worksheet (copy the SQL, replace the placeholders with `SBX_INT` and `CALC_SUPPORT`).

You should see: `nothing to do`, no new ledger row. The `DATA_CHANGE_LOG` table is DCM's (Lab 3); the migration that writes to it is Flyway's. A Flyway file may write rows into a DCM-owned table. What it must never do is `CREATE`, `ALTER` or `DROP` that table.

### Part C: a rename, the case where the tools must cooperate (about 50 min)

### Step 4.8 What DCM says when asked to define a Flyway table (Mode H only, plan only)

Run in: **Terminal**. Create a temporary file `dcm/calc_engine/sources/definitions/tables/CALC_AUDIT_EVENT.sql` that defines the live hybrid table as a standard `DEFINE TABLE` with the same columns (no `INDEX` clause is possible). Then plan:

```
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

**Do not deploy.** Delete the temporary file straight afterwards.

You should see: one of three outcomes: an adoption-style ALTER, an error that the object is a hybrid table, or an ALTER that would change its type. Write down which. A deploy of this could convert or recreate a table that holds audit history, and hybrid tables cannot be undropped.

> **TO-CONFIRM [TC-G07]:** `DEFINE TABLE` on an existing hybrid table is not covered by the docs (hybrid tables are not in the supported list). How to check at lab time: the plan above. Record the outcome in file 13.

#### If you are in Mode S (no hybrid tables)

Skip this step. You will see the same ownership clash with standard tables in Step 4.11.

### Step 4.9 The coordinated rename: `FORMULA_CATEGORIES` becomes `CALC_CATEGORIES`

This is the NAIC six-step rename protocol (architecture page, section on DCM and Flyway ownership), in sandbox size. The reason it exists: DCM reads a rename as drop plus create, which loses the data (Lab 3, Step 3.19), while Flyway can do an in-place `ALTER TABLE ... RENAME TO`.

Learner action (git, you run it): start a branch, for example `git switch -c feature/LAB4A`. The whole protocol is one change on one branch: the Flyway rename and the DCM definition updates together, so a reviewer sees both.

**Protocol step 1: write both halves.** Run in: **Terminal**.

1. Rename the DCM file to `dcm/calc_engine/sources/definitions/tables/CALC_CATEGORIES.sql` (use `git mv`; you run git) and change its `DEFINE TABLE` line to the new name. Keep every column.
2. In `procedures/SP_ADD_CATEGORY.sql` replace the two occurrences of `FORMULA_CATEGORIES` with `CALC_CATEGORIES`.
3. In `grants/access.sql` replace the app-role and reader-role lines that mention `FORMULA_CATEGORIES` with `CALC_CATEGORIES`.
4. Create `flyway/migrations/common/V4__rename_formula_categories_to_calc_categories.sql`:

```sql
ALTER TABLE ${calc_db}.${calc_schema}.FORMULA_CATEGORIES RENAME TO ${calc_db}.${calc_schema}.CALC_CATEGORIES;
```

**Protocol step 2: read the unsafe plan against the state before the rename.** Do not migrate yet.

```
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
cp dcm/calc_engine/out/plan_result.json dcm/calc_engine/out/plan_unsafe_rename.json
```

You should see: a DROP of `FORMULA_CATEGORIES`, a CREATE of `CALC_CATEGORIES`, an ALTER of `SP_ADD_CATEGORY`, and grant changes. This is the "unsafe remove/add interpretation". If you deployed it, the two category rows would be gone. Rule: reject any plan that contains a DROP of a table that has data.

> **TO-CONFIRM [TC-G06]:** this is the experiment that proves the rename reading in the main project; the scratch version was Lab 3, Step 3.19. Record the plan text in file 13.

**Protocol step 3: capture evidence.** Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SELECT COUNT(*) AS CATEGORIES_N, HASH_AGG(*) AS CATEGORIES_HASH FROM SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES;
SHOW GRANTS ON TABLE SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES;
SHOW TABLES LIKE 'FORMULA_CATEGORIES' IN SCHEMA SBX_INT.CALC_SUPPORT;
SELECT PROCEDURE_NAME FROM SBX_INT.INFORMATION_SCHEMA.PROCEDURES
 WHERE PROCEDURE_SCHEMA = 'CALC_SUPPORT' AND PROCEDURE_DEFINITION ILIKE '%FORMULA_CATEGORIES%';
```

You should see: your recorded count and hash, the grants, the owner `SBX_CICD_PIPELINE`, and `SP_ADD_CATEGORY` as the one consumer.

**Protocol step 4: apply the rename through Flyway only.** Run in: **Terminal**:

```
cd flyway
flyway validate
flyway migrate
flyway info
cd ..
```

You should see: V4 `Success`. The owner of the table does not change.

Now look at the window between the two tools. Run in: **Snowsight worksheet**:

```sql
CALL SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY('RATIO', CURRENT_USER());
```

You should see: an error saying the table `FORMULA_CATEGORIES` does not exist. The procedure body still refers to the old name until DCM redeploys it. In a pipeline the two steps run back to back, but the window is real. Larger systems avoid it with expand and contract: add the new name, move consumers, then remove the old one. The NAIC architecture lists that approach for incompatible changes.

**Protocol step 5: a fresh full plan against the final definitions.**

```
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: an ALTER on `SP_ADD_CATEGORY` (the body changed), adoption-style entries for `CALC_CATEGORIES` and the grants, and **no DROP and no CREATE of `CALC_CATEGORIES`**. Reject the plan if you see either.

> **TO-CONFIRM [TC-06-03]:** how DCM describes the old entity `FORMULA_CATEGORIES` after an out-of-band rename. The docs say an object dropped by hand disappears from `SHOW ENTITIES`, which suggests no DROP entry and a clean adoption of the new name. How to check at lab time: the plan above and `SHOW ENTITIES`. Record the result in file 13. If the plan proposes anything destructive, stop and read the entries.

**Protocol step 6: deploy DCM and require an empty re-plan.**

```
snow dcm deploy --target INT --from dcm/calc_engine --alias int-rename-categories -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

Run in: **Snowsight worksheet**:

```sql
SELECT COUNT(*) AS CATEGORIES_N, HASH_AGG(*) AS CATEGORIES_HASH FROM SBX_INT.CALC_SUPPORT.CALC_CATEGORIES;
CALL SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY('RATIO', CURRENT_USER());
SHOW GRANTS ON TABLE SBX_INT.CALC_SUPPORT.CALC_CATEGORIES;
```

You should see: `EMPTY changeset`; the same count and hash as before the rename; the procedure returns `OK: RATIO`; the grants are present.

Learner action (git): commit the branch with a message such as `feat: rename formula_categories to calc_categories (flyway V4 + dcm)`. The protocol is complete only when both halves are in one change set.

#### If DCM is not available (Mode D-off)

Do the Flyway part (V4) as written. Replace the DCM parts with `CREATE OR REPLACE PROCEDURE` of the edited `SP_ADD_CATEGORY` body in a worksheet, and read Steps 2, 5 and 6 as explanation.

> **Source note:** the six-step rename protocol comes from the NAIC architecture page `calc-engine-complete-architecture.html` (section on deployment ownership). It is NAIC's design, not a Snowflake document, so it carries no VERIFIED tag.

### Part D: two owners for one object (about 40 min)

### Step 4.10 Set up an experiment schema and see why DCM goes first

Run in: **Snowsight worksheet**, role `SYSADMIN` (or `ACCOUNTADMIN` for this one statement; switch back afterwards). Schemas belong to the platform lane, so the pipeline role cannot create them.

```sql
USE ROLE SYSADMIN;
CREATE SCHEMA IF NOT EXISTS SBX_INT.LAB4_CONFLICT;
GRANT OWNERSHIP ON SCHEMA SBX_INT.LAB4_CONFLICT TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
USE ROLE SBX_CICD_PIPELINE;
```

Now the ordering demo, in the same worksheet:

```sql
CREATE TABLE SBX_INT.LAB4_CONFLICT.ORDER_DEMO (
    ID NUMBER(38,0) DEFAULT SBX_INT.LAB4_CONFLICT.SEQ_NOT_YET.NEXTVAL,
    LABEL VARCHAR(20)
);
```

You should see: an error that the sequence does not exist. A table that points at a sequence needs the sequence first. In the engine, DCM creates sequences and Flyway creates the hybrid tables that use them, so a normal change runs DCM first and Flyway second. A rename flips this (Step 4.9) because the Flyway rename changes what DCM must see.

### Step 4.11 The conflict experiment: both tools claim one table

Scenario: a developer adds a table to the DCM project, and another developer, unaware, writes a Flyway migration for the same table.

**Part 1: DCM defines it.** Run in: **Terminal** (repository root). Create `dcm/lab4_conflict/manifest.yml`:

```yaml
manifest_version: 2
type: DCM_PROJECT
default_target: 'CONFLICT'
targets:
  CONFLICT:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_INT.LAB4_CONFLICT.CONFLICT_DCM'
    project_owner: SBX_CICD_PIPELINE
    templating_config: 'CONFLICT'
templating:
  defaults:
    db: 'SBX_INT'
  configurations:
    CONFLICT: { db: 'SBX_INT' }
```

and `dcm/lab4_conflict/sources/definitions/shared_thing.sql`:

```sql
DEFINE TABLE {{db}}.LAB4_CONFLICT.SHARED_THING (
    ID NUMBER(38,0) NOT NULL,
    LABEL VARCHAR(20),
    primary key (ID)
);
```

```
snow dcm create --target CONFLICT --from dcm/lab4_conflict --if-not-exists -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm deploy --target CONFLICT --from dcm/lab4_conflict --alias conflict-v1 -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
```

Run in: **Snowsight worksheet**: `INSERT INTO SBX_INT.LAB4_CONFLICT.SHARED_THING VALUES (1, 'one'), (2, 'two');`

**Part 2: Flyway also creates it.** Create `flyway/lab4_conflict/flyway.toml` (same environment as the main project, its own history table and schema), and `migrations/V1__create_shared_thing.sql` beside it:

```toml
[environments.target]
url = "jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE"
user = "SBX_SVC_CICD"

[flyway]
environment = "target"
locations = ["filesystem:migrations"]
defaultSchema = "LAB4_CONFLICT"
schemas = ["LAB4_CONFLICT"]
table = "FLYWAY_HISTORY_CONFLICT"
cleanDisabled = true
createSchemas = false
```

```sql
CREATE TABLE SBX_INT.LAB4_CONFLICT.SHARED_THING (
    ID NUMBER(38,0),
    LABEL VARCHAR(20),
    EXTRA VARCHAR(20)
);
```

Run in: **Terminal** (the same shell where you loaded `.env.flyway`; set `FLYWAY_LOCATIONS=filesystem:migrations` for this run, because the variable from the main project would point to other folders):

```
cd flyway/lab4_conflict
FLYWAY_LOCATIONS=filesystem:migrations flyway migrate
FLYWAY_LOCATIONS=filesystem:migrations flyway info
```

You should see: the migration fails because the table already exists, and the history shows version 1 as `Failed`. This is the lucky outcome: a plain `CREATE TABLE` is a safe collision.

**Part 3: the "fix" that causes the damage.** The developer sees "already exists" and edits the file to `CREATE OR REPLACE TABLE`, which is the usual shortcut. Because the failed version is not applied, Flyway lets you repair and rerun:

```
FLYWAY_LOCATIONS=filesystem:migrations flyway repair
# edit migrations/V1__create_shared_thing.sql: CREATE TABLE -> CREATE OR REPLACE TABLE
FLYWAY_LOCATIONS=filesystem:migrations flyway migrate
cd ../..
```

Run in: **Snowsight worksheet**:

```sql
SELECT * FROM SBX_INT.LAB4_CONFLICT.SHARED_THING;
DESC TABLE SBX_INT.LAB4_CONFLICT.SHARED_THING;
```

You should see: zero rows (the two rows are gone) and an extra column `EXTRA`. Flyway reports success. The data was destroyed silently and the table no longer matches DCM's definition.

**Part 4: the tools fight.** Run in: **Terminal**:

```
snow dcm plan --target CONFLICT --from dcm/lab4_conflict --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/lab4_conflict/out/plan_result.json
```

You should see: an ALTER on `SHARED_THING` that wants to remove `EXTRA` and restore the DCM shape. If you deployed it, the table would match DCM, the Flyway history would still claim that V1 created `EXTRA`, and `flyway validate` would still pass because it compares file checksums, not tables. Neither tool can see the disagreement.

> **TO-CONFIRM [TC-06-04]:** the exact states reported at each part: the status of the failed V1 in `flyway info`, the `CREATE OR REPLACE` outcome on a table that DCM manages (ownership and management are expected to stay with the project), and the plan text. Record the results in file 13.

**The cure** is organisational, not technical: pick one owner. The table is a standard table, so it belongs to DCM. Delete the Flyway file. If a Flyway migration needs to write rows into a DCM table, it may (as V3 does with `DATA_CHANGE_LOG`), but it may never create, alter, replace or drop it. A cheap guard in CI is a script that lists every object mentioned in `CREATE|ALTER|DROP` statements in `flyway/migrations` and fails if that object is in the DCM definitions. NAIC's architecture calls the matching idea `ownership.yml` ("exact owner/tool/role per object or prefix"); the POC branch has none yet. The sandbox's equivalent is the decision table in Step 4.12.

#### If you are in Mode H (hybrid tables)

Also run the mirror image: ask DCM to define a table that Flyway owns (done in Step 4.8). The point is symmetrical: a table has a single definition, in a single tool.

### Step 4.12 The ordering rules and the decision table

Run in: **Snowsight worksheet**: clean the experiment first.

```sql
DROP DCM PROJECT IF EXISTS SBX_INT.LAB4_CONFLICT.CONFLICT_DCM;
DROP SCHEMA IF EXISTS SBX_INT.LAB4_CONFLICT;
```

Copy this table into your notes (and later into `docs/learning/` as your own sandbox ownership map). Fill the last column from what you did in this lab.

| Change | Owner tool | Order | Why | Where you did it |
|---|---|---|---|---|
| New schema, database, role, warehouse, user, resource monitor | Platform lane (Terraform in NAIC, bootstrap SQL here) | before everything | containers and identities must exist first; DCM cannot manage users or monitors | file 02 |
| New standard table, new column on a standard table | DCM | DCM before Flyway | declarative, previewable; ALTER shown in the plan | Lab 3 |
| New sequence | DCM | DCM before Flyway | hybrid tables use them as defaults (Step 4.10) | Lab 3 |
| Edit a procedure, function or view body | DCM | any time | one ALTER per object in the plan | Lab 3, Step 3.11 |
| Grants on DCM objects | DCM (`access.sql`) | with the object | declared, reapplied on every deploy | Lab 3, Step 3.13 |
| New hybrid table, hybrid index, hybrid column | Flyway | after DCM | DCM cannot define them | Steps 4.2 and 4.3 |
| Grants on a Flyway table | Flyway (inside the migration) | with the table | the owner of the table owns its grants | Step 4.2 |
| Fix or backfill data | Flyway | after the schema change it depends on | ordered, checksummed, asserted, ledgered | Steps 4.6 and 4.7 |
| Rename a table or column | Flyway first, then DCM definition | Flyway, then DCM, then empty re-plan | DCM reads a rename as drop and create | Step 4.9 |
| Remove a table on purpose | DCM (delete the definition) | with an exact-FQN DROP approval and a backup | the plan shows a DROP, so CI must demand approval | Lab 3, Step 3.7 |
| Clone-support procedures | `snow sql -f` in the sandbox (the NAIC architecture places clone-support under Flyway) | before the first clone | they must exist before any clone is created | Lab 5 |
| The Flyway history table | Flyway (created by `migrate`) | automatic | never define it in DCM | Step 4.3 |

Ordering in one line: **normal change** is DCM plan, review, DCM deploy, Flyway migrate, DCM full re-plan empty. **Rename change** is Flyway rename, DCM definition update, full plan with no DROP, DCM deploy, empty re-plan.

## 6. Checkpoint questions

1. Why does a Flyway-owned table in the same schema not get dropped by a DCM deploy?
2. Why must a normal change run DCM before Flyway?
3. What does DCM's plan show for a renamed table, and what would deploying it do to the data?
4. In the rename protocol, why is the DCM full plan taken twice?
5. Which three properties make V3 a safe data change?
6. In the conflict experiment, why did `flyway validate` still pass after DCM changed the table?
7. Who owns the grants on `CALC_AUDIT_EVENT`, and who owns the grants on `CALCULATIONS`?
8. Which parts of this lab differ between Mode H and Mode S, and what do you lose in Mode S?

<details>
<summary>Answers</summary>

1. DCM only drops objects it manages, and it manages only what it defined (Step 4.4).
2. The Flyway tables use sequences that DCM creates (Step 4.10).
3. A DROP of the old name and a CREATE of the new name. Deploying it would give an empty new table; the data is gone.
4. First against the pre-migration state, to prove the unsafe reading and reject it; second after the Flyway rename, against the final definitions, to confirm there is no DROP or CREATE and then deploy.
5. The change touches only rows that still need it (idempotent), it asserts the expected row count and rolls back otherwise, and it leaves a ledger row. It is also never edited after being applied.
6. Flyway compares file checksums with its history, not tables with files.
7. Flyway owns the first (inside V2); DCM owns the second (`access.sql`).
8. Mode S uses the `standard` folder, has no indexes, does not enforce the primary key, and its locking behaviour cannot be shown. You still learn the ownership and ordering rules, which do not depend on the table type.

</details>

## 7. Break it

Scenario: the data fix finds a different number of rows from the one the author expected.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
INSERT INTO SBX_INT.CALC_SUPPORT.CALCULATIONS (CALC_NAME, CALC_STATUS) VALUES ('LEGACY_C', NULL), ('LEGACY_D', NULL), ('LEGACY_E', NULL);
```

Run the V3 block by hand again (copy it from the file, replace the placeholders with `SBX_INT` and `CALC_SUPPORT`).

Symptom: the block ends with an error `V3 assertion failed: unexpected number of rows`. Nothing was updated.
Diagnosis: three rows match, the author expected two. The pre-check refused to guess.
Recovery: look at the three rows. Either the new rows are wanted (then write a new migration `V3_1__...` or a new version with the correct expected number and its own ledger row) or they are mistakes (delete them). Never edit V3: it is applied, and Flyway would fail `validate` with a checksum mismatch. Then `DELETE FROM SBX_INT.CALC_SUPPORT.CALCULATIONS WHERE CALC_NAME IN ('LEGACY_C','LEGACY_D','LEGACY_E');`.

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| DCM cannot define hybrid tables | Step 4.8 (and Lab 3, Step 3.16) | Flyway | VERIFIED (supported list); outcome of `DEFINE TABLE` on a hybrid table TO-CONFIRM TC-G07 |
| DCM cannot rename | Step 4.9, protocol step 2 | Flyway rename plus coordinated DCM change | TO-CONFIRM TC-G06 |
| Flyway has no preview, no drift detection and no undo in Community | Step 4.11 (the disagreement stays invisible) | DCM plan for DCM objects; evidence queries and clones for Flyway objects | VERIFIED (feature summary) |
| Two owners silently destroy data | Step 4.11 | One owner per object, an ownership map, a CI guard | observed |
| A rename leaves a window where consumers are broken | Step 4.9 | Run both steps in one pipeline run; expand and contract for real systems | observed |
| Data-fix expected counts differ per environment | Step 4.6 | Compute expectations per environment or parameterise carefully | observed |
| The hybrid pair's DDL auto-commits, so a failure half-way leaves partial objects | Lab 2 (and Lab 8 drill) | Small migrations, manual cleanup, fix forward | VERIFIED (driver page) |
| Mode S cannot show enforced keys, indexes or locking | Step 4.5 | Upgrade the account, or read the docs | VERIFIED (trial page) |
| The `$$` parser risk in Flyway | Step 4.6 | Plain-statement fallback | TO-CONFIRM TC-G08 |

## 9. What you learned

- Every object has one owner tool, and an ownership map makes it checkable.
- DCM and Flyway coexist in one schema because DCM only manages what it defined and Flyway only runs its own files.
- A normal change runs DCM, then Flyway, then an empty DCM re-plan. A rename runs Flyway first, and the unsafe DCM plan is read and rejected before it.
- A data change is safe when it is idempotent, asserted and ledgered, and it is never edited after it is applied.
- When two tools own one object they destroy data silently and neither can see it.

## 10. Cleanup

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`), if not already done in Step 4.12:

```sql
DROP DCM PROJECT IF EXISTS SBX_INT.LAB4_CONFLICT.CONFLICT_DCM;
DROP SCHEMA IF EXISTS SBX_INT.LAB4_CONFLICT;
DELETE FROM SBX_INT.CALC_SUPPORT.CALC_WORKFLOW_STATUS;
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_AUDIT_EVENT;
```

Run in: **Terminal**: `rm -rf dcm/lab4_conflict flyway/lab4_conflict dcm/calc_engine/out`.

Keep: the hybrid pair or stand-ins, the history at version 4, the renamed category table, the DCM project and its definitions. Cost note: hybrid tables add storage and use the warehouse only when you run statements; nothing here runs while the warehouse is suspended.

Learner action (git): the repository now holds `flyway/**` and the renamed DCM files; merge your branch the way you prefer. Make sure `flyway/.env.flyway` is not staged (it is gitignored).

## 11. State left behind

- Flyway history `SBX_INT.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT` is at version 4 (V1, V2, V3, V4 applied).
- Hybrid pair `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT` exist (Mode H) or their standard stand-ins (Mode S), both empty, with grants: app role SELECT, INSERT, UPDATE, DELETE on the first and only SELECT, INSERT on the second; reader SELECT on both.
- `DATA_CHANGE_LOG` has the ledger row for version 3.
- `FORMULA_CATEGORIES` is now `CALC_CATEGORIES`; the DCM definition file is `tables/CALC_CATEGORIES.sql`; `SP_ADD_CATEGORY` and `access.sql` use the new name; the DCM deployment `int-rename-categories` is done and the re-plan is empty.
- The conflict experiment (`SBX_INT.LAB4_CONFLICT`, `dcm/lab4_conflict`, `flyway/lab4_conflict`) is removed.
- Your notes contain the ownership decision table from Step 4.12.
- `V5__add_audit_event_type_index.sql` is **not** applied yet (Lab 7 does it).

## 12. Open items for file 13

TC-G06, TC-G07, TC-G08, TC-G14, TC-G21, TC-06-01 (env var overrides the TOML environment), TC-06-02 (duplicate-key error text), TC-06-03 (DCM view of the old entity after an out-of-band rename), TC-06-04 (states in the conflict experiment). New names introduced in this file for the contract: schema `SBX_INT.LAB4_CONFLICT`, folders `dcm/lab4_conflict` and `flyway/lab4_conflict`, project `CONFLICT_DCM`, Flyway history `FLYWAY_HISTORY_CONFLICT`, plus the helper `scripts/plan_summary.py` from Lab 3.

## 13. Next

Continue with [07-lab5-clones-and-ticket-environments.md](07-lab5-clones-and-ticket-environments.md).

