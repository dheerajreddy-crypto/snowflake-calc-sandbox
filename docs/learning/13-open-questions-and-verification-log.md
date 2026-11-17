# 13. Open questions and verification log

> **File no:** 13 | **Kind:** reference | **Prerequisites:** none | **Read date for public docs facts:** 2026-11-16 | **Modes:** all

The author wrote this pack without access to a Snowflake account, so no command in it was run. Part A lists every fact that was **not** verified against a public page, with the check to run at lab time. Part B lists what **was** read on a public page and when. Fill the "Result at lab time" line of each item as you meet it. If a result differs from the docs, the lab text is wrong, not you: tell the assistant and fix the lab file.

Honest limits:

- Nothing here was executed. "Expected if the docs are right" is a prediction, not an observation.
- Facts marked VERIFIED were read on 2026-11-16 and can change. Re-check the pins (Snowflake CLI 3.33.0, Flyway 13.12.2, the image digest) at lab time.
- Snowflake error texts and numeric error codes are quoted only where a public page gives them. Others are described, and you paste the real text.
- Each block below lists the lab files that cite the id. Where the list is short, the id is used in one place only.

## Probe results (fill in during file 02)

| Probe | Date | Result | Mode set | Exact error text (paste) |
|---|---|---|---|---|
| Probe H (`CREATE HYBRID TABLE`) | 2026-11-17 | Failed as expected | `HYBRID_MODE` = S | `391404 (0A000): Hybrid tables are currently not available to trial accounts.` (run as SBX_CICD_PIPELINE through sbx_owner; query id 01c87a8c-0309-d9e2-0013-1d5200039022) |
| Probe D (`CREATE DCM PROJECT`) | 2026-11-17 | Worked | `DCM_MODE` = on | `ZZ_DCM_PROBE successfully created.` in SBX_INT.LAB3_SCRATCH (drop pending) |
| Snowflake CLI version (`snow --version`) | 2026-11-16 | 3.33.0 | | |
| Flyway version (`flyway -v`) | 2026-11-16 | 13.12.2 (image `flyway/flyway:13.12.2`) | | |

## Part A. Open questions

Order: global ids `TC-G..` first, then file-local ids by file number.

### TC-G01: Exact error when creating a hybrid table in a trial
- Where used: file 02, file 09, file 12
- What we could not verify: Exact error when creating a hybrid table in a trial.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Run Probe H, copy the error; Run `CREATE HYBRID TABLE` in the trial and paste the error. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-17 | CONFIRMED | `391404 (0A000): Hybrid tables are currently not available to trial accounts.` HYBRID_MODE = S.

### TC-G02: Whether CREATE DCM PROJECT works in a trial (see result at the end of this block)
- Where used: file 02, file 05
- What we could not verify: Whether CREATE DCM PROJECT works in a trial.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Run Probe D; Step 3.2: `snow dcm create ...`; copy error if it fails. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-17 | CONFIRMED: works | `CREATE DCM PROJECT SBX_INT.LAB3_SCRATCH.ZZ_DCM_PROBE;` via sbx_owner returned "ZZ_DCM_PROBE successfully created." DCM_MODE = on. CREATE DCM PROJECT works in a trial for the pipeline role.

### TC-G03: Exact DEFINE grammar for PROCEDURE, SEQUENCE, FUNCTION, VIEW, SCHEMA, ROLE, WAREHOUSE
- Where used: file 05, file 09, file 11, file 12
- What we could not verify: Exact DEFINE grammar for PROCEDURE, SEQUENCE, FUNCTION, VIEW, SCHEMA, ROLE, WAREHOUSE.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: deploy; read the error; follow links on the supported-entities page; Supported-entities page links; `snow init --template dcm_project`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G04: Saved plan file name (plan_result.json vs plan.json)
- Where used: file 05, file 07, file 08, file 10, file 12
- What we could not verify: Saved plan file name (plan_result.json vs plan.json).
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: `ls dcm/lab3_scratch/out/` after the first plan; `ls dcm/calc_engine/out/`; `ls dcm/calc_engine/out/` after `--save-output`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G05: Sequence counter after DEFINE redeploy
- Where used: file 05
- What we could not verify: Sequence counter after DEFINE redeploy.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Step 3.18 experiment. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G06: Rename seen by DCM as drop + create
- Where used: file 05, file 06, file 10, file 11
- What we could not verify: Rename seen by DCM as drop + create.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Step 3.19 plan text; Step 4.9 protocol step 2; Read the plan after renaming a definition. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G07: DEFINE TABLE on an existing hybrid table (adopt, fail, alter)
- Where used: file 06
- What we could not verify: DEFINE TABLE on an existing hybrid table (adopt, fail, alter).
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Step 4.8, plan only, never deploy. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G08: Flyway parsing of $$ procedure and Scripting blocks with semicolons
- Where used: file 04, file 06, file 11
- What we could not verify: Flyway parsing of $$ procedure and Scripting blocks with semicolons.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Step 5.9; Step 4.7 `flyway migrate`; Run `flyway migrate` on `V4`/`V7` of the capstone. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G09: Default baselineVersion
- Where used: file 04, file 12
- What we could not verify: Default baselineVersion.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Flyway docs or `baseline` without the flag in a throwaway schema; Flyway docs. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G10: Per-environment placeholders in flyway.toml
- Where used: file 04, file 08, file 12
- What we could not verify: Per-environment placeholders in flyway.toml.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Avoided; env vars and `-placeholders.x` used; Run `flyway info` with `FLYWAY_PLACEHOLDERS_CALC_DB`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G11: -outputType=json flag name
- Where used: file 04, file 12
- What we could not verify: -outputType=json flag name.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Run `info` with the flag; `flyway -?`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G12: One-statement CREATE USER ... TYPE = SERVICE ... RSA_PUBLIC_KEY
- Where used: file 02
- What we could not verify: One-statement CREATE USER ... TYPE = SERVICE ... RSA_PUBLIC_KEY.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: `CREATE USER` reference page (pack uses three statements). The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | PARTLY CONFIRMED | The pack's three-statement form worked in the trial: CREATE USER SBX_SVC_CICD TYPE = SERVICE, GRANT ROLE, then ALTER USER ... SET RSA_PUBLIC_KEY. `DESC USER SBX_SVC_CICD` shows RSA_PUBLIC_KEY_FP starting with SHA256:. The one-statement form was not tried.

### TC-G13: OIDC/WIF users in trial accounts
- Where used: file 08
- What we could not verify: OIDC/WIF users in trial accounts.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Run the `CREATE USER ... WORKLOAD_IDENTITY` statement. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G14: GRANT CREATE HYBRID TABLE ON SCHEMA syntax and whether schema owners need it
- Where used: file 02, file 06, file 12
- What we could not verify: GRANT CREATE HYBRID TABLE ON SCHEMA syntax and whether schema owners need it.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: GRANT reference; line is commented in bootstrap; only if Step 4.3 fails with insufficient privileges; Grant reference page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G15: Database clone with hybrid tables and retention 0
- Where used: file 07
- What we could not verify: Database clone with hybrid tables and retention 0.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Steps 5.4 and 5.9. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G16: Minimum Snowflake CLI for DCM: 3.24.0 (docs) vs 3.16 (NAIC notes)
- Where used: file 02, file 05, file 08
- What we could not verify: Minimum Snowflake CLI for DCM: 3.24.0 (docs) vs 3.16 (NAIC notes).
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: `snow --version`, DCM usage page; `snow dcm --help` on the installed version; PyPI and docs at lab time. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G17: Official GA statement for DCM Projects
- Where used: file 05
- What we could not verify: Official GA statement for DCM Projects.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Snowflake release notes. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G18: Credit equivalence of the "$400" trial balance
- Where used: file 02
- What we could not verify: Credit equivalence of the "$400" trial balance.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Balance tile after sign-up. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | CONFIRMED in dollars | Learner reports "$400 left of $400". Credit equivalence still unknown; Snowsight showed dollars. The pack's 20-credit monitor is still the cost guard.

### TC-G19: First-login MFA form in a trial
- Where used: file 02, file 12
- What we could not verify: First-login MFA form in a trial.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Observe at sign-up. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | CONFIRMED (partial) | MFA enrolled with Microsoft Authenticator. The sign-up flow gave no edition/cloud/region choice; defaults were assigned.

### TC-G20: Manifest layout for env_vars / env_secrets
- Where used: file 05
- What we could not verify: Manifest layout for env_vars / env_secrets.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: docs page dcm-projects-files. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G21: Key-pair JVM flag for encrypted keys in Flyway 13.12.2
- Where used: file 02, file 04, file 06, file 08, file 12
- What we could not verify: Key-pair JVM flag for encrypted keys in Flyway 13.12.2.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: If `info` fails with key decryption error; `flyway info` error; retry with JAVA_OPTS; Run `scripts/flyway_ci.sh info`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G22: Current SHA and inputs of snowflakedb/snowflake-actions DCM sub-actions
- Where used: file 08, file 12
- What we could not verify: Current SHA and inputs of snowflakedb/snowflake-actions DCM sub-actions.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Action README on GitHub. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G22b: actions/checkout version to pin
- Where used: file 08
- What we could not verify: actions/checkout version to pin.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Newest release on GitHub. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G23: TIMESTAMP_TZ in secondary hybrid indexes
- Where used: no lab step cites it as written (kept so the id is not lost)
- What we could not verify: TIMESTAMP_TZ in secondary hybrid indexes.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: see the TO-CONFIRM callout in the file(s) above. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G24: Edition requirement for hybrid tables
- Where used: file 02
- What we could not verify: Edition requirement for hybrid tables.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: "Hybrid tables clouds and regions" page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G25: GRANT OWNERSHIP inside DCM with pre-existing grants
- Where used: file 05, file 12
- What we could not verify: GRANT OWNERSHIP inside DCM with pre-existing grants.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Step 3.21 experiment; Try it and paste the text. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G26: Flags of snow dcm preview/purge/drop/drop-deployment/list-deployments
- Where used: file 05, file 12
- What we could not verify: Flags of snow dcm preview/purge/drop/drop-deployment/list-deployments.
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: `snow dcm <cmd> --help`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G27: connections.toml form for key-pair (private_key_file)
- Where used: file 02
- What we could not verify: connections.toml form for key-pair (private_key_file).
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: `snow connection add` for `sbx_cicd`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | CONFIRMED | The entry written by hand works: account, user, role, warehouse, database, `authenticator = "SNOWFLAKE_JWT"`, `private_key_file = "<absolute path>"`. `snow connection test -c sbx_cicd` returned Status OK for SBX_SVC_CICD, role SBX_CICD_PIPELINE, warehouse SBX_CALC_WH, with the passphrase in PRIVATE_KEY_PASSPHRASE. The encrypted key (aes-256-cbc) works with the CLI. `snow connection list` prints the connection parameters (no secrets are stored in the file).

### TC-G28: Flyway authentication with workload identity (not listed in docs)
- Where used: file 08
- What we could not verify: Flyway authentication with workload identity (not listed in docs).
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Flyway driver page and release notes. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-G29: IGNORE HYBRID TABLES when the schema has no hybrid tables (Mode S clone)
- Where used: file 07
- What we could not verify: IGNORE HYBRID TABLES when the schema has no hybrid tables (Mode S clone).
- Why it matters: The labs rely on this fact as written; a different result changes the step that uses it.
- How to verify at lab time: Step 5.2. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-02-01: Where Snowsight shows the account identifier
- Where used: file 02
- What we could not verify: Where Snowsight shows the account identifier.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Account menu. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | CONFIRMED | Snowsight account details lists account identifier (ORG-ACCOUNT form), account locator, cloud, edition and region. Account: AWS, AWS_US_EAST_2, Enterprise. Current user name is DHEERAJ.BANDA3 (login name is the email). The real identifier is kept in the learner's notes, not in the repo.

### TC-02-02: Core DDL in bootstrap (roles, DB, schema, warehouse, future grants, ownership by SECURITYADMIN) not re-read
- Where used: file 02
- What we could not verify: Core DDL in bootstrap (roles, DB, schema, warehouse, future grants, ownership by SECURITYADMIN) not re-read.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run the bootstrap. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | PARTLY CONFIRMED | Parts 1 to 4 ran without reported errors. `SHOW SCHEMAS IN DATABASE SBX_INT` shows owner SBX_CICD_PIPELINE for CALC_DEV_SUPPORT, CALC_SUPPORT, CAP_DCM, CAP_FLYWAY, CAP_HAND, LAB2_SCRATCH, LAB3_SCRATCH; PUBLIC owned by SYSADMIN; INFORMATION_SCHEMA has an empty owner. So role creation, DB/schema/warehouse creation, resource monitor, and GRANT OWNERSHIP by SECURITYADMIN on SYSADMIN-owned schemas all work. Part 5 also ran: FIN_CALCULATION_RESULT_BASE exists; SHOW GRANTS lists SBX_CALC_DEV SELECT through the part-4 future grant. Part 6 ran (service user created, key registered). Part 7 checks: DESC USER done.

### TC-02-03: Python versions supported by CLI 3.33.0
- Where used: file 02
- What we could not verify: Python versions supported by CLI 3.33.0.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `pip install` result, PyPI. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | CONFIRMED | macOS, Python 3.14.7, venv at ~/.snowflake-sandbox/venv. `pip install "snowflake-cli==3.33.0"` succeeded (pyyaml built from source). `snow --version` prints 3.33.0. DIFFERENCE: `snow dcm --help` lists 9 commands (create, deploy, describe, drop, drop-deployment, list, list-deployments, plan, purge). File 02 expects `preview` and `test` as well. They are absent in 3.33.0, so later labs that use them need a check (see TC-02-03b).

### TC-02-03b: `snow dcm preview` and `snow dcm test` missing in CLI 3.33.0
- Where used: file 02 (expected command list), later DCM labs and workflows that call them
- What we could not verify: whether these commands exist under another name or in a newer CLI.
- How to verify at lab time: `snow dcm --help` after each upgrade; the DCM command reference page.
- Result at lab time (learner fills): 2026-11-16 | OPEN | Not in `snow dcm --help` on 3.33.0. Resolve before Lab 3 (file 05).

### TC-02-04: Docker image `-v` flag and env pass-through in `fw`
- Where used: file 02, file 04
- What we could not verify: Docker image `-v` flag and env pass-through in `fw`.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `docker run --rm redgate/flyway:13.12.2 -v`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | DIFFERS, fixed | Docker 29.7.2 on an Apple Silicon Mac (arm64). (1) `docker` is not on PATH after installing Docker Desktop; binary is /Applications/Docker.app/Contents/Resources/bin/docker. Pulls fail with "docker-credential-desktop not found" unless that folder is on PATH. (2) `redgate/flyway:13.12.2` has NO linux/arm64 manifest ("no match for platform"). (3) `flyway/flyway:13.12.2` works on arm64: prints "Flyway OSS Edition 13.12.2 by Redgate". DECISION: use image `flyway/flyway:13.12.2` in the `fw` helper instead of `redgate/flyway`. The `-v` flag works. Env pass-through of `fw` is not yet tested (needs a Flyway project, Lab 2).

### TC-02-05: How interactive login works for the human user
- Where used: file 02
- What we could not verify: How interactive login works for the human user.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `snow connection test -c sbx_dev`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): 2026-11-16 | DIFFERS | `snow connection test -c sbx_dev` with no authenticator set fails with "251006: Password is empty". The CLI did not prompt for a password or MFA code. Second finding: the key-pair login for the learner's user failed with 390144 and SYSTEM$GET_LOGIN_FAILURE_DETAILS errorCode JWT_TOKEN_INVALID_USER_IN_ISSUER (username null) when `user` was the user NAME (dheeraj.banda3). The CLI needs the LOGIN NAME (the email-form `<LOGIN_NAME>`, not the user name) in `user`; `connections.toml` was changed accordingly (retest: `snow connection test` passes for sbx_dev and sbx_owner, 2026-11-17). This also means the pack's `<LEARNER_USER>` in connections.toml must be the login name, while the bootstrap SQL uses the user name. Fallback chosen: a second key pair for the learner's own user (own passphrase, outside the repo), registered with ALTER USER ... SET RSA_PUBLIC_KEY_2, used by sbx_dev and sbx_owner. Result: sbx_dev and sbx_owner now log in with the personal key pair (RSA_PUBLIC_KEY_2) and the login name. No password or MFA prompt. OUTCOME: CONFIRMED.

### TC-02-06: Learner's GitHub plan for environments
- Where used: file 02
- What we could not verify: Learner's GitHub plan for environments.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: GitHub account settings. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-03-01: Lab 1 SQL syntax (Scripting `SELECT ... INTO :v`, DEFAULT seq.NEXTVAL, INFORMATION_SCHEMA) not re-read
- Where used: file 03
- What we could not verify: Lab 1 SQL syntax (Scripting `SELECT ... INTO :v`, DEFAULT seq.NEXTVAL, INFORMATION_SCHEMA) not re-read.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run it. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-03-02: Reach and columns of `INFORMATION_SCHEMA.QUERY_HISTORY`
- Where used: file 03
- What we could not verify: Reach and columns of `INFORMATION_SCHEMA.QUERY_HISTORY`.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.8 output and docs. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-04-01: `FLYWAY_URL` overrides `[environments.target] url` in TOML
- Where used: file 04
- What we could not verify: `FLYWAY_URL` overrides `[environments.target] url` in TOML.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.2; fallback `-url=`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-04-02: Columns and wording of `flyway info` on Snowflake
- Where used: file 04
- What we could not verify: Columns and wording of `flyway info` on Snowflake.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.2 output. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-04-03: History table case on Snowflake (`FLYWAY_SCHEMA_HISTORY` vs quoted lower case)
- Where used: file 04
- What we could not verify: History table case on Snowflake (`FLYWAY_SCHEMA_HISTORY` vs quoted lower case).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `SHOW TABLES`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-04-04: Wording of Flyway's refusal on a non-empty schema without history
- Where used: file 04
- What we could not verify: Wording of Flyway's refusal on a non-empty schema without history.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.11. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-01: column names of SHOW ENTITIES / SHOW DEPLOYMENTS
- Where used: file 05
- What we could not verify: column names of SHOW ENTITIES / SHOW DEPLOYMENTS.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: read the result grid. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-02: what the ALTER does with a live column that is not defined
- Where used: file 05
- What we could not verify: what the ALTER does with a live column that is not defined.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: open `changes` in the plan JSON, Step 3.5. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-03: UNDROP of a DCM-dropped table: managed again?
- Where used: file 05
- What we could not verify: UNDROP of a DCM-dropped table: managed again?.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 3.7 `SHOW ENTITIES` after UNDROP. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-04: shape of an adoption plan (ALTER "set PROJECT" is a NAIC POC observation, not public docs)
- Where used: file 05
- What we could not verify: shape of an adoption plan (ALTER "set PROJECT" is a NAIC POC observation, not public docs).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 3.9 plan JSON. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-05: hand-made grant survives a procedure change
- Where used: file 05
- What we could not verify: hand-made grant survives a procedure change.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 3.11 `SHOW GRANTS`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-06: scalar SQL function body with a subquery reading its arguments
- Where used: file 05
- What we could not verify: scalar SQL function body with a subquery reading its arguments.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 3.12 test SELECT. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-07: PLAN privilege and temp-stage error texts for a non-owner role
- Where used: file 05
- What we could not verify: PLAN privilege and temp-stage error texts for a non-owner role.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 3.15. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-08: literal error when adopting an object owned by a role the project owner lacks
- Where used: file 05
- What we could not verify: literal error when adopting an object owned by a role the project owner lacks.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: optional experiment in Step 3.15. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-09: error texts for DEFINE RESOURCE MONITOR / DEFINE USER and for DEFINE HYBRID TABLE
- Where used: file 05
- What we could not verify: error texts for DEFINE RESOURCE MONITOR / DEFINE USER and for DEFINE HYBRID TABLE.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Steps 3.16 and 3.20. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-10: type string for `SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT` with a table
- Where used: file 05
- What we could not verify: type string for `SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT` with a table.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: run it; read error. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-05-11: form of `SHOW GRANTS` for a DCM project object
- Where used: file 05
- What we could not verify: form of `SHOW GRANTS` for a DCM project object.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 3.15. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-06-01: `FLYWAY_URL` / `FLYWAY_LOCATIONS` env vars override the TOML environment
- Where used: file 06
- What we could not verify: `FLYWAY_URL` / `FLYWAY_LOCATIONS` env vars override the TOML environment.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 4.1 `flyway info`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-06-02: duplicate-key error text on a hybrid table
- Where used: file 06
- What we could not verify: duplicate-key error text on a hybrid table.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 4.5. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-06-03: how DCM reports the old entity after an out-of-band rename
- Where used: file 06
- What we could not verify: how DCM reports the old entity after an out-of-band rename.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 4.9 protocol step 5. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-06-04: states in the conflict experiment (failed V1, CREATE OR REPLACE on a managed table, plan text)
- Where used: file 06
- What we could not verify: states in the conflict experiment (failed V1, CREATE OR REPLACE on a managed table, plan text).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 4.11. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-01: error number/text of a schema clone with hybrid tables (docs 391411 for a table clone, NAIC 392105 for a schema clone)
- Where used: file 07
- What we could not verify: error number/text of a schema clone with hybrid tables (docs 391411 for a table clone, NAIC 392105 for a schema clone).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.1 (Mode H). The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-02: `SHOW DCM PROJECTS ... IN SCHEMA` form; whether a schema clone copies the DCM project
- Where used: file 07
- What we could not verify: `SHOW DCM PROJECTS ... IN SCHEMA` form; whether a schema clone copies the DCM project.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.2. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-03: UNDROP error text for a database with retention 0
- Where used: file 07
- What we could not verify: UNDROP error text for a database with retention 0.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.5. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-04: developer role `CREATE DATABASE` error text
- Where used: file 07
- What we could not verify: developer role `CREATE DATABASE` error text.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.6. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-05: Snowflake Scripting syntax in `CREATE_CALC_DEV_CLONE`: `FOR rec IN <RESULTSET>`, `RESULTSET := (EXECUTE IMMEDIATE :q)`, the argument-signature `REGEXP_REPLACE`, `SELECT ... INTO` from `RESULT_SCAN(LAST_QUERY_ID())`, INFORMATION_SCHEMA FUNCTIONS/PROCEDURES columns
- Where used: file 07
- What we could not verify: Snowflake Scripting syntax in `CREATE_CALC_DEV_CLONE`: `FOR rec IN <RESULTSET>`, `RESULTSET := (EXECUTE IMMEDIATE :q)`, the argument-signature `REGEXP_REPLACE`, `SELECT ... INTO` from `RESULT_SCAN(LAST_QUERY_ID())`, INFORMATION_SCHEMA FUNCTIONS/PROCEDURES columns.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.7 deploy and Step 5.9 call; read `sqlerrm`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-06: composition of the first clone plan (adoption entries, ENV_CONFIG, GRANTs)
- Where used: file 07
- What we could not verify: composition of the first clone plan (adoption entries, ENV_CONFIG, GRANTs).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.10. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-07: `snow sql -D name=value` with `<% name %>` templating
- Where used: file 07, file 08
- What we could not verify: `snow sql -D name=value` with `<% name %>` templating.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Step 5.12. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-07-08: error text when redirects are forgotten and the developer role writes to INT
- Where used: file 07
- What we could not verify: error text when redirects are forgotten and the developer role writes to INT.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Break it. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-01: Which role runs `snow dcm` for the CLONE target in CI (`SBX_CICD_PIPELINE` vs `SBX_CALC_DEV`); alternative needs `GRANT ROLE SBX_CALC_DEV TO USER SBX_SVC_CICD`
- Where used: file 08
- What we could not verify: Which role runs `snow dcm` for the CLONE target in CI (`SBX_CICD_PIPELINE` vs `SBX_CALC_DEV`); alternative needs `GRANT ROLE SBX_CALC_DEV TO USER SBX_SVC_CICD`.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: First PR run; read any ownership error. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-02: Container user can read a mode-600 mounted key file; Flyway home directory
- Where used: file 08, file 12
- What we could not verify: Container user can read a mode-600 mounted key file; Flyway home directory.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run `flyway_ci.sh`; try `--user "$(id -u):$(id -g)"`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-03: Learner's GitHub plan shows reviewers on environments; `pull_request` merge checkout; fork PRs get no secrets; docker/jq/python3 on `ubuntu-24.04`; `always()` on cancel
- Where used: file 08, file 12
- What we could not verify: Learner's GitHub plan shows reviewers on environments; `pull_request` merge checkout; fork PRs get no secrets; docker/jq/python3 on `ubuntu-24.04`; `always()` on cancel.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Settings page; GitHub docs. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-04: Migrations replay on a fresh environment: `V3` constant `EXPECTED_ROWS` and `V4` rename without `IF EXISTS` fail on a new QA
- Where used: file 08, file 11
- What we could not verify: Migrations replay on a fresh environment: `V3` constant `EXPECTED_ROWS` and `V4` rename without `IF EXISTS` fail on a new QA.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run `promote-qa-prod` to `qa`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-05: `snow sql --format json` output shape (list of rows, upper-case keys); `INFORMATION_SCHEMA.DATABASES` and `.TABLES` columns
- Where used: file 08
- What we could not verify: `snow sql --format json` output shape (list of rows, upper-case keys); `INFORMATION_SCHEMA.DATABASES` and `.TABLES` columns.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run the commands by hand. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-06: Flyway `-target=<version>`
- Where used: file 08, file 11, file 12
- What we could not verify: Flyway `-target=<version>`.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Flyway docs or `-?`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-07: `snow sql -f ... -D name=value` with `<% %>` and `$$` blocks inside CI (same as TC-07-07)
- Where used: file 08
- What we could not verify: `snow sql -f ... -D name=value` with `<% %>` and `$$` blocks inside CI (same as TC-07-07).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Re-run Lab 5 step 5.12 first. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-08: Column names of the Flyway history table on Snowflake (lower-case quoted names)
- Where used: file 08, file 09, file 10
- What we could not verify: Column names of the Flyway history table on Snowflake (lower-case quoted names).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `SELECT * FROM <history table>`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-09: Flyway image digest `sha256:ccc7fe0...` (seen 2026-11-16)
- Where used: file 08
- What we could not verify: Flyway image digest `sha256:ccc7fe0...` (seen 2026-11-16).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Docker Hub tag page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-08-10: Flyway `ignoreMigrationPatterns='*:pending'` for `validate`
- Where used: file 08, file 12
- What we could not verify: Flyway `ignoreMigrationPatterns='*:pending'` for `validate`.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run with and without on a pending file. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-09-01: Duplicate key error text on a hybrid table
- Where used: file 09
- What we could not verify: Duplicate key error text on a hybrid table.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run step 5.3. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-09-02: `ALTER SESSION SET LOCK_TIMEOUT` and default wait
- Where used: file 09
- What we could not verify: `ALTER SESSION SET LOCK_TIMEOUT` and default wait.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Snowflake parameter page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-09-03: Mode S: update on a different row waits (table-level wait)
- Where used: file 09
- What we could not verify: Mode S: update on a different row waits (table-level wait).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run the two-session test. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-09-04: `CREATE INDEX <name> ON <table> (<cols>)` grammar for hybrid tables
- Where used: file 09
- What we could not verify: `CREATE INDEX <name> ON <table> (<cols>)` grammar for hybrid tables.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: CREATE INDEX reference page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-09-05: A grant cannot be limited to a column or a row
- Where used: file 09
- What we could not verify: A grant cannot be limited to a column or a row.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: GRANT reference page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-09-06: A plan reports a hand-revoked declared grant
- Where used: file 09
- What we could not verify: A plan reports a hand-revoked declared grant.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run the break-it commands. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-01: `SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT` arguments
- Where used: file 10, file 12
- What we could not verify: `SYSTEM$UNMANAGE_ENTITY_FROM_DCM_PROJECT` arguments.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: DCM manage page. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-02: Which JWT code a wrong key returns; failure-detail availability
- Where used: file 10, file 12
- What we could not verify: Which JWT code a wrong key returns; failure-detail availability.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `SYSTEM$GET_LOGIN_FAILURE_DETAILS`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-03: Plan result when only the definition is deleted and the `GRANT` stays
- Where used: file 10
- What we could not verify: Plan result when only the definition is deleted and the `GRANT` stays.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Try it in the clone. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-04: Plan treatment of an extra column not in the definition (dropped?)
- Where used: file 10
- What we could not verify: Plan treatment of an extra column not in the definition (dropped?).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Read `changes[]`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-05: Exact error when the `SBX_DEV_SET_ON_CLI` placeholder is used; outcome of the plan with the wrong default target
- Where used: file 10, file 12
- What we could not verify: Exact error when the `SBX_DEV_SET_ON_CLI` placeholder is used; outcome of the plan with the wrong default target.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run the two variants. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-06: Boolean runtime variable form (`-D "clone_redirects=false"`)
- Where used: file 10
- What we could not verify: Boolean runtime variable form (`-D "clone_redirects=false"`).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Plan output. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-07: Fractional `CREDIT_QUOTA`, `SYSTEM$WAIT`, monitor latency
- Where used: file 10, file 12
- What we could not verify: Fractional `CREDIT_QUOTA`, `SYSTEM$WAIT`, monitor latency.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `SHOW RESOURCE MONITORS`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-10-08: `SBX_CICD_PIPELINE` owns `FIN_CALCULATION_RESULT_BASE` and can grant on it
- Where used: file 10
- What we could not verify: `SBX_CICD_PIPELINE` owns `FIN_CALCULATION_RESULT_BASE` and can grant on it.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: `SHOW GRANTS ON TABLE ...`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-11-01: Hybrid `CREATE` with `NOT NULL AUTOINCREMENT` plus out-of-line `PRIMARY KEY`
- Where used: file 11
- What we could not verify: Hybrid `CREATE` with `NOT NULL AUTOINCREMENT` plus out-of-line `PRIMARY KEY`.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run the seed in `CAP_HAND`. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-11-02: Plan content after a hand-run rename (old entity gone, new table adopted)
- Where used: file 11
- What we could not verify: Plan content after a hand-run rename (old entity gone, new table adopted).
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Read the plan. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

### TC-11-03: `ALTER TABLE IF EXISTS ... RENAME TO` is a no-op for a missing table
- Where used: file 11
- What we could not verify: `ALTER TABLE IF EXISTS ... RENAME TO` is a no-op for a missing table.
- Why it matters: The step that uses it shows an expected output taken from docs or from the author's reading, not from a run.
- How to verify at lab time: Run it on a missing name. The callout in the lab file names the exact step.
- Expected if the docs are right: the output shown after the step, or the behaviour described in the callout.
- Fallback if it differs: use the fallback named in the callout; if none, record the real output here and ask the assistant to adapt the step.
- Result at lab time (learner fills): date | outcome | notes

## Further unverified items without their own id

These come from the author lists and the docs research. Each is flagged in the text with the words "note it" or "TO-CONFIRM" but has no separate id.

| Item | Where | How to verify at lab time | Result at lab time |
|---|---|---|---|
| Exact wording of Flyway errors for `validate` with ignored or missing migrations, for a repeated `migrate` after a failure, and for the `undo` refusal in Community edition | 04 | Run the step and paste the message | |
| Snowsight "Run all" behaviour with `$$` blocks (the pack uses `snow sql -f` for files) | 02, 03 | Run one multi-statement worksheet with a procedure body | |
| `--variable` quoting forms for `snow dcm` (inner single quotes for strings) | 05, 07, 10 | Try both forms from the CLI reference; see also TC-10-06 | |
| Flyway `executeInTransaction` and mixed transactions on Snowflake | 04 | Flyway docs page "executeInTransaction" | |
| Whether `snowflakedb/snowflake-cli-action` is deprecated in favour of `snowflake-actions` | 08 | Read the repository README | |
| Snowsight account identifier location (TC-02-01) and other file 02 setup items | 02 | See the TC-02-xx blocks above | |
| Learner user name contains a dot (`dheeraj.banda3`), so `<LEARNER_USER>` must be double-quoted in bootstrap lines 23 and 24 (TC-02-07) | 02 | Run `SHOW USERS;` and copy the name exactly; record whether the quoted form worked | 2026-11-16: learner ran parts 1 and 2 without reported errors |
| Whether secondary roles are active by default in new sessions (default secondary roles ALL). If yes, a session sees the union of all four granted roles' privileges, so role-restriction tests in Labs 7 and 8 may pass wrongly (TC-02-08) | 02, 09, 10 | `SHOW USERS` column `default_secondary_roles`, or `SELECT CURRENT_SECONDARY_ROLES();`; use `USE SECONDARY ROLES NONE;` before any role test | |

## Part B. VERIFIED log

Source: the author's docs research notes. "Date read" is 2026-11-16 for every row. Rows with partial reads say so.

| Doc URL | What was verified | Date read | Used in file(s) |
|---|---|---|---|
| docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-overview | all editions, cost, limits, key terms, migration tooling | 2026-11-16 | 01, 02, 05 |
| .../dcm-projects-use | CLI >= 3.24.0, privileges, PLAN/DEPLOY semantics, one config at a time | 2026-11-16 | 02, 05, 08 |
| .../dcm-projects-files | manifest v2 schema, folder layout, DEFINE rules, Jinja limits | 2026-11-16 | 05 |
| .../dcm-projects-supported-entities | entity list (no hybrid), table limits, OWNERSHIP grants | 2026-11-16 | 05, 06, 12 |
| .../dcm-projects-cicd | CI/CD flow, parallel developer pattern, GitHub actions, DROP detection | 2026-11-16 | 05, 08 |
| .../dcm-projects-manage | SHOW ENTITIES, unmanage functions, PURGE, DROP project | 2026-11-16 | 05, 10 |
| docs.snowflake.com/en/sql-reference/sql/execute-dcm-project | SQL grammar, plan JSON | 2026-11-16 | 05, 12 |
| docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dcm-commands/{overview,plan,deploy,create,list,describe,test} | flags | 2026-11-16 | 05, 08, 12 |
| docs.snowflake.com/en/user-guide/tables-hybrid, -limitations, -clone | availability, limits, clone, IGNORE HYBRID TABLES | 2026-11-16 | 02, 06, 07, 09 |
| docs.snowflake.com/en/sql-reference/sql/create-hybrid-table | DDL syntax | 2026-11-16 | 06, 09 |
| docs.snowflake.com/en/release-notes/bcr-bundles/2026_02/bcr-2157 | CREATE HYBRID TABLE privilege | 2026-11-16 | 02 |
| docs.snowflake.com/en/user-guide/admin-trial-account | trial duration, limits | 2026-11-16 | 02 |
| docs.snowflake.com/en/sql-reference/sql/create-resource-monitor, /user-guide/resource-monitors | syntax and pattern | 2026-11-16 | 02, 10 |
| docs.snowflake.com/en/user-guide/key-pair-auth, -troubleshooting | openssl, ADD KEY PAIR, RSA_PUBLIC_KEY, JWT errors | 2026-11-16 | 02, 10, 12 |
| docs.snowflake.com/en/user-guide/admin-user-management, security-mfa-rollout | TYPE=SERVICE, phases, trial exemption | 2026-11-16 | 02 |
| docs.snowflake.com/en/user-guide/workload-identity-federation | WIF concepts, drivers (partial read; GitHub specifics from CLI action page) | 2026-11-16 | 08 |
| docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/github-action; github.com/marketplace/actions/snowflake-actions | OIDC user DDL, inputs, key-pair env vars, pinning | 2026-11-16 | 08 |
| docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections | env vars, JWT connection, -x | 2026-11-16 | 02, 12 |
| documentation.red-gate.com (Snowflake driver, feature summary, commands, clean, baseline, repair, out-of-order, clean-disabled, schema-history, undo, docker, supported DBs, release notes) | see section 7 | 2026-11-16 | 02, 04, 06, 08, 12 |
| red-gate.com/hub/product-learning/flyway/using-toml-configuration-files-with-flyway | TOML vs CONF | 2026-11-16 | 02, 04, 06, 08, 12 |
| docs.github.com (manage-environments, concurrency) | environments, reviewers, plan limits, concurrency | 2026-11-16 | 08 |
| hub.docker.com API (redgate/flyway, flyway/flyway tag 13.12.2); pypi.org/pypi/snowflake-cli | tags exist; CLI 3.33.0 | 2026-11-16 | 02, 08 |

Facts verified from the research notes and used across files (all read 2026-11-16):

- DCM Projects are available in all editions; the Snowflake CLI needs 3.24.0 or newer for `snow dcm`.
- Hybrid tables are not supported in trial accounts (two pages), are generally available on AWS and Azure commercial regions, and are not in the DCM supported list.
- Flyway Community has no undo, no dry run and no drift report; `cleanDisabled` defaults to true.
- Key-pair authentication, `TYPE = SERVICE` users and the password phase-out exemption for trial accounts.
- GitHub environments with required reviewers, and `concurrency` groups.

## Next

Back to [index](index.md).
