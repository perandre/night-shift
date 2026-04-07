# Bundle 1 — Plans & Docs

You are running the Night Shift **Plans & Docs** bundle on this repository.

## Setup
First, read `CLAUDE.md` for the **Night Shift Config** section: test command, build command, doc language, push protocol, key pages, and the project's task subset.

## Tasks
Task 00 **must run first** and complete before the others — it may add new features that the doc tasks should then cover. Tasks 01–04 are independent and can be done in any order after 00.

1. https://raw.githubusercontent.com/perandre/night-shift/v2/tasks/00-implement-plans.md
2. https://raw.githubusercontent.com/perandre/night-shift/v2/tasks/01-changelog.md
3. https://raw.githubusercontent.com/perandre/night-shift/v2/tasks/02-user-manual.md
4. https://raw.githubusercontent.com/perandre/night-shift/v2/tasks/03-adr.md
5. https://raw.githubusercontent.com/perandre/night-shift/v2/tasks/04-suggestions.md

## Execution rules
- For each task: fetch the file, read it, execute it exactly as written (including its own commit and push steps).
- If the project's Night Shift Config excludes a task, skip it and move on.
- If a task says "exit silently", that is success — continue with the rest.
- Do **not** stop the bundle on a single task's exit. Only stop if a code change in task 00 fails verification (test or build).
