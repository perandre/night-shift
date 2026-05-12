# PHP support — PLAN

Make Night Shift work as well on PHP repos (Laravel, Symfony, WordPress, raw PHP libraries) as it does today on JS/TS. Today the framework is JS/TS-centric in three concrete ways: the default toolchain detection only recognises JS/TS commands, several tasks hard-code JS-only library names as detection markers, and the routine sandbox almost certainly ships without `php`/`composer` (to be confirmed by the live tests below). Doc and discovery tasks should work for PHP today; everything that needs to *run* the project will not.

This plan generalises the framework so the next PHP app added to Night Shift configures itself the same way a Next.js app does — by writing five lines to `CLAUDE.md`.

## Live experiment in flight

Two routines were spawned to learn from real behaviour, not just static analysis:

| Routine name | ID | Target repo | Shape |
|---|---|---|---|
| `nightshift-php-test-monolog` | `trig_019ZGAA7RivfqBhM3RFGi5di` | `perandre/monolog` (fork of `Seldaek/monolog`) | PHP library, phpunit, no DB |
| `nightshift-php-test-symfony-demo` | `trig_01PWvqPYFAYdWdQMg9FfwzTa` | `perandre/symfony-demo` (fork of `symfony/demo`) | Full Symfony app with Doctrine ORM + SQLite |

Both routines:
- Have `cron: 0 0 1 1 *` (Jan 1 2027) so they do not auto-fire. **Delete from https://claude.ai/code/routines after reviewing the run output.**
- Have `CLAUDE.md` with a Night Shift Config block on the fork. The forks' default branch is `main`.
- Begin with a verbose environment probe (`php --version`, `composer --version`, `apt-get`, etc.) so the toolchain question gets a clear answer.
- Run `bundles/all.md` end-to-end so every task hits its decision point on a PHP codebase.
- End with a `## PHP support findings` (or `## PHP+DB support findings`) section the operator can paste back into this plan.

Inspect:
- Routines: `RemoteTrigger {action: list}` or the dashboard.
- PRs that ran open against: `gh pr list --label night-shift -R perandre/monolog` and `gh pr list --label night-shift -R perandre/symfony-demo`.

A "Findings — live runs" section near the bottom of this plan is the place to write the eventual report into.

## Context — what currently happens on a PHP repo

### Tasks that already work unchanged

These are language-agnostic — they read PRs / issues / commits / general source and propose changes:

- `find-bugs`, `find-security-issues` — pattern-based code review. **Caveat:** the prompt names JS-specific markers as examples (`dangerouslySetInnerHTML`, `NEXT_PUBLIC_*`); the LLM tends to generalise but a PHP-aware example list would lift signal.
- `suggest-improvements`, `document-decisions`, `update-changelog`, `update-user-guide` — doc-only tasks, no code execution.
- `work-on-issues`, `work-on-jira-issues` — issue → PR pipeline, no language assumptions.
- `triage-ci-failures` — operates on GitHub check outputs; CI provider matters, language doesn't.
- `improve-seo` — concepts are universal (titles, meta, OG, JSON-LD, sitemap, robots), but the *auth-detection heuristics* are Next.js-flavored (`middleware.ts`, `(dashboard)/`, `getServerSession`). Symfony/Laravel auth conventions need to be added so the "public vs. authenticated" classification works correctly.

### Tasks that silently self-skip on PHP

- `build-planned-features`, `add-tests` — need a working test command. The default list at `bundles/_multi-runner.md:455-456` is JS/TS + Rust + Python + Go. No `composer test`, no `vendor/bin/phpunit`, no `php artisan test`, no `vendor/bin/simple-phpunit`. So if a PHP repo has no `CLAUDE.md`, these self-skip silently. With an explicit `Test command: composer test` in `CLAUDE.md`, they will *attempt* to run — and then likely fail because the sandbox has no `php` (TBD by the live runs).
- `translate-ui` — the task hard-codes JS i18n libraries (`next-intl`, `react-i18next`, `formatjs`) as detection markers (`tasks/translate-ui.md:27`). On a Symfony app using `symfony/translator` with XLIFF in `translations/`, or a Laravel app with `lang/`, the task exits silently because it does not recognise the setup.
- `improve-accessibility` — example test libs are JS-only (`jest-axe`, `@axe-core/react`). The task is conceptually fine for PHP — it audits HTML — but Twig and Blade templates use different syntax than JSX, so the grep patterns inside the task need to be generalised.
- `improve-performance` — has hard JS/Next.js framing: bundle size, `next/image`, font-display swap, client/server boundary. None of this maps to a PHP backend's perf concerns (N+1 queries, missing indexes, opcache warm-up, twig cache config). For PHP this task needs either a separate prompt or a tagged section.

### Tasks that may fail noisily

- Anything that runs `Test command` / `Build command` from the config. If the routine sandbox lacks `php` (the live runs will confirm), every PR-producing task that includes a verify step will fail at the `composer test` step and abandon its branch. That's the dominant failure mode to expect.

## JS-bias inventory — specific edits

Each row below is a concrete file:line to change, with the proposed replacement framing. Wording reflects the spirit: don't hard-code language enumerations into tasks; teach the framework to consult config plus a per-language defaults table.

### 1. Default test/build command detection — `bundles/_multi-runner.md:455-456`

Current:
> Test command — First of: `npm test`, `pnpm test`, `yarn test`, `bun test`, `cargo test`, `pytest`, `go test ./...`. If none, test-needing tasks self-skip.
> Build command — First of: `npm run build`, `pnpm build`, `yarn build`, `bun run build`, `cargo build`, `go build ./...`. If none, build-needing tasks self-skip.

Change: add PHP detection at the start of the search order (composer.json beats package.json on a PHP repo, but the list is ordered by *what's checked first*, so order doesn't actually matter as long as detection is correct):

- **Test command — additional candidates:** `composer test` (if `composer.json` defines a `scripts.test` entry — check it first), `vendor/bin/phpunit`, `vendor/bin/simple-phpunit` (Symfony convention), `php artisan test` (Laravel), `vendor/bin/pest` (Pest test framework).
- **Build command — additional candidates:** `composer install --no-dev --no-progress`, `php bin/console assets:install` (Symfony), `php artisan optimize` (Laravel). For pure PHP libraries there is no "build" — leave build empty.

Tighten the wording so "first match wins" reads correctly across mixed projects (e.g. a Symfony app with both `composer.json` and `package.json` for its frontend assets).

### 2. i18n detection — `tasks/translate-ui.md:27`

Current:
> Detect the project's i18n setup (e.g. `next-intl`, `react-i18next`, `formatjs`, custom). Prefer an i18n setup inside `<app_path>` when scoped. If there is no i18n setup at all, exit silently — that is a config decision, not a night-shift fix.

Change to a *language-aware* detection list that's grouped by ecosystem and points to where translation files actually live:

```
Detect the project's i18n setup. Look first at composer.json/package.json
dependencies, then for canonical file shapes:

JS/TS  — next-intl, react-i18next, formatjs, vue-i18n. Translation files under
         locales/, messages/, or i18n/. UI strings live in JSX attributes.
PHP    — symfony/translator (XLIFF/YAML under translations/), Laravel
         Illuminate\Translation (PHP arrays under lang/ or resources/lang/),
         WordPress gettext (.po/.mo files), Drupal core translation.
Python — Django gettext (.po), Babel.
Ruby   — Rails I18n (config/locales/*.yml).
```

Then change Step 3 (grep for hardcoded strings) to:

> Grep UI templates for hardcoded user-visible strings. The relevant patterns depend on the template language detected:
> - JSX/TSX — `>literal<` text inside JSX, `placeholder=`, `aria-label=`, `title=`, `alt=`.
> - Twig — text inside `<tag>...</tag>` not already wrapped in `{{ … | trans }}` or `{% trans %}…{% endtrans %}`; attribute values not wrapped in `{{ … | trans }}`.
> - Blade — text outside `{{ __('…') }}` / `@lang('…')`; attributes likewise.
> - PHP — `echo "…"` / `'…'` returned from controller/template helpers that look user-facing.

### 3. Security audit examples — `tasks/find-security-issues.md:30-31`

Current:
> - XSS — `dangerouslySetInnerHTML`, `innerHTML`, unescaped output in templates
> - Exposed secrets — API keys, tokens, private URLs in client bundles, public repos, or `NEXT_PUBLIC_*`

Change to keep the existing examples and add PHP-specific markers (these belong in the *examples*, not as exhaustive detectors — the LLM generalises):

- XSS — `dangerouslySetInnerHTML`, `innerHTML`, `{!! $var !!}` (Blade raw), `{{ var | raw }}` (Twig raw), `echo $_GET[...]` without `htmlspecialchars()`.
- Injection — raw SQL string building in any language (`mysqli_query($conn, "SELECT … $var")`, missing prepared statements in PDO; raw string interpolation in Eloquent's `DB::raw()`).
- Exposed secrets — committed `.env`, `NEXT_PUBLIC_*`, Laravel `config/services.php` hard-coded keys, Symfony parameter files with literal tokens.

### 4. SEO auth-classification heuristics — `tasks/improve-seo.md:28-31`

Current (paraphrased): page is authenticated if it sits under `middleware.ts` auth matcher, uses `getServerSession`, lives in `(dashboard)/` route group, etc.

Change to add PHP-framework equivalents alongside the Next.js ones:

```
A page is authenticated if any of the following is true:
- (Next.js) sits under a middleware.ts auth matcher, calls getServerSession, etc.
- (Symfony)  the route's controller class or method has #[IsGranted('ROLE_USER')]
             or is configured in config/packages/security.yaml under
             access_control as requiring authentication.
- (Laravel)  the route is wrapped in Route::middleware('auth') or the controller
             has $this->middleware('auth') in __construct.
- (WordPress) the page is in wp-admin/ or behind is_user_logged_in().
- (any)      sits under a directory named admin/, dashboard/, account/, settings/.
```

### 5. Performance task framing — `tasks/improve-performance.md:26-33`

Current focuses on frontend/Next.js perf (bundle size, next/image, fonts, client/server boundary, Lighthouse).

Change to split into ecosystem-aware audit areas:

- **Always-on (any language):** N+1 query patterns, missing indexes implied by query shape, missing caching layers, blocking I/O on hot paths.
- **Frontend (JS app or app with frontend assets):** bundle size, image sizing & format, fonts, render-blocking resources, client/server boundary.
- **PHP backend:** opcache configuration, eager-loading of ORM relations (Doctrine `fetch: EAGER`, Eloquent `with()` placement), missing covering indexes against query patterns, sessions in DB without TTL.
- **DB (any backend that uses one):** missing indexes for actual query patterns observed in code, full table scans, lack of LIMIT on listing routes.

This makes the task useful for a Symfony backend even if it has no compiled frontend assets.

### 6. Manifest-level project-shape declaration (new — `manifest.yml`)

Today `manifest.yml` describes tasks; it doesn't describe project shapes. Add an optional `shapes:` block (or extend `Night Shift Config`) so a repo can declare *which* shape it is, and each task can consult that to enable/disable language-specific behaviour:

```yaml
# In CLAUDE.md → ## Night Shift Config
shape: php-symfony  # or: php-laravel, php-wordpress, php-library, next-app, sveltekit, rails…
```

Tasks that have shape-specific behaviour read this; tasks that don't, ignore it. The shape can default to "infer from composer.json / package.json" when unset. This is the cleanest way to keep tasks short while supporting many ecosystems.

## The toolchain question — routine sandbox vs. GitHub Actions

The Anthropic Cloud routine sandbox almost certainly does not ship with `php` and `composer` pre-installed (Node, Python, Go, Rust are the standard fare). The live runs will confirm; the report section at the bottom is where the answer lands.

There are three possible end-states:

### A. `php` and `composer` are already in the sandbox → no change needed

Best case. All test-running tasks work as soon as `CLAUDE.md` declares `Test command: composer test`. Nothing else to do beyond the JS-bias inventory above.

### B. They are not, but `apt-get install -y php-cli composer` works inside the routine

The wrapper prompts (or `_multi-runner.md`) need a pre-step that detects PHP repos and runs:

```bash
if [ -f composer.json ] && ! command -v php >/dev/null 2>&1; then
  apt-get update >/dev/null && apt-get install -y --no-install-recommends \
    php-cli php-mbstring php-xml php-curl php-sqlite3 unzip >/dev/null
  curl -fsSL https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
fi
```

This adds ~30-60s to the first repo per night per routine. The wrapper's "discover repos" loop already iterates per repo, so the install can live inside that loop with a `composer.json` guard.

Risk: extensions vary by app (Symfony demo needs `php-sqlite3`, Laravel apps may need `php-mysql` or `php-pgsql`). Either install a generous default set, or let the project declare them in `Night Shift Config`:

```yaml
PHP extensions: sqlite3, mbstring, xml, curl
```

### C. They are not, and apt-get is locked down → GitHub Actions backend wins for PHP

If routine sandboxes refuse package installation, the routine path stops being viable for PHP repos that need a verifying test run. Two options:

1. **Recommend GH Actions backend for PHP repos.** `.github/workflows/night-shift.yml` (already exists in this repo) can use `shivammathur/setup-php@v2` to install whatever PHP + extensions the repo declares. The workflow runs `claude -p` with full shell access to a `setup-php`-prepared toolchain. This is the cleanest path. The skill's setup flow should detect PHP repos and default-recommend Actions for them.
2. **Routine for read-only tasks, Actions for verify-needing tasks.** A hybrid: doc tasks + audits run via routine (cheap, no test execution), build/test/code-fixes tasks run via Actions. Hard to maintain — two configs per repo.

**Recommendation regardless of A/B/C:** ship the JS-bias fixes from the inventory unchanged. They are pure prompt edits, low risk, and make the framework better for *all* languages, including JS where the current prompts are over-specific.

## New PHP-specific tasks worth considering

These are net-new tasks (not generalisations of existing ones) that would make Night Shift visibly useful on PHP repos:

- **`composer-audit`** — runs `composer audit` (the built-in advisory checker) and opens a PR pinning to safe versions when a known-vulnerable dependency is in use. Mirrors `npm audit` for the JS side, which Night Shift doesn't currently have either — could be a shared "dep-audit" task that branches on detected ecosystem.
- **`phpstan-baseline-shrink`** — for repos with a phpstan baseline file, attempt to remove the easiest baseline entries by fixing the underlying issue. Mirrors the spirit of `add-tests` (chipping away at debt nightly).
- **`migration-safety-audit`** — pre-flight check for Doctrine/Laravel migrations that touch large tables without batching, drop columns referenced by code, or rename without compat-shim. Parallels the existing Shopify `audit-migration-risk` task.
- **`php-cs-fixer-or-pint`** — auto-format with the project's chosen formatter (`php-cs-fixer`, `laravel/pint`) only when CI would block on formatting. High bar — silent if formatting is already clean.

All four are opt-in (Night Shift Config: `bundles: [..., php]`).

## Critical files

| File | Change |
|---|---|
| `bundles/_multi-runner.md` | Add PHP test/build command candidates to the defaults table at line 455-456. Add the optional PHP toolchain install block to the per-repo prelude. |
| `tasks/translate-ui.md` | Generalise i18n detection beyond JS libraries; widen the string-grep patterns to Twig/Blade/PHP. |
| `tasks/find-security-issues.md` | Add PHP-flavored XSS / injection / exposed-secrets examples. |
| `tasks/improve-seo.md` | Add Symfony/Laravel/WordPress auth-detection heuristics alongside the Next.js ones. |
| `tasks/improve-performance.md` | Restructure into always-on / frontend / PHP-backend / DB sections. |
| `tasks/improve-accessibility.md` | Generalise template-language references (Twig/Blade/JSX). |
| `manifest.yml` + `HOW-TO.md` | Document the optional `shape:` and `PHP extensions:` fields in Night Shift Config. |
| `skills/night-shift/SKILL.md` | When the picker detects `composer.json` and no `package.json` in a target repo, recommend the GitHub Actions backend (per the toolchain question above). Bump `NIGHT_SHIFT_VERSION` per `CLAUDE.md` workflow rule. |
| `tasks/composer-audit.md` (new) | New optional task. |
| `tasks/phpstan-baseline-shrink.md` (new) | New optional task. |

## Verification

1. **Reproduce on monolog** — after applying the JS-bias fixes, re-trigger `nightshift-php-test-monolog` and confirm: at least the audit bundle opens a meaningful security / bug / suggestion PR; `add-tests` and `build-planned-features` either run end-to-end or self-skip with a *correct* reason (no plan file, etc.) rather than the current "unrecognised test command".
2. **Reproduce on symfony-demo** — re-trigger `nightshift-php-test-symfony-demo` and confirm: `improve-seo` correctly classifies the admin routes (`/en/admin/...`) as authenticated and only audits `/`, `/en/blog/`, etc.; `improve-performance` either runs and identifies real N+1 risk in the Doctrine code, or self-skips with a *correct* reason; `translate-ui` recognises the Symfony translator setup or, if no hardcoded strings remain, self-skips silently (the demo is already localized).
3. **Toolchain test** — if path B (apt-get install) was chosen, time a cold-start install on a fresh routine; should be under 90s.
4. **Negative test on a JS repo** — re-trigger an existing Night Shift routine (e.g. `night-shift-build` against frisk) and confirm no behaviour change. The JS-bias generalisations must not regress JS support.

## Findings — live runs (to fill in after the routines complete)

### `nightshift-php-test-monolog` — `perandre/monolog`

- **Toolchain probe result:** _(paste output here — php / composer / apt-get availability)_
- **Per-task outcomes:** _(paste table from routine output)_
- **JS-bias surfaces actually hit:** _(citations the routine called out)_
- **PRs opened:** `gh pr list --label night-shift -R perandre/monolog --state all`
- **Notable observations:** _(free text)_

### `nightshift-php-test-symfony-demo` — `perandre/symfony-demo`

- **Toolchain probe result:** _(PHP extensions enumerated, sqlite3 presence)_
- **Per-task outcomes:** _(paste table)_
- **DB-shape observations:** _(N+1 detection, migration handling, Doctrine awareness)_
- **App-shape observations:** _(Twig SEO, translate-ui on Symfony translator, etc.)_
- **PRs opened:** `gh pr list --label night-shift -R perandre/symfony-demo --state all`
- **Notable observations:** _(free text)_

### Decision matrix — routine vs. GitHub Actions for PHP

| Criterion | Routine wins | Actions wins |
|---|---|---|
| Toolchain control (PHP version, extensions) | _(fill in)_ | _(fill in)_ |
| Cold-start time per night | _(fill in)_ | _(fill in)_ |
| Setup ergonomics (steps for a new repo) | _(fill in)_ | _(fill in)_ |
| Cost (Claude Max vs. ANTHROPIC_API_KEY) | _(fill in)_ | _(fill in)_ |
| Failure visibility (where errors surface) | _(fill in)_ | _(fill in)_ |

## Scope estimate

- JS-bias inventory edits (six files): ~2-3 hours of careful prompt work.
- Optional PHP toolchain install block in `_multi-runner.md`: ~1 hour, gated on path B above.
- New PHP-specific tasks (composer-audit, phpstan-baseline-shrink): ~2 hours each.
- Skill picker UX for PHP-shape default-to-Actions: ~1 hour.
- Documentation and verification re-run: ~1 hour.

Total: half a day to two days depending on how many new PHP-specific tasks land in the first cut. The JS-bias inventory alone (no new tasks, no toolchain install) is a worthwhile first PR — it makes Night Shift better on JS repos too.

## Open questions

1. Should `Night Shift Config` add an explicit `language:` or `shape:` field, or rely on inference from `composer.json` / `package.json`? Inference is friendlier; explicit is more debuggable. Recommend: infer with an override.
2. For path B (apt-get install), what's the minimum extension set we should install by default? Recommend the union of what the live test runs needed (probably `mbstring xml curl sqlite3` covers most apps).
3. Are there PHP frameworks beyond Symfony / Laravel / WordPress / Drupal that should be first-class? `CakePHP`, `Yii`, `Phalcon` exist but with much smaller footprints — probably handled adequately by generic "PHP library" shape.
