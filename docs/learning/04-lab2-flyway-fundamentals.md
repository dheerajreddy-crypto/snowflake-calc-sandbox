# Lab 2 (file 04): Flyway fundamentals on Snowflake

> Lab no: 2 | File no: 04 | Time: 2.5 to 3 hours | Prerequisites: file 03 (for the baseline part), Flyway runner from file 02 | Modes supported: H/S, D (Flyway works on standard tables in every mode) | Leaves behind: `SBX_INT.LAB2_SCRATCH` with Flyway history and mini objects (may be cleaned), `SBX_INT.LAB2_BASELINE` baselined at version 1 then dropped, the main `flyway/flyway.toml` written but not yet migrated.

## 1. Goal

Learn Flyway from nothing: configuration, versioned (`V`) and repeatable (`R__`) migrations, the history table, checksums, the commands `info`, `validate`, `migrate`, `baseline`, `repair`, `clean`, out-of-order, placeholders, a migration that fails half way (Snowflake DDL auto-commit), fix-forward, and baselining an existing database (your Lab 1 objects). Then expose Flyway's limits with exercises.

## 2. Why this matters in calc-engine and calc-admin

> **Real world.** Flyway is the planned tool for the Calc Admin lifecycle schema. The two hybrid tables `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT` exist in INT, created by hand on 2026-11-03, with no scripts. The design: scripts V001 to V006 (sequence, tables, indexes, grants); INT is **baselined** at V006 after verifying the live shape equals the scripts; QA and PROD run the scripts for real. Flyway never runs at app startup in the target (`calc-admin-with-flyway.html`). EDP already runs Flyway for `NAIC_<tier>_FINANCIAL_EDL.FIN_CALCULATION` with history table `flyway_bus_history_table`. The calc-engine branch even hand-built a mini Flyway: `data_changes/V<timestamp>__*.sql` with a ledger `DATA_CHANGE_LOG` and a SHA-256 checksum, which the architecture moves into the Flyway lane. This lab teaches what that mini Flyway was copying.

## 3. Prerequisites and state check

Run in: **Terminal** (repository folder, venv active) and **Snowsight worksheet**.

```bash
snow connection test -c sbx_owner
docker run --rm flyway/flyway:13.12.2 -v     # confirmed on Apple Silicon, see TC-02-04
```

```sql
USE ROLE SBX_CICD_PIPELINE;
SHOW TABLES IN SCHEMA SBX_INT.CALC_SUPPORT;       -- expect CALCULATIONS, FORMULAS, FORMULA_CATEGORIES from Lab 1
SHOW SCHEMAS LIKE 'LAB2%' IN DATABASE SBX_INT;     -- expect LAB2_SCRATCH
```

You should see: Flyway prints version 13.12.2; the three Lab 1 tables exist; `LAB2_SCRATCH` exists. If the Lab 1 tables are missing you can still do everything except Step 5.11 (baseline); do Lab 1 first.

The helper `fw` is defined in file 02. Repeat it in the shell you use now. Set the connection URL once per shell session (the account identifier stays in your shell, never in a committed file) and the passphrase through a prompt that does not echo:

```bash
export FLYWAY_URL="jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE&authenticator=snowflake_jwt&private_key_file=/keys/sbx_svc_cicd_rsa_key.p8"
read -s PRIVATE_KEY_PASSPHRASE && export PRIVATE_KEY_PASSPHRASE
```

(`/keys/...` is the path inside the Docker container; `fw` mounts your key folder read-only.)

## 4. Concepts

- **Migration.** A SQL file that changes the database one step, applied once and in order. Flyway keeps the list of applied steps in a table.
- **Versioned migration.** File name `V<version>__<description>.sql` (two underscores). Applied exactly once, in version order, then its **checksum** (a fingerprint of the file text) is stored. Versions can use dots or underscores: `V3_1` is version 3.1.
- **Repeatable migration.** File name `R__<description>.sql`. Re-applied whenever its checksum changes, after all pending versioned migrations. Good for views and procedures, whose definition you want to keep as one current file.
- **History table.** Flyway's own table, `flyway_schema_history` by default, created by `migrate` in the default schema. One row per applied migration with version, description, checksum, who, when and success. Snowflake stores the name in upper case.
- **Checksum validation.** Before migrating, Flyway compares the checksum of each applied file with the file on disk. A mismatch is an error. That is how it protects "applied versions are immutable".
- **Fix-forward.** To change something that was already applied, you add a new migration. You do not edit the old file.
- **Baseline.** Tells Flyway "this database already has everything up to version N". Used to adopt an existing database.
- **Placeholders.** `${name}` tokens in SQL files, replaced by values from configuration.
- **Snowflake auto-commits DDL.** Each `CREATE`/`ALTER` commits on its own, so a file that fails halfway leaves its earlier statements applied.

> **VERIFIED:** Flyway Community includes migrate, info, validate, baseline, repair, clean, versioned and repeatable migrations, and placeholders. Undo migrations and dry run need Teams; drift detection needs Enterprise (https://documentation.red-gate.com/fd/feature-summary-165740620.html and https://documentation.red-gate.com/flyway/reference/commands, read 2026-11-16).

> **VERIFIED:** since Flyway 10 the default configuration format is TOML (`flyway.toml`); the old `flyway.conf` is read only if no TOML file is found, and the two formats cannot be mixed (https://www.red-gate.com/hub/product-learning/flyway/using-toml-configuration-files-with-flyway, read 2026-11-16).

> **VERIFIED:** the Snowflake JDBC URL needs `db` and `warehouse`; Flyway commits no DDL rollback on Snowflake, so a half-failed migration leaves earlier statements in place and you clean them up by hand (https://documentation.red-gate.com/flyway/reference/database-driver-reference/snowflake, read 2026-11-16).

## 5. Step-by-step

### Step 5.1 Create the scratch project

Run in: **Terminal**. Create `flyway/lab2_scratch/flyway.toml`:

```toml
[environments.target]
url = "jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE"
user = "SBX_SVC_CICD"
# The private key path and your real account come from the FLYWAY_URL variable you exported above.
# Never write a key, passphrase or your real account identifier into this file.

[flyway]
environment = "target"
locations = ["filesystem:migrations"]
schemas = ["LAB2_SCRATCH"]
defaultSchema = "LAB2_SCRATCH"
createSchemas = false
# table: left at its default (flyway_schema_history) on purpose, to show the default
# cleanDisabled is NOT set here: its default is true

[flyway.placeholders]
calc_db = "SBX_INT"
```

For reference only, the same settings in the older `flyway.conf` format (do not create this file; the two formats cannot be mixed):

```text
flyway.url=jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE
flyway.user=SBX_SVC_CICD
flyway.locations=filesystem:migrations
flyway.schemas=LAB2_SCRATCH
flyway.defaultSchema=LAB2_SCRATCH
flyway.placeholders.calc_db=SBX_INT
```

> **TO-CONFIRM [TC-04-01]:** that the environment variable `FLYWAY_URL` overrides `[environments.target] url` in a TOML file. The docs say environment variables override files, and the pack's contract relies on `FLYWAY_URL`. How to check: run Step 5.2; if Flyway reports it cannot connect to the placeholder account, pass the URL as a command-line parameter instead: `fw flyway/lab2_scratch -url="$FLYWAY_URL" info`. Record the working form in file 13.

> **TO-CONFIRM [TC-G10]:** per-environment placeholders in `flyway.toml`. An article from 2025 said they were not possible. The pack avoids the question: it uses `[flyway.placeholders]`, environment variables (`FLYWAY_PLACEHOLDERS_CALC_DB`) or `-placeholders.calc_db=...`. How to check: not needed in this lab.

Create the first migration `flyway/lab2_scratch/migrations/V1__create_mini_calculations.sql`:

```sql
CREATE TABLE ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI (
  CALC_ID     NUMBER(38,0) NOT NULL,
  CALC_NAME   VARCHAR(200) NOT NULL,
  CALC_STATUS VARCHAR(10)  DEFAULT 'DRAFT',
  PRIMARY KEY (CALC_ID)
);

INSERT INTO ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI (CALC_ID, CALC_NAME)
VALUES (1, 'MINI_ONE'), (2, 'MINI_TWO');
```

You should see: two files. Nothing has run.

### Step 5.2 `info` before anything is applied

Run in: **Terminal**, repository root.

```bash
fw flyway/lab2_scratch info
```

You should see: a connection to `SBX_INT` as `SBX_SVC_CICD`, and a table listing `V1 create mini calculations` with state **Pending**. No history table exists yet; `info` does not create one.

> **TO-CONFIRM [TC-04-02]:** the exact columns and wording of `flyway info` on Snowflake. How to check: the output. Record a copy in file 13.

If you see a key-decryption error, see TC-G21 in file 02 (JVM flag). If you see an authentication error with code 390144, see file 12.

### Step 5.3 First `migrate` and reading the history table

Run in: **Terminal**.

```bash
fw flyway/lab2_scratch migrate
fw flyway/lab2_scratch info
```

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`.

```sql
SHOW TABLES IN SCHEMA SBX_INT.LAB2_SCRATCH;
SELECT * FROM SBX_INT.LAB2_SCRATCH.CALCULATIONS_MINI;
SELECT * FROM SBX_INT.LAB2_SCRATCH."flyway_schema_history";   -- or FLYWAY_SCHEMA_HISTORY; see TC-04-03
```

You should see: `V1` state **Success** in `info`; tables `CALCULATIONS_MINI` and the history table in `LAB2_SCRATCH`; two data rows; one history row (plus possibly a schema-creation row) with `version` 1, a numeric `checksum`, `installed_by` `SBX_SVC_CICD`, and `success` true.

> **TO-CONFIRM [TC-04-03]:** whether Snowflake shows the history table as `FLYWAY_SCHEMA_HISTORY` or `flyway_schema_history` (quoted lower case). How to check: the `SHOW TABLES` output. Record in file 13. The pack's contract assumes upper case.

Write down the history `checksum` for V1. You need it in Step 5.6.

### Step 5.4 A second versioned migration and a placeholder

Run in: **Terminal**. Create `flyway/lab2_scratch/migrations/V2__add_notes_column.sql`:

```sql
ALTER TABLE ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI ADD COLUMN NOTES VARCHAR(500);
```

```bash
fw flyway/lab2_scratch migrate
```

You should see: only `V2` applied (V1 is skipped because it is already in the history), and `NOTES` added to the table (`SHOW COLUMNS IN TABLE SBX_INT.LAB2_SCRATCH.CALCULATIONS_MINI;`). The text `${calc_db}` in your file was replaced by `SBX_INT` from the configuration. Placeholders can come from the config file, from `-placeholders.<name>=...` on the command line or from `FLYWAY_PLACEHOLDERS_<NAME>` in the environment (later sources win).

### Step 5.5 A repeatable migration

Run in: **Terminal**. Create `flyway/lab2_scratch/migrations/R__calc_summary_view.sql`:

```sql
CREATE OR REPLACE VIEW ${calc_db}.LAB2_SCRATCH.CALC_SUMMARY AS
SELECT CALC_STATUS, COUNT(*) AS CALC_COUNT
  FROM ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI
 GROUP BY CALC_STATUS;
```

```bash
fw flyway/lab2_scratch migrate     # applies the R__ file
fw flyway/lab2_scratch migrate     # nothing to do
```

Now edit the view file (add `, MAX(CALC_ID) AS LAST_CALC_ID` to the select list) and run `migrate` again.

You should see: the first run applies the repeatable (`info` shows it as `Repeatable`); the second run says the schema is up to date; after your edit the repeatable is applied again because its checksum changed. Repeatables have no version, so they are re-applied, never skipped as "older".

### Step 5.6 Checksums: edit an applied migration

Run in: **Terminal**. Open `V1__create_mini_calculations.sql` and add a comment line at the top (`-- edited after the fact`). Then:

```bash
fw flyway/lab2_scratch validate
fw flyway/lab2_scratch migrate
```

You should see: `validate` fails and says a checksum mismatch was detected for migration version 1 (applied checksum versus resolved checksum), and `migrate` refuses to run because `validateOnMigrate` is on by default. Even a comment changes the checksum.

Two ways out. Do **A** now:

- **A. Revert the file** to its exact original text. `validate` passes again. This is the intended path.
- **B. `repair`** realigns the stored checksum to the edited file: `fw flyway/lab2_scratch repair`. It works, and that is the danger. In a team, `repair` makes the history say whatever the files say now, so the protection disappears. NAIC's rules: no automatic repair, manual repair only with approval.

> **VERIFIED:** `validate` compares applied migrations with available ones and reports checksum mismatches and missing migrations; `repair` removes failed entries, realigns checksums, descriptions and types with the available files, and marks missing migrations as deleted; it must be given the same locations as `migrate` (https://documentation.red-gate.com/flyway/reference/commands/repair, read 2026-11-16).

### Step 5.7 A migration that fails half way

Run in: **Terminal**. Create `flyway/lab2_scratch/migrations/V3__bad_migration_half_fails.sql`:

```sql
CREATE TABLE ${calc_db}.LAB2_SCRATCH.LAB2_PARTIAL (ID NUMBER);
ALTER TABLE ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI ADD COLUMN NOTES VARCHAR(500);   -- fails: NOTES already exists from V2
```

```bash
fw flyway/lab2_scratch migrate
fw flyway/lab2_scratch info
```

Run in: **Snowsight worksheet**.

```sql
SHOW TABLES LIKE 'LAB2_PARTIAL' IN SCHEMA SBX_INT.LAB2_SCRATCH;
```

You should see: `migrate` fails on the second statement with an error that the column already exists. `info` shows `V3` as **Failed**. **And the table `LAB2_PARTIAL` exists**: Snowflake committed the first statement before the second one failed. This is the proof that Flyway cannot roll back a half-failed Snowflake migration.

Run `migrate` again. You should see: it refuses because of the failed entry in the history (the wording is not recorded here; note it).

Recovery (do these in order):

```sql
DROP TABLE IF EXISTS SBX_INT.LAB2_SCRATCH.LAB2_PARTIAL;
```

```bash
rm flyway/lab2_scratch/migrations/V3__bad_migration_half_fails.sql
fw flyway/lab2_scratch repair
fw flyway/lab2_scratch info
```

You should see: `repair` removes the failed row; `info` shows V1, V2 and the repeatable as applied and no V3. Deleting the bad file is allowed here because it **never succeeded**. The rule "never edit an applied file" applies to applied files only.

Now make the change you actually wanted, as a new migration (**fix-forward**). Suppose 500 characters turned out too short for notes. Create `flyway/lab2_scratch/migrations/V3_1__fix_forward_calc_notes.sql`:

```sql
ALTER TABLE ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI ALTER COLUMN NOTES SET DATA TYPE VARCHAR(2000);
```

```bash
fw flyway/lab2_scratch migrate
```

You should see: version 3.1 applied. You did not touch V2; the history shows the real sequence of events.

> **Lesson in one line:** keep each migration to one DDL statement where you can. A one-statement file either applies or does not, and the half-applied state cannot happen.

> **VERIFIED:** the history entry of a failed migration is recorded as failed; Flyway's docs say to clean up partial changes by hand and use repair on databases without DDL transactions; undo migrations "do not help with failed versioned migrations" there (https://documentation.red-gate.com/flyway/flyway-concepts/migrations/undo-migrations, read 2026-11-16).

### Step 5.8 Out-of-order migrations

Run in: **Terminal**. Create a helper that jumps ahead, `flyway/lab2_scratch/migrations/V5__out_of_order_helper.sql`:

```sql
CREATE TABLE ${calc_db}.LAB2_SCRATCH.LAB2_HELPER (ID NUMBER);
```

```bash
fw flyway/lab2_scratch migrate
```

Now create the "late" file with a lower version, `flyway/lab2_scratch/migrations/V4__out_of_order_demo.sql` (this simulates a colleague's branch merged after yours):

```sql
CREATE TABLE ${calc_db}.LAB2_SCRATCH.LAB2_LATE (ID NUMBER);
```

```bash
fw flyway/lab2_scratch info
fw flyway/lab2_scratch migrate
fw flyway/lab2_scratch validate
```

You should see: `info` shows `V4` as **Ignored** (its version is lower than the highest applied, V5); `migrate` does not apply it; `validate` flags it (the exact message is not recorded; note it). With the default `outOfOrder=false` this is a deliberate stop: the order of application must equal the order of versions.

Now allow it, once:

```bash
export FLYWAY_OUT_OF_ORDER=true
fw flyway/lab2_scratch migrate
unset FLYWAY_OUT_OF_ORDER
fw flyway/lab2_scratch info
```

You should see: V4 applied, shown as **Out of Order** in `info`. In NAIC's target config `outOfOrder=false` is fixed. fpr-admin (Postgres) sets `out-of-order=true`, which is a convenience that hides ordering mistakes.

> **VERIFIED:** `outOfOrder` defaults to false; with versions 1 and 3 applied and 2 found later, `true` applies version 2 and `false` ignores it; the environment variable is `FLYWAY_OUT_OF_ORDER` (https://documentation.red-gate.com/flyway/reference/configuration/flyway-namespace/flyway-out-of-order-setting, read 2026-11-16).

### Step 5.9 A procedure body in a migration

Run in: **Terminal**. Create `flyway/lab2_scratch/migrations/R__sp_mini_count.sql`:

```sql
CREATE OR REPLACE PROCEDURE ${calc_db}.LAB2_SCRATCH.SP_MINI_COUNT()
RETURNS NUMBER
LANGUAGE SQL
AS
$$
DECLARE
    v_n NUMBER;
BEGIN
    SELECT COUNT(*) INTO :v_n FROM ${calc_db}.LAB2_SCRATCH.CALCULATIONS_MINI;
    RETURN v_n;
END;
$$;
```

```bash
fw flyway/lab2_scratch migrate
```

```sql
CALL SBX_INT.LAB2_SCRATCH.SP_MINI_COUNT();
```

> **TO-CONFIRM [TC-G08]:** whether Flyway parses a `$$ ... $$` body that contains semicolons and a Snowflake Scripting block correctly. This matters a great deal for NAIC, where almost everything is a procedure. How to check: this step. Record the result and any error text in file 13. If Flyway splits the body at the inner semicolons and fails, try the same content with the body on fewer lines, or note that procedure files may need a different handling approach.

You should see (if it parses): the repeatable applied and the call returns `2` (two seed rows). Whatever you see, write it down: this decides how comfortable Flyway is for procedure-heavy code.

### Step 5.10 `clean`: why it is disabled, and rebuilding from nothing

Run in: **Terminal**.

```bash
fw flyway/lab2_scratch clean
```

You should see: Flyway refuses because clean is disabled (the default). Only in this scratch sandbox, enable it for one command:

```bash
export FLYWAY_CLEAN_DISABLED=false
fw flyway/lab2_scratch clean
unset FLYWAY_CLEAN_DISABLED
```

Run in: **Snowsight worksheet**.

```sql
SHOW TABLES IN SCHEMA SBX_INT.LAB2_SCRATCH;
SHOW VIEWS IN SCHEMA SBX_INT.LAB2_SCRATCH;
```

You should see: no tables and no views. `clean` dropped **every object in the configured schemas**, including the history table. Now rebuild everything from the files:

```bash
fw flyway/lab2_scratch migrate
```

You should see: all versions applied in order, now V4 before V5 because the history is empty. A fresh build from files gives the same end state (apart from the order difference), which is the real value of migrations: the database can be rebuilt in a clone.

> **VERIFIED:** `clean` drops all objects in the configured schemas and schemas Flyway created; its documentation says never to use it against production; `cleanDisabled` defaults to true (https://documentation.red-gate.com/flyway/reference/commands/clean and https://documentation.red-gate.com/flyway/reference/configuration/flyway-namespace/flyway-clean-disabled-setting, read 2026-11-16).

> **Safety rule.** Keep `schemas` in the configuration to the scratch schema only. If you ever configured `CALC_SUPPORT` with clean enabled, `clean` would drop every engine object. In NAIC `cleanDisabled=true` is fixed in shared environments.

### Step 5.11 Baseline an existing database (your Lab 1 objects)

**Why.** `SBX_INT.CALC_SUPPORT` already has objects with no Flyway history, like INT's hand-made hybrid tables. You cannot run `V1__create...` on it because the table exists. You baseline instead. You practise on a zero-copy clone so the real Lab 1 objects stay untouched.

Run in: **Snowsight worksheet**, role `SBX_CICD_PIPELINE`.

```sql
USE ROLE SBX_CICD_PIPELINE;
CREATE SCHEMA SBX_INT.LAB2_BASELINE CLONE SBX_INT.CALC_SUPPORT;
SHOW TABLES IN SCHEMA SBX_INT.LAB2_BASELINE;
```

You should see: a schema with the same tables and the seed rows. (A clone of a schema without hybrid tables works; see Lab 5 for what changes with hybrid tables.)

Run in: **Terminal**. Create `flyway/lab2_scratch/baseline_migrations/V1__lab1_objects.sql` (the script that, in an ideal world, created the objects; it must not run now):

```sql
-- Represents everything that Lab 1 created by hand. Flyway must NEVER execute this on the baselined schema.
CREATE TABLE ${calc_db}.LAB2_BASELINE.CALCULATIONS (CALC_ID NUMBER(38,0) NOT NULL);
```

And `flyway/lab2_scratch/baseline_migrations/V2__add_approved_by.sql`:

```sql
ALTER TABLE ${calc_db}.LAB2_BASELINE.CALCULATIONS ADD COLUMN APPROVED_BY VARCHAR(255);
```

Point Flyway at the clone and the new folder for these commands only:

```bash
BASE_ARGS='-schemas=LAB2_BASELINE -defaultSchema=LAB2_BASELINE -locations=filesystem:baseline_migrations'
fw flyway/lab2_scratch $BASE_ARGS migrate
```

You should see: an error. Either Flyway refuses because the schema is not empty and has no history table, or (if it tried) V1 fails because the table already exists.

> **TO-CONFIRM [TC-04-04]:** the exact wording of that refusal. How to check: the output. Record in file 13. The pack expects a message that suggests `baseline` or `baselineOnMigrate`.

Now baseline, and then migrate:

```bash
fw flyway/lab2_scratch $BASE_ARGS baseline -baselineVersion=1 -baselineDescription="Lab 1 objects"
fw flyway/lab2_scratch $BASE_ARGS info
fw flyway/lab2_scratch $BASE_ARGS migrate
fw flyway/lab2_scratch $BASE_ARGS info
```

You should see: after `baseline`, `V1` shown as **Baseline** and `V2` as **Pending**; after `migrate`, `V2` applied and `V1` never executed (check that `CALCULATIONS` in the clone still has its original rows and gained `APPROVED_BY`). A history table `FLYWAY_SCHEMA_HISTORY` now exists in `SBX_INT.LAB2_BASELINE`.

> **TO-CONFIRM [TC-G09]:** the default value of `baselineVersion`. The pack passes `-baselineVersion=1` explicitly, so the lab works. How to check: the Flyway docs page for the setting, or run `baseline` without the flag in a throwaway schema. Record in file 13.

What can go wrong with a baseline: you baseline at a version while the live objects do **not** match the scripts up to that version. Flyway cannot check. That is why NAIC's design says verify the live shape equals the scripts first, and requires approval for a baseline (`baselineOnMigrate=false`). Setting `baselineOnMigrate=true` would baseline silently on the first `migrate` against any non-empty schema, which is convenient and dangerous.

### Step 5.12 Write the main project configuration (do not migrate yet)

Run in: **Terminal**. Create `flyway/flyway.toml`. This is the file Labs 4, 6, 7 and 8 use. It contains **no secrets**.

```toml
[environments.target]
url = "jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE"
user = "SBX_SVC_CICD"
# private key and url overrides come from environment variables / CLI parameters, never from this file

[flyway]
environment = "target"
locations = ["filesystem:migrations/common", "filesystem:migrations/standard"]   # Mode S. Mode H: migrations/hybrid
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

Also create `flyway/.env.flyway.example` (committed; the real `.env.flyway` is gitignored):

```text
FLYWAY_URL=jdbc:snowflake://<ACCOUNT_IDENTIFIER>.snowflakecomputing.com/?db=SBX_INT&warehouse=SBX_CALC_WH&role=SBX_CICD_PIPELINE&authenticator=snowflake_jwt&private_key_file=/keys/sbx_svc_cicd_rsa_key.p8
FLYWAY_LOCATIONS=filesystem:migrations/common,filesystem:migrations/standard
FLYWAY_PLACEHOLDERS_CALC_DB=SBX_INT
FLYWAY_PLACEHOLDERS_CALC_SCHEMA=CALC_SUPPORT
```

Compare each setting with the NAIC contract in file 01: `validateOnMigrate=true`, `outOfOrder=false`, `baselineOnMigrate=false`, `cleanDisabled=true`, a dedicated history table per boundary and environment. Do not run `migrate` with this file yet: Lab 4 does that after DCM owns the tables Flyway migrations depend on.

### Step 5.13 Limitations exercises

**L1. Flyway does not detect drift.** Run in **Snowsight worksheet**, then **Terminal**:

```sql
ALTER TABLE SBX_INT.LAB2_SCRATCH.CALCULATIONS_MINI ADD COLUMN SNEAKY VARCHAR(10);
```

```bash
fw flyway/lab2_scratch validate
fw flyway/lab2_scratch info
```

You should see: both succeed with no complaint. The history says everything matches, but the table has a column no migration created. Run `SHOW COLUMNS IN TABLE SBX_INT.LAB2_SCRATCH.CALCULATIONS_MINI;` to see it. Flyway compares **files with its history**, not files with the database. Drift detection exists only in the Enterprise edition (public preview for Snowflake).

**L2. Repeatables miss hand edits.** Run in **Snowsight worksheet**:

```sql
CREATE OR REPLACE VIEW SBX_INT.LAB2_SCRATCH.CALC_SUMMARY AS SELECT 'hand edit' AS WHO;
```

```bash
fw flyway/lab2_scratch migrate
```

You should see: "up to date". The hand edit survives because the repeatable file did not change, so Flyway never re-applies it. The live view is wrong until someone edits the file.

**L3. No preview.** Add a new `V6__x.sql` with `ALTER TABLE ... ADD COLUMN`. Run `fw flyway/lab2_scratch info`. You see **Pending** and a file name, not what the change will do to the live database. The dry-run feature is in Teams. Delete the file again.

**L4. Renames are manual.** Flyway has no concept of "rename". You write `ALTER TABLE <old> RENAME TO <new>` as a plain migration; nothing links it to any definition. You use this in Lab 4 where a declarative tool would otherwise drop and recreate.

**L5. No rollback in Community.** Run:

```bash
fw flyway/lab2_scratch undo
```

You should see: an error that the command is not available in this edition (the exact text is not recorded; note it). Undo needs Teams, and even there it does not repair a half-failed Snowflake migration. Recovery in Community is fix-forward, restoring from a clone, or Time Travel for data.

> **TO-CONFIRM [TC-G11]:** the flag name for JSON output (`-outputType=json`). It is not needed in this lab but Lab 6 uses it in CI. How to check: run `fw flyway/lab2_scratch -outputType=json info`. Record in file 13.

**L6. Procedure-heavy code.** Count what you did in Step 5.9: a procedure fits an `R__` file, but there is no check that the live procedure still matches the file. A codebase of 31 procedures with 6,700 lines is a lot of repeatable files with no plan output. This is where DCM (Lab 3) is stronger.

Remove the extra column when you are done with L1:

```sql
ALTER TABLE SBX_INT.LAB2_SCRATCH.CALCULATIONS_MINI DROP COLUMN SNEAKY;
```

## 6. Checkpoint questions

1. Why does a comment-only edit to an applied file make `validate` fail?
2. What is the difference between a versioned and a repeatable migration? When would you choose each?
3. After a migration fails half way on Snowflake, what is in the database, what is in the history table, and what do you do?
4. Why is `repair` risky as a routine tool?
5. `clean` is disabled by default. What did it do in Step 5.10, and why must it stay disabled in shared environments?
6. What does `baseline` do, and what can Flyway **not** check about it?
7. Name two things Flyway Community cannot tell you about the live database.
8. Why does NAIC run Flyway from a pipeline job and not when the application starts?

Answers:

1. The checksum is a fingerprint of the whole file text. Any change, including whitespace or comments, changes it.
2. Versioned: applied once, in order (tables, data fixes, renames). Repeatable: re-applied when changed, after versioned ones (views, procedures).
3. The statements before the failure remain (DDL commits at once); the history has a failed row; drop the partial objects by hand, run `repair`, then apply a corrected or new migration.
4. It realigns recorded checksums to the current files, which erases the protection against edited applied migrations.
5. It dropped every object in the schema, including the history. A shared environment's data would be gone with no undo.
6. It records that the database is at version N. It cannot check that the live objects really match the scripts up to N.
7. Any two of: drift (live differs from history), what a change will do before it runs (no preview), whether a repeatable's live object was hand-edited.
8. The application credential would need DDL rights; parallel instances race; a failed migration blocks every pod; PROD changes need approval in a protected pipeline; Snowflake DDL cannot be rolled back.

## 7. Break it

**Exercise: delete an applied migration file.** Run in **Terminal**:

```bash
mv flyway/lab2_scratch/migrations/V2__add_notes_column.sql /tmp/V2.sql.bak
fw flyway/lab2_scratch validate
```

- **Symptom:** `validate` fails and reports an applied migration (version 2) that is not found among the local files.
- **Diagnosis:** the history lists V2 as applied, but no file with that version exists. In a team this happens when someone deletes a file or works on an older branch.
- **Recovery:** put the file back: `mv /tmp/V2.sql.bak flyway/lab2_scratch/migrations/V2__add_notes_column.sql`. Run `validate` again; it passes.
- **Lesson:** history and files must stay in step. Restoring the file is correct. `repair` would mark V2 as deleted, which would hide the problem.

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| No drift detection | L1 (Step 5.13) | DCM plan (Lab 3), drift detective queries | VERIFIED by docs (feature summary); behaviour to be seen by you |
| No preview or plan | L3 | DCM plan (Lab 3); Flyway dry run is Teams only | VERIFIED by docs |
| Repeatables miss hand edits | L2 | DCM plan sees live versus defined | To be observed by you |
| Half-failed migration leaves partial changes | Step 5.7 | One statement per file; clone or Time Travel recovery | VERIFIED by docs; behaviour to be seen by you |
| Renames are hand-written | L4 | Coordinated protocol (Lab 4) | VERIFIED by reasoning |
| No rollback in Community | L5 | Fix-forward, clones, Time Travel | VERIFIED by docs (undo is Teams) |
| Procedure-heavy codebases | Step 5.9, L6 | DCM for routines (Lab 3) | TO-CONFIRM TC-G08 for parsing |
| `repair` and `outOfOrder` can hide mistakes | Steps 5.6, 5.8 | Config contract: no auto repair, `outOfOrder=false` | VERIFIED by running |

## 9. What you learned

- Flyway is an ordered list of files plus a history table with checksums; it knows files and history, not the live database.
- Applied files are immutable; mistakes are fixed forward, and `repair` and `clean` are dangerous tools.
- Snowflake DDL commits at once, so a failed file can leave partial changes; one statement per file avoids it.
- Baselining adopts an existing database but trusts you that the shapes match.
- Flyway cannot drift-check, preview, or roll back in Community edition.

## 10. Cleanup

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
DROP SCHEMA IF EXISTS SBX_INT.LAB2_BASELINE;
-- Optional: keep LAB2_SCRATCH for rereading, or empty it:
-- DROP TABLE / VIEW / PROCEDURE objects in SBX_INT.LAB2_SCRATCH
ALTER WAREHOUSE SBX_CALC_WH SUSPEND;
```

Keep: `flyway/flyway.toml`, `flyway/.env.flyway.example`, the `flyway/lab2_scratch` project files (useful for reference), and the Lab 1 objects in `CALC_SUPPORT`. Unset `FLYWAY_*` variables in your shell.

## 11. State left behind

`SBX_INT.LAB2_SCRATCH` has Flyway history and mini objects (may be cleaned). `SBX_INT.LAB2_BASELINE` was baselined at version 1, then dropped in cleanup. The main `flyway/flyway.toml` is written but **not yet migrated**.

**NEW NAMES in this lab (not in the contract):** `flyway/lab2_scratch/migrations/V5__out_of_order_helper.sql` (jumps ahead for the out-of-order demo), `flyway/lab2_scratch/migrations/R__sp_mini_count.sql` (procedure parsing test), `flyway/lab2_scratch/baseline_migrations/` with `V1__lab1_objects.sql` and `V2__add_approved_by.sql` (baseline exercise), tables `LAB2_PARTIAL`, `LAB2_HELPER`, `LAB2_LATE` (scratch only), view `CALC_SUMMARY`, procedure `SP_MINI_COUNT`.

## 12. Open items for file 13

TC-G08, TC-G09, TC-G10, TC-G11, TC-G21 (key decryption, from file 02), TC-04-01, TC-04-02, TC-04-03, TC-04-04.

## 13. Next

Go to [05-lab3-dcm-fundamentals.md](05-lab3-dcm-fundamentals.md).
