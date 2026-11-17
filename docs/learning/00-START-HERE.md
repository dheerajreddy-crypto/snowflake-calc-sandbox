# 00. Start here: hands-on DCM + Flyway + CI/CD learning pack

> Read time: 20 minutes. Pack written 2026-11-16. Public documentation facts in this pack were read on 2026-11-16. Anything not read is marked **TO-CONFIRM** and collected in file 13.

## Purpose

You (Dheeraj) understand DCM Projects and Flyway only at a high level today. This pack lets you build a small personal sandbox in your own Snowflake trial account and your own GitHub repository, then learn both tools by doing, one lab at a time.

By the end you should be able to:

- Explain, from first principles, why the NAIC calc engine and Calc Admin need **DCM** (a Snowflake feature that deploys a declared end state of objects) and **Flyway** (a tool that applies numbered, ordered migration scripts).
- Show how the same work looks **without** either tool, and what each tool cannot do.
- Run a change through a pipeline: pull request, clone environment, plan, deploy, test, cleanup, promotion with approval.
- Decide which tool owns which kind of change, and say why.

Status of the two tools in NAIC, stated once and plainly: **both DCM and Flyway are part of the target design and both are being implemented.** DCM owns the supported engine objects (standard tables, procedures, functions, views, sequences, approved grants, ordinary tasks). Flyway owns hybrid tables, the Calc Admin lifecycle schema, explicit renames, data changes, clone support and the `FIN_CALCULATION` migrations. A working note dated 2026-11-11 recorded pausing DCM and dropping the INT project object. The team has since confirmed that both tools are in the design (architecture pages `calc-engine-complete-architecture.html`, `calc-engine-pipeline.html`, `calc-admin-with-flyway.html`), so the INT project is to be re-adopted. File 01 explains this in full.

## How the pack is organised

Every file is a Markdown file with an HTML copy next to it. Read them in order. Labs are separate files, each doing one concept.

| File | Content | Type |
|---|---|---|
| 00 | This file: how to use the pack, the assistant rules, the first prompt | Read |
| 01 | The real calc engine and Calc Admin, mapped to the sandbox | Read |
| 02 | Accounts, security, cost controls, tools, repository, bootstrap SQL | Setup |
| 03 | Lab 1: the old way (no tooling) | Hands-on |
| 04 | Lab 2: Flyway fundamentals | Hands-on |
| 05 | Lab 3: DCM fundamentals | Hands-on |
| 06 | Lab 4: both tools together, one owner per object | Hands-on |
| 07 | Lab 5: clones as ticket environments | Hands-on |
| 08 | Lab 6: CI/CD with GitHub Actions | Hands-on |
| 09 | Lab 7: Calc Admin lifecycle in miniature | Hands-on |
| 10 | Lab 8: failure drills and limitations | Hands-on |
| 11 | Capstone: one change four ways, comparison, quiz | Hands-on |
| 12 | Glossary, cheat sheets, troubleshooting | Reference |
| 13 | Open questions and verification log | Reference |
| index | Links to everything | Reference |

Each lab file uses the same 13 headings: Goal, Why this matters, Prerequisites and state check, Concepts, Step-by-step, Checkpoint questions, Break it, Limitations, What you learned, Cleanup, State left behind, Open items, Next.

Conventions used everywhere:

- Placeholders are written in angle brackets: `<ACCOUNT_IDENTIFIER>`, `<LEARNER_USER>`, `<GITHUB_OWNER>`, `<PASSPHRASE>`, `<PUBLIC_KEY_BODY>`. You replace them. Never paste real secrets into any file in the repository.
- Each step says where to run it: **Snowsight worksheet** (the Snowflake web UI), **Terminal**, or **GitHub UI**.
- Each step has a "You should see" block.
- Callouts: `VERIFIED` means the fact was seen on an official page on 2026-11-16 (URL given). `TO-CONFIRM [TC-id]` means it was not verified; the callout says how to check, and you record the result in file 13.
- Modes: `H` = hybrid tables available, `S` = standard-table stand-ins, `D` = DCM available, `D-off` = DCM not available. File 02 tells you how to find your mode. The default for a free trial is **Mode S + D**.

## Prerequisites

- A laptop with a terminal (macOS, Linux or Windows with WSL) and permission to install software.
- An email address for the Snowflake trial: use `dheeraj.banda3@gmail.com` (your personal address). No payment card is needed to start a trial.
- A GitHub account (personal). The sandbox repository must be **public** so that environment approvals work on a free plan (see file 02). It contains no secrets.
- Basic SQL. You do not need to know Snowflake scripting, YAML, Docker or GitHub Actions in advance. Each is introduced where it is first used.
- Roughly 20 to 30 hours in total, spread over several sessions (see "Time per lab").

## Rules for the assistant

These rules are for the AI assistant in the new session. They also tell you what to expect from it.

1. **Never ask for, accept, store or print passwords, private keys, passphrases, tokens or one-time codes.** If you paste one by mistake, the assistant tells you to rotate it and does not repeat it.
2. **Never write a secret into any file or commit one.** Placeholders only. The `.gitignore` in file 02 blocks the usual key and env file names.
3. **Use only the sandbox Snowflake account.** The connection names `sbx_admin`, `sbx_dev`, `sbx_owner`, `sbx_cicd` are the only ones allowed. If any other connection exists on the laptop (for example a NAIC one), the assistant does not use it and says so.
4. **Explain before acting.** For every step the assistant says what it will do, why, and where it runs, then waits for your go-ahead.
5. **Wait for your confirmation before every destructive step** (drops, `clean`, deleting a definition and deploying, `DROP DATABASE`, rotating keys). Say the exact object names first.
6. **You run git.** The assistant never runs `git commit`, `git push`, `git pull` or `git fetch` unless you ask for that specific operation in that request. It may run read-only git commands (`status`, `diff`, `log`).
7. **Snowflake statements:** the assistant may propose SQL and may run it only through the `sbx_*` connections, only after you agree. `ACCOUNTADMIN` is used only for the bootstrap and the resource monitor, never as a default.
8. **Mark unverified facts.** If a command or behaviour is not confirmed, the assistant says "TO-CONFIRM", runs the check with you and writes the result into file 13.
9. **Check for cost.** Before starting a long warehouse task, the assistant reminds you of the resource monitor and suspends the warehouse at the end of a session.
10. **Keep the repository clean.** Files go only into the folders named in the repository tree in file 02. The assistant tells you which files it created so you can commit them.

## First prompt to paste in the new session

Open the workspace that contains your `snowflake-calc-sandbox` repository, then paste this as your first message. Replace the bracketed parts.

```text
I am Dheeraj. I am following the hands-on DCM + Flyway learning pack in
docs/learning/ of this repository (snowflake-calc-sandbox). Start by reading
docs/learning/00-START-HERE.md, then 01, then 02, then the file that matches
the first "todo" or "doing" row in the Resume here table in 00.

Follow the "Rules for the assistant" section in 00 exactly. In particular:
- never ask me for or store passwords, private keys, passphrases or tokens;
- use only the sandbox connections sbx_admin, sbx_dev, sbx_owner, sbx_cicd;
- explain each step before you do it and wait for my yes before any destructive step;
- I run git myself; do not commit, push, pull or fetch;
- mark anything unverified as TO-CONFIRM and add it to file 13.

My current state: [for example "Snowflake trial not created yet" or
"finished file 02, HYBRID_MODE=S, DCM_MODE=on, snow version X, Flyway 13.12.2"].
Tell me which file we do next, what I need to have ready, and how long it takes.
Then guide me one step at a time. After each lab, help me update the Resume here
table and the verification log in file 13.
```

How to resume in a later session: paste the same prompt, say which row of the Resume here table you stopped at, and add what is true now (for example "Lab 3 step 5.7 done, project exists"). The assistant re-runs the "Prerequisites and state check" section of the lab to confirm the sandbox matches.

## Who runs what

| Action | Who |
|---|---|
| Create the Snowflake trial and GitHub repository, complete MFA | You |
| Generate the key pair, paste the public key into Snowsight, add the secrets to GitHub | You (the assistant guides but never sees the private key or passphrase) |
| Run SQL in Snowsight or through `snow sql` | You, or the assistant after you say yes, only on sandbox connections |
| Create and edit repository files | The assistant, after explaining; you review |
| `git add`, `commit`, `push`, `pull`, `fetch`, pull requests, merges, approvals | You |
| Destructive steps | You confirm; then either of you runs them |

## Time per lab

| File | What | Time |
|---|---|---|
| 00 | Start here | 20 min read |
| 01 | Context | 40 min read |
| 02 | Accounts and setup | 2 to 3 h |
| 03 | Lab 1, the old way | 1 to 1.5 h |
| 04 | Lab 2, Flyway fundamentals | 2.5 to 3 h |
| 05 | Lab 3, DCM fundamentals | 3 h |
| 06 | Lab 4, both together | 2.5 to 3 h |
| 07 | Lab 5, clones | 2.5 h |
| 08 | Lab 6, CI/CD | 3 h (+1 h OIDC stretch) |
| 09 | Lab 7, Calc Admin lifecycle | 2 to 2.5 h |
| 10 | Lab 8, failure drills | 2 to 3 h |
| 11 | Capstone | 3 h |
| 12, 13 | Reference | as needed |

> **Trial clock.** A Snowflake trial lasts 30 days from sign-up or until the free credits are used, whichever comes first. Plan the sessions so labs 03 to 11 fit in that window. See file 02 for details and cost controls.

> **VERIFIED:** the trial lasts 30 days or until free credits run out, whichever is first, and then the account is suspended (https://docs.snowflake.com/en/user-guide/admin-trial-account, read 2026-11-16).

## Progress checklist

Tick each box (edit this file in your copy) when done.

- [ ] Read 00 and 01
- [ ] 02: trial account created, MFA set, resource monitor active
- [ ] 02: `snow`, Flyway and git installed and verified
- [ ] 02: key pair generated, public key registered, secrets set in GitHub
- [ ] 02: bootstrap SQL run, repository created with the folder tree
- [ ] 02: probes run, `HYBRID_MODE` and `DCM_MODE` recorded
- [ ] Lab 1 (03): old way done, pain points written
- [ ] Lab 2 (04): Flyway fundamentals done
- [ ] Lab 3 (05): DCM fundamentals done, Lab 1 objects adopted
- [ ] Lab 4 (06): ownership split and decision table done
- [ ] Lab 5 (07): clone procedures and isolation proof done
- [ ] Lab 6 (08): workflows green, int approved, QA and PROD promoted
- [ ] Lab 7 (09): lifecycle miniature and role checks done
- [ ] Lab 8 (10): failure drills done
- [ ] Capstone (11): four ways done, quiz answered, teach-back done
- [ ] 13: verification log filled with your results

## Resume here

Update this table at the end of every session. Keep one row per file.

| Step | Status (todo/doing/done) | Date | Mode (H/S, D) | Notes |
|---|---|---|---|---|
| 00 Start here | todo | | | |
| 01 Context | todo | | | |
| 02 Accounts and setup | todo | | | |Mode not yet probed. Done: trial, MFA, snow 3.33.0, Docker + Flyway image (flyway/flyway:13.12.2), key pair, bootstrap parts 1-7, key-pair login as SBX_SVC_CICD works with flags. connections.toml done: sbx_dev, sbx_owner, sbx_cicd tested OK (sbx_admin skipped; personal key sbx_dheeraj_rsa_key, login name in `user`). Todo: probes H and D, GitHub repo + .gitignore + secrets, verification checklist. |
| 03 Lab 1 the old way | todo | | | |
| 04 Lab 2 Flyway fundamentals | todo | | | |
| 05 Lab 3 DCM fundamentals | todo | | | |
| 06 Lab 4 both together | todo | | | |
| 07 Lab 5 clones | todo | | | |
| 08 Lab 6 CI/CD | todo | | | |
| 09 Lab 7 Calc Admin lifecycle | todo | | | |
| 10 Lab 8 failure drills | todo | | | |
| 11 Capstone | todo | | | |
| 12 Glossary | todo | | | |
| 13 Open questions and log | todo | | | |

## Never-run-calculations note

This applies to the real **NAIC** environments only. On NAIC you never call `POST /executions`, `SP_RUN_CALCULATION`, any other `SP_RUN_*` procedure, or anything that calls them indirectly (pipelines, `SP_RUN_ALL`, Streamlit actions). Only the calc owner runs those. This pack never asks you to.

In your personal sandbox the rule does not apply. The sandbox compute procedure is deliberately named `SP_COMPUTE_MINI_RESULT` (not `SP_RUN_...`) so you never confuse it with the real one. It writes one harmless row into a table you own. Use the sandbox account only for this pack.

## Next

Go to [01-calc-engine-and-calc-admin-context.md](01-calc-engine-and-calc-admin-context.md).
