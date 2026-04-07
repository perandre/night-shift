# Night Shift

![Night Shift](night-shift.png)

While you sleep, an AI agent runs maintenance jobs across your repositories. You wake up to commits and PRs, and a one-line summary in each project's history file.

## Get started

Open Claude Code in any project and paste this:

```
Fetch https://raw.githubusercontent.com/perandre/night-shift/main/SETUP.md and follow it to set up Night Shift on my account. Ask me which repos to include.
```

Claude will ask you which repositories to manage, confirm the schedule, and create three nightly scheduled jobs in your account. Two minutes, no manual config.

Want to try it once before scheduling? Paste this in any project instead — runs all four bundles against the current repo, no scheduling:

```
Fetch https://raw.githubusercontent.com/perandre/night-shift/main/bundles/all.md and execute it against this repository.
```

## What you'll find in your repo tomorrow morning

- **Planned features partly built** — picks the next phase from a `docs/*-PLAN.md` file and opens a PR
- **Updated docs** — changelog entries, user guide pages, decision records
- **New test coverage** — fills coverage gaps following your existing test patterns
- **Fixed accessibility issues** — WCAG AA violations on key pages
- **Translated UI strings** — moves hardcoded text into your i18n system
- **Audit PRs** — security, bug, SEO, and performance issues, each as its own PR

Each affected repo gets a one-line entry in `docs/NIGHTSHIFT-HISTORY.md`.

## The bundles

| Bundle | What it does | Mode |
|---|---|---|
| **plans** | Implements the next phase of a planning document | One PR per plan |
| **docs** | Updates changelog, user guide, decision records, suggestions | Direct to main |
| **code-fixes** | Adds tests, fixes accessibility, completes translations | Direct to main |
| **audits** | Finds security / bug / SEO / performance issues | One PR per area |

`manifest.yml` is the single source of truth for what tasks exist, what they do, what bundle they belong to, and what order they run in. Edit one file to add, rename, reorder, or move tasks.

## Stopping Night Shift on a project

Add **either**:
- An empty file `.nightshift-skip` at the repo root
- A line `Night Shift: skip` in `CLAUDE.md`, `AGENTS.md`, or `README.md`

The wrapper reports `opted-out` for that repo and moves on.

## Customising per project

Add a `## Night Shift Config` section to the project's `CLAUDE.md`. All fields are optional — anything unset uses sensible defaults autodetected from your `package.json` and repo layout.

```markdown
## Night Shift Config
- Tasks: build-planned-features, update-changelog, add-tests, find-security-issues
- Doc language: Norwegian (nb)
- Test command: npm test
- Build command: npm run build
- Push: git push mirror main && git push origin main
- Key pages: /dashboard, /surveys, /people
```

## How to add a project, add a task, or run something now

See **[HOW-TO.md](HOW-TO.md)** — five copy-paste recipes covering the common operations.
