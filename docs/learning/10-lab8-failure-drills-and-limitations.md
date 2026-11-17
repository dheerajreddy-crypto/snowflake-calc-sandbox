# Lab 8: failure drills and limitations (file 10)

> **Lab no:** 8 | **File no:** 10 | **Time:** 2 to 3 h | **Prerequisites:** files 08 and 09 done | **Modes supported:** H or S, D (DCM drills become read-along in Mode D-off; hybrid-specific notes are read-only in Mode S) | **Leaves behind:** everything restored to the end state of Lab 7; a table of nine drills filled in by you; any triggered drill resource monitor removed.

## 1. Goal

Break things on purpose, in a safe place, and practise the full cycle for each failure: symptom, diagnosis, recovery, lesson.

The nine drills:

1. A migration that fails half-way.
2. An applied migration that someone edited.
3. A definition that was deleted (DCM drops the object).
4. A change made by hand (drift).
5. The wrong target (the manifest safety placeholder).
6. A stale plan.
7. Clone leakage (a clone that writes to the source).
8. Expired or wrong credentials.
9. A cost runaway (resource monitor).

You finish with a consolidated table of limitations for four ways of working: no tooling, Flyway only, DCM only, and both.

## 2. Why this matters in calc-engine and calc-admin

> **Real world (NAIC).** `calc-engine-complete-architecture.html` section 12 states there is no universal rollback: "Snowflake DDL commits independently. DCM definition recovery and Flyway history do not restore DML or historical table contents." It names the drills the platform must rehearse: a DCM failure (small deployment, restore prior definitions), a Flyway failure ("failure after first DDL and recovery rehearsal"), clone leakage ("zero INT writes assertion, cleanup always, scheduled sweeper"), source drift ("controlled out-of-band edit detected and blocked") and data recovery (Time Travel or backups).
>
> The DCM branch notes record the real hazards in the same list: DCM drops anything not defined, hand edits in PROD are reverted by the next deploy, a clone that forgets its redirects writes to INT, and a forgotten command-line override must fail instead of touching INT (that is why the clone target uses a database name that does not exist, `CALC_DEV_SET_ON_CLI`). On JWT login, the only NAIC mention found was a negative observation in a test note ("Key was valid; no JWT/390144 error occurred"). There is no NAIC incident write-up, so the credential drill is built from the public Snowflake error table.
>
> Nothing in this lab touches NAIC. Every drill runs in your sandbox, and every destructive one runs in a throwaway clone or a throwaway object, never in `SBX_INT` itself.

## 3. Prerequisites and state check

Run in: **Terminal** (repository root).

```bash
snow connection test -c sbx_dev          # OK
snow connection test -c sbx_owner        # OK
ls scripts/refresh_dev_clone.sh scripts/check_plan_for_drops.py sql/lab1/drift_detective.sql
```

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SHOW DATABASES LIKE 'SBX_DEV_%';                                              -- expect 0 rows
SELECT "version", "success" FROM SBX_INT.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT ORDER BY "installed_rank";   -- 1 to 5, all success (column names: TC-08-08)
SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_CATEGORIES;                   -- 2 (or FORMULA_CATEGORIES if Lab 4 is unfinished)
```

If the history shows more than version 5, or a failed row, stop and fix that first: these drills assume the Lab 7 end state.

Set up one shell with helper variables. Use it for every drill. Run in: **Terminal**.

```bash
export CLONE_DB=SBX_DEV_DRILL8
export PROJECT="$CLONE_DB.CALC_SUPPORT.CALC_ENGINE_DCM"
dplan()   { snow dcm plan   "$PROJECT" --target CLONE --from dcm/calc_engine -D "calc_db='$CLONE_DB'" -D "fin_db='$CLONE_DB'" -c sbx_dev --role SBX_CALC_DEV --save-output "$@"; }
ddeploy() { snow dcm deploy "$PROJECT" --target CLONE --from dcm/calc_engine -D "calc_db='$CLONE_DB'" -D "fin_db='$CLONE_DB'" -c sbx_dev --role SBX_CALC_DEV --save-output "$@"; }
flyclone() { ( cd flyway && set -a && source .env.flyway && set +a \
    && export FLYWAY_URL="${FLYWAY_URL/db=SBX_INT/db=$CLONE_DB}" FLYWAY_PLACEHOLDERS_CALC_DB="$CLONE_DB" \
    && flyway "$@" ); }
```

`flyclone` runs Flyway against the drill clone by rewriting only the database name in your local, git-ignored settings. Every Flyway command in this lab goes through it, so `SBX_INT` is never the target.

> **TO-CONFIRM [TC-G04]:** The file name `--save-output` writes under `dcm/calc_engine/out/`. This lab uses `ls dcm/calc_engine/out/` each time you need it. Record the result in file 13.

## 4. Concepts

**Drill.** A rehearsed failure. You know in advance what will break so you can watch the mechanism instead of panicking.

**Blast radius.** How much a mistake can damage. Each drill states it. Clones keep the blast radius at "one throwaway database".

**Fail forward.** After a Flyway migration was applied, you never edit it. You add a new migration that fixes the result.

**Fail closed.** A check that cannot decide treats the situation as a failure, as `check_plan_for_drops.py` does when the plan file looks wrong.

**Idempotent recovery.** A recovery step that is safe to repeat. DEPLOY after a partial failure is the DCM example.

**Time Travel and retention.** Snowflake can keep old data for a number of days so you can query or restore the past. Clone databases in this pack are created with `DATA_RETENTION_TIME_IN_DAYS = 0`, so a table dropped in a clone cannot be restored. That is deliberate (it saves storage) and it matters in drill 3.

### 4.1 Drill map

```
 Flyway lane              DCM lane                      Identity and cost
 1 failed half-way        3 deleted definition (DROP)   8 wrong or expired key
 2 edited applied file    4 hand edit (drift)           9 cost runaway
                          5 wrong target
                          6 stale plan
 both lanes + clones: 7 clone leakage
```

## 5. Step-by-step

### Step 5.1 Create the drill clone

Run in: **Terminal**. This runs the five-step script from Lab 5: drop any old clone, create it, create its DCM project, plan, deploy, re-plan.

```bash
SNOW_CONNECTION=sbx_dev scripts/refresh_dev_clone.sh DRILL8 --yes
```

You should see: the script ends with an empty re-plan ("No changes detected" or an empty `changeset`) and the clone `SBX_DEV_DRILL8` exists.

Run in: **Snowsight worksheet**:

```sql
SHOW DATABASES LIKE 'SBX_DEV_DRILL8';
```

If you are in Mode D-off the script skips the DCM steps. Drills 3 to 7 then become read-along: read each "You should see" block and answer the questions without running the DCM commands.

For drills 3 to 7 you will refresh the clone between drills so each starts from a clean state. A refresh is the same command as above. It drops the old clone and builds a new one, which is the point of throwaway environments.

### Drill 1: a migration fails half-way

Blast radius: the clone only. Lane: Flyway. This repeats Lab 2 on the main project.

Run in: **Terminal**. Create the file `flyway/migrations/common/V99__drill_half_fail.sql`. Do not commit it. Delete it at the end of the drill.

```sql
CREATE TABLE DRILL_LEFTOVER (ID NUMBER);
INSERT INTO DRILL_TABLE_THAT_DOES_NOT_EXIST VALUES (1);
```

Check what Flyway sees, then migrate the clone:

```bash
flyclone info
flyclone migrate
flyclone info
```

You should see: `info` lists version 99 as `Pending`. `migrate` stops with an error on the second statement (the table does not exist). The second `info` shows version 99 as `Failed`.

**Symptom.** `migrate` ends with an error and every later `migrate` refuses to run.

**Diagnosis.** Run in a **Snowsight worksheet** (role `SBX_CALC_DEV`):

```sql
SHOW TABLES LIKE 'DRILL_LEFTOVER' IN SCHEMA SBX_DEV_DRILL8.CALC_SUPPORT;                    -- 1 row: the first statement stayed
SELECT * FROM SBX_DEV_DRILL8.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT ORDER BY 1 DESC LIMIT 3;   -- version 99, success = false
```

Snowflake commits DDL as soon as it runs, so Flyway cannot roll the file back. The first statement stayed. The history row says the migration failed.

> **VERIFIED:** Snowflake commits DDL immediately, so a migration that fails part-way leaves the earlier statements applied; clean them up by hand (https://documentation.red-gate.com/flyway/reference/database-driver-reference/snowflake, read 2026-11-16). `repair` removes failed entries from the history and leaves any objects behind to be cleaned manually, and it must be given the same locations as `migrate` (https://documentation.red-gate.com/flyway/reference/commands, read 2026-11-16). Undo migrations need the paid Teams edition and, per the docs, do not help with failed versioned migrations on databases without DDL transactions.

**Recovery.** Three actions, in this order.

```sql
DROP TABLE SBX_DEV_DRILL8.CALC_SUPPORT.DRILL_LEFTOVER;       -- 1. clean the objects the failed file left
```

```bash
flyclone repair                                              # 2. remove the failed history row
# 3. fix the never-applied file: replace the second statement with
#    INSERT INTO DRILL_LEFTOVER VALUES (1);
flyclone migrate
```

You should see: version 99 is now `Success`. Because the failed version never ran fully, correcting its file is allowed. A version that was applied successfully is never edited (drill 2).

**Lesson.** Keep DDL migrations small, one idea per file, so a failure leaves at most one thing to clean. Flyway's job ends at recording the failure. The cleanup is human work. NAIC's design says exactly this: "keep DDL migrations small because Snowflake DDL commits independently".

Cleanup: delete `V99__drill_half_fail.sql` from `flyway/migrations/common/`. The clone still has the version 99 row, so refresh the clone before the next drill (step 5.1 command).

### Drill 2: an applied migration was edited

Blast radius: the clone. Lane: Flyway.

Run in: **Terminal**. Open `flyway/migrations/common/V3__data_fix_default_calc_status.sql` and add one comment line at the bottom, for example `-- harmless note`. Save it.

```bash
flyclone validate
```

You should see: `validate` fails and reports that the checksum of version 3 differs between the file and the database.

**Symptom.** `validate` (and therefore `migrate`, because `validateOnMigrate = true`) refuses to run.

**Diagnosis.** Flyway stores a checksum of every applied file. Even a comment changes it. The file on disk no longer matches what ran.

**Recovery, the right way.** Undo your edit in the editor. If you need the change itself, add a new migration `V6__...` that does it. Run `flyclone validate` again: it passes.

**Recovery, the dangerous way (read it, then do it once in the clone).**

```bash
flyclone repair
flyclone validate        # now passes with your edited file
```

You should see: `repair` realigns the stored checksum to the edited file. Nothing re-ran. The database still holds the result of the old text. Now the history says the edited text ran when it did not. Undo the edit in the editor and run `flyclone repair` once more to restore the original checksum.

**Lesson.** `repair` makes the history agree with the files. It does not make the database agree with either. Treat it as a controlled act with approval, which is exactly why NAIC's design forbids automatic repair. (fpr-admin's dev profile sets `repair-on-migrate=true` and `validate-on-migrate=false`, and the contrast is the lesson.)

### Drill 3: a deleted definition (DCM drops the object)

Blast radius: the clone. Lane: DCM. Mode D only.

Refresh the clone (step 5.1 command), then count the rows you are about to lose.

Run in: **Snowsight worksheet** (role `SBX_CALC_DEV`):

```sql
SELECT COUNT(*) FROM SBX_DEV_DRILL8.CALC_SUPPORT.CALC_CATEGORIES;   -- 2 rows: RATIO, TOTAL
```

Run in: **Terminal**. Delete the definition file `dcm/calc_engine/sources/definitions/tables/CALC_CATEGORIES.sql` (use `FORMULA_CATEGORIES.sql` if Lab 4 is unfinished). Open `dcm/calc_engine/sources/definitions/grants/access.sql` and remove the lines that grant on that table. A developer who really wanted the table gone would do both.

> **TO-CONFIRM [TC-10-03]:** What the plan does if you delete only the definition and leave the `GRANT` on that table. Try it first if you are curious and write down the result. The pack does not predict it: DCM may fail the plan or treat the object as external. Record the result in file 13.

```bash
dplan
python3 scripts/check_plan_for_drops.py dcm/calc_engine/out/plan*.json
echo "exit code: $?"
```

You should see: the plan has one `DROP` for the table, `check_plan_for_drops.py` prints `drops: 1` with the table name and exits `1`.

**Symptom.** None, if nobody reads the plan. The deploy just succeeds.

Now do the thing the check exists to prevent, in the clone only:

```bash
ddeploy
```

```sql
SELECT COUNT(*) FROM SBX_DEV_DRILL8.CALC_SUPPORT.CALC_CATEGORIES;   -- error: object does not exist
UNDROP TABLE SBX_DEV_DRILL8.CALC_SUPPORT.CALC_CATEGORIES;            -- fails: the clone has retention 0
```

You should see: the table and its two rows are gone, and `UNDROP` fails because clones are created with `DATA_RETENTION_TIME_IN_DAYS = 0`.

**Diagnosis.** DEPLOY drops objects that are managed by the project but no longer defined. The docs warn: "always run and review PLAN before DEPLOY".

> **VERIFIED:** Removing a `DEFINE` and deploying drops the object, and removing a `GRANT` revokes it (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage, read 2026-11-16). The sample CI workflow in the docs blocks DROP of databases, schemas, tables and stages by reading the saved plan.

**Recovery.** In the clone: put the definition and the grant lines back, refresh the clone. In a real environment: restore from Time Travel (not available for hybrid tables, which also cannot be `UNDROP`ped) or from a backup, then re-add the definition. If you only wanted DCM to stop managing a table, do not delete the definition. Use the unmanage function, then remove the definition:

```sql
SELECT SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT('SBX_DEV_DRILL8.CALC_SUPPORT.CALC_ENGINE_DCM', 'TABLE', 'SBX_DEV_DRILL8.CALC_SUPPORT.CALC_CATEGORIES');
```

> **TO-CONFIRM [TC-10-01]:** The exact argument spelling of `SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT` (project, entity type, fully qualified name). The docs show the form with `'DYNAMIC TABLE'` as the type. How to check at lab time: the manage page of the DCM docs. Do not run this line against anything you care about. Record the result in file 13.

**Lesson.** The plan is the only moment the drop is visible. In NAIC the exact fully qualified name of every approved drop is listed in an approval file, with no wildcards, and an unused approval also fails.

#### If DCM is not available (Mode D-off)

Read-along: a hand-run equivalent is `DROP TABLE SBX_DEV_DRILL8.CALC_SUPPORT.CALC_CATEGORIES;`. With no tool there is no plan and no check. Compare in your notes.

### Drill 4: an out-of-band change (drift)

Blast radius: the clone. Lane: DCM, and what Flyway cannot see.

Restore the file and grant lines from drill 3 (editor), then refresh the clone (step 5.1 command).

Run in: **Snowsight worksheet** (role `SBX_CALC_DEV`, which owns the clone's objects):

```sql
ALTER TABLE SBX_DEV_DRILL8.CALC_SUPPORT.EXECUTIONS ADD COLUMN DRIFT_NOTE VARCHAR;
UPDATE SBX_DEV_DRILL8.CALC_SUPPORT.EXECUTIONS SET DRIFT_NOTE = 'edited by hand' WHERE 1 = 0;   -- touches no rows, but the column exists
CREATE OR REPLACE PROCEDURE SBX_DEV_DRILL8.CALC_SUPPORT.SP_ADD_CATEGORY(P_CATEGORY_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS $$ BEGIN RETURN 'HAND_EDITED'; END $$;
```

**Symptom.** Nothing. Nothing complains. Everything still runs.

**Diagnosis.** Three ways to look, in order of how much they see.

```bash
dplan --delta     # looks only at changed definitions and what depends on them
dplan             # full plan against the live state
flyclone validate # Flyway's view
```

You should see:

- The `--delta` plan shows **no changes**. You did not edit any definition, so a delta plan has nothing to compare.
- The full plan shows changes: the procedure body is replaced back to the definition, and the extra column is reconciled. Read the `changes[]` of the `EXECUTIONS` entry and check whether the column `DRIFT_NOTE` is removed. If it is, DCM would delete a column and its data to make the object match the definition.
- `validate` passes. Flyway knows only its history table.
- The grants of the procedure may also show as a change, because `CREATE OR REPLACE` drops grants.

> **VERIFIED:** `PLAN DELTA` does not detect drift. It evaluates only changed definitions and what depends on them, so always run a full plan before deploy. A plan against the current account state shows changes made outside DCM, and a redeploy restores the definitions (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use, read 2026-11-16). Flyway Community has no drift check; drift reports are Enterprise (https://documentation.red-gate.com/fd/feature-summary-165740620.html, read 2026-11-16).

> **TO-CONFIRM [TC-10-04]:** What the plan does with an extra column that is not in the definition (the docs say `DEFINE` follows `CREATE OR ALTER` rules, which suggests the column is dropped). How to check at lab time: read the `changes[]` of the table entry. Record the result in file 13.

Also run the hand-made detective from Lab 1 (`sql/lab1/drift_detective.sql`) against the clone to see that a tool and a query find the same thing, and that only the tool can also repair it.

**Recovery.** `ddeploy`, then `dplan` must show an empty change set. The hand edit is gone. In PROD this is why NAIC's rule is "no hand edits": the next deploy reverts them.

**Lesson.** Drift detection exists only for what DCM owns. Flyway-owned hybrid tables, and grants applied by Flyway, are not checked by anything. A shape check (Lab 7, step 5.7) is the manual substitute.

#### If DCM is not available (Mode D-off)

Read-along: only the Lab 1 detective query finds this, and only if someone runs it.

### Drill 5: the wrong target

Blast radius: none. Plans never run DDL. Lane: DCM. Mode D only.

Refresh the clone if you ran `ddeploy` after changing definitions. Then try two common mistakes.

**Variant A: forget the database override.**

```bash
snow dcm plan "$PROJECT" --target CLONE --from dcm/calc_engine -c sbx_dev --role SBX_CALC_DEV
```

You should see: the plan fails, because the CLONE configuration renders the database name as `SBX_DEV_SET_ON_CLI`, which does not exist.

**Symptom.** An error that mentions a missing database or an object that cannot be found.

**Diagnosis.** The manifest is not rendered, so a project name cannot take a ticket. The CLONE target therefore carries a placeholder database that never exists. Forgetting `-D` fails loudly. If the placeholder were `SBX_INT`, forgetting `-D` would aim a clone job at INT.

> **TO-CONFIRM [TC-10-05]:** The exact error text. The pack does not assert it. Record it in file 13.

**Variant B: forget `--target`, plan only, never deploy.**

```bash
snow dcm plan "$PROJECT" --from dcm/calc_engine -c sbx_dev --role SBX_CALC_DEV --save-output
python3 scripts/check_plan_for_drops.py dcm/calc_engine/out/plan*.json
```

Without `--target`, the CLI uses the manifest's `default_target`, which is `INT`. That renders the INT names against the clone's project object.

You should see one of two outcomes, both safe: an error, or a large plan that wants to create `SBX_INT` objects and drop the clone's objects. If it is a plan, `check_plan_for_drops.py` prints many drops and exits `1`.

**Recovery.** Run the right command. Never deploy a plan you did not mean to run.

**Lesson.** Targets do not choose the connection, account or role. They only choose a project name and a template set. Two safety layers cover the mistakes: a placeholder that cannot exist, and a drop check that fails closed.

> **VERIFIED:** The CLI resolves the project from an explicit identifier, then `--target`, then `default_target`; targets do not select the connection, account or role (https://docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dcm-commands/overview, read 2026-11-16).

### Drill 6: a stale plan

Blast radius: the clone. Lane: DCM. Mode D only.

Refresh the clone. The idea: a plan you reviewed is not what DEPLOY runs. DEPLOY always recomputes.

1. Make one reviewed change. In `dcm/calc_engine/sources/definitions/procedures/SP_ADD_CATEGORY.sql` add a comment line inside the body (a comment above `DEFINE` may not register as a change; put it inside).
2. Plan and save a copy:

```bash
dplan
ls dcm/calc_engine/out/
cp dcm/calc_engine/out/plan*.json "$TMPDIR/reviewed_plan.json"
python3 -c "import json,sys; p=json.load(open(sys.argv[1])); print(len(p['changeset']), 'change(s) reviewed')" "$TMPDIR/reviewed_plan.json"
```

You should see: one change (the procedure).

3. After you reviewed it, someone else changes the clone. Run in a **Snowsight worksheet**:

```sql
ALTER TABLE SBX_DEV_DRILL8.CALC_SUPPORT.EXECUTIONS ADD COLUMN STALE_TEST VARCHAR;
```

4. Deploy without re-planning:

```bash
ddeploy
ls dcm/calc_engine/out/
python3 -c "import json,glob; p=json.load(open(glob.glob('dcm/calc_engine/out/deploy*.json')[0])); print(len(p['changeset']), 'change(s) deployed')"
```

You should see: more than one change deployed. The table was also altered, although your reviewed plan did not mention it.

**Symptom.** A deploy that did more than anyone approved.

**Diagnosis.** DEPLOY first runs its own internal compile and plan, then applies it. The saved plan from step 2 is only feedback.

**Recovery.** Run `dplan` again right before deploying and compare it with the reviewed plan. A pipeline can do the comparison: fail if the set of (type, fqn) entries differs. NAIC's contract is "a fresh full plan immediately before every deploy; reject stale or failed plans; exact-FQN drop approval; deploy, then a fresh full re-plan must show no changes".

**Lesson.** Approving a plan file does not bind the deploy. What you approve has to be the thing that runs, and in DCM that means plan and deploy close together, with a machine check in between.

### Drill 7: clone leakage

Blast radius: a few sandbox rows in `SBX_INT_FIN`, which you remove at the end. Lane: clone plus DCM. Mode D only.

The rule from Lab 5: a clone must write only to itself. The mechanism is the redirect map in `ENV_CONFIG`. The CLONE configuration sets `clone_redirects: true`, so the map points INT names at the clone. In this drill you turn it off by mistake.

Refresh the clone, then record the INT baseline. Run in a **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SELECT MAX(RESULT_ID) AS LAST_RESULT_ID, COUNT(*) AS RESULT_ROWS FROM SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE;
SELECT COUNT(*) AS EXEC_ROWS FROM SBX_INT.CALC_SUPPORT.EXECUTIONS;
```

Write the three numbers down (the max may be NULL if the table is empty). Now redeploy the clone's `ENV_CONFIG` with redirects off. Run in: **Terminal**:

```bash
dplan -D "clone_redirects=false"      # expect one ALTER on the view ENV_CONFIG
ddeploy -D "clone_redirects=false"
```

> **TO-CONFIRM [TC-10-06]:** The quoting form for a boolean runtime variable (`-D "clone_redirects=false"`). The docs show quoted strings and bracketed lists. How to check at lab time: the plan should show the changed view. If it errors, try `-D "clone_redirects='false'"` only if your template compares to a string, and record which worked. Record the result in file 13.

Run the harmless sandbox compute procedure inside the clone, as the clone's developer role. Run in a **Snowsight worksheet** (role `SBX_CALC_DEV`):

```sql
CALL SBX_DEV_DRILL8.CALC_SUPPORT.SP_COMPUTE_MINI_RESULT(2001, OBJECT_CONSTRUCT('note', 'drill 7'));
```

You should see first: an **insufficient privileges error**. The procedure runs as its owner, `SBX_CALC_DEV`, and that role has only `SELECT` on the source results table (bootstrap grant 6). Privileges stopped the leak. That is a second safety layer, and it is not guaranteed in every environment. To see the leak itself, simulate a developer role that also holds write rights on the source results, then repeat the call. Run in a **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
GRANT INSERT ON TABLE SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE TO ROLE SBX_CALC_DEV;   -- simulation only; revoked in the recovery step
```

> **TO-CONFIRM [TC-10-08]:** That `SBX_CICD_PIPELINE` owns `FIN_CALCULATION_RESULT_BASE` and can grant on it. The bootstrap creates it, and the owner may be another role. How to check at lab time: `SHOW GRANTS ON TABLE SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE;`. If the grant fails, run it once with the `sbx_admin` connection. Record the result in file 13.

Then run the `CALL` again as `SBX_CALC_DEV`.

> **Reminder.** `SP_COMPUTE_MINI_RESULT` exists only in your sandbox and writes one harmless row. On NAIC, the "never run calculations" rule is absolute and you never do anything like this.

Compare INT with your baseline (the same two queries as above).

You should see: `RESULT_ROWS` in `SBX_INT_FIN` went up by 1. The clone ran, and the rows landed in the source. (`EXEC_ROWS` may or may not change, depending on whether the procedure qualifies the executions table; either way the result table proves the leak.)

**Symptom.** None in the clone. The clone's own tables look normal.

**Diagnosis.** The only detection is a comparison of the source before and after, which is the zero-writes assertion the pull request pipeline runs (step 11 in file 08). `QUERY_HISTORY` would show DML statements against `SBX_INT_FIN`.

**Recovery.** Remove the leaked rows as the owner. Run in a **Snowsight worksheet** (role `SBX_CICD_PIPELINE`), using the `LAST_RESULT_ID` you noted:

```sql
DELETE FROM SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE WHERE RESULT_ID > <LAST_RESULT_ID>;
```

If the table was empty before, use `WHERE 1 = 1` instead (only in the sandbox, only if you noted it was empty). Take back the simulated grant: `REVOKE INSERT ON TABLE SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE FROM ROLE SBX_CALC_DEV;`. Then redeploy with redirects on (`ddeploy` without the override) or refresh the clone.

**Lesson.** The NAIC team rejected passing redirect JSON on the command line because forgetting it silently turned redirects off. Their design derives the redirects from the database name inside the CLONE template so they cannot be forgotten, and still keeps the before and after assertion, because a guard that only checks the home schema, not the map, can pass while the map is wrong.

#### If DCM is not available (Mode D-off)

Read-along. Without DCM you rewrite names by hand in the clone, and a missed name is the same leak. The assertion is still your only detector.

### Drill 8: expired or wrong credentials

Blast radius: none. You do not change anything in Snowflake. You present a key Snowflake has never seen. Lane: identity.

Run in: **Terminal**. Make a throwaway key pair that is registered nowhere. It stands in for "the key in the CI secret no longer matches the key on the user".

```bash
mkdir -p ~/.snowflake-sandbox/keys
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out ~/.snowflake-sandbox/keys/drill_wrong_key.p8 -nocrypt
snow connection test -x --user SBX_SVC_CICD --account <ACCOUNT_IDENTIFIER> --authenticator SNOWFLAKE_JWT --private-key-file ~/.snowflake-sandbox/keys/drill_wrong_key.p8
```

> **VERIFIED:** The unencrypted key command (`openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt`) and the connection options `--user`, `--account`, `--authenticator`, `--private-key-file` (https://docs.snowflake.com/en/user-guide/key-pair-auth and .../snowflake-cli/command-reference/dcm-commands/overview, read 2026-11-16). The throwaway key is unencrypted on purpose because it protects nothing. Never do this for a real key.

You should see: the test fails with `JWT token is invalid.` followed by an identifier in brackets (a UUID) and the vendor code 390144.

**Symptom.** Every Snowflake call from CI or the laptop fails at login. Nothing about SQL, grants or objects is mentioned.

**Diagnosis.** 390144 is the general wrapper. The detailed reason is on Snowflake's side and only an administrator can read it. Copy the UUID from the message. Run in a **Snowsight worksheet** (role `ACCOUNTADMIN`, connection `sbx_admin`):

```sql
SELECT JSON_EXTRACT_PATH_TEXT(SYSTEM$GET_LOGIN_FAILURE_DETAILS('<UUID_FROM_THE_ERROR>'), 'errorCode');
DESC USER SBX_SVC_CICD;     -- read RSA_PUBLIC_KEY_FP, the fingerprint of the key that is registered
```

You should see: a more specific code. The documented table:

| Code | Meaning |
|---|---|
| 394300 | the user in the token does not exist (use the user's `LOGIN_NAME`, not its name) |
| 394301 | token missing an issue or expiry time |
| 394302 | token issued more than 60 seconds ago (clock skew or latency) |
| 394303 | token expired |
| 394304 | public key fingerprint does not match the registered key |
| 394305 | token not signed with RS256 |
| 394306 | signature invalid (private key does not match the registered public key) |
| 394307 | account in the token does not match |

For this drill the likely answer is a fingerprint or signature mismatch (394304 or 394306).

> **VERIFIED:** The JWT error table and the use of `SYSTEM$GET_LOGIN_FAILURE_DETAILS` by an administrator (https://docs.snowflake.com/en/user-guide/key-pair-auth-troubleshooting, read 2026-11-16).

> **TO-CONFIRM [TC-10-02]:** Which specific code your wrong-key attempt returns, and how long the login-failure details stay available. How to check at lab time: run the query within a few minutes of the failure. NAIC's own notes mention 390144 only in passing, and the public table says an expired token is 394303, so what NAIC saw as "expired key" may be a different code. Record the result in file 13.

**Recovery.** In a real incident the matching steps are:

1. Compare the fingerprint of the key in the CI secret with `RSA_PUBLIC_KEY_FP`. Compute the local value with the command from the docs: `openssl rsa -pubin -in <public key file> -outform DER | openssl dgst -sha256 -binary | openssl enc -base64`.
2. Check the clock (`394302`), the account identifier (`394307`) and the user's login name (`394300`).
3. If the key must be replaced, register the new public key in the second slot, switch CI, test, then remove the old one. Slots are `RSA_PUBLIC_KEY` and `RSA_PUBLIC_KEY_2`:

```sql
ALTER USER SBX_SVC_CICD SET RSA_PUBLIC_KEY_2 = '<NEW_PUBLIC_KEY_BODY>';     -- both keys work now
-- update the GitHub secret SNOWFLAKE_PRIVATE_KEY_RAW, run a workflow, then:
ALTER USER SBX_SVC_CICD UNSET RSA_PUBLIC_KEY;                                -- retire the old key
```

Do not run step 3 unless you really want to rotate the sandbox key. If you do, the named form `ALTER USER ... ROTATE KEY PAIR` also exists, with the old key valid for 24 hours by default.

Remove the throwaway key: `rm ~/.snowflake-sandbox/keys/drill_wrong_key.p8`.

**Lesson.** A login failure looks the same for every cause until an administrator reads the detail. Know the three most common causes (wrong key, clock skew, wrong identifier). The cure for secret rotation is two key slots, so there is no moment with no valid key. NAIC's target removes the stored key by federation, which turns this drill into "check the trust subject" instead.

### Drill 9: a cost runaway

Blast radius: a few hundredths of a credit on a throwaway warehouse. Lane: cost. Role: `ACCOUNTADMIN`, the only role that can create resource monitors. Use connection `sbx_admin` for this drill only.

Do not use `SBX_CALC_WH` or `SBX_CALC_RM` for the experiment. A monitor that suspends your working warehouse would stop the rest of the pack. Use a separate warehouse and monitor. These two names are new to this lab: `SBX_DRILL_WH` and `SBX_DRILL_RM`.

Run in: **Snowsight worksheet** (role `ACCOUNTADMIN`):

```sql
USE ROLE ACCOUNTADMIN;
CREATE WAREHOUSE SBX_DRILL_WH WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE;
CREATE RESOURCE MONITOR SBX_DRILL_RM WITH CREDIT_QUOTA = 0.05
  FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 50 PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND_IMMEDIATE;
ALTER WAREHOUSE SBX_DRILL_WH SET RESOURCE_MONITOR = SBX_DRILL_RM;
```

> **VERIFIED:** `CREATE RESOURCE MONITOR ... WITH CREDIT_QUOTA ... FREQUENCY ... START_TIMESTAMP ... TRIGGERS ON <n> PERCENT DO { SUSPEND | SUSPEND_IMMEDIATE | NOTIFY }`; only `ACCOUNTADMIN` can create one; assign it with `ALTER WAREHOUSE ... SET RESOURCE_MONITOR`; `SUSPEND` waits for running queries, `SUSPEND_IMMEDIATE` cancels them (https://docs.snowflake.com/en/sql-reference/sql/create-resource-monitor, read 2026-11-16).

> **TO-CONFIRM [TC-10-07]:** A fractional `CREDIT_QUOTA` such as `0.05`, the function `SYSTEM$WAIT(<seconds>)` used below, and how soon after the credits are used a monitor acts. The docs say the quota is a number. Usage is tracked, not instantly: wait several minutes before you decide that nothing happened. How to check at lab time: run it and look at `SHOW RESOURCE MONITORS`. Record the result in file 13.

Burn a few cents of credit on purpose. An X-Small warehouse uses roughly one credit per hour while running, so five minutes is about 0.08 credits, above the 0.05 quota. Run:

```sql
USE WAREHOUSE SBX_DRILL_WH;
CALL SYSTEM$WAIT(300);
SELECT 1;
```

You should see: the first call keeps the warehouse running for five minutes. Then check:

```sql
SHOW RESOURCE MONITORS LIKE 'SBX_DRILL_RM';     -- used_credits and remaining_credits, level of the last trigger
SHOW WAREHOUSES LIKE 'SBX_DRILL_WH';             -- state: SUSPENDED (by auto-suspend) or by the monitor
```

If the monitor fired, a later `USE WAREHOUSE SBX_DRILL_WH; SELECT 1;` fails or the warehouse refuses to resume. If it has not fired after about 15 minutes, move on, do the cleanup, and look at the monitor again later.

**Symptom.** A warehouse that is suddenly suspended, or a bill that grew while nobody was working.

**Diagnosis.** `SHOW RESOURCE MONITORS` shows used credits and the percentage reached. `SHOW WAREHOUSES` shows the state. Account usage views show history, with a delay of some hours (TC-10-07).

**Recovery.** Raise the quota or wait for the next period: `ALTER RESOURCE MONITOR SBX_DRILL_RM SET CREDIT_QUOTA = 1;`. For your real sandbox monitor use the same statement on `SBX_CALC_RM` only if it ever triggers, and only after you understand why.

**Lesson.** A monitor is a brake, not a gauge. Warehouse-level monitors do not cap serverless or cloud-services credits. The causes of real runaway in this pack are small and listed: a warehouse with a long auto-suspend, a pipeline that loops, clones that nobody drops (clones cost almost nothing until data diverges), and a trial that runs out of credits. The quota of 20 credits in `SBX_CALC_RM` is the pack's choice, not a Snowflake default.

Cleanup for the drill (run as `ACCOUNTADMIN`):

```sql
DROP WAREHOUSE IF EXISTS SBX_DRILL_WH;
DROP RESOURCE MONITOR IF EXISTS SBX_DRILL_RM;
```

## 6. Checkpoint questions

1. Why does a Flyway migration that fails on its second statement leave its first statement behind?
2. After `repair` removes a failed row, why is it allowed to edit the failed file, but not an applied one?
3. What did `repair` change in drill 2, and what did it not change?
4. What is the difference between a `--delta` plan and a full plan, and which one found the drift?
5. Why does the CLONE target in the manifest name a database that does not exist?
6. Why did the deploy in drill 6 do more than the reviewed plan said?
7. Which two layers prevented, or would have detected, a clone leak in drill 7?
8. A CI login fails with 390144. What do you do first?
9. A monitor on `SBX_DRILL_WH` did not fire after five minutes. Give two reasons that are not "it is broken".

**Answers**

1. Snowflake commits DDL immediately. Flyway cannot wrap a file in a transaction that rolls back DDL.
2. A failed version never completed, so no result depends on its text. An applied version has a stored checksum and its result exists in the database. Editing it makes the history lie.
3. It realigned the stored checksum to the edited file. It did not re-run the file or change any object.
4. A delta plan checks only changed definitions and what depends on them. A full plan compares every definition with live state. The full plan found the drift.
5. So that forgetting the command-line override fails instead of aiming the job at a real database such as INT.
6. DEPLOY recomputes its own plan against the live state, which now included a change made after the review.
7. Privileges (the developer role could not write to the source results until you granted it), and the before and after count comparison.
8. Ask an administrator for the detailed code with the UUID (`SYSTEM$GET_LOGIN_FAILURE_DETAILS`) and compare key fingerprints, clock, account and login name.
9. Usage is tracked with a delay, and a fractional quota may behave differently from what you assumed (TC-10-07). The warehouse might also have auto-suspended before usage was counted. Check again later.

## 7. Break it

**Two failures in one pull request.** Only if you finished Lab 6.

1. Create a branch `feature/LAB8A` and, in the same commit, edit the comment of the applied `V3` migration and delete a definition file.
2. Open a pull request.
3. Symptom: two checks fail for different reasons. `validate` fails in the `flyway-validate` job (checksum mismatch). `pr-clone-check` fails at the plan step on the DROP, and the `cleanup` job still removes the clone.
4. Diagnosis: read the failing step name of each job. A failure in a different lane needs its own fix.
5. Recovery: revert the V3 edit, restore the definition, push. Both go green.

Lesson: lanes fail independently, and the pipeline reports them separately. You fix one, you do not assume the other is fine.

## 8. Limitations seen in this lab

Per-drill limitations are in the drill sections. The consolidated table below is the product of the lab. Fill the right-hand notes from what you saw.

| Capability | No tooling (hand-run SQL) | Flyway only (Community) | DCM only | Both, with a pipeline |
|---|---|---|---|---|
| Preview before change | None. You read the SQL. | None. `info` lists pending files only. No dry run in Community (Teams has one). | Full `PLAN` with a change list. | Plan for DCM objects; Flyway pending list; pipeline shows both. |
| Drift detection | Only a hand-made query (Lab 1 detective) | None in Community. Drift check is Enterprise. | Full plan shows changes made by hand. Delta plan does not. | DCM objects only. Flyway-owned objects have no detector. |
| Who changed what, when | Query history, with limited retention | History table: version, time, user, checksum | Deployment history with alias and stored artifacts | Both, plus Git history and pipeline logs |
| Rollback | None | No undo in Community. Undo is Teams and does not repair half-failed DDL. Fix forward. | Re-deploy an earlier deployment's definitions (`FROM DEPLOYMENT`). Restores definitions, not data. | Same as each tool, plus Time Travel or backups for data. No universal rollback. |
| Rename | By hand, no record | Explicit migration, clear | Seen as drop plus create (TC-G06) | Flyway rename first, then definition update |
| Hybrid tables and indexes | By hand | Yes, as migrations | Not supported | Flyway lane |
| Row data changes | By hand, unlogged | Yes, versioned, with assertions | Not supported | Flyway lane |
| Procedures and functions | `CREATE OR REPLACE`, silent overwrite | Repeatable `R__` files rerun whole when the checksum changes | One definition per object, per-object diff | DCM lane |
| Grants | By hand; lost on `CREATE OR REPLACE` | In migrations | Declared, re-applied each deploy | DCM for its objects, Flyway for hybrid tables |
| Accidental drop protection | None | Flyway does not drop unknown objects. `clean` is disabled by default. | None by itself. A deploy drops undefined objects. | Drop check in the pipeline plus review |
| Parallel developers | They overwrite each other | Version number collisions at merge | One project object per developer or ticket | Clone per ticket plus project per clone |
| Test environment | None, or INT itself | Clone by hand | Clone by hand | Clone per pull request, created and dropped by the pipeline |
| Learning curve | Lowest | Low | Medium | Highest |

Status of the claims: Flyway Community feature list and DCM plan/deploy semantics are VERIFIED against the docs (read 2026-11-16). `FROM DEPLOYMENT` is described in the DCM docs as re-planning or re-deploying a previous deployment's rendered definitions. Rename as drop plus create is TO-CONFIRM (TC-G06).

## 9. What you learned

- Every failure has a mechanism you can name: DDL auto-commit, checksums, undefined means dropped, delta plans do not see drift, deploy recomputes, placeholders that cannot exist, JWT detail codes, monitors with delay.
- Recovery is rarely automatic. Flyway records, DCM reconciles definitions, and neither restores data.
- Layers of safety overlap on purpose: privileges, drop checks, before and after counts, and a clone with no retention.
- Credential failures are diagnosed by an administrator reading the detail code.
- The consolidated table is your map for the capstone: each cell tells you what a way of working cannot do.

## 10. Cleanup

Run in: **Terminal** and **Snowsight worksheet**.

```sql
CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('DRILL8');     -- as SBX_CICD_PIPELINE or SBX_CALC_DEV
```

Also: delete `flyway/migrations/common/V99__drill_half_fail.sql` if it still exists; revoke any simulated grant from drill 7; drop `SBX_DRILL_WH` and `SBX_DRILL_RM`; remove `drill_wrong_key.p8`; restore any definition or grant lines you removed.

Verify `SBX_INT` is untouched: `git status` shows no changed tracked files you did not intend (you run git yourself), and `SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.CALC_CATEGORIES;` still returns 2.

## 11. State left behind

Everything restored to the end state of Lab 7. The table of nine drills filled in by you. `SBX_CALC_RM` unchanged. If a drill touched `SBX_CALC_RM` (it should not), reset it with `ALTER RESOURCE MONITOR SBX_CALC_RM SET CREDIT_QUOTA = 20;` as `ACCOUNTADMIN`.

## 12. Open items for file 13

TC-G04, TC-G06, TC-10-01, TC-10-02, TC-10-03, TC-10-04, TC-10-05, TC-10-06, TC-10-07, TC-10-08, TC-08-08.

Names new to this lab: warehouse `SBX_DRILL_WH`, resource monitor `SBX_DRILL_RM`, clone ticket `DRILL8`, migration file `V99__drill_half_fail.sql` (never committed).

## 13. Next

Go on to [the capstone and comparison](11-capstone-and-comparison.md).

### Drill log (copy and fill in)

| Drill | Symptom you saw | Diagnosis | Recovery | One-line lesson |
|---|---|---|---|---|
| 1 failed half-way | | | | |
| 2 edited applied file | | | | |
| 3 deleted definition | | | | |
| 4 hand edit | | | | |
| 5 wrong target | | | | |
| 6 stale plan | | | | |
| 7 clone leakage | | | | |
| 8 wrong credentials | | | | |
| 9 cost runaway | | | | |
