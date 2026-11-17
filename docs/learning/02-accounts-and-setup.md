# 02. Accounts and setup (file 02)

> Lab no: setup | File no: 02 | Time: 2 to 3 hours | Prerequisites: 00, 01, an email address, a laptop with a terminal | Modes supported: H/S, D (this file finds your mode) | Leaves behind: trial account, bootstrap run, tools installed, repository with the folder tree, secrets set, hybrid and DCM probes done and recorded.

> **READ THIS BEFORE YOU START: trial limits.**
> 1. **Hybrid tables are not supported in Snowflake trial accounts** (VERIFIED on two Snowflake pages, read 2026-11-16). The default path is **Mode S**: standard tables stand in for the two hybrid tables. Primary-key enforcement, row-level locking and secondary indexes are then shown as read-only steps. Probe H below records the real error (TC-G01).
> 2. **DCM Projects are available in all editions, but trial availability is not stated** on the pages read. Probe D below tests it (TC-G02). If it fails, you are in **Mode D-off** and the DCM parts become read-along.
> 3. **The trial lasts 30 days or until the free credits run out, whichever comes first.** The balance is about $400 (TC-G18, unconfirmed). Plan the labs inside that window. The resource monitor `SBX_CALC_RM` (20 credits, a choice of this pack) protects the credits.
> 4. Upgrading to a paid account reaches Mode H but costs money. It is optional, never required.

## Order of work (read this first)

The headings below follow the topic order of the brief. Do the work in this order:

1. Create the trial account (section "Create the Snowflake trial account").
2. First login safety: MFA, no daily use of `ACCOUNTADMIN`.
3. Install the tools (section "Install the tools").
4. Generate the key pair for the service user (section "Service user and key pair").
5. Run the bootstrap SQL in a worksheet (last section), after replacing `<LEARNER_USER>` and `<PUBLIC_KEY_BODY>`.
6. Create the connections, then run `snow connection test`.
7. Create the GitHub repository, copy the pack into `docs/learning/`, set secrets and variables.
8. Run the two probes (hybrid, DCM) and record your mode.
9. Complete the verification checklist.

## Create the Snowflake trial account

### What the trial gives you, and what it does not

> **VERIFIED:** sign-up needs a valid email only, no payment details. You choose cloud platform, region and edition at sign-up. The trial ends 30 days after sign-up or when free credits are used up, whichever is first, and then the account is suspended (https://docs.snowflake.com/en/user-guide/admin-trial-account, read 2026-11-16).

> **VERIFIED:** features not available in trial accounts include external network access, **hybrid tables**, outbound private connectivity, Snowflake Openflow and Duo for MFA. AI features stay off until a credit card is added (same page, read 2026-11-16).

> **TO-CONFIRM [TC-G18]:** the free balance. Snowflake's sign-up page (seen only as a search snippet) says about $400 of free usage. The docs page gives no number. How to check at lab time: read the balance tile in Snowsight after sign-up. Record the result in file 13. Do not rely on a credit count.

### Decisions to record

Write these three decisions in the Notes column of the Resume table in file 00.

| Decision | Recommendation | Reason |
|---|---|---|
| Cloud | **AWS**, a commercial region near you | AWS and Azure are the clouds where hybrid tables are generally available; Google Cloud is preview only. Matters only if you upgrade later. |
| Region | Any commercial region near you; avoid special government regions | Hybrid tables are not available in US SnowGov regions. |
| Edition | **Enterprise** if the form offers it, otherwise **Standard** | Neither edition is required for DCM. Enterprise features may burn extra credits, so do not turn on optional features. |

> **TO-CONFIRM [TC-G24]:** whether hybrid tables need a particular edition. The pages read on 2026-11-16 state no edition requirement. How to check: open the Snowflake page "Hybrid tables clouds and regions" at lab time. Record in file 13.

> **VERIFIED:** hybrid tables are generally available in commercial AWS and Azure regions, in preview in Google Cloud regions, not available in US SnowGov regions, and "not supported in trial accounts" (https://docs.snowflake.com/en/user-guide/tables-hybrid-limitations, read 2026-11-16).

### DCM and hybrid tables in a trial: what to expect and the fallback

| Capability | Trial status | What the pack does |
|---|---|---|
| DCM Projects | Available in all editions. Trial availability is not stated explicitly on the pages read. | Probe D below tests it. Default is Mode D (on). |
| Hybrid tables | **Not supported in trial accounts** (two Snowflake pages say so). | Probe H below tests it. Default is **Mode S**: standard tables stand in for the two hybrid tables. |

> **VERIFIED:** DCM Projects are available for all Snowflake editions, with no DCM-specific cost beyond compute (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-overview, read 2026-11-16).

> **TO-CONFIRM [TC-G02]:** whether `CREATE DCM PROJECT` works in a trial account. How to check: run Probe D below. Record the exact error if it fails.

**Plan A (default, free): Mode S + D.** Everything runs except the steps that need real hybrid tables. In those steps you see a "read-only" block with the expected behaviour taken from the Snowflake documentation, plus a table of what differs.

**Plan B (optional, costs money): upgrade to reach Mode H.** Adding a credit card upgrades the account. Remaining free credits still apply (stated on the trial page you read; re-check at upgrade time). Use an AWS or Azure region. The resource monitor `SBX_CALC_RM` keeps working. Never treat the upgrade as required. If you switch modes later, you need a fresh Flyway history because the checksums of the `hybrid/` and `standard/` migrations differ. The labs that change under Plan B are 06, 07, 09 and 11.

**If DCM is unavailable (Mode D-off):** labs 05 to 11 become read-along for the DCM parts. The `CREATE OR ALTER` statement reproduces the per-object behaviour of `DEFINE` by hand (a `DEFINE` behaves like `CREATE OR ALTER`), so you can still see the diff idea.

### Password phases do not apply to trials

> **VERIFIED:** Snowflake's password phase-outs (mandatory MFA, no new legacy service users, passwords blocked for non-human users) do not apply to reader accounts, trial accounts or Snowflake Postgres; single-factor password sign-in continues to work there (https://docs.snowflake.com/en/user-guide/security-mfa-rollout, read 2026-11-16).

You still build a `TYPE = SERVICE` user with a key pair, because that is NAIC's reality (passwords for service users are being blocked there) and it is what your GitHub workflows will use.

## First login safety

1. Sign up with `dheeraj.banda3@gmail.com`. Pick a strong, unique password in a password manager.
2. Set up multi-factor authentication (MFA) on your human user. Duo is not available in trials; use an authenticator app or passkey, whatever the sign-up screen offers.

> **TO-CONFIRM [TC-G19]:** what the first-login MFA prompt looks like in a fresh trial and which factors are offered. How to check: observe at sign-up. Record in file 13.

3. Your first user holds `ACCOUNTADMIN`. Use it only for the bootstrap and the resource monitor. Snowflake recommends at least two `ACCOUNTADMIN` users and never making it the default role. The bootstrap changes your **default role** to `SBX_CALC_DEV`.
4. Write your account identifier into your own notes (not into any repository file). It has the form `<ORGANISATION>-<ACCOUNT_NAME>`. Wherever this pack writes `<ACCOUNT_IDENTIFIER>`, you substitute it locally.

> **TO-CONFIRM [TC-02-01]:** the exact place in Snowsight where the account identifier is shown. How to check: open the account menu at the bottom left and look for "Account details" or the account locator. Record in file 13.

## Cost controls

Cost is controlled by four things, all set by the bootstrap:

| Control | Value | Why |
|---|---|---|
| Warehouse `SBX_CALC_WH` | size `XSMALL`, `AUTO_SUSPEND = 60`, `AUTO_RESUME = TRUE`, `INITIALLY_SUSPENDED = TRUE` | Credits are used only while it runs. Docs recommend short auto-suspend. |
| Resource monitor `SBX_CALC_RM` | quota 20 credits, monthly, notify at 50 and 80 percent, suspend at 90, suspend immediately at 100 | Hard stop on runaway cost. The quota of 20 is this pack's choice, not a Snowflake default. |
| Clone retention | `DATA_RETENTION_TIME_IN_DAYS = 0` on clones and scratch schemas | Dropped clones leave no Time Travel storage. |
| Cleanup section in every lab | drop what you no longer need | Storage is deducted from the trial balance at on-demand rates. |

> **VERIFIED:** only `ACCOUNTADMIN` can create resource monitors. You assign a monitor to a warehouse with `ALTER WAREHOUSE <wh> SET RESOURCE_MONITOR = <rm>`. `SUSPEND` waits for running queries to finish; `SUSPEND_IMMEDIATE` cancels them. A warehouse-level monitor cannot stop cloud-services credits (https://docs.snowflake.com/en/sql-reference/sql/create-resource-monitor and https://docs.snowflake.com/en/user-guide/resource-monitors, read 2026-11-16).

> **VERIFIED:** warehouse credits are consumed only while the warehouse runs; serverless features also use credits; storage cost is deducted from the balance (trial page, read 2026-11-16).

Habits: check the balance tile at the start of each session. Run `ALTER WAREHOUSE SBX_CALC_WH SUSPEND;` at the end of a session. Never leave a clone database around overnight.

## Roles and users mirroring NAIC

| Sandbox role | Mirrors in NAIC | What it can do |
|---|---|---|
| `SBX_CICD_PIPELINE` | `CICD_PIPELINE` | Owns and deploys every engine object. DCM project owner for INT, QA and PROD. Flyway runs as it. Owns the clone procedures. |
| `SBX_CALC_DEV` | `COMMON_INT_CALC_DEV` | Developer role. Owns the objects inside a ticket clone. Calls the clone procedures. Cannot create databases. |
| `SBX_FN_CALC_ENGINE_NP` | `FN_NAIC_CALC_ENGINE_NP` | Runtime app role. DML only, no DDL. Used in Lab 7. |
| `SBX_CALC_READER` | a read-only role | `SELECT` only. |

Hierarchy: `SBX_CALC_DEV` is granted to `SBX_CICD_PIPELINE` (this mirrors the Terraform grant that lets the owner's-rights clone procedure hand objects to the developer role); `SBX_CICD_PIPELINE` is granted to `SYSADMIN` (so `SYSADMIN` can drop stray clones).

Your learner user holds all four roles so you can play both sides. In NAIC developers do not hold the pipeline role. The default role is `SBX_CALC_DEV`.

Rules the bootstrap follows: databases are created by `SYSADMIN` (this stands in for Terraform); schemas are then owned by `SBX_CICD_PIPELINE`; databases stay owned by `SYSADMIN`; no role gets `ACCOUNTADMIN`, `MANAGE GRANTS` or any `CREATE ... ON ACCOUNT` privilege except that `SBX_CICD_PIPELINE` gets `CREATE DATABASE ON ACCOUNT` (the clone procedure needs it, as in NAIC where the procedure owner creates databases).

> **VERIFIED:** a new hybrid table needs the separate `CREATE HYBRID TABLE` privilege on the schema since behaviour change bundle 2026_02 (BCR-2157), unless you own the schema (https://docs.snowflake.com/en/release-notes/bcr-bundles/2026_02/bcr-2157, read 2026-11-16). The pipeline role owns the schema, so it does not need the grant; the bootstrap contains it as a commented teaching line.

> **TO-CONFIRM [TC-G14]:** exact `GRANT CREATE HYBRID TABLE ON SCHEMA` syntax. How to check: SQL reference for `GRANT`, or try it in a paid account. Record in file 13.

> **TO-CONFIRM [TC-02-02]:** core Snowflake DDL used by the bootstrap (`CREATE ROLE`, `CREATE DATABASE`, `CREATE SCHEMA`, `CREATE WAREHOUSE`, `GRANT ... ON FUTURE ...`, `GRANT OWNERSHIP ... COPY CURRENT GRANTS`, `GRANT CREATE DATABASE ON ACCOUNT`) is long-standing Snowflake syntax that was not re-read on 2026-11-16. How to check: run the bootstrap; any syntax error names the line. Also confirm that `SECURITYADMIN` can run `GRANT OWNERSHIP` on schemas owned by `SYSADMIN` (it holds the `MANAGE GRANTS` privilege by default). Record in file 13.

## Service user and key pair

### Why a service user with a key pair

Automation (GitHub Actions) needs an identity that is not a person. Snowflake `TYPE = SERVICE` users cannot sign in with a password, cannot enroll in MFA and cannot have a password set. Snowflake is blocking passwords for all non-human users in production accounts (phases from 2026), which is why NAIC moves to key pairs or workload identity federation. A key pair works like this: you keep a **private key** secret; Snowflake stores the matching **public key**; the client proves it holds the private key by signing a short-lived token (JWT).

> **VERIFIED:** a `TYPE = SERVICE` user cannot log in with a password or use MFA; `ALTER USER ... RESET PASSWORD` is not allowed for it (https://docs.snowflake.com/en/user-guide/admin-user-management, read 2026-11-16).

### Generate the key pair (Terminal, on your laptop, not in the repository)

```bash
mkdir -p ~/.snowflake-sandbox/keys
chmod 700 ~/.snowflake-sandbox/keys
cd ~/.snowflake-sandbox/keys

# Encrypted private key, AES-256-CBC (recommended). It will ask you for a passphrase.
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes-256-cbc -inform PEM -out sbx_svc_cicd_rsa_key.p8

# Public key
openssl rsa -in sbx_svc_cicd_rsa_key.p8 -pubout -out sbx_svc_cicd_rsa_key.pub

chmod 600 sbx_svc_cicd_rsa_key.p8
```

You should see: two files, `sbx_svc_cicd_rsa_key.p8` and `sbx_svc_cicd_rsa_key.pub`. Choose a strong passphrase `<PASSPHRASE>` and keep it in your password manager. The assistant never sees it.

> **VERIFIED:** minimum 2048-bit RSA; use `aes-256-cbc` for the encrypted key, not `des3`; commands as above (https://docs.snowflake.com/en/user-guide/key-pair-auth, read 2026-11-16).

Print the public key body for the bootstrap SQL (no header or footer lines, one line):

```bash
grep -v -- '-----' ~/.snowflake-sandbox/keys/sbx_svc_cicd_rsa_key.pub | tr -d '\n'; echo
```

Paste the output in place of `<PUBLIC_KEY_BODY>` in the bootstrap (last statement of part 6). The private key never goes into the repository, a chat or a ticket.

> **VERIFIED:** register the public key with `ALTER USER <user> SET RSA_PUBLIC_KEY = '<key body without BEGIN/END lines>'`. Check the registration with `DESC USER`, which shows `RSA_PUBLIC_KEY_FP`. A newer named form exists: `ALTER USER <user> ADD KEY PAIR <name> PUBLIC_KEY = '...'` (same key-pair page, read 2026-11-16). The pack uses the classic form because it is shorter; try the named form as a follow-up if you like.

> **TO-CONFIRM [TC-G12]:** whether `CREATE USER ... TYPE = SERVICE ... RSA_PUBLIC_KEY = '...'` works in one statement. The pack avoids the question by using three statements (create, grant role, set key). How to check: the `CREATE USER` reference page. Record in file 13.

### Store the key as a GitHub secret (later, in the repository section)

You will paste the contents of the `.p8` file into the GitHub secret `SNOWFLAKE_PRIVATE_KEY_RAW` and the passphrase into `SNOWFLAKE_PRIVATE_KEY_PASSPHRASE`. You paste them yourself in the browser. Never put them in a file in the repository.

### If a JWT login fails

The error is `JWT token is invalid` with code 390144, which is a general wrapper. An admin can read the detailed code with `SELECT JSON_EXTRACT_PATH_TEXT(SYSTEM$GET_LOGIN_FAILURE_DETAILS('<uuid>'), 'errorCode');`. Frequent causes: wrong account identifier (394307), user name used instead of `LOGIN_NAME` (394300), clock skew (394302), expired token (394303), fingerprint mismatch (394304), private key does not match the registered public key (394306). File 12 has the table.

> **VERIFIED:** codes and causes from https://docs.snowflake.com/en/user-guide/key-pair-auth-troubleshooting (read 2026-11-16).

## Install the tools

You need `git`, the Snowflake CLI (`snow`), a Flyway runner and (for Flyway via Docker) Docker.

### git

Terminal: `git --version`. Install from your package manager if missing. You run all git commands yourself.

### Snowflake CLI

> **VERIFIED:** DCM Projects require Snowflake CLI **3.24.0 or higher** for the commands in the guide (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use, read 2026-11-16). PyPI showed `snowflake-cli` 3.33.0 on 2026-11-16. This pack pins 3.33.0 and tells you to re-check the pin at lab time.

> **TO-CONFIRM [TC-G16]:** NAIC working notes mention 3.16 as the minimum for DCM; the public docs say 3.24.0. Use the docs figure. How to check: `snow --version` after install, and the DCM usage page at lab time. Record in file 13.

Terminal:

```bash
python3 -m venv ~/.snowflake-sandbox/venv
. ~/.snowflake-sandbox/venv/bin/activate
pip install "snowflake-cli==3.33.0"
snow --version
snow dcm --help
```

You should see: a version number 3.33.0 (or newer if you changed the pin) and a list of DCM commands: `create`, `deploy`, `describe`, `drop`, `drop-deployment`, `list`, `list-deployments`, `plan`, `preview`, `purge`, `test`.

> **TO-CONFIRM [TC-02-03]:** which Python versions the CLI 3.33.0 supports. How to check: `pip install` error text or the PyPI page. If install fails on a very new Python, use Python 3.12 (NAIC laptops use a 3.12 virtual environment because of a TLS issue that does not affect your home laptop). Record in file 13.

### Flyway (Community edition)

> **VERIFIED:** Flyway 13.12.2 was the current stable release on 2026-11-16 (released 2026-11-12). Docker tags `redgate/flyway:13.12.2` and `flyway/flyway:13.12.2` exist. Community edition needs no license. The Snowflake JDBC driver ships with the command line (https://documentation.red-gate.com/flyway/release-notes-and-older-versions/release-notes-for-flyway-engine and https://documentation.red-gate.com/flyway/reference/usage/flyway-docker, read 2026-11-16).

The pack uses Docker, so there is nothing to install except Docker itself. Pin the version, never `latest`. Docker usage seen in the docs: mount your project folder and set the working directory:

```bash
docker run --rm -v "$PWD/flyway/lab2_scratch:/flyway/project" flyway/flyway:13.12.2 -workingDirectory=project info
```

Create a shell helper so later labs are shorter. Terminal (add to your shell profile if you like):

```bash
export FLYWAY_VERSION=13.12.2
fw () {  # usage: fw <project-folder> <flyway arguments...>
  local proj="$1"; shift
  docker run --rm \
    -v "$PWD/$proj:/flyway/project" \
    -v "$HOME/.snowflake-sandbox/keys:/keys:ro" \
    -e PRIVATE_KEY_PASSPHRASE \
    -e FLYWAY_URL -e FLYWAY_LOCATIONS -e FLYWAY_OUT_OF_ORDER -e FLYWAY_CLEAN_DISABLED \
    -e FLYWAY_PLACEHOLDERS_CALC_DB -e FLYWAY_PLACEHOLDERS_CALC_SCHEMA \
    "flyway/flyway:${FLYWAY_VERSION}" -workingDirectory=project "$@"
}
```

Check it with `fw flyway/lab2_scratch -v` after the repository exists (Lab 2 does this again).

> **TO-CONFIRM [TC-02-04]:** that the Docker image accepts `-v` as the version flag and that the environment variable pass-through above works as written. How to check: run `docker run --rm flyway/flyway:13.12.2 -v`. Record in file 13. (Learner result 2026-11-16: `redgate/flyway:13.12.2` has no arm64 build on Apple Silicon; use `flyway/flyway:13.12.2`. See TC-02-04.) Fallback: download the Flyway command-line archive from the Redgate site and run `flyway -v` locally; adjust `fw` to call it.

> **TO-CONFIRM [TC-G21]:** the JVM flag for encrypted private keys. The Snowflake driver page says encrypted keys on Linux may need `JAVA_OPTS='-Dnet.snowflake.jdbc.useBundledBouncyCastleForPrivateKeyDecryption=true'`. How to check: if `flyway info` fails with a key-decryption error, add `-e JAVA_OPTS` to `fw` and set that value. Record in file 13. Fallback for the sandbox only: use a second, unencrypted key kept outside the repository.

Optional: pin by digest. Note: this digest is for the `redgate/flyway` image, which has no arm64 build; read the digest of `flyway/flyway:13.12.2` yourself before using one. Seen on 2026-11-16 (re-check): `redgate/flyway:13.12.2` manifest-list digest `sha256:ccc7fe019c49ea1f7d1a63bab7c4f5d490570783cc0ccf8edccc49a209f5f7c0`.

### Snowflake CLI connections

File `~/.snowflake/connections.toml` (create it with `snow connection add` or by hand). **Never write a password in this file.** Four connections, all with warehouse `SBX_CALC_WH` and database `SBX_INT`:

| Connection | User | Role | Auth | Use |
|---|---|---|---|---|
| `sbx_admin` | `<LEARNER_USER>` | `ACCOUNTADMIN` | interactive | Bootstrap and resource monitor only |
| `sbx_dev` | `<LEARNER_USER>` | `SBX_CALC_DEV` | interactive | Everyday labs |
| `sbx_owner` | `<LEARNER_USER>` | `SBX_CICD_PIPELINE` | interactive | Acting as pipeline or owner from the laptop |
| `sbx_cicd` | `SBX_SVC_CICD` | `SBX_CICD_PIPELINE` | key pair | Same login GitHub Actions uses |

Example shape (a guide; the exact keys may differ):

```toml
[sbx_dev]
account = "<ACCOUNT_IDENTIFIER>"
user = "<LEARNER_USER>"
role = "SBX_CALC_DEV"
warehouse = "SBX_CALC_WH"
database = "SBX_INT"

[sbx_cicd]
account = "<ACCOUNT_IDENTIFIER>"
user = "SBX_SVC_CICD"
role = "SBX_CICD_PIPELINE"
warehouse = "SBX_CALC_WH"
database = "SBX_INT"
authenticator = "SNOWFLAKE_JWT"
private_key_file = "<ABSOLUTE_PATH_TO>/sbx_svc_cicd_rsa_key.p8"   # TO-CONFIRM TC-G27
```

> **VERIFIED:** `snow connection add --connection-name jwt --authenticator SNOWFLAKE_JWT --private-key-file <path>` is the documented way to add a key-pair connection. The environment variable `PRIVATE_KEY_PASSPHRASE` takes precedence over a passphrase stored in the connection, so keep the passphrase in your shell, not in the file. `snow connection test` tests a connection (https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections, read 2026-11-16).

> **TO-CONFIRM [TC-G27]:** the exact `connections.toml` key for the private key file. How to check: run `snow connection add` for `sbx_cicd` and open the generated file. Record in file 13.

> **TO-CONFIRM [TC-02-05]:** how the interactive login works for your human user (password plus MFA prompt, a browser flow, or a second key pair registered to your own user). How to check: run `snow connection test -c sbx_dev` and watch the prompts. If prompts are awkward, register a second key pair for `<LEARNER_USER>` with `ALTER USER <LEARNER_USER> SET RSA_PUBLIC_KEY_2 = '...'` and use `private_key_file` in `sbx_dev`, `sbx_owner` and `sbx_admin`. Record in file 13.

Always pass `-c <connection>` explicitly in later labs. Connection targets in a DCM manifest do not select the connection or role for you.

> **VERIFIED:** DCM manifest targets do not select the CLI connection, account or role; the CLI warns on an account mismatch but does not switch (https://docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dcm-commands/overview, read 2026-11-16).

## GitHub repository

### Visibility: public

Create a repository named **`snowflake-calc-sandbox`** under your account, default branch `main`, visibility **public**.

> **VERIFIED:** required reviewers and wait timers on environments are available for public repositories on all current plans. For private repositories you need GitHub Pro, Team or Enterprise features, and a Free-plan user who makes a public repository private loses the configured rules and secrets (https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments, read 2026-11-16).

> **TO-CONFIRM [TC-02-06]:** your own GitHub plan and whether private-repository environments would work for you. The page read is slightly inconsistent about Pro and Team. How to check: your GitHub account settings, plan section. The pack assumes public, which is safe because the repository contains no secrets (the `.gitignore` below blocks keys and env files).

### Target layout

```text
snowflake-calc-sandbox/
  README.md
  .gitignore                         # includes: out/ , .env.flyway , *.p8 , *.pem , rsa_key*
  dcm/
    calc_engine/
      manifest.yml
      sources/
        definitions/
          sequences/    SEQ_CALC_ID.sql  SEQ_FORMULA_ID.sql  SEQ_EXECUTION_ID.sql  SEQ_WORKFLOW_ID.sql  SEQ_AUDIT_EVENT_ID.sql
          tables/       CALCULATIONS.sql  FORMULAS.sql  FORMULA_CATEGORIES.sql  EXECUTIONS.sql  DATA_CHANGE_LOG.sql
          functions/    FN_APPLY_REDIRECTS.sql
          views/        ENV_CONFIG.sql
          procedures/   SP_ADD_CATEGORY.sql  SP_REGISTER_CALCULATION.sql  SP_ACTIVATE_DRAFT_CALCULATION.sql  SP_COMPUTE_MINI_RESULT.sql
          grants/       access.sql
        macros/                       # optional, only if a lab needs one
      out/                            # gitignored, written by --save-output
    lab3_scratch/                     # throwaway DCM project used only in Lab 3 (own manifest.yml)
  flyway/
    flyway.toml
    .env.flyway.example               # committed; the real .env.flyway is gitignored
    migrations/
      common/       (files identical in both hybrid modes)
      hybrid/       (Mode H variants: real hybrid tables and indexes)
      standard/     (Mode S variants: standard-table stand-ins)
    lab2_scratch/                     # separate small Flyway project for Lab 2 only
      flyway.toml
      migrations/
  sql/
    bootstrap/      snowflake-sandbox-bootstrap.sql   (same text as the code block at the end of this file)
    lab1/           procedures_v1.sql  tables_v1.sql  seed_v1.sql  drift_detective.sql
    clone_support/  calc_dev_clone_procs.sql          (deployed with snow sql, not by DCM or Flyway)
    capstone/       capstone_seed.sql
  scripts/
    refresh_dev_clone.sh
    check_plan_for_drops.py
    assert_zero_int_writes.sql
    smoke_clone.sql
  tests/            (optional placeholder folder for smoke SQL referenced by workflows)
  .github/workflows/
    validate.yml
    pr-clone-check.yml
    deploy-int.yml
    promote-qa-prod.yml
    sweep-stale-clones.yml
    oidc-connection-test.yml          # optional stretch, Lab 6
  docs/learning/                      # this pack (00 to 13, index)
```

Local folder name: `snowflake-calc-sandbox`. The key folder `~/.snowflake-sandbox/keys/` stays **outside** the repository.

### Create the skeleton (Terminal; you run git)

```bash
git clone https://github.com/<GITHUB_OWNER>/snowflake-calc-sandbox.git
cd snowflake-calc-sandbox
mkdir -p dcm/calc_engine/sources/definitions/{sequences,tables,functions,views,procedures,grants} \
         dcm/calc_engine/sources/macros dcm/lab3_scratch \
         flyway/migrations/{common,hybrid,standard} flyway/lab2_scratch/migrations \
         sql/{bootstrap,lab1,clone_support,capstone} scripts tests .github/workflows docs/learning
cp -R <PATH_TO_THIS_PACK>/*.md <PATH_TO_THIS_PACK>/*.html docs/learning/
```

Create a `.gitignore` with this content (placeholders only, no secrets):

```text
out/
.env.flyway
*.p8
*.pem
rsa_key*
.DS_Store
```

Empty folders are not stored by git; add a `.gitkeep` file in the ones you want kept. Then you run `git add`, `git commit` and `git push` yourself. The assistant does not.

Copy the bootstrap block from the end of this file to `sql/bootstrap/snowflake-sandbox-bootstrap.sql` with the placeholders left in (never commit your substituted copy; the public key body is not secret, but your user name is personal).

### Secrets and variables (GitHub UI)

Settings, then Secrets and variables, then Actions (check the current menu names in the UI).

| Kind | Name | Value |
|---|---|---|
| Secret | `SNOWFLAKE_ACCOUNT` | your `<ACCOUNT_IDENTIFIER>` |
| Secret | `SNOWFLAKE_USER` | `SBX_SVC_CICD` |
| Secret | `SNOWFLAKE_PRIVATE_KEY_RAW` | the full contents of `sbx_svc_cicd_rsa_key.p8` |
| Secret | `SNOWFLAKE_PRIVATE_KEY_PASSPHRASE` | your `<PASSPHRASE>` (the workflow maps it to `PRIVATE_KEY_PASSPHRASE`) |
| Variable | `SNOWFLAKE_ROLE` | `SBX_CICD_PIPELINE` |
| Variable | `SNOWFLAKE_WAREHOUSE` | `SBX_CALC_WH` |
| Variable | `HYBRID_MODE` | `H` or `S` (from Probe H) |
| Variable | `DCM_MODE` | `on` or `off` (from Probe D) |
| Variable | `SNOW_CLI_VERSION` | `3.33.0` |
| Variable | `FLYWAY_VERSION` | `13.12.2` |

> **VERIFIED:** `snow` reads `SNOWFLAKE_PRIVATE_KEY_RAW` for CI use, and the passphrase from the environment variable `PRIVATE_KEY_PASSPHRASE` (configure-connections page, read 2026-11-16).

GitHub environments `int`, `qa`, `prod` and `pr-clone` are created in Lab 6, step by step.

## Probes: find your mode

Run in **Snowsight worksheet** after the bootstrap, as the pipeline role. Use the role selector or `USE ROLE`.

```sql
USE ROLE SBX_CICD_PIPELINE;
USE WAREHOUSE SBX_CALC_WH;

-- Probe H: hybrid tables
CREATE HYBRID TABLE SBX_INT.CALC_SUPPORT.ZZ_HYBRID_PROBE (ID NUMBER PRIMARY KEY);
-- success: run the next line, then HYBRID_MODE = H
DROP TABLE IF EXISTS SBX_INT.CALC_SUPPORT.ZZ_HYBRID_PROBE;

-- Probe D: DCM projects
CREATE DCM PROJECT SBX_INT.LAB3_SCRATCH.ZZ_DCM_PROBE;
-- success: run the next line, then DCM_MODE = on
DROP DCM PROJECT SBX_INT.LAB3_SCRATCH.ZZ_DCM_PROBE;
```

You should see: Probe H most likely fails in a trial with an error saying the feature is not supported. **Copy the exact error text** into file 13 under TC-G01. That means `HYBRID_MODE = S`. Probe D should succeed; if it fails, copy the error under TC-G02 and set `DCM_MODE = off`.

> **TO-CONFIRM [TC-G01]:** the exact error when creating a hybrid table in a trial. Record the real text.

> **TO-CONFIRM [TC-G02]:** whether `CREATE DCM PROJECT` works in a trial. Record the result.

> **VERIFIED:** the create syntax `CREATE [OR REPLACE] DCM PROJECT [IF NOT EXISTS] <name> [COMMENT = '...']`, the privilege `CREATE DCM PROJECT` on a schema, and `DROP DCM PROJECT [IF EXISTS] <name>` (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use and .../dcm-projects-manage, read 2026-11-16).

Record `HYBRID_MODE` and `DCM_MODE` in the Resume table in file 00 and set the GitHub variables.

## Verification checklist

Tick each item.

- [ ] Trial account created; MFA set; I can sign in to Snowsight.
- [ ] Resource monitor `SBX_CALC_RM` exists and is assigned to `SBX_CALC_WH` (see the check query below).
- [ ] `snow --version` prints 3.24.0 or higher.
- [ ] `snow connection test -c sbx_dev` works.
- [ ] `snow connection test -c sbx_owner` works.
- [ ] `snow connection test -c sbx_cicd` works (key pair, passphrase from `PRIVATE_KEY_PASSPHRASE`).
- [ ] `fw flyway/lab2_scratch -v` (or `flyway -v`) prints 13.12.2.
- [ ] `DESC USER SBX_SVC_CICD` shows a value for `RSA_PUBLIC_KEY_FP`.
- [ ] Repository `snowflake-calc-sandbox` cloned with the layout above; `.gitignore` committed; the private key is **not** in the repository.
- [ ] GitHub secrets and variables set.
- [ ] Probes H and D run; modes recorded; results copied to file 13.

Check queries (Snowsight, role `SBX_CALC_DEV` for the first two, `ACCOUNTADMIN` for the monitor):

```sql
SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE();
SHOW GRANTS TO USER <LEARNER_USER>;
SHOW RESOURCE MONITORS;   -- ACCOUNTADMIN only; look for SBX_CALC_RM
DESC USER SBX_SVC_CICD;   -- look for RSA_PUBLIC_KEY_FP
```

## State left behind

Trial account; bootstrap run; all databases, schemas, roles, `SBX_CALC_WH`, `SBX_CALC_RM`, `SBX_SVC_CICD` exist; `snow connection test` is OK for `sbx_dev`, `sbx_owner`, `sbx_cicd`; Flyway runs `-v`; the repository is cloned with the tree above; secrets are set; the hybrid probe is done, `HYBRID_MODE` recorded, probe table dropped.

## Open items for file 13

TC-G01, TC-G02, TC-G12, TC-G14, TC-G16, TC-G18, TC-G19, TC-G21, TC-G24, TC-G27, TC-02-01, TC-02-02, TC-02-03, TC-02-04, TC-02-05, TC-02-06.

## Next

Go to [03-lab1-the-old-way.md](03-lab1-the-old-way.md).

---

## Bootstrap SQL: `snowflake-sandbox-bootstrap.sql`

How to run it: open a Snowsight worksheet. Replace `<LEARNER_USER>` (your login name) and `<PUBLIC_KEY_BODY>` (the one-line public key from the key section). Run it **part by part** in the order shown, each with the role stated in its first line. Run every statement in a part (use "Run all" within the part). Each part is safe to rerun.

```sql
-- =====================================================================
-- snowflake-sandbox-bootstrap.sql
-- Sandbox mirror of the NAIC calc-engine layout. Run in YOUR OWN trial
-- account only. Placeholders: <LEARNER_USER>, <PUBLIC_KEY_BODY>.
-- No secrets belong in this file.
-- =====================================================================

-- ---------------------------------------------------------------------
-- PART 1 (SECURITYADMIN): roles and the learner's grants
-- ---------------------------------------------------------------------
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS SBX_CICD_PIPELINE      COMMENT = 'Owner and deployer of engine objects (mirrors CICD_PIPELINE)';
CREATE ROLE IF NOT EXISTS SBX_CALC_DEV           COMMENT = 'Developer role, owns clone objects (mirrors COMMON_INT_CALC_DEV)';
CREATE ROLE IF NOT EXISTS SBX_FN_CALC_ENGINE_NP  COMMENT = 'Runtime app role, DML only (mirrors FN_NAIC_CALC_ENGINE_NP)';
CREATE ROLE IF NOT EXISTS SBX_CALC_READER        COMMENT = 'Read-only role';

-- Role hierarchy
GRANT ROLE SBX_CALC_DEV      TO ROLE SBX_CICD_PIPELINE;  -- lets owner's-rights clone procedures hand objects to the dev role
GRANT ROLE SBX_CICD_PIPELINE TO ROLE SYSADMIN;           -- lets SYSADMIN drop stray clones

-- The learner holds all four roles (in NAIC developers do not hold the pipeline role)
GRANT ROLE SBX_CALC_DEV, SBX_CICD_PIPELINE, SBX_FN_CALC_ENGINE_NP, SBX_CALC_READER TO USER <LEARNER_USER>;
ALTER USER <LEARNER_USER> SET DEFAULT_ROLE = SBX_CALC_DEV;   -- never ACCOUNTADMIN as default

-- ---------------------------------------------------------------------
-- PART 2 (SYSADMIN): warehouse, databases, schemas  (stands in for Terraform)
-- ---------------------------------------------------------------------
USE ROLE SYSADMIN;

CREATE WAREHOUSE IF NOT EXISTS SBX_CALC_WH
  WAREHOUSE_SIZE      = 'XSMALL'
  AUTO_SUSPEND        = 60
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Sandbox warehouse, tiny on purpose';

CREATE DATABASE IF NOT EXISTS SBX_INT      COMMENT = 'Mirrors COMMON_INT';
CREATE DATABASE IF NOT EXISTS SBX_INT_FIN  COMMENT = 'Mirrors NAIC_INT_FINANCIAL_EDL (results)';
CREATE DATABASE IF NOT EXISTS SBX_QA       COMMENT = 'Simulated QA';
CREATE DATABASE IF NOT EXISTS SBX_PROD     COMMENT = 'Simulated PROD';

CREATE SCHEMA IF NOT EXISTS SBX_INT.CALC_SUPPORT;       -- engine objects, DCM project, Flyway history
CREATE SCHEMA IF NOT EXISTS SBX_INT.CALC_DEV_SUPPORT;   -- clone procedures
CREATE SCHEMA IF NOT EXISTS SBX_INT.LAB2_SCRATCH;       -- Lab 2 Flyway playground
CREATE SCHEMA IF NOT EXISTS SBX_INT.LAB3_SCRATCH;       -- Lab 3 DCM drop experiments
CREATE SCHEMA IF NOT EXISTS SBX_INT.CAP_HAND;           -- capstone way 1
CREATE SCHEMA IF NOT EXISTS SBX_INT.CAP_FLYWAY;         -- capstone way 2
CREATE SCHEMA IF NOT EXISTS SBX_INT.CAP_DCM;            -- capstone way 3
CREATE SCHEMA IF NOT EXISTS SBX_INT_FIN.FIN_CALCULATION;
CREATE SCHEMA IF NOT EXISTS SBX_QA.CALC_SUPPORT;
CREATE SCHEMA IF NOT EXISTS SBX_PROD.CALC_SUPPORT;

-- ---------------------------------------------------------------------
-- PART 3 (ACCOUNTADMIN): resource monitor and one account-level privilege
-- Only ACCOUNTADMIN may create resource monitors.
-- The quota of 20 credits is this pack's choice, not a Snowflake default.
-- ---------------------------------------------------------------------
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE RESOURCE MONITOR SBX_CALC_RM WITH
  CREDIT_QUOTA    = 20
  FREQUENCY       = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 50  PERCENT DO NOTIFY
           ON 80  PERCENT DO NOTIFY
           ON 90  PERCENT DO SUSPEND
           ON 100 PERCENT DO SUSPEND_IMMEDIATE;

ALTER WAREHOUSE SBX_CALC_WH SET RESOURCE_MONITOR = SBX_CALC_RM;

-- The clone procedure (owned by the pipeline role) creates databases
GRANT CREATE DATABASE ON ACCOUNT TO ROLE SBX_CICD_PIPELINE;

-- ---------------------------------------------------------------------
-- PART 4 (SECURITYADMIN): ownership of schemas and container grants
-- Databases stay owned by SYSADMIN. Schemas go to the pipeline role.
-- ---------------------------------------------------------------------
USE ROLE SECURITYADMIN;

GRANT OWNERSHIP ON SCHEMA SBX_INT.CALC_SUPPORT      TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT.CALC_DEV_SUPPORT  TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT.LAB2_SCRATCH      TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT.LAB3_SCRATCH      TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT.CAP_HAND          TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT.CAP_FLYWAY        TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT.CAP_DCM           TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_INT_FIN.FIN_CALCULATION TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_QA.CALC_SUPPORT       TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA SBX_PROD.CALC_SUPPORT     TO ROLE SBX_CICD_PIPELINE COPY CURRENT GRANTS;

-- Pipeline role: warehouse, databases. Schema-level create rights come with ownership.
GRANT USAGE ON WAREHOUSE SBX_CALC_WH TO ROLE SBX_CICD_PIPELINE;
GRANT USAGE ON DATABASE SBX_INT      TO ROLE SBX_CICD_PIPELINE;
GRANT USAGE ON DATABASE SBX_INT_FIN  TO ROLE SBX_CICD_PIPELINE;
GRANT USAGE ON DATABASE SBX_QA       TO ROLE SBX_CICD_PIPELINE;
GRANT USAGE ON DATABASE SBX_PROD     TO ROLE SBX_CICD_PIPELINE;
-- Needed in Lab 2 so the pipeline role can clone a schema (LAB2_BASELINE) inside SBX_INT
GRANT CREATE SCHEMA ON DATABASE SBX_INT TO ROLE SBX_CICD_PIPELINE;

-- Explicit teaching line: schema owners do not need it, shown so you see the privilege (TC-G14)
-- GRANT CREATE HYBRID TABLE ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CICD_PIPELINE;   -- TO-CONFIRM TC-G14

-- DCM project creation privilege (owners do not strictly need it; explicit for learning)
GRANT CREATE DCM PROJECT ON SCHEMA SBX_INT.CALC_SUPPORT  TO ROLE SBX_CICD_PIPELINE;
GRANT CREATE DCM PROJECT ON SCHEMA SBX_QA.CALC_SUPPORT   TO ROLE SBX_CICD_PIPELINE;
GRANT CREATE DCM PROJECT ON SCHEMA SBX_PROD.CALC_SUPPORT TO ROLE SBX_CICD_PIPELINE;
GRANT CREATE DCM PROJECT ON SCHEMA SBX_INT.LAB3_SCRATCH  TO ROLE SBX_CICD_PIPELINE;

-- Developer role
GRANT USAGE ON WAREHOUSE SBX_CALC_WH TO ROLE SBX_CALC_DEV;
GRANT USAGE ON DATABASE SBX_INT      TO ROLE SBX_CALC_DEV;
GRANT USAGE ON DATABASE SBX_INT_FIN  TO ROLE SBX_CALC_DEV;
GRANT USAGE ON SCHEMA SBX_INT.CALC_SUPPORT          TO ROLE SBX_CALC_DEV;
GRANT USAGE ON SCHEMA SBX_INT.CALC_DEV_SUPPORT      TO ROLE SBX_CALC_DEV;
GRANT USAGE ON SCHEMA SBX_INT_FIN.FIN_CALCULATION   TO ROLE SBX_CALC_DEV;
GRANT SELECT ON ALL TABLES    IN SCHEMA SBX_INT.CALC_SUPPORT        TO ROLE SBX_CALC_DEV;
GRANT SELECT ON FUTURE TABLES IN SCHEMA SBX_INT.CALC_SUPPORT        TO ROLE SBX_CALC_DEV;
GRANT SELECT ON ALL TABLES    IN SCHEMA SBX_INT_FIN.FIN_CALCULATION TO ROLE SBX_CALC_DEV;
GRANT SELECT ON FUTURE TABLES IN SCHEMA SBX_INT_FIN.FIN_CALCULATION TO ROLE SBX_CALC_DEV;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CALC_DEV;
GRANT USAGE ON FUTURE FUNCTIONS  IN SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CALC_DEV;
GRANT USAGE ON FUTURE SEQUENCES  IN SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CALC_DEV;
-- USAGE on CREATE_CALC_DEV_CLONE(VARCHAR) and DROP_CALC_DEV_CLONE(VARCHAR) is granted by
-- sql/clone_support/calc_dev_clone_procs.sql after each CREATE OR REPLACE (Lab 5).

-- App role and read role: schema-level access only.
-- Table-level grants are explicit per table (DCM access.sql in Lab 3, Flyway migrations in Lab 4),
-- NOT future grants, so Lab 7 can give the audit table INSERT+SELECT only.
GRANT USAGE ON WAREHOUSE SBX_CALC_WH TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT USAGE ON DATABASE SBX_INT      TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT USAGE ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT USAGE ON DATABASE SBX_INT_FIN  TO ROLE SBX_FN_CALC_ENGINE_NP;
GRANT USAGE ON SCHEMA SBX_INT_FIN.FIN_CALCULATION TO ROLE SBX_FN_CALC_ENGINE_NP;

GRANT USAGE ON WAREHOUSE SBX_CALC_WH TO ROLE SBX_CALC_READER;
GRANT USAGE ON DATABASE SBX_INT      TO ROLE SBX_CALC_READER;
GRANT USAGE ON SCHEMA SBX_INT.CALC_SUPPORT TO ROLE SBX_CALC_READER;

-- ---------------------------------------------------------------------
-- PART 5 (SBX_CICD_PIPELINE): the results table that EDP's own Flyway owns in NAIC
-- Created here by the platform lane so the sandbox has something to write to.
-- ---------------------------------------------------------------------
USE ROLE SBX_CICD_PIPELINE;

CREATE TABLE IF NOT EXISTS SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE (
  RESULT_ID    NUMBER(38,0) NOT NULL AUTOINCREMENT START 1 INCREMENT 1 NOORDER,
  CALC_ID      NUMBER(38,0),
  EXECUTION_ID NUMBER(38,0),
  VALUE_NUMBER NUMBER(38,6),
  LOADED_AT    TIMESTAMP_NTZ(9) DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (RESULT_ID)
);

GRANT INSERT, SELECT ON TABLE SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE
  TO ROLE SBX_FN_CALC_ENGINE_NP;

-- ---------------------------------------------------------------------
-- PART 6 (SECURITYADMIN): service user for automation (key pair, no password)
-- Run this part after generating the key pair. Replace <PUBLIC_KEY_BODY>
-- with the one-line key body (no BEGIN/END lines).
-- ---------------------------------------------------------------------
USE ROLE SECURITYADMIN;

CREATE USER IF NOT EXISTS SBX_SVC_CICD
  TYPE              = SERVICE
  DEFAULT_ROLE      = SBX_CICD_PIPELINE
  DEFAULT_WAREHOUSE = SBX_CALC_WH
  COMMENT           = 'Automation identity used by GitHub Actions and the sbx_cicd connection';

GRANT ROLE SBX_CICD_PIPELINE TO USER SBX_SVC_CICD;

ALTER USER SBX_SVC_CICD SET RSA_PUBLIC_KEY = '<PUBLIC_KEY_BODY>';

-- ---------------------------------------------------------------------
-- PART 7: checks (read-only)
-- ---------------------------------------------------------------------
SHOW GRANTS TO ROLE SBX_CICD_PIPELINE;
SHOW SCHEMAS IN DATABASE SBX_INT;
DESC USER SBX_SVC_CICD;   -- RSA_PUBLIC_KEY_FP must be filled
```

You should see after Part 7: `SHOW SCHEMAS` lists `CALC_SUPPORT`, `CALC_DEV_SUPPORT`, `LAB2_SCRATCH`, `LAB3_SCRATCH`, `CAP_HAND`, `CAP_FLYWAY`, `CAP_DCM` and `PUBLIC`, with owner `SBX_CICD_PIPELINE` for the first seven. `DESC USER SBX_SVC_CICD` shows a non-empty `RSA_PUBLIC_KEY_FP`.

Notes on what differs from the plain contract: the bootstrap adds `GRANT CREATE SCHEMA ON DATABASE SBX_INT TO ROLE SBX_CICD_PIPELINE` because Lab 2 clones a schema into `SBX_INT.LAB2_BASELINE` as the pipeline role.
