# Night Shift — Setup runbook

You are Claude Code, running in a user's terminal. The user has pasted the Night Shift setup one-liner. Your job is to create three scheduled triggers in their account that will run Night Shift nightly across the repositories they specify. Follow this runbook step by step. Be brief, conversational, and concrete.

## Step 1 — Greet and explain

Tell the user, in 2-3 lines:

> I'll set up Night Shift on your account. It will create three scheduled jobs that run nightly: one for plans, one for docs + code fixes, one for audits. I need a list of GitHub repositories to include. You can change them later.

## Step 2 — Collect the repo list

Ask the user:

> Which GitHub repositories should Night Shift manage? Paste the URLs (one per line, or comma-separated). Personal and org repos both work, as long as the remote environment can clone and push to them.

Accept any of: `https://github.com/owner/repo`, `owner/repo`, `git@github.com:owner/repo.git`. Normalise each to `https://github.com/owner/repo` before using it. Strip any trailing `.git`.

If the user gives zero repos, abort and tell them to come back when they have at least one.

## Step 3 — Confirm the schedule

Default schedule (Europe/Oslo local time, converted to UTC for cron):

| Job | Local | UTC cron |
|---|---|---|
| plans | 01:00 | `0 23 * * *` |
| docs + code-fixes | 03:00 | `0 1 * * *` |
| audits | 05:00 | `0 3 * * *` |

Show the table to the user and ask:

> Schedule looks like this. Press enter to accept, or tell me a different timezone or hour for any of them.

If the user changes anything, recompute the cron in UTC and confirm before proceeding. (The cron must be in UTC — if the user gives local times, ask their timezone first if you don't already know it from prior context.)

## Step 4 — Create the three triggers

Use the `/schedule` skill (or the `RemoteTrigger` tool directly if available) to create these three scheduled jobs. **All three must use:**

- `model`: `claude-sonnet-4-6`
- `allowed_tools`: `["Bash", "Read", "Write", "Edit", "Glob", "Grep", "Task"]`
- `enabled`: `true`
- `sources[]`: every repo from Step 2 **plus** `https://github.com/perandre/night-shift` (always — the wrappers write a run log to it)

### Trigger 1 — Plans

- **name**: `night-shift-bundle-plans`
- **cron** (UTC, default): `0 23 * * *`
- **prompt**:
  ```
  Fetch https://raw.githubusercontent.com/perandre/night-shift/main/bundles/multi-plans.md and execute it. The wrapper auto-discovers all target repositories cloned into this session (excluding the night-shift repo itself), dispatches a Task subagent per target repo, and writes a run log to runs/YYYY-MM.md in the night-shift repo at the end.
  ```

### Trigger 2 — Docs + code-fixes

- **name**: `night-shift-bundle-docs-and-code-fixes`
- **cron** (UTC, default): `0 1 * * *`
- **prompt**:
  ```
  Fetch https://raw.githubusercontent.com/perandre/night-shift/main/bundles/multi-docs-and-code-fixes.md and execute it. The wrapper auto-discovers all target repositories cloned into this session (excluding the night-shift repo itself), dispatches a Task subagent per target repo to run the docs bundle then the code-fixes bundle in sequence, and writes a run log to runs/YYYY-MM.md in the night-shift repo at the end.
  ```

### Trigger 3 — Audits

- **name**: `night-shift-bundle-audits`
- **cron** (UTC, default): `0 3 * * *`
- **prompt**:
  ```
  Fetch https://raw.githubusercontent.com/perandre/night-shift/main/bundles/multi-audits.md and execute it. The wrapper auto-discovers all target repositories cloned into this session (excluding the night-shift repo itself), dispatches a Task subagent per target repo to run find-security-issues, find-bugs, improve-seo, and improve-performance (each opening its own PR), and writes a run log to runs/YYYY-MM.md in the night-shift repo at the end.
  ```

## Step 5 — Handle the trigger cap

If the user's plan rejects the create with `trigger_limit_reached`, tell them which existing triggers they have and ask which to delete to free slots. The Night Shift cap is 3 enabled triggers — disabled ones may also count, depending on plan. They'll need to delete via https://claude.ai/code/scheduled (the API does not allow delete).

## Step 6 — Summarise

Once all three triggers are created, print a final summary to the user:

```
✓ Night Shift is set up.

| Job | Schedule | Repos |
|---|---|---|
| plans | <local time> | <N> |
| docs + code-fixes | <local time> | <N> |
| audits | <local time> | <N> |

Tomorrow morning, check docs/NIGHTSHIFT-HISTORY.md in each repo for what
happened. To pause Night Shift on any project, drop a .nightshift-skip file
at its root. To customise per project, add a Night Shift Config section
to that project's CLAUDE.md — see https://github.com/perandre/night-shift
```

Stop here. Do not run any test runs proactively — let the user trigger one if they want.

## If anything fails

- API rejection on create → see Step 5.
- User unsure which repos → suggest they list their active personal/work repos and pick the 2-3 they touch most often.
- User wants to test before scheduling → suggest they paste the one-liner from `bundles/all.md` (single repo, no scheduling) into Claude Code in one project first.
