# Lab 3: DCM Projects from scratch (file 05)

> **Lab no:** 3 | **File no:** 05 | **Time estimate:** 3 h (Parts A to C are required, Parts D to F can be split over a second sitting) | **Prerequisites:** file 03 (Lab 1 objects exist), Snowflake CLI `snow` 3.24.0 or newer, `SBX_CICD_PIPELINE` usable through connection `sbx_owner` | **Modes supported:** H or S, D required (Mode D-off has a read-along path) | **Leaves behind:** project `SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM` adopting every Lab 1 object plus the remaining engine definitions, deployment alias `int-adopt`, empty re-plan; scratch schema and scratch project dropped

## 1. Goal

By the end of this lab you can:

1. Explain what a DCM project object is and how it differs from a folder of SQL files.
2. Write a `manifest.yml` and definition files that use `DEFINE`, and run `snow dcm create`, `plan` and `deploy`.
3. Read a plan, including the empty plan that means "nothing to do".
4. Adopt the hand-made Lab 1 objects into a project without changing them.
5. Edit a procedure in place, add a column, and watch drift get detected.
6. Delete a definition on purpose and see what that does to data (the main danger of DCM).
7. Reuse one set of definition files for several environments with templating.
8. Name what DCM cannot manage, each proven by an experiment, and say what covers it instead.

## 2. Why this matters in calc-engine and calc-admin

> **Real world.** In NAIC, the engine used to be deployed by pasting `sql/procedures.sql` (about 6,700 lines, 31 `CREATE OR REPLACE PROCEDURE` statements) into Snowsight, in INT only. Live objects drifted from Git (`SP_PROMOTE_METADATA`, `SP_TOP_STATES` and several tables had no DDL on `main`). The `feature/dcm-poc` branch split that file into one definition file per object under `dcm/calc_engine/sources/definitions/`, created the project `COMMON_INT.CALCULATION_SUPPORT.CALC_ENGINE_DCM`, and adopted 70 live objects with zero data change and an empty re-plan.
>
> A working note dated 2026-11-11 recorded pausing DCM and dropping the INT project. The team has since confirmed that both DCM and Flyway are part of the target design (architecture pages: complete architecture, pipeline, calc-admin-with-flyway), so the INT project is to be re-adopted. `DROP DCM PROJECT` left every object, row and grant in place. You will repeat that adoption in miniature in Part B, and you will prove the "left in place" fact in Part D.
>
> Roadmap phase "2 DCM core re-adoption in INT" has this exit evidence: an adoption-only plan, unchanged data and grants, and an empty re-plan. Those three checks are exactly what this lab practises.

## 3. Prerequisites and state check

Run in: **Terminal** (from the repository root `snowflake-calc-sandbox`).

```
snow --version
snow connection test -c sbx_owner
snow dcm --help
git status --short
```

You should see: a version of 3.24.0 or higher (the pack pins `3.33.0` for CI; record the version you actually use in the table at the end of Step 3.1), a successful connection test, the list of `snow dcm` subcommands (`create`, `deploy`, `describe`, `drop`, `drop-deployment`, `list`, `list-deployments`, `plan`, `preview`, `purge`, `test`), and a clean working tree.

> **VERIFIED:** DCM Projects need Snowflake CLI 3.24.0 or higher, and the command list above is the one in the CLI reference (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use and https://docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dcm-commands/overview, read 2026-11-16).

> **TO-CONFIRM [TC-G16]:** NAIC working notes say DCM needed CLI 3.16 or newer; the public docs say 3.24.0. Use 3.24.0 as the minimum. How to check at lab time: `snow dcm --help` works on your version. Record the result in file 13.

Run in: **Snowsight worksheet** as role `SBX_CICD_PIPELINE`. Confirm the Lab 1 objects exist and are owned by the pipeline role (the project owner must own what it adopts).

```sql
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;
SHOW TABLES IN SCHEMA SBX_INT.CALC_SUPPORT;
SHOW PROCEDURES IN SCHEMA SBX_INT.CALC_SUPPORT;
SHOW SEQUENCES IN SCHEMA SBX_INT.CALC_SUPPORT;
```

You should see: tables `CALCULATIONS`, `FORMULAS`, `FORMULA_CATEGORIES`; procedures `SP_REGISTER_CALCULATION`, `SP_ADD_CATEGORY` (plus any built-ins in the list); sequence `SEQ_CALC_ID`. The `owner` column shows `SBX_CICD_PIPELINE` for each. If an object is missing or owned by another role, go back to file 03 and repeat the step that created it. Ownership matters: see Step 3.15.

Also decide your mode. If your Probe D in file 02 failed (`DCM_MODE = off`), read Parts A to F as a transcript and use the `#### If DCM is not available (Mode D-off)` blocks. Everything else in the pack still works.

## 4. Concepts

**Declarative.** You describe the end state ("this table has these columns") and the tool works out the steps. The opposite is imperative: you write the steps ("add this column") and you must know the current state. Lab 1 was imperative; Flyway (Lab 2) is imperative with a bookkeeping table; DCM is declarative.

**DCM project object.** A Snowflake object that lives in a schema (here `SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM`). It runs plan and deploy, and it stores a snapshot of every deployment (the deployment history). You need one project object per environment. The state lives inside Snowflake, not in a state file on your laptop.

**Definition file.** A text file under `sources/definitions/` made of `DEFINE`, `GRANT` and `ATTACH` statements. Nothing else is allowed in it: no `INSERT`, no `CREATE`. It is a Jinja template, so `{{calc_db}}` is replaced before it is read. `DEFINE` behaves like `CREATE OR ALTER`: create the object if missing, alter it if different, leave it if identical. An object may be defined once across all files.

**Manifest.** The file `manifest.yml` next to `sources/`. It lists targets (one per environment: account, project name, owner role, templating configuration) and templating variables.

**Plan.** A dry run. DCM renders the templates, compares the definitions with what the last deployment recorded, and lists CREATE, ALTER and DROP changes. It runs no DDL. **Deploy.** Runs the changes. Important: deploy computes its own fresh changeset. It does not replay your saved plan (you prove this in Step 3.6).

**Managed object.** An object that a deployment created or adopted. DCM only ever drops managed objects. An object that exists but was never defined is invisible to the project. That is why Flyway objects in the same schema are safe (Lab 4).

```
 definition files (Git)           DCM project object (in Snowflake)          live objects
 sources/definitions/*.sql  --->  render Jinja ---> compare with     --->   tables, procs,
 manifest.yml (targets, vars)     last deployment snapshot                  sequences, grants
                                  PLAN = list changes (no DDL)
                                  DEPLOY = recompute, then run DDL
```

**Project owner role.** The role named as `project_owner` in the manifest. The deploy runs as that role, and every object it creates is owned by it. It must also own, or hold the role that owns, any object it adopts.

> **VERIFIED:** DCM Projects are declarative with plan then deploy, state is stored in the project object, definition files allow only `DEFINE`, `GRANT`, `ATTACH`, `DEFINE` behaves like `CREATE OR ALTER`, and DEPLOY recomputes its changeset (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-overview and .../dcm-projects-use, read 2026-11-16).

> **TO-CONFIRM [TC-G17]:** NAIC documents say DCM Projects reached general availability in August 2026. The overview page read for this pack does not state a date, and some parts (for example plan delta, Streamlit and code bundle entities) are marked preview. How to check at lab time: the Snowflake release notes. Record the result in file 13.

## 5. Step-by-step

### Part A: a throwaway project to learn the mechanics (about 40 min)

You start in `SBX_INT.LAB3_SCRATCH` with a tiny project so that mistakes cost nothing. The real engine comes in Part B.

### Step 3.1 Create the scratch project files

Run in: **Terminal**. Create the folders and the files (use your editor).

```
mkdir -p dcm/lab3_scratch/sources/definitions
snow --version
```

`dcm/lab3_scratch/manifest.yml`:

```yaml
manifest_version: 2
type: DCM_PROJECT
default_target: 'SCRATCH'
targets:
  SCRATCH:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_INT.LAB3_SCRATCH.LAB3_DCM'
    project_owner: SBX_CICD_PIPELINE
    templating_config: 'SCRATCH'
templating:
  defaults:
    db: 'SBX_INT'
  configurations:
    SCRATCH: { db: 'SBX_INT' }
```

`dcm/lab3_scratch/sources/definitions/scratch.sql`:

```sql
DEFINE SEQUENCE {{db}}.LAB3_SCRATCH.SEQ_DEMO start with 100 increment by 1 noorder;

DEFINE TABLE {{db}}.LAB3_SCRATCH.DEMO_ORDERS (
    ORDER_ID NUMBER(38,0) NOT NULL,
    ORDER_STATUS VARCHAR(20) DEFAULT 'NEW',
    primary key (ORDER_ID)
);
```

> **TO-CONFIRM [TC-G03]:** the exact `DEFINE SEQUENCE` and `DEFINE PROCEDURE` grammar. The shapes in this lab are copied from definition files that worked in the NAIC DCM proof of concept (for example `DEFINE SEQUENCE ... start with 2001 increment by 1 noorder;`). The per-object public grammar pages were not reachable when this pack was written. How to check at lab time: follow the links on https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities. If a statement is rejected, the error message shows the expected form. Record the result in file 13.

The repository `.gitignore` from file 02 already contains `out/`, so saved plans are never committed.

You should see: two files, no output. Replace `<ACCOUNT_IDENTIFIER>` with your real org-account value in the manifest. An account identifier is not a secret, but if you prefer to keep the committed file generic, keep the placeholder in Git and edit a local copy.

### Step 3.2 Create the project object and read the first plan

Run in: **Terminal**.

```
snow dcm create --target SCRATCH --from dcm/lab3_scratch --if-not-exists -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm plan --target SCRATCH --from dcm/lab3_scratch --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
ls dcm/lab3_scratch/out/
```

Why the extra flags: targets do not choose the connection, account or role. The CLI only warns on a mismatch. You always pass `-c` and `--role` yourself.

You should see: a plan with CREATE entries for the sequence and the table, and nothing dropped. `ls` shows the saved plan file. Write down its exact name.

> **TO-CONFIRM [TC-G04]:** the docs disagree on the saved file name (`plan.json` on the CLI page, `plan_result.json` on the files page). The rest of the pack assumes `plan_result.json`. How to check at lab time: the `ls` above. Record the real name in file 13 and use it everywhere below.

> **TO-CONFIRM [TC-G02]:** whether `CREATE DCM PROJECT` works in your trial. If this step fails with an error that mentions the DCM project, copy the exact text into file 13, set `DCM_MODE = off` in the resume table in file 00 and switch to the Mode D-off blocks.

Create the helper that summarises a plan. Run in: **Terminal**. Create `scripts/plan_summary.py`:

```python
#!/usr/bin/env python3
"""Print a short summary of a DCM plan JSON (docs shape: version 2, changeset[]).
Usage: python3 scripts/plan_summary.py <plan.json>"""
import collections
import json
import sys


def main():
    if len(sys.argv) != 2:
        sys.exit("Usage: plan_summary.py <plan.json>")
    with open(sys.argv[1]) as fh:
        plan = json.load(fh)
    changes = plan.get("changeset", [])
    counts = collections.Counter(c.get("type", "?") for c in changes)
    print("changes: %d  %s" % (len(changes), "  ".join("%s=%d" % kv for kv in sorted(counts.items()))))
    for c in changes:
        oid = c.get("object_id", {})
        print("%-7s %-10s %s" % (c.get("type", "?"), oid.get("domain", "?"), oid.get("fqn", oid.get("name", "?"))))
    if not changes:
        print("EMPTY changeset: the project is in sync")


main()
```

```
python3 scripts/plan_summary.py dcm/lab3_scratch/out/plan_result.json
```

You should see: `changes: 2  CREATE=2` and two lines (one SEQUENCE, one TABLE). If a key name differs from the docs shape, open the file with `python3 -m json.tool dcm/lab3_scratch/out/plan_result.json` and adjust the script. Lab 6 builds a stricter `check_plan_for_drops.py` from this script.

> **VERIFIED:** the plan JSON has `version: 2`, `metadata`, and `changeset[]` with `type` (CREATE, ALTER or DROP) and `object_id{domain,name,fqn,database,schema}`; an empty array means the project definitions are already in sync (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use, read 2026-11-16).

#### If DCM is not available (Mode D-off)

Replace Steps 3.2 to 3.7 with a worksheet transcript. Because `DEFINE` is `CREATE OR ALTER`, you can reproduce the per-object result by hand:

```sql
CREATE OR ALTER TABLE SBX_INT.LAB3_SCRATCH.DEMO_ORDERS (
    ORDER_ID NUMBER(38,0) NOT NULL,
    ORDER_STATUS VARCHAR(20) DEFAULT 'NEW',
    primary key (ORDER_ID)
);
```

Run it twice. The second run changes nothing, which is what an empty re-plan means. What you cannot reproduce is the project-wide plan, the drop of undefined objects and the deployment history; read those parts as explanation only.

### Step 3.3 Deploy, then re-plan (the empty plan)

Run in: **Terminal**.

```
snow dcm deploy --target SCRATCH --from dcm/lab3_scratch --alias scratch-v1 --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm plan --target SCRATCH --from dcm/lab3_scratch --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/lab3_scratch/out/plan_result.json
```

You should see: the deploy reports the two objects created, and the second plan ends with `EMPTY changeset`. This empty plan is the success signal used all the way through the pack ("deploy, then a fresh full re-plan must show no changes").

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SHOW ENTITIES IN DCM PROJECT SBX_INT.LAB3_SCRATCH.LAB3_DCM;
SHOW DEPLOYMENTS IN DCM PROJECT SBX_INT.LAB3_SCRATCH.LAB3_DCM;
```

You should see: two managed entities (the sequence and the table), and one deployment named `DEPLOYMENT$1` with the alias `scratch-v1`.

> **TO-CONFIRM [TC-05-01]:** exact column names of `SHOW ENTITIES` and `SHOW DEPLOYMENTS` output. How to check at lab time: read the result grid. Record the result in file 13.

### Step 3.4 Add a column to a table that holds data

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`). Put some rows in the table by hand. Rows are not something DCM manages, so this is allowed.

```sql
INSERT INTO SBX_INT.LAB3_SCRATCH.DEMO_ORDERS (ORDER_ID, ORDER_STATUS) VALUES (1, 'NEW'), (2, 'DONE'), (3, 'NEW');
SELECT COUNT(*) FROM SBX_INT.LAB3_SCRATCH.DEMO_ORDERS;
```

Run in: **Terminal**. Edit `scratch.sql`: add one column inside the table, after `ORDER_STATUS`:

```sql
    ORDER_NOTE VARCHAR(100),
```

```
snow dcm plan --target SCRATCH --from dcm/lab3_scratch --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/lab3_scratch/out/plan_result.json
snow dcm deploy --target SCRATCH --from dcm/lab3_scratch --alias scratch-v2 -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
```

You should see: exactly one change, an ALTER on `DEMO_ORDERS`, no CREATE and no DROP. After the deploy, `SELECT COUNT(*)` still returns 3 and `DESC TABLE SBX_INT.LAB3_SCRATCH.DEMO_ORDERS` shows `ORDER_NOTE`.

Compare with Lab 1: there you wrote `ALTER TABLE ... ADD COLUMN` yourself and had to know the table did not yet have the column. Here you edited the end state and the tool worked out the ALTER.

> **VERIFIED:** DEPLOY creates missing objects, alters differing ones, skips identical ones; the documented table limitations include reordering columns and changing to incompatible types (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities, read 2026-11-16). Try moving `ORDER_NOTE` before `ORDER_STATUS` and plan again to see the reorder limitation; record what the plan says. Do not deploy that edit.

### Step 3.5 Drift: change an object by hand, then plan

Drift means the live object no longer matches its definition. In Lab 1 you needed a hand-written "drift detective" query. A DCM plan is that detective for free.

Run in: **Snowsight worksheet**:

```sql
ALTER TABLE SBX_INT.LAB3_SCRATCH.DEMO_ORDERS ADD COLUMN SNEAKY_COLUMN VARCHAR(10);
```

Run in: **Terminal**:

```
snow dcm plan --target SCRATCH --from dcm/lab3_scratch --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/lab3_scratch/out/plan_result.json
```

You should see: one ALTER on `DEMO_ORDERS`. The definition does not mention `SNEAKY_COLUMN`, so the plan proposes to bring the table back to the definition.

> **TO-CONFIRM [TC-05-02]:** what exactly the ALTER does with a column that exists live but not in the definition (the expectation is that it removes the column, which would lose that column's data). How to check at lab time: open the `changes` list of that entry with `python3 -m json.tool dcm/lab3_scratch/out/plan_result.json`. Record the result in file 13. Do not deploy yet; Step 3.6 uses this drift.

Contrast with the Flyway lesson from Lab 2: Flyway never looks at the live schema, so it would report nothing here.

### Step 3.6 Plan versus deploy: deploy recomputes

Right now the live table has the drifted column. Your last plan showed one ALTER. Now change the world a second time, after that plan, and deploy without planning again.

Run in: **Snowsight worksheet**:

```sql
ALTER TABLE SBX_INT.LAB3_SCRATCH.DEMO_ORDERS ADD COLUMN SECOND_SNEAKY VARCHAR(10);
```

Run in: **Terminal**:

```
snow dcm deploy --target SCRATCH --from dcm/lab3_scratch --alias scratch-v3 --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
ls dcm/lab3_scratch/out/
```

You should see: the deploy handled both sneaky columns, although the plan you read only knew about the first. The saved deploy result lists the actual changes. Run `snow dcm plan ...` once more: the changeset is empty.

The lesson: a plan is a review aid, not a contract. In CI you must run a fresh full plan immediately before deploying and apply your checks (such as "no DROP") to that plan, and then check the deploy result too. Lab 6 does this.

> **VERIFIED:** DEPLOY recomputes a fresh changeset and does not replay a saved plan; PLAN DELTA does not detect drift, so run a full PLAN before deploy (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use, read 2026-11-16).

> **TO-CONFIRM [TC-G04]:** the file name for the saved deploy result (`deploy_result.json` is the name on the files page). Check with `ls`.

### Step 3.7 The danger: remove a definition and the object is dropped

First restore the table to its definition by deploying (this clears the two sneaky columns from Steps 5.5 and 5.6 if any remain), then add a second table that will hold rows.

Run in: **Terminal**. Append to `scratch.sql`:

```sql
DEFINE TABLE {{db}}.LAB3_SCRATCH.DEMO_PRECIOUS (
    ITEM_ID NUMBER(38,0) NOT NULL,
    ITEM_NAME VARCHAR(50),
    primary key (ITEM_ID)
);
```

```
snow dcm deploy --target SCRATCH --from dcm/lab3_scratch --alias scratch-v4 -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
```

Run in: **Snowsight worksheet**:

```sql
INSERT INTO SBX_INT.LAB3_SCRATCH.DEMO_PRECIOUS VALUES (1, 'irreplaceable row one'), (2, 'irreplaceable row two');
SELECT COUNT(*) FROM SBX_INT.LAB3_SCRATCH.DEMO_PRECIOUS;
```

Now delete the `DEFINE TABLE ... DEMO_PRECIOUS` block from `scratch.sql` (as if a merge conflict or a tidy-up removed it), then read the plan:

```
snow dcm plan --target SCRATCH --from dcm/lab3_scratch --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/lab3_scratch/out/plan_result.json
```

You should see: a line starting `DROP` for `DEMO_PRECIOUS`. Nothing in the plan says "this table has 2 rows". A plan shows structure, not data.

Deploy it (this is a throwaway schema, so this is the experiment):

```
snow dcm deploy --target SCRATCH --from dcm/lab3_scratch --alias scratch-v5-drop -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
```

Run in: **Snowsight worksheet**:

```sql
SELECT COUNT(*) FROM SBX_INT.LAB3_SCRATCH.DEMO_PRECIOUS;
```

You should see: an error that the table does not exist. The rows went with it.

Discussion to write in your notes:

1. Who would have caught this in review? A reviewer sees a deleted file, not a dropped table with data.
2. What is the recovery? Standard tables have Time Travel, so `UNDROP TABLE SBX_INT.LAB3_SCRATCH.DEMO_PRECIOUS;` can bring it back within the retention period. Try it. Hybrid tables cannot be undropped, and clone databases are created with retention 0 (Lab 5), so in those places there is no way back.
3. After `UNDROP`, is the table managed again? Run `SHOW ENTITIES IN DCM PROJECT SBX_INT.LAB3_SCRATCH.LAB3_DCM;`.

> **TO-CONFIRM [TC-05-03]:** whether an undropped table is managed by the project again, and what a plan says about it now that its definition is gone (a second DROP is expected). How to check at lab time: the two commands above. Record the result in file 13.

> **VERIFIED:** removing a `DEFINE` and deploying drops the object, and the docs say to always review PLAN before DEPLOY; the sample CI workflow blocks drops of databases, schemas, tables and stages by reading the saved plan JSON (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage and .../dcm-projects-cicd, read 2026-11-16).

How NAIC deals with it (from the target architecture): an exact-FQN DROP approval file (no wildcards, an unused approval fails the run), checked by a script against the fresh full plan. You build the sandbox version in Lab 6 (`scripts/check_plan_for_drops.py`).

To keep the object you wanted: restore the `DEFINE` block, or unmanage without dropping:

```sql
SELECT SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT('SBX_INT.LAB3_SCRATCH.LAB3_DCM', 'TABLE', 'SBX_INT.LAB3_SCRATCH.DEMO_PRECIOUS');
```

> **VERIFIED:** this function removes an entity from management without dropping it, after which you must also delete the statement from the files or the next deploy manages it again (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage, read 2026-11-16). 

> **TO-CONFIRM [TC-05-10]:** the `'TABLE'` type string in that call is an assumption from the docs example `'DYNAMIC TABLE'`. How to check at lab time: run it; if it errors, read the message and try `'TABLE'` in another spelling from the docs page. Record the working form in file 13.

### Part B: adopt the Lab 1 objects into the real engine project (about 60 min)

Adoption means putting an existing, hand-made object under management without changing it. The rule: write a definition that matches the live object exactly, plan, and check that the plan changes nothing real.

### Step 3.8 Write definitions from the live objects

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`). Ask Snowflake for the real DDL. This is what NAIC did (the sequence file in the POC carries the comment "Source: GET_DDL").

```sql
SELECT GET_DDL('SEQUENCE',  'SBX_INT.CALC_SUPPORT.SEQ_CALC_ID');
SELECT GET_DDL('TABLE',     'SBX_INT.CALC_SUPPORT.CALCULATIONS', TRUE);
SELECT GET_DDL('TABLE',     'SBX_INT.CALC_SUPPORT.FORMULAS', TRUE);
SELECT GET_DDL('TABLE',     'SBX_INT.CALC_SUPPORT.FORMULA_CATEGORIES', TRUE);
SELECT GET_DDL('PROCEDURE', 'SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(VARCHAR, VARCHAR)', TRUE);
SELECT GET_DDL('PROCEDURE', 'SBX_INT.CALC_SUPPORT.SP_REGISTER_CALCULATION(VARCHAR, VARCHAR)', TRUE);
```

Convert each result into a file by four rules: (1) change `create or replace` to `DEFINE`; (2) replace the text `SBX_INT.CALC_SUPPORT` with `{{calc_db}}.{{calc_schema}}`; (3) one object per file, file name equal to the object name; (4) keep the body text exactly as it is.

Run in: **Terminal**:

```
mkdir -p dcm/calc_engine/sources/definitions/{sequences,tables,functions,views,procedures,grants}
```

Start with this minimal `dcm/calc_engine/manifest.yml` (Step 3.14 expands it):

```yaml
manifest_version: 2
type: DCM_PROJECT
default_target: 'INT'
targets:
  INT:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM'
    project_owner: SBX_CICD_PIPELINE
    templating_config: 'INT'
templating:
  defaults:
    calc_db: 'SBX_INT'
    calc_schema: 'CALC_SUPPORT'
  configurations:
    INT: { calc_db: 'SBX_INT', calc_schema: 'CALC_SUPPORT' }
```

Expected shapes (your Lab 1 columns are the truth; if `GET_DDL` shows a different type, follow `GET_DDL`). `sequences/SEQ_CALC_ID.sql`:

```sql
-- Source: GET_DDL on SBX_INT.CALC_SUPPORT.SEQ_CALC_ID
DEFINE SEQUENCE {{calc_db}}.{{calc_schema}}.SEQ_CALC_ID start with 2001 increment by 1 noorder;
```

`tables/CALCULATIONS.sql`:

```sql
DEFINE TABLE {{calc_db}}.{{calc_schema}}.CALCULATIONS (
    CALC_ID NUMBER(38,0) NOT NULL DEFAULT {{calc_db}}.{{calc_schema}}.SEQ_CALC_ID.NEXTVAL,
    CALC_NAME VARCHAR(255),
    CALC_STATUS VARCHAR(10) DEFAULT 'ACTIVE',
    SOURCE_CALC_ID NUMBER(38,0),
    CREATED_BY VARCHAR(255),
    CREATED_AT TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
    primary key (CALC_ID)
);
```

`tables/FORMULA_CATEGORIES.sql`:

```sql
DEFINE TABLE {{calc_db}}.{{calc_schema}}.FORMULA_CATEGORIES (
    CATEGORY_ID NUMBER(38,0) NOT NULL autoincrement start 1 increment 1 noorder,
    CATEGORY_NAME VARCHAR(255),
    CREATED_BY VARCHAR(255),
    primary key (CATEGORY_ID)
);
```

`tables/FORMULAS.sql` follows the same method (`FORMULA_ID`, `CALC_ID`, `FORMULA_TEXT` plus whatever Lab 1 created). `procedures/SP_ADD_CATEGORY.sql` (idempotent insert; this is the shape used in the NAIC POC, with names qualified so the procedure does not depend on the caller's current schema):

```sql
DEFINE PROCEDURE {{calc_db}}.{{calc_schema}}.SP_ADD_CATEGORY(P_CATEGORY_NAME VARCHAR, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$BEGIN
    INSERT INTO {{calc_db}}.{{calc_schema}}.FORMULA_CATEGORIES (CATEGORY_NAME, CREATED_BY)
    SELECT :P_CATEGORY_NAME, :P_USER
    WHERE NOT EXISTS (SELECT 1 FROM {{calc_db}}.{{calc_schema}}.FORMULA_CATEGORIES WHERE CATEGORY_NAME = :P_CATEGORY_NAME);
    RETURN 'OK';
END$$;
```

`procedures/SP_REGISTER_CALCULATION.sql`: take the body from `sql/lab1/procedures_v1.sql` (the restored v1) and wrap it the same way.

If your Lab 1 body differs from the text above (for example it used unqualified names), a first plan will show an ALTER for that procedure, because DCM treats a changed handler body as a full replace. Either copy your live body exactly (pure adoption) or accept the change on purpose. Both are valid; deciding on purpose is the habit being trained.

> **TO-CONFIRM [TC-G03]:** `DEFINE PROCEDURE ... AS $$...$$` mirrors `CREATE OR ALTER PROCEDURE`. The shape above is from the NAIC POC. If rejected, read the error and check the supported-entities page.

### Step 3.9 Create the project, plan, and read an adoption plan

Run in: **Terminal**:

```
snow dcm create --target INT --from dcm/calc_engine --if-not-exists -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
```

Run in: **Snowsight worksheet**: record evidence before you touch anything.

```sql
SELECT COUNT(*) AS ROWS_N, HASH_AGG(*) AS ROWS_HASH FROM SBX_INT.CALC_SUPPORT.CALCULATIONS;
SHOW GRANTS ON TABLE SBX_INT.CALC_SUPPORT.CALCULATIONS;
```

Write down `ROWS_N`, `ROWS_HASH` and how many grant rows the `SHOW GRANTS` returned.

Run in: **Terminal**:

```
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: no `DROP` and no `CREATE` lines. In the NAIC POC the first plan of an existing set of objects showed every object as an ALTER (70 entities, 0 create, 70 alter, 0 drop; 69 of them adoption-only, one real change). Expect the same kind of picture: one ALTER per defined object, and any entry that is more than adoption points at a definition that does not match the live object.

> **TO-CONFIRM [TC-05-04]:** how the public docs describe an adoption plan. The docs say objects that existed before and were newly added to the project become managed; the "ALTER with set PROJECT" picture above comes from the NAIC POC, not from the public pages. How to check at lab time: open one entry with `python3 -m json.tool` and read `changes`. Record the result in file 13.

Pass criteria for adoption (use them every time):

1. Zero DROP.
2. Zero CREATE (everything already exists).
3. Every ALTER is explained: pure adoption, or a change you decided on.
4. After deploy: same row count and hash, same grants, and an empty re-plan.

If a table entry proposes a real change, stop. Fix the definition to match `GET_DDL` instead of deploying the difference.

### Step 3.10 Deploy the adoption and prove nothing changed

Run in: **Terminal**:

```
snow dcm deploy --target INT --from dcm/calc_engine --alias int-adopt --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

Run in: **Snowsight worksheet**: repeat the evidence queries from Step 3.9.

You should see: the final summary says `EMPTY changeset`, `ROWS_N` and `ROWS_HASH` are identical, and the grant list is the same. `SHOW ENTITIES IN DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;` lists your six adopted objects.

You have now done what the NAIC roadmap calls "DCM core re-adoption in INT" in miniature.

### Step 3.11 Edit a procedure in place (and keep its grants)

In Lab 1 a `CREATE OR REPLACE PROCEDURE` silently overwrote the other developer's version and wiped grants. See the difference.

Run in: **Snowsight worksheet**: add a hand-made grant that DCM does not know about.

```sql
GRANT USAGE ON PROCEDURE SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(VARCHAR, VARCHAR) TO ROLE SBX_CALC_READER;
SHOW GRANTS ON PROCEDURE SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(VARCHAR, VARCHAR);
```

Run in: **Terminal**. First make a comment-only edit **inside the body** (NAIC found that a comment above `DEFINE` may not register as a change, so test edits go inside the body). Add `-- touched` as the first line after `$$BEGIN`, then plan:

```
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: exactly one ALTER, on `SP_ADD_CATEGORY`. Per the docs, the plan does not show a line-by-line body difference for procedures: a changed handler is treated as a full replace.

Now make a real change: replace `RETURN 'OK';` with `RETURN 'OK: ' || :P_CATEGORY_NAME;`, remove the `-- touched` line, then deploy and re-plan:

```
snow dcm deploy --target INT --from dcm/calc_engine --alias int-proc-edit -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

Run in: **Snowsight worksheet**:

```sql
CALL SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY('RATIO', CURRENT_USER());
SHOW GRANTS ON PROCEDURE SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(VARCHAR, VARCHAR);
```

You should see: the call returns `OK: RATIO`, the re-plan is empty, and the grant to `SBX_CALC_READER` is still listed.

> **TO-CONFIRM [TC-05-05]:** that the hand-made grant survives the procedure change. The NAIC POC observed that grants are "altered in place" and survive edits, but the public docs do not say so explicitly. If the grant disappears, that is also a finding: write it down and see Step 3.13, where grants are declared so they are reapplied on every deploy.

### Step 3.12 Add the remaining engine definitions

The contract for later labs expects the rest of the miniature engine to be under DCM. Add these files. Everything here is a CREATE, so the plan must show CREATE lines only.

`sequences/SEQ_FORMULA_ID.sql`, `SEQ_EXECUTION_ID.sql`, `SEQ_WORKFLOW_ID.sql`, `SEQ_AUDIT_EVENT_ID.sql` (one file each, same pattern):

```sql
DEFINE SEQUENCE {{calc_db}}.{{calc_schema}}.SEQ_FORMULA_ID start with 1 increment by 1 noorder;
```

`tables/EXECUTIONS.sql`:

```sql
DEFINE TABLE {{calc_db}}.{{calc_schema}}.EXECUTIONS (
    EXECUTION_ID NUMBER(38,0) NOT NULL DEFAULT {{calc_db}}.{{calc_schema}}.SEQ_EXECUTION_ID.NEXTVAL,
    CALC_ID NUMBER(38,0),
    STATUS VARCHAR(20),
    ROWS_WRITTEN NUMBER(38,0),
    STARTED_AT TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
    primary key (EXECUTION_ID)
);
```

`tables/DATA_CHANGE_LOG.sql` (the ledger that Lab 4 data changes write to):

```sql
DEFINE TABLE {{calc_db}}.{{calc_schema}}.DATA_CHANGE_LOG (
    CHANGE_ID NUMBER(38,0) NOT NULL autoincrement start 1 increment 1 noorder,
    CHANGE_VERSION VARCHAR(40) NOT NULL,
    DESCRIPTION VARCHAR(500),
    EXPECTED_ROWS NUMBER(38,0),
    ROWS_AFFECTED NUMBER(38,0),
    APPLIED_BY VARCHAR(255) DEFAULT CURRENT_USER(),
    APPLIED_AT TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
    primary key (CHANGE_ID)
);
```

`views/ENV_CONFIG.sql` (same Jinja `if` shape as the NAIC file; it shows that templating can contain logic, not only names). In INT the redirect map is `[]`; in a clone (Lab 5) it maps INT names to the clone:

```sql
DEFINE VIEW {{calc_db}}.{{calc_schema}}.ENV_CONFIG(CONFIG_KEY, CONFIG_VALUE) as
SELECT 'HOME_SCHEMA' AS CONFIG_KEY, TO_VARIANT('{{calc_db}}.{{calc_schema}}') AS CONFIG_VALUE
UNION ALL
SELECT 'RESULT_SCHEMA', TO_VARIANT('{{fin_db}}.{{fin_schema}}')
UNION ALL
SELECT 'SCHEMA_REDIRECTS', PARSE_JSON('{% if clone_redirects %}[{"from": "SBX_INT.CALC_SUPPORT.", "to": "{{calc_db}}.CALC_SUPPORT."}, {"from": "SBX_INT_FIN.FIN_CALCULATION.", "to": "{{calc_db}}.FIN_CALCULATION."}]{% else %}{{schema_redirects}}{% endif %}');
```

`functions/FN_APPLY_REDIRECTS.sql`. The NAIC function uses `REDUCE` and `REGEXP_REPLACE` over any number of map entries. This miniature handles up to two entries with plain `REPLACE`, which is enough for the sandbox and easy to read:

```sql
DEFINE FUNCTION {{calc_db}}.{{calc_schema}}.FN_APPLY_REDIRECTS(P_TEXT VARCHAR, P_MAP ARRAY)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
SELECT IFF(ARRAY_SIZE(P_MAP) > 1,
           REPLACE(L1, P_MAP[1]:"from"::VARCHAR, P_MAP[1]:"to"::VARCHAR),
           L1)
FROM (SELECT IFF(ARRAY_SIZE(P_MAP) > 0,
                 REPLACE(P_TEXT, P_MAP[0]:"from"::VARCHAR, P_MAP[0]:"to"::VARCHAR),
                 P_TEXT) AS L1)
$$;
```

> **TO-CONFIRM [TC-05-06]:** that this scalar SQL function body (a subquery that reads the arguments) is accepted. A plan validates creation only. How to check at lab time: after the deploy below, run `SELECT SBX_INT.CALC_SUPPORT.FN_APPLY_REDIRECTS('SBX_INT.CALC_SUPPORT.X', PARSE_JSON('[{"from":"SBX_INT.","to":"SBX_DEV_T."}]'));` and expect `SBX_DEV_T.CALC_SUPPORT.X`. With an empty map `[]` the text must come back unchanged.

`procedures/SP_ACTIVATE_DRAFT_CALCULATION.sql` (miniature of the real one: promote the draft, retire the source):

```sql
DEFINE PROCEDURE {{calc_db}}.{{calc_schema}}.SP_ACTIVATE_DRAFT_CALCULATION(P_DRAFT_CALC_ID NUMBER, P_USER VARCHAR)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE
    v_src NUMBER;
BEGIN
    SELECT SOURCE_CALC_ID INTO :v_src
      FROM {{calc_db}}.{{calc_schema}}.CALCULATIONS
     WHERE CALC_ID = :P_DRAFT_CALC_ID AND CALC_STATUS = 'DRAFT';
    IF (:v_src IS NOT NULL) THEN
        UPDATE {{calc_db}}.{{calc_schema}}.CALCULATIONS SET CALC_STATUS = 'INACTIVE' WHERE CALC_ID = :v_src;
    END IF;
    UPDATE {{calc_db}}.{{calc_schema}}.CALCULATIONS SET CALC_STATUS = 'ACTIVE'
     WHERE CALC_ID = :P_DRAFT_CALC_ID AND CALC_STATUS = 'DRAFT';
    RETURN 'ACTIVATED ' || :P_DRAFT_CALC_ID;
END;
$$;
```

`procedures/SP_COMPUTE_MINI_RESULT.sql`. Note the deliberately hard-coded INT result table name inside the generated text: the real engine builds SQL from data that contains INT names and rewrites them at run time with the redirect map. There is no `SP_RUN_` prefix on purpose, to avoid confusion with the real engine procedures you must never call on NAIC.

```sql
DEFINE PROCEDURE {{calc_db}}.{{calc_schema}}.SP_COMPUTE_MINI_RESULT(P_CALC_ID NUMBER, P_PARAMS VARIANT)
RETURNS VARCHAR LANGUAGE SQL EXECUTE AS OWNER
AS
$$
DECLARE
    v_map ARRAY;
    v_exec_id NUMBER;
    v_sql VARCHAR;
BEGIN
    SELECT CONFIG_VALUE::ARRAY INTO :v_map
      FROM {{calc_db}}.{{calc_schema}}.ENV_CONFIG WHERE CONFIG_KEY = 'SCHEMA_REDIRECTS';
    INSERT INTO {{calc_db}}.{{calc_schema}}.EXECUTIONS (CALC_ID, STATUS, ROWS_WRITTEN)
    SELECT :P_CALC_ID, 'SUCCESS', 1;
    SELECT MAX(EXECUTION_ID) INTO :v_exec_id
      FROM {{calc_db}}.{{calc_schema}}.EXECUTIONS WHERE CALC_ID = :P_CALC_ID;
    v_sql := 'INSERT INTO SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE (CALC_ID, EXECUTION_ID, VALUE_NUMBER) SELECT '
             || :P_CALC_ID || ', ' || :v_exec_id || ', 42';
    v_sql := {{calc_db}}.{{calc_schema}}.FN_APPLY_REDIRECTS(:v_sql, :v_map);
    EXECUTE IMMEDIATE :v_sql;
    RETURN 'OK execution ' || :v_exec_id;
END;
$$;
```

Run in: **Terminal**: plan, read, deploy, re-plan. Because `ENV_CONFIG` now uses `fin_db`, `fin_schema`, `clone_redirects` and `schema_redirects`, first replace the manifest with the full version from Step 3.14, then:

```
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
snow dcm deploy --target INT --from dcm/calc_engine --alias int-complete -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
snow dcm plan --target INT --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: the first plan has CREATE lines for 4 sequences, 2 tables, 1 view, 1 function and 2 procedures, plus no DROP; the last plan is empty. Do not call `SP_COMPUTE_MINI_RESULT` yet; Lab 5 uses it inside a clone.

### Step 3.13 Declare grants in the project

In NAIC, a `CREATE OR REPLACE` wiped grants, so people re-granted by hand. The POC fixed this with a declared grant in `access.sql` ("declared grant, reapplied on every deploy"). Do the same for the app role. The role names come from template variables so QA and PROD can use different roles.

`dcm/calc_engine/sources/definitions/grants/access.sql`:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE {{calc_db}}.{{calc_schema}}.CALCULATIONS TO ROLE {{app_role}};
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE {{calc_db}}.{{calc_schema}}.FORMULAS TO ROLE {{app_role}};
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE {{calc_db}}.{{calc_schema}}.FORMULA_CATEGORIES TO ROLE {{app_role}};
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE {{calc_db}}.{{calc_schema}}.EXECUTIONS TO ROLE {{app_role}};
GRANT SELECT, INSERT ON TABLE {{calc_db}}.{{calc_schema}}.DATA_CHANGE_LOG TO ROLE {{app_role}};
GRANT USAGE ON SEQUENCE {{calc_db}}.{{calc_schema}}.SEQ_CALC_ID TO ROLE {{app_role}};
GRANT USAGE ON SEQUENCE {{calc_db}}.{{calc_schema}}.SEQ_FORMULA_ID TO ROLE {{app_role}};
GRANT USAGE ON SEQUENCE {{calc_db}}.{{calc_schema}}.SEQ_EXECUTION_ID TO ROLE {{app_role}};
GRANT USAGE ON PROCEDURE {{calc_db}}.{{calc_schema}}.SP_REGISTER_CALCULATION(VARCHAR, VARCHAR) TO ROLE {{app_role}};
GRANT USAGE ON PROCEDURE {{calc_db}}.{{calc_schema}}.SP_ADD_CATEGORY(VARCHAR, VARCHAR) TO ROLE {{app_role}};
GRANT USAGE ON PROCEDURE {{calc_db}}.{{calc_schema}}.SP_ACTIVATE_DRAFT_CALCULATION(NUMBER, VARCHAR) TO ROLE {{app_role}};
GRANT USAGE ON PROCEDURE {{calc_db}}.{{calc_schema}}.SP_COMPUTE_MINI_RESULT(NUMBER, VARIANT) TO ROLE {{app_role}};
GRANT SELECT ON TABLE {{calc_db}}.{{calc_schema}}.CALCULATIONS TO ROLE {{reader_role}};
GRANT SELECT ON TABLE {{calc_db}}.{{calc_schema}}.FORMULAS TO ROLE {{reader_role}};
GRANT SELECT ON TABLE {{calc_db}}.{{calc_schema}}.FORMULA_CATEGORIES TO ROLE {{reader_role}};
GRANT SELECT ON TABLE {{calc_db}}.{{calc_schema}}.EXECUTIONS TO ROLE {{reader_role}};
```

The two Flyway-owned hybrid tables are not in this file. Their grants live inside the Flyway migrations (Lab 4). Note the sequence grants: a table with `DEFAULT seq.NEXTVAL` fails for a role that has no `USAGE` on the sequence. The real Calc Admin audit table hit exactly this on 2026-11-03.

Run in: **Terminal**: plan (expect GRANT creates only, and a plan entry for the earlier hand grant is not expected because DCM ignores grants it did not deploy), deploy with `--alias int-grants`, re-plan (empty).

> **TO-CONFIRM [TC-G25]:** public docs say `GRANT OWNERSHIP` fails in plan or deploy when the object already has other grants unless those are also defined or revoked. Do not add `GRANT OWNERSHIP` to this file. Part E has an experiment on a scratch object.

### Part C: one set of definitions, several environments (about 20 min)

### Step 3.14 Templating and the one-configuration rule

Run in: **Terminal**. Replace `dcm/calc_engine/manifest.yml` with the full version (this is the version the later labs and the CI workflows use):

```yaml
manifest_version: 2
type: DCM_PROJECT
default_target: 'INT'
targets:
  INT:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM'
    project_owner: SBX_CICD_PIPELINE
    templating_config: 'INT'
  QA:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_QA.CALC_SUPPORT.CALC_ENGINE_DCM'
    project_owner: SBX_CICD_PIPELINE
    templating_config: 'QA'
  PROD:
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_PROD.CALC_SUPPORT.CALC_ENGINE_DCM'
    project_owner: SBX_CICD_PIPELINE
    templating_config: 'PROD'
  CLONE:      # always pass the clone project FQN and -D "calc_db='SBX_DEV_<TICKET>'"; SBX_DEV_SET_ON_CLI does not exist on purpose
    account_identifier: <ACCOUNT_IDENTIFIER>
    project_name: 'SBX_DEV_SET_ON_CLI.CALC_SUPPORT.CALC_ENGINE_DCM'
    project_owner: SBX_CALC_DEV
    templating_config: 'CLONE'
templating:
  defaults:
    calc_db: 'SBX_INT'
    calc_schema: 'CALC_SUPPORT'
    fin_db: 'SBX_INT_FIN'
    fin_schema: 'FIN_CALCULATION'
    app_role: 'SBX_FN_CALC_ENGINE_NP'
    reader_role: 'SBX_CALC_READER'
    schema_redirects: '[]'
    clone_redirects: false
  configurations:
    INT:   { calc_db: 'SBX_INT',  fin_db: 'SBX_INT_FIN' }
    QA:    { calc_db: 'SBX_QA',   fin_db: 'SBX_INT_FIN' }
    PROD:  { calc_db: 'SBX_PROD', fin_db: 'SBX_INT_FIN' }
    CLONE: { calc_db: 'SBX_DEV_SET_ON_CLI', fin_db: 'SBX_DEV_SET_ON_CLI', clone_redirects: true }
```

Variable resolution order, lowest to highest: global defaults, the chosen configuration, then runtime values given with `-D`/`--variable`. The CLONE target uses a database name that does not exist (`SBX_DEV_SET_ON_CLI`) so that forgetting the command-line override fails loudly instead of touching INT. This is the pattern in the NAIC manifest. The manifest itself is not rendered, so the project name cannot take a ticket id; you pass the clone's project name on the command line (Lab 5).

String values given with `-D` need inner single quotes, for example `-D "calc_db='SBX_DEV_LAB5A'"`. Without them the CLI reports an error (NAIC saw error 1510).

Now the one-configuration rule. This is a plan only. **Never deploy it.**

```
snow dcm plan --target INT -D "calc_db='SBX_QA'" --from dcm/calc_engine --save-output -c sbx_owner --role SBX_CICD_PIPELINE --warehouse SBX_CALC_WH
python3 scripts/plan_summary.py dcm/calc_engine/out/plan_result.json
```

You should see: CREATE lines for every object in `SBX_QA.CALC_SUPPORT` and DROP lines for the INT objects the INT project manages. The project object remembers what it deployed; switching its rendering to another database reads as "remove INT, create QA".

The lesson: one project object, one environment. Switching a project between configurations can drop objects. For several environments use one project object each (`SBX_INT...`, `SBX_QA...`, `SBX_PROD...`), exactly as the manifest targets say. Lab 6 deploys QA and PROD that way.

> **VERIFIED:** only one configuration can be deployed per project object at a time and switching can drop objects missing from the new rendering; use distinct project objects for coexisting environments; variable precedence is defaults, configuration, runtime (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use and .../dcm-projects-files, read 2026-11-16).

> **TO-CONFIRM [TC-G20]:** the `env_vars` and `env_secrets` manifest layout is not used in this pack. Never put secrets in templating variables because rendered SQL is stored unredacted.

### Part D: who may do what (about 20 min)

### Step 3.15 Privileges on the project object

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SHOW GRANTS ON DCM PROJECT SBX_INT.LAB3_SCRATCH.LAB3_DCM;  -- TO-CONFIRM TC-05-11
```

> **TO-CONFIRM [TC-05-11]:** the exact form of `SHOW GRANTS` for a DCM project object. How to check at lab time: if it is rejected, use `SHOW GRANTS ON DCM PROJECT` from the manage page of the docs, or `SHOW GRANTS TO ROLE SBX_CICD_PIPELINE`. Record the working form in file 13.

Run in: **Terminal**. Try to plan as the developer role, who has no privilege on the scratch project yet (the project exists only if you have not yet cleaned up the scratch schema; do this before Step 3.22):

```
snow dcm plan --target SCRATCH --from dcm/lab3_scratch -c sbx_dev --role SBX_CALC_DEV --warehouse SBX_CALC_WH
```

You should see: a privilege error. Record the exact text.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
GRANT PLAN ON DCM PROJECT SBX_INT.LAB3_SCRATCH.LAB3_DCM TO ROLE SBX_CALC_DEV;
```

Repeat the plan.

You should see: either a plan, or an error about a temporary stage. Both are findings. The CLI uploads your sources to a temporary stage in the project's schema, so the role also needs a privilege to create one there.

The key facts to write in your notes:

| Privilege on the project | Lets the role |
|---|---|
| READ | list and describe the project, `SHOW ENTITIES` |
| MONITOR | read deployment history and files (includes READ) |
| PLAN | run `EXECUTE DCM PROJECT ... PLAN` (includes MONITOR) |
| OWNERSHIP | deploy, preview, test, drop, alter, grant the others |

PLAN runs with the privileges of the project owner role, not the caller. So a role with PLAN can read the DDL of anything the owner can read, through the plan output. Grant it deliberately.

> **VERIFIED:** privilege hierarchy READ < MONITOR < PLAN < OWNERSHIP, and PLAN executes with the owner role's privileges (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use, read 2026-11-16).

> **TO-CONFIRM [TC-05-07]:** the exact error texts above and whether `SBX_CALC_DEV` needs an extra stage privilege on `LAB3_SCRATCH`. Record the results in file 13.

Ownership is the other half. Everything DCM deploys is owned by the project owner role. Adoption works only because Lab 1 created the objects as `SBX_CICD_PIPELINE`. If a Lab 1 object had been created as `SBX_CALC_DEV`, the adoption plan or deploy would fail until the pipeline role held that role (the pipeline role does hold `SBX_CALC_DEV`, see bootstrap) and ownership was handed over. NAIC met this: objects made by `SYSADMIN` or by an AZ role blocked clones and adoption until everything was moved to `CICD_PIPELINE`.

> **TO-CONFIRM [TC-05-08]:** the literal DCM error text for adopting an object owned by a role the project owner does not hold. No such message was quoted in the sources used for this pack. Optional experiment: as `SBX_CALC_DEV` create a table in `LAB3_SCRATCH`, define it in the scratch project and plan. Record the message in file 13.

### Part E: what DCM cannot manage, each proven (about 40 min)

All experiments use the scratch project (`dcm/lab3_scratch`). Reset `scratch.sql` to the version that deployed cleanly (the sequence, `DEMO_ORDERS` with `ORDER_NOTE`, no `DEMO_PRECIOUS`) before each one, and run `snow dcm plan` only unless the step says deploy. Use the same flags as Step 3.2.

### Step 3.16 Hybrid tables are not supported

Run in: **Terminal**. Append to `scratch.sql`, then plan:

```sql
DEFINE HYBRID TABLE {{db}}.LAB3_SCRATCH.DEMO_HYBRID (
    ID NUMBER(38,0) NOT NULL,
    primary key (ID)
);
```

You should see: a syntax error from the plan (the NAIC POC saw `unexpected 'HYBRID'`). This works in both Mode H and Mode S, because it is a parse failure, not a table creation. Remove the block again.

> **VERIFIED:** hybrid tables are not in the supported-entities list (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities, read 2026-11-16). The error text above comes from the NAIC POC. How to check at lab time: your own plan output. Record the exact text in file 13.

Covered by: Flyway (Lab 4). In NAIC this is why `CALC_WORKFLOW_STATUS` and `CALC_AUDIT_EVENT` sit in the Flyway lane.

### Step 3.17 Row data is not supported

Append to `scratch.sql` and plan:

```sql
INSERT INTO {{db}}.LAB3_SCRATCH.DEMO_ORDERS (ORDER_ID) VALUES (99);
```

You should see: a rejection, because definition files allow only `DEFINE`, `GRANT` and `ATTACH`. Remove the block. Covered by: Flyway data-change migrations (Lab 4), which in NAIC replaced the hand-built `data_changes/` runner and its `DATA_CHANGE_LOG` ledger.

### Step 3.18 Sequence counters on redeploy

Run in: **Snowsight worksheet**:

```sql
SELECT SBX_INT.LAB3_SCRATCH.SEQ_DEMO.NEXTVAL;
SELECT SBX_INT.LAB3_SCRATCH.SEQ_DEMO.NEXTVAL;
SELECT SBX_INT.LAB3_SCRATCH.SEQ_DEMO.NEXTVAL;
```

Run in: **Terminal**. Make an unrelated change (add `ORDER_TAG VARCHAR(10),` to `DEMO_ORDERS`), deploy with `--alias scratch-seq1`. In the worksheet call `NEXTVAL` once and write down the value. Then change the sequence definition from `increment by 1` to `increment by 10`, deploy with `--alias scratch-seq2`, and call `NEXTVAL` once more.

You should see: the first value after the unrelated change continues from the old counter (`103` if you started at 100 and called it three times). After the sequence change, record what happens to the counter.

> **TO-CONFIRM [TC-G05]:** whether a sequence definition change keeps or resets the counter. The docs do not say. This matters in NAIC because the 8 engine sequences must not reset (reuse of `CALC_ID` values would be a data bug). How to check at lab time: the experiment above. Record the result in file 13.

### Step 3.19 Rename is a drop and a create

Run in: **Terminal**. In `scratch.sql` rename `DEMO_ORDERS` to `DEMO_ORDERS_V2` (change the name in the `DEFINE` line only), then plan and read it:

```
python3 scripts/plan_summary.py dcm/lab3_scratch/out/plan_result.json
```

You should see: a DROP for `DEMO_ORDERS` and a CREATE for `DEMO_ORDERS_V2`. DCM sees two different objects. It has no notion of "same table, new name". If you deploy (do so in this scratch schema), the new table is empty and the 3 rows are gone, unless you recover with `UNDROP TABLE SBX_INT.LAB3_SCRATCH.DEMO_ORDERS;`.

> **TO-CONFIRM [TC-G06]:** the docs do not describe renames at all. The drop plus create reading is inferred by NAIC and the architecture's rename protocol; this experiment is the proof. Record the plan text in file 13.

Covered by: an explicit Flyway rename plus a coordinated DCM definition change. You do that in Lab 4 with `FORMULA_CATEGORIES` becoming `CALC_CATEGORIES`.

### Step 3.20 Users, resource monitors and other account objects

Append each of these on its own to `scratch.sql` and plan. They are expected to be rejected:

```sql
DEFINE RESOURCE MONITOR SBX_ZZ_RM WITH CREDIT_QUOTA = 1;
DEFINE USER SBX_ZZ_USER;
```

> **TO-CONFIRM [TC-05-09]:** the exact errors. The supported-entities page lists no users, integrations, resource monitors or clones, and says a grant that mentions a user does not mean DCM manages the user. Record the messages in file 13. Covered by: the platform lane (Terraform in NAIC, the bootstrap SQL in the sandbox).

### Step 3.21 Grant subtleties

Two experiments in the scratch project.

1. DCM ignores grants it did not deploy. Hand-grant `SELECT` on `DEMO_ORDERS_V2` (or `DEMO_ORDERS`, whichever exists) to `SBX_CALC_READER`, then plan. You should see no change for that grant. (The NAIC POC found about 214 Terraform-made future grants on the engine objects that never appeared in plans.)
2. `GRANT OWNERSHIP` with pre-existing grants. While that hand grant exists, append `GRANT OWNERSHIP ON TABLE {{db}}.LAB3_SCRATCH.DEMO_ORDERS_V2 TO ROLE SBX_CALC_DEV;` and plan. The project owner (`SBX_CICD_PIPELINE`) holds `SBX_CALC_DEV` through the bootstrap hierarchy, so the owner-lockout check passes. Expect the plan to fail because the table has other grants.

> **VERIFIED:** `COPY CURRENT GRANTS` and `REVOKE CURRENT GRANTS` are not available for `GRANT OWNERSHIP` in DCM, and a pre-existing grant makes PLAN or DEPLOY of the ownership grant fail unless the grant is also defined or revoked first (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-supported-entities, read 2026-11-16).

> **TO-CONFIRM [TC-G25]:** the observed behaviour and error text in your account. Record them in file 13.

### Part F: what `DROP DCM PROJECT` does (about 10 min)

### Step 3.22 Drop the project and look at what is left

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`), against the scratch project only:

```sql
SELECT COUNT(*) FROM SBX_INT.LAB3_SCRATCH.DEMO_ORDERS_V2;
DROP DCM PROJECT SBX_INT.LAB3_SCRATCH.LAB3_DCM;
SHOW TABLES IN SCHEMA SBX_INT.LAB3_SCRATCH;
```

You should see: the table is still there and can still be queried. Dropping the project object removes the management and the deployment history, not the managed objects. (This is what happened in NAIC INT on 2026-11-11: engine objects, rows and grants stayed in place and unmanaged.) What you lose is the project's memory of what it managed, so a later project must adopt the objects again, which is why the Part B adoption routine matters.

> **VERIFIED:** dropping the project object alone leaves managed objects in place as unmanaged and deletes the deployment-history artifacts; `EXECUTE DCM PROJECT <p> PURGE` drops everything it manages (non-production only) (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-manage, read 2026-11-16).

> **TO-CONFIRM [TC-G26]:** the flags of `snow dcm purge`, `drop` and `drop-deployment`. Check with `snow dcm purge --help` before using them; this lab uses the SQL `DROP DCM PROJECT`.

## 6. Checkpoint questions

1. What is the difference between a DCM project object and the definition files in Git?
2. After a clean deploy, what must a fresh full plan show, and why do you run it?
3. You delete `FORMULAS.sql` in a merge by mistake. What does the next deploy do, and what will the plan not tell you?
4. Why can a saved plan not be treated as an approval of what a deploy will do?
5. Why did adoption of the Lab 1 objects work without ownership changes?
6. Your colleague hand-created a hybrid table in the same schema. Does DCM drop it on the next deploy? Why or why not?
7. Why does the manifest's CLONE target point at a database that does not exist?
8. Name three things DCM cannot manage and the tool that covers each.

<details>
<summary>Answers</summary>

1. The project object lives in Snowflake and holds the deployment history and the record of managed objects; the files are the desired state you give it each time.
2. An empty changeset. It proves the live state equals the definitions, including after any out-of-band change.
3. It drops the table (and its rows) because it is managed and no longer defined. The plan shows a DROP but not how many rows are lost. Time Travel `UNDROP` is the recovery for a standard table.
4. Deploy recomputes its own changeset from the live state at deploy time (Step 3.6).
5. Lab 1 created them as `SBX_CICD_PIPELINE`, which is the project owner role.
6. No. DCM only drops objects it manages. An object that was never defined is invisible to the project.
7. So that forgetting the `-D` override fails with "database not found" instead of deploying over INT.
8. Hybrid tables, row data and renames go to Flyway; users and resource monitors go to the platform lane (Terraform or bootstrap SQL). Also acceptable: grants on objects it does not own, and package or Streamlit artifacts (artifact pipeline).

</details>

## 7. Break it

Scenario: a developer hot-fixes a procedure in Snowsight, as the team used to do in INT.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`): edit `SP_ADD_CATEGORY` by hand with `CREATE OR REPLACE PROCEDURE`, changing only the `RETURN` text to `'HOTFIX'`. Call it and see `HOTFIX`.

Run in: **Terminal**: plan.

Symptom: the plan shows one ALTER on `SP_ADD_CATEGORY` that you did not make in Git.
Diagnosis: the live object differs from the definition (drift), so DCM wants to restore the definition. A hot fix that is not in Git is reverted by the next deploy. This is NAIC risk number 5, "no hand edits in PROD".
Recovery: decide. If the hot fix is wanted, copy it into `procedures/SP_ADD_CATEGORY.sql`, plan (it should be empty or adoption-only), deploy. If not, deploy the definition as it is and confirm the procedure returns `OK: RATIO` again. Then check the grant to `SBX_CALC_READER` (Step 3.11): a `CREATE OR REPLACE` may have removed it.

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| Removing a definition drops the object and its data | Step 3.7 | Plan review plus an exact-FQN DROP check in CI (Lab 6); `UNDROP` for standard tables | VERIFIED (docs) and observed |
| A plan shows structure only, not row counts | Step 3.7 | Evidence queries before and after; backups | observed |
| Deploy does not replay the plan | Step 3.6 | Fresh full plan immediately before deploy; check the deploy result | VERIFIED (docs) |
| Hybrid tables not supported | Step 3.16 | Flyway | VERIFIED (supported list); error text TO-CONFIRM |
| No row data, no data fixes | Step 3.17 | Flyway data migrations | VERIFIED (only DEFINE, GRANT, ATTACH allowed) |
| Rename is drop plus create | Step 3.19 | Flyway rename plus coordinated definition change (Lab 4) | TO-CONFIRM TC-G06 |
| One configuration per project object | Step 3.14 | One project object per environment | VERIFIED (docs) |
| Users, resource monitors, integrations not manageable | Step 3.20 | Platform lane | VERIFIED (supported list); error text TO-CONFIRM |
| `GRANT OWNERSHIP` fails with pre-existing grants | Step 3.21 | Revoke or define the other grants first | VERIFIED (docs); behaviour TO-CONFIRM TC-G25 |
| Changed procedure body is a full replace, with no body diff in the plan | Step 3.11 | Code review of the definition file in Git | VERIFIED (docs) |
| Plan validates creation, not runtime success | Step 3.12 | Smoke tests after deploy (Lab 5, Lab 6) | VERIFIED (docs) |
| Sequence counter behaviour on redeploy unknown | Step 3.18 | Check before relying on it | TO-CONFIRM TC-G05 |

## 9. What you learned

- DCM is declarative: you edit the end state, a plan lists CREATE, ALTER and DROP, and an empty plan means in sync.
- Adoption of existing objects needs a definition that matches the live object, an owner role match, and three checks: no DROP, no real change, empty re-plan.
- Deploy recomputes. A plan is for review, not a promise.
- Removing a definition drops the object. The plan does not show the data at risk.
- DCM does not manage hybrid tables, row data, renames, users or monitors. Those need Flyway or the platform lane (Lab 4).

## 10. Cleanup

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
DROP DCM PROJECT IF EXISTS SBX_INT.LAB3_SCRATCH.LAB3_DCM;
DROP SCHEMA IF EXISTS SBX_INT.LAB3_SCRATCH;
REVOKE USAGE ON PROCEDURE SBX_INT.CALC_SUPPORT.SP_ADD_CATEGORY(VARCHAR, VARCHAR) FROM ROLE SBX_CALC_READER;
```

Run in: **Terminal**: delete the local saved plans with `rm -rf dcm/lab3_scratch/out dcm/calc_engine/out`. Keep `dcm/lab3_scratch/` as a reference if you like; it is never deployed again.

Keep: the INT project `CALC_ENGINE_DCM`, all `dcm/calc_engine/` files. Cost note: DCM plan and deploy are mostly metadata work. Warehouse use here is a few seconds per command.

Learner action (git, you run it): create a branch and commit the new files `dcm/calc_engine/**`, `dcm/lab3_scratch/**` and `scripts/plan_summary.py` with a message such as `feat: adopt lab1 objects into dcm project`. The assistant does not run git for you.

## 11. State left behind

- Project `SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM` exists. Deployments `int-adopt`, `int-proc-edit`, `int-complete` (and `int-grants`) are in its history. A fresh full re-plan is empty.
- `dcm/calc_engine/` holds the full manifest and definitions that adopt ALL Lab 1 objects, plus: sequences `SEQ_FORMULA_ID`, `SEQ_EXECUTION_ID`, `SEQ_WORKFLOW_ID`, `SEQ_AUDIT_EVENT_ID`; tables `EXECUTIONS`, `DATA_CHANGE_LOG`; view `ENV_CONFIG`; function `FN_APPLY_REDIRECTS`; procedures `SP_ACTIVATE_DRAFT_CALCULATION`, `SP_COMPUTE_MINI_RESULT`; `grants/access.sql`.
- `SBX_INT.LAB3_SCRATCH` and `LAB3_DCM` are dropped.
- New helper: `scripts/plan_summary.py`.

## 12. Open items for file 13

TC-G02, TC-G03, TC-G04, TC-G05, TC-G06, TC-G16, TC-G17, TC-G20, TC-G25, TC-G26, TC-05-01 (SHOW ENTITIES and SHOW DEPLOYMENTS columns), TC-05-02 (hand-added column drift behaviour), TC-05-03 (UNDROP of a dropped managed table), TC-05-04 (adoption plan shape), TC-05-05 (grant survives procedure edit), TC-05-06 (SQL function body), TC-05-07 (PLAN privilege and temp stage errors), TC-05-08 (ownership error text), TC-05-09 (unsupported DEFINE error texts), TC-05-10 (type string for the unmanage function), TC-05-11 (`SHOW GRANTS` form for a DCM project). New names introduced here for the contract: helper `scripts/plan_summary.py`; the scratch manifest target `SCRATCH`; template variable `db` in `dcm/lab3_scratch`; `ENV_CONFIG` row key `RESULT_SCHEMA`.

## 13. Next

Continue with [06-lab4-both-together-ownership.md](06-lab4-both-together-ownership.md).
