# Task 00 — Implement Plans

Scan `docs/` for plan files matching `*-PLAN.md`. If any contain unimplemented phases with concrete file paths and code specs, implement the **next pending phase only** to limit blast radius.

## Read project config first
Read `CLAUDE.md` for the **Night Shift Config** section (test command, build command, push protocol). If task 0 is not in the project's task list, exit.

## Steps
1. List `docs/*-PLAN.md`. Skip plans marked **deferred**, **blocked**, or **on hold**.
2. For each remaining plan, identify phases. A phase is "implemented" if its referenced files/migrations/exports already exist with the described shape. Skip phases already done.
3. Pick the **first** plan with a pending phase. Implement only that one phase tonight.
4. Follow the plan's file paths and specs literally. Do not invent scope.
5. Run the project's **test command** and **build command**. Both must pass.
6. If anything fails, do not commit. Leave a note in the plan file under a `## Night Shift Notes` section describing what blocked you.

## Commit
Only if test + build pass:
```
git add -A
git commit -m "nightshift(plans): implement <plan-name> phase <N> — <short title>"
```
Push using the project's push protocol from CLAUDE.md.

## Idempotency
- One phase per night, ever. Never implement two phases in one run.
- If no pending phases exist across all plans, exit silently.
