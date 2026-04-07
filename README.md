# Night Shift

Centralized prompt library for scheduled Claude Code maintenance tasks across multiple projects.

Each file in `tasks/` is a self-contained prompt that a scheduled trigger can fetch and execute against any repo. Project-specific details (test commands, key pages, doc language, push protocol) live in each project's `CLAUDE.md` under a **Night Shift Config** section — task prompts here stay client-agnostic.

## Tasks

| # | Task | Mode |
|---|------|------|
| 00 | Implement plans | code, self-verified, direct to main |
| 01 | Changelog | docs, direct to main |
| 02 | User manual | docs, direct to main |
| 03 | ADR | docs, direct to main |
| 04 | Suggestions | docs, direct to main |
| 05 | Tests | code, self-verified, direct to main |
| 06 | Accessibility | code, self-verified, direct to main |
| 07 | i18n completeness | code, self-verified, direct to main |
| 08 | Security audit | one PR per issue |
| 09 | Bug hunt | one PR per issue |
| 10 | SEO & metadata | one big PR |
| 11 | Performance | one big PR |

Recommended execution order: `00 → 01-04 → 05 → 06 → 07 → 08 → 09 → 10 → 11`.

## How triggers use it

In each project's scheduled trigger, use a thin wrapper prompt:

```
Fetch the task instructions from
https://raw.githubusercontent.com/perandre/night-shift/v1/tasks/05-tests.md
and execute them against this repository. Read CLAUDE.md for project-specific
configuration (test commands, build commands, key pages, push protocol, etc.).
```

The trigger contains no task logic — all of it lives here.

## Per-project config

Each project's `CLAUDE.md` should include a section like:

```markdown
## Night Shift Config
- Tasks: 1, 2, 3, 5, 8, 10
- Doc language: Norwegian
- Changelog format: "## Uke NN / ### Title / - Bullet"
- Test command: npm test
- Build command: npm run build
- Push: git push mirror main && git push origin main
- Key pages: /dashboard, /surveys, /people
```

## Version pinning

Triggers reference a tagged version in the raw URL (`v1`, `v2`, ...).

- **Update all projects:** create a new tag, bump the version in each project's triggers.
- **Test changes:** point one project at a branch URL before tagging.
- **Roll back:** point triggers at the previous tag.

## Adding a new project

1. Add the **Night Shift Config** section to the project's `CLAUDE.md`.
2. Create scheduled triggers for the desired task subset, each fetching from `https://raw.githubusercontent.com/perandre/night-shift/v1/tasks/NN-*.md`.
3. Run each task once manually (daytime) to validate output quality.
4. Enable the schedule.
