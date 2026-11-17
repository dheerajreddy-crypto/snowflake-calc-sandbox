# Lab 6: CI/CD with GitHub Actions (file 08)

> **Lab no:** 6 | **File no:** 08 | **Time:** 3 h (plus 1 h for the optional OIDC stretch) | **Prerequisites:** file 07 done; GitHub secrets and variables set (file 02); GitHub environments created (step 5.1) | **Modes supported:** H or S, D (steps that need DCM skip themselves when the repository variable `DCM_MODE` is `off`) | **Leaves behind:** six workflow files committed by you, one green pull request run, `int` deployed through an approval, `SBX_QA` and `SBX_PROD` filled by the promote workflow, no `SBX_DEV_PR*` database left.

## 1. Goal

Turn the manual flow you ran by hand in Labs 3 to 5 into automatic, repeatable pipelines, using GitHub Actions as the sandbox stand-in for NAIC's GitLab CI.

By the end you can:

- explain every GitLab CI concept NAIC plans to use and name its GitHub Actions counterpart;
- run a pull request pipeline that creates a private clone for the PR, applies Flyway and DCM to it, fails on an unexpected DROP, proves an empty re-plan, runs a smoke test, proves the source was not written to, and always cleans up;
- deploy to `int` after a human approval, then promote to `qa` and `prod`;
- authenticate with a key-pair secret, and (stretch) with GitHub OIDC and no stored key;
- list the supply-chain rules NAIC cares about and show where each one lives in the workflow files.

## 2. Why this matters in calc-engine and calc-admin

> **Real world (NAIC).** Today nobody deploys the calc engine with a pipeline. The old way was pasting `sql/procedures.sql` into Snowsight, in INT only. There is no root `.gitlab-ci.yml` yet, and the Flyway module repository has no implementation (research-context section 12). The target design in `calc-engine-pipeline.html` and `calc-engine-complete-architecture.html` (sections 8 and 9) is one pinned GitLab profile that includes, in order: source checks, security, deployment, a new Snowflake DCM module, a Snowflake Flyway module, a Jira change-request fragment, and the core module last.
>
> The planned job flow is: static checks, a ready merged-result merge request, a guarded review clone `CALC_DEV_<ticket>` (or `CALC_DEV_MR<iid>`), Flyway info and validate, a full DCM plan with an exact DROP policy, prerequisite migrations, DCM deploy, data changes, an empty re-plan, smoke tests, cleanup with `when: always`, then protected manual INT, QA and PROD jobs. Review, INT, QA and PROD use separate identities and separate `resource_group` values. The target identity is a GitLab OIDC token federated into Snowflake.
>
> This lab builds that same shape on GitHub, with `SBX_` names. Everything here is a miniature. You are not changing NAIC, and the "never run calculations" rule from file 00 is about NAIC only. In the sandbox the compute procedure is `SP_COMPUTE_MINI_RESULT` and it is harmless.

## 3. Prerequisites and state check

Run in: **Terminal** (in the clone `snowflake-calc-sandbox`).

```bash
snow --version                              # expect 3.24.0 or higher; the pack pins 3.33.0
snow connection test -c sbx_cicd            # expect: connection OK for SBX_SVC_CICD
docker run --rm flyway/flyway:13.12.2 -v   # expect: a Flyway 13.12.2 version banner
ls scripts/refresh_dev_clone.sh sql/clone_support/calc_dev_clone_procs.sql
```

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`, warehouse `SBX_CALC_WH`).

```sql
SHOW PROCEDURES LIKE 'CREATE_CALC_DEV_CLONE' IN SCHEMA SBX_INT.CALC_DEV_SUPPORT;   -- expect 1 row
SHOW PROCEDURES LIKE 'DROP_CALC_DEV_CLONE'   IN SCHEMA SBX_INT.CALC_DEV_SUPPORT;   -- expect 1 row
SHOW DATABASES LIKE 'SBX_DEV_%';                                                    -- expect 0 rows
SHOW DCM PROJECTS IN SCHEMA SBX_INT.CALC_SUPPORT;                                   -- expect CALC_ENGINE_DCM (skip if mode D-off)
```

Run in: **GitHub UI** (repository, Settings, Secrets and variables, Actions). You should see the four secrets `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, `SNOWFLAKE_PRIVATE_KEY_RAW`, `SNOWFLAKE_PRIVATE_KEY_PASSPHRASE`, and the variables `SNOWFLAKE_ROLE`, `SNOWFLAKE_WAREHOUSE`, `HYBRID_MODE`, `SNOW_CLI_VERSION`, `FLYWAY_VERSION`. Also add `DCM_MODE` (`on` or `off`) from your probe result in file 02. If something is missing, go back to file 02 section "GitHub repository" before continuing.

If a clone from Lab 5 still exists, drop it now: `CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('LAB5A');`.

## 4. Concepts

**CI/CD.** Continuous integration means every change is built and tested automatically when it is proposed. Continuous delivery means a tested change can be deployed by a repeatable pipeline, with a human approval where you want one. A **pipeline** is the automated sequence. In GitHub it is called a **workflow run**.

**Workflow, job, step, runner.** A workflow is one YAML file in `.github/workflows/`. It contains jobs. Jobs run on a **runner**, which is a machine GitHub (or you) provide. A job contains steps, which are shell commands or reusable actions. Jobs run in parallel unless one declares `needs: <other job>`.

**Trigger.** The `on:` block says when a workflow starts: `pull_request`, `push`, `workflow_dispatch` (a manual button), `schedule` (cron).

**Environment.** A named deployment target in GitHub (`int`, `qa`, `prod`). A job that names `environment: int` waits until the environment's protection rules pass, for example a **required reviewer**, and only then can it read that environment's secrets.

**Concurrency group.** `concurrency: group: deploy-int` lets at most one job in that group run at once. It is the counterpart of the GitLab `resource_group` that NAIC uses to keep two deploys from racing.

**OIDC (OpenID Connect) token.** A short-lived signed token that the runner can ask GitHub for. Snowflake can be told to trust it, so no stored key is needed. This is "workload identity federation".

**Supply chain.** Everything your pipeline downloads and runs: actions, container images, CLI packages. Pinning means naming an exact immutable version so that nobody can change what runs under you.

### 4.1 GitLab CI to GitHub Actions

| NAIC GitLab CI idea | GitHub Actions counterpart in the sandbox | Notes |
|---|---|---|
| Pipeline | Workflow run | One run per trigger. |
| Stage | `needs:` between jobs | GitHub has no `stages:` keyword. Order is expressed by `needs`. |
| Job | Job | Same word. |
| Runner | Runner (`runs-on:`) | NAIC uses its own runners and a controlled image. The sandbox uses GitHub-hosted `ubuntu-24.04`. |
| `include`, component, profile `snowflake-calc-engine` | Reusable workflow (`workflow_call`) or composite action | The sandbox keeps steps inline so each file stands alone. In a real repo you would move repeated steps into a composite action. |
| Protected environment with manual approval | Environment with required reviewers | Only on public repos for Free plans (see step 5.1). |
| `when: manual` job | Environment approval, or `workflow_dispatch` | `promote-qa-prod.yml` uses both. |
| `resource_group` | `concurrency: group:` | Use `cancel-in-progress: false` for deploys so a running deploy is never killed. |
| `id_tokens:` (OIDC) | `permissions: id-token: write` | Only the OIDC job asks for it. |
| CI/CD variables (masked, protected) | Secrets and variables, optionally per environment | A secret is hidden in logs. A variable is plain text. |
| Merge request pipeline, merged results | `pull_request` event | GitHub tests a merge of your branch into the base by default (see TC-08-03). |
| `rules:` | `on:` plus `if:` | `if:` works on jobs and on single steps. |
| `when: always` cleanup | `if: always()` on a step or job | Runs also when earlier steps failed. |
| Scheduled pipeline | `schedule:` cron | Used by the clone sweeper. |
| Artifacts and 30-day evidence | `$GITHUB_STEP_SUMMARY` and artifacts | The sandbox writes plan summaries to the step summary. |
| Internal Nexus mirror for tools | Tools from your own mirror or pinned public sources | See section 5.14. In NAIC, Snowflake CLI comes from the PyPI proxy, there is no CLI image, and Flyway must be an internally mirrored immutable image. |
| Vault plus JWT auth to read secrets | GitHub encrypted secrets, or OIDC straight to Snowflake | NAIC's target removes stored Snowflake secrets. |

### 4.2 The pipeline you will build

```
 pull request opened or updated
        |
        v
 validate.yml        lint + flyway validate (reads INT only)
 pr-clone-check.yml  record INT counts
                     drop old PR clone, CREATE_CALC_DEV_CLONE('PR<n>')
                     [pre-DCM migrations if flyway/PRE_DCM_TARGET is set]
                     dcm plan (full) -> check_plan_for_drops.py  <-- fails here on any DROP
                     dcm deploy to the clone
                     flyway migrate on the clone
                     dcm plan again -> assert_plan_empty.py
                     smoke_clone.sql, then assert_zero_int_writes.sql
                     cleanup job (always): DROP_CALC_DEV_CLONE('PR<n>')
        |
 merge to main
        v
 deploy-int.yml      environment int (you approve) -> same steps against SBX_INT
        |
 manual button
        v
 promote-qa-prod.yml environment qa or prod (you approve) -> SBX_QA / SBX_PROD
 sweep-stale-clones.yml  daily: drop PR clones older than one day
```

### 4.3 Ordering rules the workflows follow

The pack's two ordering rules (file 06) are both implemented:

1. Normal change: DCM full plan, review, DCM deploy, Flyway migrate, DCM full re-plan must be empty. This is the default order in every workflow.
2. Rename change: Flyway rename first, then the DCM definition update, then an empty re-plan. The workflows support this through one small file, `flyway/PRE_DCM_TARGET`. If it contains a version number such as `6`, the workflow runs `flyway migrate -target=6` before the DCM plan. This mirrors the "apply prerequisite migrations" step of NAIC's planned job flow. If the file is empty, the step does nothing.

> **NEW NAME (sandbox convention).** `flyway/PRE_DCM_TARGET`, `scripts/pre_dcm_migrations.sh`, `scripts/flyway_ci.sh` and `scripts/assert_plan_empty.py` are not in the contract tree. They are small helpers of this lab.

### 4.4 What a green pipeline proves, and what it does not

It proves that this commit, applied to a copy of INT, plans without drops, migrates, re-plans empty and passes the smoke test. It does not prove that INT itself is unchanged since the plan (DEPLOY recomputes its own plan), and it does not roll anything back. Lab 8 shows both gaps.

> **VERIFIED:** DEPLOY computes a fresh changeset and does not replay a saved PLAN (https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use, read 2026-11-16).

## 5. Step-by-step

### Step 5.1 Create the GitHub environments

Run in: **GitHub UI** (repository, Settings, Environments).

1. Create four environments: `int`, `qa`, `prod`, `pr-clone`.
2. On `int`, `qa` and `prod`, tick **Required reviewers** and add yourself. **Untick "Prevent self-review"**. You are the only person in the sandbox, so with self-review blocked nothing could ever be approved.
3. Leave `pr-clone` without any protection rule. The pull request job must start without a human click, and it reads secrets only through the repository level.
4. Optional on `prod`: set a wait timer of one minute to feel what a cooling-off period is like.

You should see: four environments, three of them with "Required reviewers: 1".

> **VERIFIED:** A job that names an environment must pass that environment's protection rules before it runs or can read its secrets. Up to 6 reviewers, only one has to approve, and "Prevent self-review" is optional. A workflow that names a non-existent environment creates it with no protection (https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments, read 2026-11-16).

> **VERIFIED:** Environments with required reviewers are available for public repositories on current plans. For private repositories you need GitHub Pro, Team or Enterprise (same page, read 2026-11-16). This is why the contract makes `snowflake-calc-sandbox` PUBLIC. Nothing secret is ever committed.

> **TO-CONFIRM [TC-08-03]:** Your own plan and behaviour: whether the environments page shows the reviewer option for your repository. How to check at lab time: open Settings, Environments, `int`. If the reviewer option is missing, use the fallback in section 8. Record the result in file 13.

### Step 5.2 Create the helper scripts

Run in: **Terminal** (repository root). Create each file with your editor, then make the shell scripts executable with `chmod +x scripts/flyway_ci.sh scripts/pre_dcm_migrations.sh`.

**`scripts/check_plan_for_drops.py`**. Exits 1 if the saved plan has any top-level DROP. It fails closed: if the file does not look like a plan, it also fails.

```python
#!/usr/bin/env python3
"""Fail if a saved DCM plan contains any DROP. Usage: check_plan_for_drops.py <plan_json>"""
import json
import sys


def main(path):
    with open(path, encoding="utf-8") as handle:
        plan = json.load(handle)
    if "changeset" not in plan:
        print("ERROR: no 'changeset' key; not a plan file, failing closed")
        return 2
    changes = plan["changeset"]
    drops = [c for c in changes if str(c.get("type", "")).upper() == "DROP"]
    print(f"plan changes: {len(changes)}  drops: {len(drops)}")
    for c in drops:
        oid = c.get("object_id", {})
        print(f"  DROP {oid.get('domain')} {oid.get('fqn')}")
    return 1 if drops else 0


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("usage: check_plan_for_drops.py <plan_json>")
        sys.exit(2)
    sys.exit(main(sys.argv[1]))
```

> **VERIFIED:** A saved plan has `version: 2`, `metadata`, and a `changeset` array. Each entry has `type` (CREATE, ALTER or DROP) and `object_id` with `domain`, `name` and `fqn`. An empty `changeset` means the project is in sync (https://docs.snowflake.com/en/sql-reference/sql/execute-dcm-project, read 2026-11-16).

> **TO-CONFIRM [TC-G04]:** The file name that `--save-output` writes (`plan_result.json` or `plan.json`). The workflows below use a glob that accepts either. How to check at lab time: `snow dcm plan ... --save-output` then `ls dcm/calc_engine/out/`. Record the result in file 13.

**`scripts/assert_plan_empty.py`**. Exits 1 if the plan has any change at all. This is the "empty re-plan" check.

```python
#!/usr/bin/env python3
"""Fail unless a saved DCM plan has an empty changeset. Usage: assert_plan_empty.py <plan_json>"""
import json
import sys

if len(sys.argv) != 2:
    print("usage: assert_plan_empty.py <plan_json>")
    sys.exit(2)
with open(sys.argv[1], encoding="utf-8") as handle:
    plan = json.load(handle)
if "changeset" not in plan:
    print("ERROR: no 'changeset' key; failing closed")
    sys.exit(2)
changes = plan["changeset"]
if changes:
    print(f"NOT EMPTY: {len(changes)} change(s) after deploy")
    for c in changes:
        oid = c.get("object_id", {})
        print(f"  {c.get('type')} {oid.get('domain')} {oid.get('fqn')}")
    sys.exit(1)
print("re-plan is empty: definitions and live objects agree")
```

**`scripts/flyway_ci.sh`**. Runs the pinned Flyway image in Docker. It builds the JDBC URL from environment variables so no secret is ever written into the repository.

```bash
#!/usr/bin/env bash
# Usage: FLY_DB=<database> KEY_FILE=<path to private key> scripts/flyway_ci.sh <flyway args...>
set -euo pipefail
: "${FLY_DB:?set FLY_DB}" "${KEY_FILE:?set KEY_FILE}"
: "${SNOWFLAKE_ACCOUNT:?}" "${SNOWFLAKE_WAREHOUSE:?}" "${SNOWFLAKE_ROLE:?}"
IMAGE="${FLY_IMAGE:-flyway/flyway:13.12.2}"      # pin by digest too, see section 5.14
if [ "${HYBRID_MODE:-H}" = "H" ]; then
  LOC="filesystem:migrations/common,filesystem:migrations/hybrid"
else
  LOC="filesystem:migrations/common,filesystem:migrations/standard"
fi
URL="jdbc:snowflake://${SNOWFLAKE_ACCOUNT}.snowflakecomputing.com/?db=${FLY_DB}&warehouse=${SNOWFLAKE_WAREHOUSE}&role=${SNOWFLAKE_ROLE}&authenticator=snowflake_jwt&private_key_file=/keys/key.p8"
docker run --rm \
  -v "$PWD/flyway:/flyway/project" \
  -v "${KEY_FILE}:/keys/key.p8:ro" \
  -e FLYWAY_URL="$URL" \
  -e FLYWAY_LOCATIONS="$LOC" \
  -e FLYWAY_PLACEHOLDERS_CALC_DB="$FLY_DB" \
  -e FLYWAY_PLACEHOLDERS_CALC_SCHEMA="CALC_SUPPORT" \
  -e PRIVATE_KEY_PASSPHRASE \
  -e JAVA_OPTS="-Dnet.snowflake.jdbc.useBundledBouncyCastleForPrivateKeyDecryption=true" \
  "$IMAGE" -workingDirectory=project "$@"
```

> **VERIFIED:** Docker form `docker run --rm -v <project>:/flyway/project redgate/flyway -workingDirectory=project migrate`; JDBC URL with mandatory `db` and `warehouse`; key-pair URL parameters `authenticator=snowflake_jwt&private_key_file=<absolute path>`; env var rule `FLYWAY_<UPPER_SNAKE>` (https://documentation.red-gate.com/flyway/reference/usage/flyway-docker and .../database-driver-reference/snowflake, read 2026-11-16).

> **TO-CONFIRM [TC-G21]:** Whether Flyway 13.12.2 needs the `JAVA_OPTS` flag for an encrypted key and how it receives the passphrase from `PRIVATE_KEY_PASSPHRASE`. The docs mention the flag for encrypted keys on Linux. How to check at lab time: run `scripts/flyway_ci.sh info` (section 5.9). If it asks for or rejects the passphrase, read the Snowflake driver page and the JDBC `private_key_file_pwd` property. Record the result in file 13.

> **TO-CONFIRM [TC-G10]:** Per-environment placeholders inside `flyway.toml` are not used on purpose. The script sets `FLYWAY_PLACEHOLDERS_CALC_DB` instead. How to check: `flyway info` output shows no placeholder error. Record the result in file 13.

> **TO-CONFIRM [TC-08-02]:** Whether the container user can read a mode-600 key file that belongs to the runner user, and whether Flyway needs a writable home directory. The mount is read-only. How to check at lab time: if you see "permission denied" on the key, add `--user "$(id -u):$(id -g)"` to the `docker run` line, or run `chmod 644` on the temporary key file inside the job (the file lives only for the job and is deleted by `if: always()`). Record the result in file 13.

**`scripts/pre_dcm_migrations.sh`**. Implements ordering rule 2 for environments that already have a Flyway history.

```bash
#!/usr/bin/env bash
# Run prerequisite migrations (for example a rename) before the DCM plan.
set -euo pipefail
: "${FLY_DB:?set FLY_DB}"
FILE="flyway/PRE_DCM_TARGET"
if [ ! -s "$FILE" ]; then echo "PRE_DCM_TARGET empty: no prerequisite migrations"; exit 0; fi
TARGET="$(tr -d '[:space:]' < "$FILE")"
[[ "$TARGET" =~ ^[0-9]+$ ]] || { echo "PRE_DCM_TARGET must be a whole version number"; exit 1; }
HAS="$(snow sql -x --format json -q "SELECT COUNT(*) AS N FROM ${FLY_DB}.INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'CALC_SUPPORT' AND TABLE_NAME = 'FLYWAY_HISTORY_CALC_SUPPORT'" | jq -r '.[0].N')"
if [ "$HAS" = "0" ]; then echo "$FLY_DB has no Flyway history yet (fresh environment): DCM goes first"; exit 0; fi
echo "Applying prerequisite migrations up to version $TARGET in $FLY_DB"
scripts/flyway_ci.sh migrate -target="$TARGET"
```

> **TO-CONFIRM [TC-08-06]:** The Flyway option `-target=<version>` (standard Flyway setting, not read in the docs this pack used). How to check at lab time: `docker run --rm redgate/flyway:13.12.2 -?` or the Flyway "target" setting page on documentation.red-gate.com. Record the result in file 13.

> **TO-CONFIRM [TC-08-05]:** The shape of `snow sql --format json` output (a list of row objects with upper-case column names is expected), and the columns of `INFORMATION_SCHEMA.TABLES` and `INFORMATION_SCHEMA.DATABASES` used here. How to check at lab time: run the `snow sql` command by hand and read the output. Record the result in file 13.

**`scripts/smoke_clone.sql` and `scripts/assert_zero_int_writes.sql`**. You wrote both in Lab 5 (file 07, step 5.12). Use them unchanged. The smoke script refuses to run against anything that is not an `SBX_DEV_` database and takes `-D clone_db=<database>`. The assertion script takes `-D before_results=<n> -D before_exec=<n>` and raises an error if the two INT counts changed. The workflow below passes the numbers it recorded before the clone was created.

> **TO-CONFIRM [TC-08-07]:** That `snow sql -f ... -D name=value` with the `<% name %>` template form and a `$$ ... $$` block behaves in CI as it did on your laptop in Lab 5 (see TC-07-07). How to check at lab time: re-run the two commands from Lab 5 step 5.12 against a clone before you rely on them in a workflow. Record the result in file 13.

### Step 5.3 Resolve the action pins

Run in: **GitHub UI**.

The workflows pin third-party actions by full 40-character commit SHA, with the version in a comment. A tag can be moved by its owner, a full SHA cannot. This pack never invents SHAs, so each workflow contains a placeholder and fails at startup until you replace it. That is on purpose.

1. Open `https://github.com/actions/checkout`, Releases, pick the newest release, click its commit, copy the 40-character SHA. Replace every `<FULL_SHA_ACTIONS_CHECKOUT>` with it.
2. Do the same on `https://github.com/snowflakedb/snowflake-actions` for `<FULL_SHA_SNOWFLAKE_ACTIONS>`. Use a `v3` release.
3. Write both SHAs and dates in the "Result at lab time" lines of file 13.

> **TO-CONFIRM [TC-G22]:** Current SHA and inputs of `snowflakedb/snowflake-actions`. The Snowflake docs name `snowflakedb/snowflake-actions@v3`, with input `cli-version`, and DCM sub-actions `dcm/plan@v3`, `dcm/deploy@v3`. The sub-actions are not used here, because the CLI commands are clearer to learn. How to check: the action's README on GitHub. Record the result in file 13.

> **TO-CONFIRM [TC-G22b]:** Which `actions/checkout` major version to use. Snowflake pages show both `@v4` and `@v7`. Pin by SHA from the newest release you can see. Record the result in file 13.

> **TO-CONFIRM [TC-G16]:** The Snowflake CLI minimum for DCM. The public docs say 3.24.0. NAIC's working notes first said 3.16. The workflows install the version in the repository variable `SNOW_CLI_VERSION` (3.33.0). Re-check the pin at lab time with `pip index versions snowflake-cli` or PyPI. Record the result in file 13.

### Step 5.4 `validate.yml`: lint and Flyway validate

Run in: **Terminal** (create `.github/workflows/validate.yml`).

This workflow runs on every pull request and on pushes to non-main branches. It never writes to Snowflake.

```yaml
name: validate
on:
  pull_request:
  push:
    branches-ignore: [main]
permissions:
  contents: read
jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<FULL_SHA_ACTIONS_CHECKOUT>   # version tag here, TC-G22b
        with:
          persist-credentials: false
      - name: No key material in the repository
        run: |
          if git ls-files | grep -E '\.(p8|pem)$|rsa_key|\.env\.flyway$'; then echo "key-like file is tracked"; exit 1; fi
          if git grep -lE 'BEGIN (ENCRYPTED )?PRIVATE KEY' -- . ':!docs' ':!.github'; then echo "private key text found"; exit 1; fi
      - name: Flyway file names
        run: |
          bad=$(find flyway/migrations -name '*.sql' | xargs -n1 basename | grep -vE '^(V[0-9]+(_[0-9]+)?__[a-z0-9_]+|R__[a-z0-9_]+)\.sql$' || true)
          if [ -n "$bad" ]; then echo "bad migration names:"; echo "$bad"; exit 1; fi
      - name: DCM definitions never contain a hybrid table
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          if grep -RInE 'DEFINE[[:space:]]+HYBRID' dcm/; then echo "hybrid tables belong to Flyway"; exit 1; fi
      - name: Manifest keeps the clone safety placeholder
        if: ${{ vars.DCM_MODE != 'off' }}
        run: grep -q 'SBX_DEV_SET_ON_CLI' dcm/calc_engine/manifest.yml
      - name: Helper scripts compile
        run: python3 -m py_compile scripts/check_plan_for_drops.py scripts/assert_plan_empty.py

  flyway-validate:
    # Fork pull requests get no secrets, so only run for pushes and same-repository pull requests.
    if: ${{ github.event_name == 'push' || github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: ubuntu-24.04
    env:
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
      HYBRID_MODE: ${{ vars.HYBRID_MODE }}
      PRIVATE_KEY_PASSPHRASE: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_PASSPHRASE }}
    steps:
      - uses: actions/checkout@<FULL_SHA_ACTIONS_CHECKOUT>
        with:
          persist-credentials: false
      - name: Write key file
        env:
          KEY_CONTENT: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
        run: |
          umask 077
          printf '%s\n' "$KEY_CONTENT" > "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
          echo "KEY_FILE=$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8" >> "$GITHUB_ENV"
      - name: Flyway info and validate against INT (pending migrations allowed)
        env:
          FLY_DB: SBX_INT
        run: |
          scripts/flyway_ci.sh info
          scripts/flyway_ci.sh validate -ignoreMigrationPatterns='*:pending'
      - name: Delete key file
        if: always()
        run: rm -f "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
```

Why `-ignoreMigrationPatterns='*:pending'`: a plain `validate` treats a migration that exists in the folder but is not yet applied as an error. A pull request is exactly that situation. What you want to catch here is an applied migration whose file was edited or deleted.

> **TO-CONFIRM [TC-08-10]:** Flyway's `ignoreMigrationPatterns` setting and its `*:pending` pattern (standard Flyway setting, not read in the docs this pack used). How to check at lab time: add a new `V99__test.sql`, run `flyway validate` with and without the option, and compare. Delete the test file afterwards. Record the result in file 13.

> **VERIFIED:** `validate` compares available migrations to applied ones; a checksum mismatch or a missing migration is an error (https://documentation.red-gate.com/flyway/reference/commands, read 2026-11-16).

Security note: `SNOWFLAKE_PRIVATE_KEY_RAW` is only read inside the "Write key file" step. The passphrase is in the job environment, so every step of this job can see it. That is a trade-off kept for readability. The defence is that all actions are pinned by SHA and the job has `contents: read` only.

### Step 5.5 `pr-clone-check.yml`: the pull request pipeline

Run in: **Terminal** (create `.github/workflows/pr-clone-check.yml`).

Read the comments. The numbered comments are the order the workflow follows.

```yaml
name: pr-clone-check
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
# One run per pull request at a time. Never cancel a running job: its cleanup must not race the next run.
concurrency:
  group: pr-clone-${{ github.event.number }}
  cancel-in-progress: false
jobs:
  clone-plan-deploy-test:
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: ubuntu-24.04
    environment: pr-clone
    env:
      PR_NUMBER: ${{ github.event.number }}
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
      SNOWFLAKE_AUTHENTICATOR: SNOWFLAKE_JWT
      SNOWFLAKE_PRIVATE_KEY_RAW: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
      PRIVATE_KEY_PASSPHRASE: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_PASSPHRASE }}
      SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
      HYBRID_MODE: ${{ vars.HYBRID_MODE }}
    steps:
      - uses: actions/checkout@<FULL_SHA_ACTIONS_CHECKOUT>
        with:
          persist-credentials: false
      - uses: snowflakedb/snowflake-actions@<FULL_SHA_SNOWFLAKE_ACTIONS>   # v3, TC-G22
        with:
          cli-version: ${{ vars.SNOW_CLI_VERSION }}
      - name: Names and key file
        env:
          KEY_CONTENT: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
        run: |
          umask 077
          printf '%s\n' "$KEY_CONTENT" > "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
          {
            echo "KEY_FILE=$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
            echo "TICKET=PR${PR_NUMBER}"
            echo "CLONE_DB=SBX_DEV_PR${PR_NUMBER}"
            echo "PROJECT=SBX_DEV_PR${PR_NUMBER}.CALC_SUPPORT.CALC_ENGINE_DCM"
            echo "SHORT_SHA=${GITHUB_SHA::7}"
          } >> "$GITHUB_ENV"
      - name: 0. Connection test
        run: snow connection test -x

      - name: 1. Record INT row counts before
        run: |
          snow sql -x --format json -q "SELECT (SELECT COUNT(*) FROM SBX_INT_FIN.FIN_CALCULATION.FIN_CALCULATION_RESULT_BASE) AS R, (SELECT COUNT(*) FROM SBX_INT.CALC_SUPPORT.EXECUTIONS) AS E" > "$RUNNER_TEMP/int_before.json"
          cat "$RUNNER_TEMP/int_before.json"
          {
            echo "BEFORE_R=$(jq -r '.[0].R' "$RUNNER_TEMP/int_before.json")"
            echo "BEFORE_E=$(jq -r '.[0].E' "$RUNNER_TEMP/int_before.json")"
          } >> "$GITHUB_ENV"

      - name: 2. Fresh clone for this pull request
        run: |
          snow sql -x -q "CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('${TICKET}')"
          snow sql -x -q "CALL SBX_INT.CALC_DEV_SUPPORT.CREATE_CALC_DEV_CLONE('${TICKET}')" | tee "$RUNNER_TEMP/clone_result.txt"
          if grep -q '"status": *"ERROR"' "$RUNNER_TEMP/clone_result.txt"; then echo "clone creation failed"; exit 1; fi

      - name: 3. Flyway info and validate on the clone
        run: |
          FLY_DB="$CLONE_DB" scripts/flyway_ci.sh info
          FLY_DB="$CLONE_DB" scripts/flyway_ci.sh validate -ignoreMigrationPatterns='*:pending'

      - name: 4. Prerequisite migrations (rename protocol, ordering rule 2)
        run: FLY_DB="$CLONE_DB" scripts/pre_dcm_migrations.sh

      - name: 5. DCM project object in the clone
        if: ${{ vars.DCM_MODE != 'off' }}
        run: snow dcm create "$PROJECT" --target CLONE --from dcm/calc_engine --if-not-exists -x

      - name: 6. Full DCM plan, then fail on any DROP
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          snow dcm plan "$PROJECT" --target CLONE --from dcm/calc_engine \
            -D "calc_db='${CLONE_DB}'" -D "fin_db='${CLONE_DB}'" --save-output -x
          PLAN="$(ls dcm/calc_engine/out/plan*.json | head -n1)"
          cp "$PLAN" "$RUNNER_TEMP/plan_full.json"
          python3 scripts/check_plan_for_drops.py "$RUNNER_TEMP/plan_full.json" | tee -a "$GITHUB_STEP_SUMMARY"

      - name: 7. DCM deploy to the clone
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          snow dcm deploy "$PROJECT" --target CLONE --from dcm/calc_engine \
            -D "calc_db='${CLONE_DB}'" -D "fin_db='${CLONE_DB}'" --alias "pr${PR_NUMBER}-${SHORT_SHA}" -x

      - name: 8. Flyway migrate on the clone
        run: FLY_DB="$CLONE_DB" scripts/flyway_ci.sh migrate

      - name: 9. Re-plan must be empty
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          snow dcm plan "$PROJECT" --target CLONE --from dcm/calc_engine \
            -D "calc_db='${CLONE_DB}'" -D "fin_db='${CLONE_DB}'" --save-output -x
          PLAN="$(ls dcm/calc_engine/out/plan*.json | head -n1)"
          python3 scripts/assert_plan_empty.py "$PLAN" | tee -a "$GITHUB_STEP_SUMMARY"

      - name: 10. Smoke test inside the clone
        run: snow sql -x -f scripts/smoke_clone.sql -D clone_db="$CLONE_DB"

      - name: 11. INT must not have changed
        run: snow sql -x -f scripts/assert_zero_int_writes.sql -D before_results="$BEFORE_R" -D before_exec="$BEFORE_E"

      - name: Delete key file
        if: always()
        run: rm -f "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"

  cleanup:
    needs: clone-plan-deploy-test
    if: always() && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-24.04
    environment: pr-clone
    env:
      PR_NUMBER: ${{ github.event.number }}
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
      SNOWFLAKE_AUTHENTICATOR: SNOWFLAKE_JWT
      SNOWFLAKE_PRIVATE_KEY_RAW: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
      PRIVATE_KEY_PASSPHRASE: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_PASSPHRASE }}
      SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
    steps:
      - uses: snowflakedb/snowflake-actions@<FULL_SHA_SNOWFLAKE_ACTIONS>   # v3, TC-G22
        with:
          cli-version: ${{ vars.SNOW_CLI_VERSION }}
      - name: Drop the clone, whatever happened before
        run: snow sql -x -q "CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('PR${PR_NUMBER}')"
      - name: Prove no clone is left
        run: snow sql -x -q "SHOW DATABASES LIKE 'SBX_DEV_PR${PR_NUMBER}'"
```

What to notice in `pr-clone-check.yml`:

- **Ticket from a number, never from text.** The ticket is `PR` plus the pull request number, which GitHub supplies as an integer. The clone procedure still validates it (`^[A-Za-z0-9]{1,40}$`). Never build a ticket from a branch name or a title, because those are free text typed by a person.
- **Values go through environment variables**, not `${{ ... }}` pasted into a `run:` line. Pasting untrusted text into a shell line is the classic Actions injection mistake.
- **Concurrency.** `cancel-in-progress: false` means a new push queues behind the running job. By default GitHub replaces an older queued run with the newest one, so you never get a long queue.

> **VERIFIED:** At most one running and one pending job per concurrency group; a newer pending run cancels the older pending one; `cancel-in-progress: true` would also cancel the running one (https://docs.github.com/en/actions/how-tos/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs, read 2026-11-16).

- **Cleanup is its own job** with `if: always()` and the same environment. If the first job is cancelled or fails, the clone is still dropped. A scheduled sweeper (step 5.8) covers the case where even that did not run.
- **Variables for DCM** use `-D "calc_db='...'"` with inner single quotes, as in Lab 3. Without the quotes you get error 1510.
- **Role.** The job runs as `SBX_CICD_PIPELINE` (from `SNOWFLAKE_ROLE`). The manifest's CLONE target names `SBX_CALC_DEV` as `project_owner`, but the CLI never switches roles for you.

> **VERIFIED:** Targets do not select the connection, account or role. The CLI warns on account mismatch, and at create time on role mismatch with `project_owner`, but does not switch (https://docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dcm-commands/overview, read 2026-11-16).

> **TO-CONFIRM [TC-08-01]:** Which role should run `snow dcm create/plan/deploy` for the CLONE target in CI. The clone procedure hands all clone objects to `SBX_CALC_DEV`, and `SBX_CICD_PIPELINE` holds `SBX_CALC_DEV` (grant 1 of the bootstrap), so the pipeline role should inherit enough. If `plan` or `deploy` fail with an ownership or privilege error, the alternative is to also grant `SBX_CALC_DEV` to the service user (`GRANT ROLE SBX_CALC_DEV TO USER SBX_SVC_CICD;` as `ACCOUNTADMIN`) and run the DCM steps with `--role SBX_CALC_DEV`. How to check at lab time: run the first PR and read the error. Record the result in file 13.

> **TO-CONFIRM [TC-08-03]:** General GitHub behaviours used here that the pack's docs reading did not cover: a `pull_request` run checks out the merge of your branch into the base branch; pull requests from forks receive no secrets; `docker`, `jq` and `python3` are installed on `ubuntu-24.04`; `if: always()` also runs when the run is cancelled. How to check at lab time: GitHub docs for the `pull_request` event and the runner image README. Record the result in file 13.

### Step 5.6 `deploy-int.yml`: merge to main deploys to `int`

Run in: **Terminal** (create `.github/workflows/deploy-int.yml`).

```yaml
name: deploy-int
on:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: deploy-int
  cancel-in-progress: false
jobs:
  deploy-int:
    runs-on: ubuntu-24.04
    environment: int          # a required reviewer must approve before anything below runs
    env:
      FLY_DB: SBX_INT
      PROJECT: SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
      SNOWFLAKE_AUTHENTICATOR: SNOWFLAKE_JWT
      SNOWFLAKE_PRIVATE_KEY_RAW: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
      PRIVATE_KEY_PASSPHRASE: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_PASSPHRASE }}
      SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
      HYBRID_MODE: ${{ vars.HYBRID_MODE }}
    steps:
      - uses: actions/checkout@<FULL_SHA_ACTIONS_CHECKOUT>
        with:
          persist-credentials: false
      - uses: snowflakedb/snowflake-actions@<FULL_SHA_SNOWFLAKE_ACTIONS>   # v3, TC-G22
        with:
          cli-version: ${{ vars.SNOW_CLI_VERSION }}
      - name: Key file and short sha
        env:
          KEY_CONTENT: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
        run: |
          umask 077
          printf '%s\n' "$KEY_CONTENT" > "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
          echo "KEY_FILE=$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8" >> "$GITHUB_ENV"
          echo "SHORT_SHA=${GITHUB_SHA::7}" >> "$GITHUB_ENV"
      - name: Connection test
        run: snow connection test -x
      - name: Prerequisite migrations (ordering rule 2)
        run: scripts/pre_dcm_migrations.sh
      - name: Full DCM plan, then fail on any DROP
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          snow dcm plan "$PROJECT" --target INT --from dcm/calc_engine --save-output -x
          PLAN="$(ls dcm/calc_engine/out/plan*.json | head -n1)"
          python3 scripts/check_plan_for_drops.py "$PLAN" | tee -a "$GITHUB_STEP_SUMMARY"
      - name: DCM deploy
        if: ${{ vars.DCM_MODE != 'off' }}
        run: snow dcm deploy "$PROJECT" --target INT --from dcm/calc_engine --alias "int-${SHORT_SHA}" -x
      - name: Flyway migrate
        run: scripts/flyway_ci.sh migrate
      - name: Re-plan must be empty
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          snow dcm plan "$PROJECT" --target INT --from dcm/calc_engine --save-output -x
          PLAN="$(ls dcm/calc_engine/out/plan*.json | head -n1)"
          python3 scripts/assert_plan_empty.py "$PLAN" | tee -a "$GITHUB_STEP_SUMMARY"
      - name: Delete key file
        if: always()
        run: rm -f "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
```

Two design points to discuss:

- The approval happens before the plan runs. The reviewer approves a commit, not a plan. A stricter design has a first job without an environment that plans and publishes the summary, and a second job with `environment: int` that deploys. Because DEPLOY recomputes its own plan, the drop check runs again inside the deploy job anyway.
- `concurrency` with `cancel-in-progress: false` is the `resource_group` idea: two merges in a row queue up, they do not run side by side.

### Step 5.7 `promote-qa-prod.yml`: manual promotion to `qa` or `prod`

Run in: **Terminal** (create `.github/workflows/promote-qa-prod.yml`).

```yaml
name: promote-qa-prod
on:
  workflow_dispatch:
    inputs:
      target:
        description: "Environment to promote to"
        type: choice
        options: [qa, prod]
        required: true
permissions:
  contents: read
concurrency:
  group: promote-${{ inputs.target }}
  cancel-in-progress: false
jobs:
  promote:
    runs-on: ubuntu-24.04
    environment: ${{ inputs.target }}      # qa or prod, each with its own reviewer
    env:
      TARGET: ${{ inputs.target }}
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
      SNOWFLAKE_AUTHENTICATOR: SNOWFLAKE_JWT
      SNOWFLAKE_PRIVATE_KEY_RAW: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
      PRIVATE_KEY_PASSPHRASE: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_PASSPHRASE }}
      SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
      HYBRID_MODE: ${{ vars.HYBRID_MODE }}
    steps:
      - uses: actions/checkout@<FULL_SHA_ACTIONS_CHECKOUT>
        with:
          persist-credentials: false
      - uses: snowflakedb/snowflake-actions@<FULL_SHA_SNOWFLAKE_ACTIONS>   # v3, TC-G22
        with:
          cli-version: ${{ vars.SNOW_CLI_VERSION }}
      - name: Map the input to names (fixed list, no free text)
        env:
          KEY_CONTENT: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
        run: |
          case "$TARGET" in
            qa)   echo "FLY_DB=SBX_QA"   >> "$GITHUB_ENV"; echo "DCM_TARGET=QA"   >> "$GITHUB_ENV" ;;
            prod) echo "FLY_DB=SBX_PROD" >> "$GITHUB_ENV"; echo "DCM_TARGET=PROD" >> "$GITHUB_ENV" ;;
            *) echo "unknown target"; exit 1 ;;
          esac
          umask 077
          printf '%s\n' "$KEY_CONTENT" > "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
          echo "KEY_FILE=$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8" >> "$GITHUB_ENV"
          echo "SHORT_SHA=${GITHUB_SHA::7}" >> "$GITHUB_ENV"
      - name: Prerequisite migrations (only if this environment already has a history)
        run: scripts/pre_dcm_migrations.sh
      - name: DCM project object, plan, drop check, deploy
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          PROJECT="SBX_$( [ "$DCM_TARGET" = QA ] && echo QA || echo PROD ).CALC_SUPPORT.CALC_ENGINE_DCM"
          snow dcm create "$PROJECT" --target "$DCM_TARGET" --from dcm/calc_engine --if-not-exists -x
          snow dcm plan "$PROJECT" --target "$DCM_TARGET" --from dcm/calc_engine --save-output -x
          PLAN="$(ls dcm/calc_engine/out/plan*.json | head -n1)"
          python3 scripts/check_plan_for_drops.py "$PLAN" | tee -a "$GITHUB_STEP_SUMMARY"
          snow dcm deploy "$PROJECT" --target "$DCM_TARGET" --from dcm/calc_engine --alias "${TARGET}-${SHORT_SHA}" -x
      - name: Flyway migrate (own history table in this database)
        run: scripts/flyway_ci.sh migrate
      - name: Re-plan must be empty
        if: ${{ vars.DCM_MODE != 'off' }}
        run: |
          PROJECT="SBX_$( [ "$DCM_TARGET" = QA ] && echo QA || echo PROD ).CALC_SUPPORT.CALC_ENGINE_DCM"
          snow dcm plan "$PROJECT" --target "$DCM_TARGET" --from dcm/calc_engine --save-output -x
          PLAN="$(ls dcm/calc_engine/out/plan*.json | head -n1)"
          python3 scripts/assert_plan_empty.py "$PLAN" | tee -a "$GITHUB_STEP_SUMMARY"
      - name: Delete key file
        if: always()
        run: rm -f "$RUNNER_TEMP/sbx_svc_cicd_rsa_key.p8"
```

A fresh `SBX_QA` has an empty `CALC_SUPPORT` schema (created by the bootstrap). The first run therefore creates everything: DCM first (sequences, standard tables, procedures, grants), then Flyway (hybrid pair or stand-ins, indexes, data change, rename). This is the "baseline INT, create QA and PROD for real" story from NAIC's calc-admin plan, in miniature (Lab 7 revisits it).

> **TO-CONFIRM [TC-08-04]:** Whether Flyway migrations written in Lab 4 replay cleanly on a brand-new environment. Two risks. First, `V3` asserts an `EXPECTED_ROWS` constant on `CALCULATIONS`, but a new QA database has different rows than INT, so the assertion can fail. Second, `V4` renames `FORMULA_CATEGORIES`, but in a new environment DCM has already created `CALC_CATEGORIES` and the old table never existed, so a plain `ALTER TABLE ... RENAME` fails (an `ALTER TABLE IF EXISTS ... RENAME` form would be a no-op). How to check at lab time: run the promote to `qa` and read the Flyway error, if any. Fix forward: make those two migrations tolerant, and record what you changed. Never edit an applied migration in INT. Record the result in file 13.

Note that the `QA` and `PROD` DCM targets in the manifest use `fin_db: SBX_INT_FIN`, the shared results database. Check that your manifest matches.

### Step 5.8 `sweep-stale-clones.yml`: the safety net

Run in: **Terminal** (create `.github/workflows/sweep-stale-clones.yml`).

A job can die before cleanup (runner lost, quota). The sweeper drops any `SBX_DEV_PR<number>` database older than one day. It only ever builds a ticket from a name that matches `^SBX_DEV_PR[0-9]+$`, then calls the guarded drop procedure, never a raw `DROP DATABASE`.

```yaml
name: sweep-stale-clones
on:
  schedule:
    - cron: "17 5 * * *"
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: sweep
  cancel-in-progress: false
jobs:
  sweep:
    runs-on: ubuntu-24.04
    environment: pr-clone
    env:
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
      SNOWFLAKE_AUTHENTICATOR: SNOWFLAKE_JWT
      SNOWFLAKE_PRIVATE_KEY_RAW: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_RAW }}
      PRIVATE_KEY_PASSPHRASE: ${{ secrets.SNOWFLAKE_PRIVATE_KEY_PASSPHRASE }}
      SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
    steps:
      - uses: snowflakedb/snowflake-actions@<FULL_SHA_SNOWFLAKE_ACTIONS>   # v3, TC-G22
        with:
          cli-version: ${{ vars.SNOW_CLI_VERSION }}
      - name: Drop PR clones older than one day
        run: |
          snow sql -x --format json -q "SELECT DATABASE_NAME FROM SBX_INT.INFORMATION_SCHEMA.DATABASES WHERE DATABASE_NAME LIKE 'SBX_DEV_PR%' AND CREATED < DATEADD(day, -1, CURRENT_TIMESTAMP())" > old.json
          jq -r '.[].DATABASE_NAME' old.json | while read -r NAME; do
            if [[ "$NAME" =~ ^SBX_DEV_PR[0-9]+$ ]]; then
              TICKET="${NAME#SBX_DEV_}"
              echo "dropping stale clone $NAME"
              snow sql -x -q "CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('${TICKET}')"
            fi
          done
```

> **TO-CONFIRM [TC-08-05]:** The `INFORMATION_SCHEMA.DATABASES` view and its `CREATED` column, and the JSON key `DATABASE_NAME`. How to check at lab time: run the `SELECT` in a worksheet and read the columns. Fallback: `SHOW DATABASES LIKE 'SBX_DEV_PR%'` and read `created_on`. Record the result in file 13.

### Step 5.9 First pull request run

Run in: **Terminal**. You run git yourself. The assistant never runs `git commit`, `git push`, `git pull` or `git fetch` for you unless you ask for that exact command in that request.

```bash
git switch -c feature/LAB6A
# edit README.md: add one line, for example "Lab 6 first pipeline run"
git add .github/workflows scripts README.md
git commit -m "ci: add six workflows and helper scripts"
git push -u origin feature/LAB6A
```

Run in: **GitHub UI**. Open a pull request from `feature/LAB6A` to `main`. Open the "Checks" tab.

You should see:

- `validate` with two jobs, green.
- `pr-clone-check` with job `clone-plan-deploy-test`, then job `cleanup`. The first run is the one most likely to fail on a small setup detail. Read the failing step name: it tells you which numbered stage broke. Typical first failures are the placeholder SHAs (workflow does not start), a passphrase or key-file problem in the Flyway step (TC-G21, TC-08-02), and a role problem in the DCM steps (TC-08-01).
- On success, the step summary of step 6 shows `plan changes: N  drops: 0`. N is greater than zero on a fresh clone because the new project adopts every cloned object. Step 9 shows `re-plan is empty`.

Run in: **Snowsight worksheet** while the job runs:

```sql
SHOW DATABASES LIKE 'SBX_DEV_PR%';       -- one row while the job runs: SBX_DEV_PR1 (your PR number)
```

After the `cleanup` job finishes, run it again. You should see 0 rows.

If the first run turns a TO-CONFIRM item into a fix, write it into the "Result at lab time" line for that id in file 13.

### Step 5.10 Read what the pipeline proved

Run in: **GitHub UI**, open the job log of `clone-plan-deploy-test`.

1. Step 1 prints the INT counts before the clone exists. Step 11 runs `assert_zero_int_writes.sql` with those numbers and must print `PASS: INT row counts unchanged`. That is your "zero writes to INT" evidence.
2. Step 6 prints `plan changes` and lists any DROP. With no definition removed, there are none. The other changes are adoptions of the cloned objects.
3. Step 8 prints the Flyway result. For a branch that adds no migration it says the schema is up to date.
4. Step 9 prints `re-plan is empty`.

Checkpoint: if the PR touches only a README, why does step 6 still list plan changes? (Answer 4 in section 6.)

### Step 5.11 Merge and approve `int`

Run in: **GitHub UI**. Merge the pull request. Open the Actions tab, run `deploy-int`. The run shows "Waiting for review". Click **Review deployments**, tick `int`, approve.

You should see: the job runs, step "DCM deploy" prints a deployment with alias `int-<7 characters of the commit>`, and the final re-plan is empty.

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SHOW DEPLOYMENTS IN DCM PROJECT SBX_INT.CALC_SUPPORT.CALC_ENGINE_DCM;   -- your alias int-<sha> is in the list
```

Mode D-off: the DCM steps are skipped, you only see the Flyway step.

### Step 5.12 Promote to `qa` and `prod`

Run in: **GitHub UI**, Actions, `promote-qa-prod`, Run workflow, target `qa`. Approve the `qa` environment. Repeat with `prod`.

Run in: **Snowsight worksheet**:

```sql
SHOW TABLES IN SCHEMA SBX_QA.CALC_SUPPORT;
SELECT "version", "description", "success"
FROM SBX_QA.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT ORDER BY "installed_rank";
```

You should see the engine tables, the hybrid pair (Mode H) or the stand-ins (Mode S), and a history that lists the same versions as INT, in the QA database, with its own rows.

> **TO-CONFIRM [TC-08-08]:** The column names of the Flyway history table on Snowflake (lower-case quoted names such as `"version"` are expected). How to check at lab time: `SELECT * FROM SBX_QA.CALC_SUPPORT.FLYWAY_HISTORY_CALC_SUPPORT;` and read the header. Record the result in file 13.

If the promote fails in Flyway, see TC-08-04 above. This is a real finding about replaying migrations on a new environment, not a typo of yours.

#### If you are in Mode S (no hybrid tables)

Set the repository variable `HYBRID_MODE` to `S`. `flyway_ci.sh` then reads `migrations/common` plus `migrations/standard`. Everything else in this lab runs unchanged. Remember that switching modes after a history exists needs a fresh history, because the checksums of the two variants differ.

#### If DCM is not available (Mode D-off)

Set `DCM_MODE` to `off`. The steps with `if: ${{ vars.DCM_MODE != 'off' }}` are skipped. The Flyway, clone, smoke and zero-write steps still run. Replace the DROP check with a read-along: in file 05, look at the sample plan output and read what a DROP entry looks like.

### Step 5.13 Optional stretch: GitHub OIDC, no stored key (1 hour)

Run in: **Snowsight worksheet**, role `ACCOUNTADMIN` (connection `sbx_admin`), only for this stretch.

```sql
CREATE USER SBX_SVC_OIDC
  TYPE = SERVICE
  DEFAULT_ROLE = SBX_CALC_READER
  DEFAULT_WAREHOUSE = SBX_CALC_WH
  WORKLOAD_IDENTITY = (
    TYPE = OIDC
    ISSUER = 'https://token.actions.githubusercontent.com'
    SUBJECT = 'repo:<GITHUB_OWNER>/snowflake-calc-sandbox:environment:int'
  );
GRANT ROLE SBX_CALC_READER TO USER SBX_SVC_OIDC;
```

> **VERIFIED:** The `WORKLOAD_IDENTITY = (TYPE = OIDC ISSUER = ... SUBJECT = ...)` form on a `TYPE = SERVICE` user. `SUBJECT` must equal the `sub` claim GitHub issues. When a job sets `environment:`, GitHub always uses the form `repo:<owner>/<repo>:environment:<name>` regardless of trigger (https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/github-action, read 2026-11-16).

> **TO-CONFIRM [TC-G13]:** Whether OIDC or workload-identity users can be created and used in a trial account. The docs do not say either way. How to check at lab time: run the `CREATE USER` above. If it fails, copy the exact error into file 13 and stop the stretch. Record the result in file 13.

Run in: **Terminal** (create `.github/workflows/oidc-connection-test.yml`):

```yaml
name: oidc-connection-test
on: workflow_dispatch
permissions:
  id-token: write      # allows the job to request a GitHub OIDC token
  contents: read
jobs:
  oidc-test:
    runs-on: ubuntu-24.04
    environment: int
    env:
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_ROLE: SBX_CALC_READER
      SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
    steps:
      - uses: snowflakedb/snowflake-actions@<FULL_SHA_SNOWFLAKE_ACTIONS>   # v3, TC-G22
        with:
          cli-version: ${{ vars.SNOW_CLI_VERSION }}
          use-oidc: true
      - name: Who am I (no key, no password in this job)
        run: snow sql -x -q "SELECT CURRENT_USER(), CURRENT_ROLE()"
```

> **VERIFIED:** With `use-oidc: true` the action sets `SNOWFLAKE_AUTHENTICATOR=WORKLOAD_IDENTITY`, `SNOWFLAKE_WORKLOAD_IDENTITY_PROVIDER=OIDC`, `SNOWFLAKE_AUDIENCE=snowflakecomputing.com` and `SNOWFLAKE_TOKEN`; the workflow needs `permissions: id-token: write`; the Snowflake CLI must be 3.11.0 or higher; you still pass the account (https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/github-action and https://github.com/marketplace/actions/snowflake-actions, read 2026-11-16).

Run the workflow from the Actions tab and approve `int`. You should see one row with `SBX_SVC_OIDC` and `SBX_CALC_READER`.

Limits to remember:

- **Flyway.** The Flyway Snowflake driver page lists key pair, PAT, OAuth and SSO, and does not list workload identity. So today a pipeline that uses OIDC for the Snowflake CLI can still need a key for Flyway. NAIC's target says Flyway also moves to federation, which is why this item is open.

> **TO-CONFIRM [TC-G28]:** Flyway authentication with workload identity. Not listed in the Flyway Snowflake driver page. How to check at lab time: search the Flyway release notes and the driver page for "workload identity" or "OIDC". Record the result in file 13.

- **Subject strings.** GitHub issues a different `sub` for pull requests without an environment, branch pushes and environment jobs. One Snowflake user matches one subject. NAIC solves it with separate identities for review, INT, QA and PROD.
- **Wildcards.** A trailing `*` wildcard for the subject was announced in a 2026_08 behaviour change bundle and is disabled by default. Do not rely on it. (Seen in a search snippet only, not opened.)

### Step 5.14 Supply-chain review

Run in: **Terminal** and **editor**. This is a reading and checking step. Make a table in your notes with three columns: rule, where the sandbox enforces it, how NAIC would enforce it.

| Rule NAIC cares about | Where it lives in the sandbox | How NAIC applies it |
|---|---|---|
| No `latest` tags | `redgate/flyway:13.12.2` in `flyway_ci.sh`; `cli-version` from `SNOW_CLI_VERSION` | Internally mirrored, immutable images; versions in a reviewed profile (profiles 9.22.1, core 3.4.2, deployment 3.0.0). |
| Pin by digest | Tag plus digest `sha256:ccc7fe019c49ea1f7d1a63bab7c4f5d490570783cc0ccf8edccc49a209f5f7c0` for `redgate/flyway:13.12.2` | Digest recorded in the mirror, SBOM and scan evidence kept. |
| Pin actions by commit SHA | `uses: ...@<FULL_SHA>` with version comment | Include pinned by version in a profile, not by branch. |
| No runtime downloads in deploy jobs | The only download is the image pull and the action install. A hardened version pre-bakes them. | No `curl`, no package install inside deploy jobs. |
| Tools from the internal mirror | The sandbox pulls from public sources | Snowflake CLI 3.33.0 is in the internal PyPI proxy. There is no Snowflake CLI image. Flyway must come as an internal immutable image. |
| Least privilege tokens | `permissions: contents: read`, `id-token: write` only in the OIDC job | MR jobs never receive INT, QA or PROD credentials. |
| Separate identities per environment | One service user in the sandbox (limit) | Review, INT, QA and PROD each have an identity. |

> **VERIFIED:** Pinning styles for `snowflakedb/snowflake-actions` are a 40-character SHA (most secure), a patch tag, or a floating major tag; the action does not pin the Snowflake CLI itself, so set `cli-version` (https://github.com/marketplace/actions/snowflake-actions, read 2026-11-16).

> **TO-CONFIRM [TC-08-09]:** The Flyway image digest. The digest above was seen on 2026-11-16 for `redgate/flyway:13.12.2` and must be re-read before you use it. How to check at lab time: Docker Hub tag page for `redgate/flyway`. To pin by digest, set `FLY_IMAGE=redgate/flyway:13.12.2@sha256:<digest you read>`. Record the result in file 13.

Optional exercise: change `FLY_IMAGE` in `flyway_ci.sh` to use the digest, run `info`, and confirm nothing else changes.

## 6. Checkpoint questions

1. Which GitLab feature does `concurrency: group: deploy-int` with `cancel-in-progress: false` replace, and why is cancelling a running deploy dangerous?
2. Why is the ticket for a pull request clone built from the pull request number and not from the branch name?
3. In `pr-clone-check.yml`, why is cleanup a separate job with `if: always()`?
4. A pull request that only edits `README.md` still shows plan changes in step 6. Name one reason.
5. What does step 11 compare, and why are the "before" numbers recorded in step 1 and not read at the end?
6. Why can the DROP check pass in the pull request and still the deploy to `int` remove an object?
7. Why does the promote workflow map `qa` and `prod` to names with a `case` statement instead of using the input directly in a command?
8. What makes OIDC safer than a key-pair secret, and what still needs a key in the sandbox?

**Answers**

1. The GitLab `resource_group`. Cancelling a running deploy can stop it between DDL statements, and Snowflake DDL commits on its own, so you would be left with half the change applied.
2. The number is an integer chosen by GitHub. A branch name or title is free text typed by a person and could contain anything. The procedure also validates the ticket, but the first line of defence is not to build it from text.
3. If the first job fails or is cancelled, later steps in it do not run. A separate job with `if: always()` runs anyway, so the clone is dropped.
4. The clone gets a brand-new DCM project object with no deployment history. DCM does not yet manage any of the cloned objects, so the first plan lists them as changes (adoption of existing objects, as in Lab 3). Only a DROP is a problem. The re-plan in step 9 is the one that must be empty.
5. The two counts of INT result rows and executions, before and after the run. The "before" values must be taken before the clone is created, otherwise a leak that happened early would already be part of the baseline.
6. DEPLOY recomputes its own plan against whatever INT looks like at that moment. If INT changed after the PR plan, or the PR plan ran against a clone and INT differs, the deploy can differ. The deploy job runs the drop check again, but only a human who reads the plan sees the intent.
7. The input comes from a choice list, but defence in depth says never place any input text into a shell command or an object name. A fixed `case` list gives only known values.
8. OIDC removes a long-lived secret: the token is short-lived and tied to the repository and environment. Flyway still needs a key because the driver page does not list workload identity (TC-G28).

## 7. Break it

**Drill: delete a definition in a pull request.**

1. Run in **Terminal**: `git switch -c feature/LAB6B`. Delete `dcm/calc_engine/sources/definitions/tables/FORMULAS.sql`. Commit and push. You run git yourself.
2. Open a pull request.
3. Symptom: `pr-clone-check` fails at step 6 with `plan changes: ... drops: 1` and a line `DROP TABLE SBX_DEV_PR<n>.CALC_SUPPORT.FORMULAS` (the domain text may differ).
4. Diagnosis: read the step summary. DEPLOY drops objects that are managed but no longer defined. Step 7 never ran, so the clone still has the table.
5. Check the cleanup job ran and `SHOW DATABASES LIKE 'SBX_DEV_PR%'` returns nothing.
6. Recovery: restore the file in your editor (or ask the assistant to restore it, then you run the git command), push, and watch the run go green.

Lesson: the drop check turns a silent data-loss deploy into a red pull request. It cannot tell a deliberate drop from an accident, so a human still has to decide. NAIC's version uses an exact-FQN approval file and fails if an approval is unused.

## 8. Limitations seen in this lab

| Limitation | Experiment that exposed it | What covers it | Status |
|---|---|---|---|
| Approval happens before the plan, not on the plan | Step 5.6 design point | Split into plan job and deploy job; drop check inside deploy | VERIFIED by docs: DEPLOY recomputes its plan |
| DROP check sees only the saved plan, not the intent | Break it | Exact-FQN approval file (NAIC design) | VERIFIED (design) |
| Environments with reviewers need a public repository on Free plans | Step 5.1 | Make the repo public, or use `workflow_dispatch` plus branch protection | VERIFIED; learner plan TO-CONFIRM (TC-08-03) |
| Fork pull requests get no secrets | Step 5.4 `if:` guard | Only same-repository PRs run clone jobs | TO-CONFIRM (TC-08-03) |
| Flyway has no workload identity login documented | Step 5.13 | Key file for Flyway until supported | TO-CONFIRM (TC-G28) |
| Migrations may not replay on a new environment | Step 5.12 | Tolerant migrations, `IF EXISTS` renames, pending-count assertions | TO-CONFIRM (TC-08-04) |
| One service user and role for all environments | Whole lab | NAIC uses separate identities per environment | VERIFIED (design) |
| Job-level secrets are visible to every step | Step 5.4 security note | SHA pins, least `permissions`, or step-level `env:` | VERIFIED (design) |
| Clone cost and time scale with data | Step 5.9 | `DATA_RETENTION_TIME_IN_DAYS = 0`, sweeper | VERIFIED (docs: clone is size-of-data for hybrid tables) |

## 9. What you learned

- Every GitLab concept NAIC plans (stages, protected environments, `resource_group`, OIDC, merged results, `rules:`) has a GitHub Actions counterpart, and you built each one.
- A pull request pipeline can create a private clone, run Flyway and DCM into it, fail on a DROP, prove an empty re-plan and a zero-write source, then always clean up.
- Ordering matters: DCM first for new objects, Flyway first for renames, and the pipeline needs a switch for the second case.
- Approval gates protect environments, but only a reviewed plan protects data. DEPLOY recomputes.
- Pin everything, give each job the least permission, and prefer a short-lived identity to a stored key.

## 10. Cleanup

Run in: **Snowsight worksheet** (role `SBX_CICD_PIPELINE`):

```sql
SHOW DATABASES LIKE 'SBX_DEV_%';
-- for every row left over:  CALL SBX_INT.CALC_DEV_SUPPORT.DROP_CALC_DEV_CLONE('<ticket>');
```

Keep: the six workflow files, the helper scripts, `SBX_QA` and `SBX_PROD` contents (Lab 7 and the drills use `SBX_INT`, QA and PROD are optional later). If you did the OIDC stretch and are finished, as `ACCOUNTADMIN`: `DROP USER IF EXISTS SBX_SVC_OIDC;`.

Cost: each pull request run creates a clone (near zero storage until data diverges) and runs on `SBX_CALC_WH` for a few minutes at X-Small. Check the resource monitor `SBX_CALC_RM` in Snowsight after the lab.

## 11. State left behind

Six workflow files committed by you (`validate.yml`, `pr-clone-check.yml`, `deploy-int.yml`, `promote-qa-prod.yml`, `sweep-stale-clones.yml`, `oidc-connection-test.yml`); helper scripts `scripts/check_plan_for_drops.py`, `scripts/assert_plan_empty.py`, `scripts/flyway_ci.sh`, `scripts/pre_dcm_migrations.sh` (plus `scripts/smoke_clone.sql` and `scripts/assert_zero_int_writes.sql` from Lab 5); an empty `flyway/PRE_DCM_TARGET` file; at least one green PR run with the clone created and dropped; `int` deployed through approval; `SBX_QA` and `SBX_PROD` filled; no `SBX_DEV_PR*` database left.

## 12. Open items for file 13

TC-G04, TC-G10, TC-G13, TC-G16, TC-G21, TC-G22, TC-G22b, TC-G28, TC-08-01, TC-08-02, TC-08-03, TC-08-04, TC-08-05, TC-08-06, TC-08-07, TC-08-08, TC-08-09, TC-08-10.

## 13. Next

Go on to [Lab 7: the calc-admin lifecycle in miniature](09-lab7-calc-admin-lifecycle-in-miniature.md).
