# Dependency Audit

Run the ecosystem's dependency audit and open a PR pinning known-vulnerable transitive deps to safe versions. **One PR per ecosystem found.**

## Read project config first
Read `CLAUDE.md` for **Night Shift Config**: test command, build command, default branch, push protocol. If the dispatcher passed `allowed_tasks` and `dep-audit` is not in it, exit silently.

**Audit scope.** Honor `Audit scope`, `Exclude`, and the **hardcoded baseline exclude** defined in `bundles/_multi-runner.md` → "Optional config fields" (single source of truth — do not inline the list here). Only consider manifests inside `Audit scope` (when set); never modify lockfiles under `Exclude` or the baseline list.

**Scoping.** If the dispatching multi-runner passes an `app_path` (non-empty, not `—`), operate inside that app only:
- Discover manifest files **under `<app_path>`** first.
- Use the app-scoped test and build commands from the scoped config.
- Branch: `night-shift/deps-<app-slug>-YYYY-MM-DD`.
- PR title: `night-shift/deps: <app_path> — pin <N> vulnerable packages`.

Without an `app_path`, behave repo-wide (root-level manifest).

## High bar — default is silent
Open a PR **only** when:
1. The ecosystem's official audit tool returned a known CVE / advisory (not a "low-confidence" or "informational" finding), **and**
2. The advisory has a stated fixed-in version, **and**
3. Bumping to that fixed-in version is a minor/patch bump under semver (or the project already pins to a range that admits the bump).

Major-version bumps, deprecation warnings, and license-policy violations do **not** qualify — they belong in a manual review, not a nightly automated PR.

**Deliberately stricter than `add-tests`.** `add-tests` has an exception that allows writing unit-only tests when dependency install fails (so the work isn't lost). This task has **no** such exception: pinning a vulnerable dep you cannot verify is worse than not pinning it at all. If `composer install` / `npm install` / `pip install -r requirements.txt` / etc. fails — or if the scoped test or build command fails after the bump — exit silently. Leave a note in `docs/SUGGESTIONS.md` listing the advisory so a human picks it up.

## Steps

1. Check for an existing open night-shift deps PR for this app (or repo when unscoped):
   ```
   gh pr list --search "night-shift/deps in:title" --state open
   ```
   If one exists for the same app, exit silently — do not stack PRs.

2. **Detect the ecosystem(s) by manifest file.** Pick the first manifest found inside the scoped path (or the repo root when unscoped). If more than one ecosystem is present (a polyglot repo), pick the one with the largest dependency surface tonight and leave the others for the next run — one PR per task per night.

   | Manifest | Ecosystem | Audit command | Apply-fix command | Audit tool ships with the toolchain? |
   |---|---|---|---|---|
   | `composer.json` + `composer.lock` | PHP | `composer audit --format=json` | Edit `composer.json` constraint, run `composer update --with-dependencies <pkg>` | Yes — built into Composer ≥ 2.4. |
   | `package.json` + `package-lock.json` | npm | `npm audit --json` | `npm audit fix` (only on the affected packages; never `--force`) | Yes — built into npm. |
   | `package.json` + `pnpm-lock.yaml` | pnpm | `pnpm audit --json` | `pnpm update <pkg>@<version>` | Yes — built into pnpm. |
   | `package.json` + `yarn.lock` | yarn | Detect version first (see note below). Berry (`yarn --version` ≥ 2, or `.yarnrc.yml` present): `yarn npm audit --json`. Classic (1.x, only `.yarnrc` or no rc file): `yarn audit --json`. | Berry: `yarn up <pkg>@<version>`. Classic: edit `package.json` + `yarn install`. | Yes — built into yarn. |
   | `pyproject.toml` / `requirements.txt` | Python | `pip-audit -f json` (preferred) or `safety check --json` | Edit pin, run `pip install -r requirements.txt` / `poetry update <pkg>` | **No** — needs `pip install pip-audit` (or `safety`) in the sandbox first. |
   | `Cargo.toml` + `Cargo.lock` | Rust | `cargo audit --json` | `cargo update -p <pkg> --precise <version>` | **No** — needs `cargo install cargo-audit` first. |
   | `Gemfile` + `Gemfile.lock` | Ruby | `bundle audit check --update` | `bundle update --conservative <pkg>` | **No** — needs the `bundler-audit` gem (`gem install bundler-audit`) first. |
   | `go.mod` + `go.sum` | Go | `govulncheck -json ./...` | `go get <pkg>@<version> && go mod tidy` | **No** — needs `go install golang.org/x/vuln/cmd/govulncheck@latest` first. |

   **Yarn version detection.** Lockfile presence alone doesn't tell you Berry vs Classic. Check, in order: (1) is `.yarnrc.yml` present in the repo? → Berry. (2) does `yarn --version` start with `1.`? → Classic. (3) otherwise (≥2) → Berry. Never run `yarn audit` against a Berry repo or `yarn npm audit` against a Classic repo — both will error with a non-obvious message and the task will exit silently instead of running.

   If none of these manifests are present, exit silently — there is nothing to audit.

3. **Run the audit.**
   - For tools that ship with the toolchain (composer, npm, yarn, pnpm — see the "ships with the toolchain?" column above), the command should just work; if it doesn't, the toolchain itself is missing and you should exit silently rather than try to repair the sandbox.
   - For tools that don't ship with the toolchain (pip-audit, cargo-audit, bundler-audit, govulncheck), attempt the documented install path **once** in a transient way that doesn't touch the project's own dep manifests (`pipx install pip-audit` or `pip install --user pip-audit`; `cargo install cargo-audit`; `gem install bundler-audit`; `go install golang.org/x/vuln/cmd/govulncheck@latest`). Never edit `composer.json`, `Gemfile`, `pyproject.toml`, etc. to add the audit tool as a project dep — that's an unrelated change that would land in the wrong PR. In particular, `composer require --dev roave/security-advisories` is **not** a substitute for `composer audit` — use the audit subcommand.
   - If the install step fails or the command is still unavailable afterwards, exit silently and leave a note in `docs/SUGGESTIONS.md` (repo-root): "Run `<audit cmd>` locally; sandbox missing the tool tonight." Do not fabricate findings.

4. **Filter findings.** Discard anything that doesn't meet all three "high bar" criteria above. If nothing remains, exit silently — most repos with a recently-run audit will have nothing actionable.

5. **Apply the smallest possible fix.** Bump only the affected package(s) to the lowest version that resolves the advisory. Do not "modernize" unrelated deps in the same PR. Re-run the audit to confirm the finding is gone and no new ones appeared.

6. **Verify nothing broke.** Run the scoped **test suite** and the scoped **build command**. Both must pass. If either regresses, revert the bump and exit silently — record the package + advisory in `docs/SUGGESTIONS.md` for human review.

7. Create the branch:
   ```
   # scoped:
   git checkout -b night-shift/deps-<app-slug>-YYYY-MM-DD
   # unscoped:
   git checkout -b night-shift/deps-YYYY-MM-DD
   ```

8. Push and open the PR (prefix title with `<app_path> — ` when scoped). The wrapper has already created the standard labels for this repo — just attach them. **Always use `--body-file`, never inline `--body`.** End the body with the Night Shift footer:
   ```
   cat > /tmp/night-shift-pr-body.md <<'EOF'
   ## Plain summary
   <1-2 sentences in English (PR review is always in English, regardless of the product's user language). Which packages were vulnerable, who could exploit them, and how the bump addresses it. No CVE jargon — paraphrase the impact in human terms. See bundles/_multi-runner.md → "Body header — Plain summary".>

   ## Summary
   Pinned <N> packages flagged by `<audit cmd>`.

   ## Advisories addressed
   - <pkg> <vulnerable range> → <fixed version> — <advisory id> — <one-line impact>

   ## Verification
   - `<audit cmd>` re-run: clean.
   - Test suite and build: passing.

   ---
   _Run by Night Shift • audits/dep-audit_
   EOF

   # Stagger PR creation. Spec: bundles/_multi-runner.md → "PR creation throttle".
   LAST=/tmp/night-shift-pr-last-created
   if [ -f "$LAST" ]; then
     ELAPSED=$(( $(date +%s) - $(cat "$LAST") ))
     [ "$ELAPSED" -lt 90 ] && sleep "$((90 - ELAPSED))"
   fi
   PR_URL=$(gh pr create --title "night-shift/deps: <app_path> — pin <N> vulnerable packages" \
     --label night-shift \
     --body-file /tmp/night-shift-pr-body.md)
   date +%s > /tmp/night-shift-pr-last-created
   # Post-create ritual — REQUIRED after every gh pr create. Do NOT return to the wrapper without running every line below. Skipping leaves PR bodies flattened (literal \n on GitHub) or auto-merge unarmed. Spec: bundles/_multi-runner.md.
   gh pr edit "$PR_URL" --add-label night-shift
   BODY=$(gh pr view "$PR_URL" --json body -q .body)
   case "$BODY" in *'\n'*) printf '%s' "$BODY" | python3 -c "import sys;sys.stdout.write(sys.stdin.read().replace(chr(92)+chr(110),chr(10)))" > /tmp/night-shift-body-fix.md && gh pr edit "$PR_URL" --body-file /tmp/night-shift-body-fix.md ;; esac
   gh pr merge "$PR_URL" --auto --squash 2>/dev/null || gh pr merge "$PR_URL" --auto || true
   ```

   **Self-review.** After the post-create ritual above, run the **Self-review + one revision** step from `_multi-runner.md` before returning your one-line result. One review, at most one revision commit, same branch; if the revision breaks tests, revert with `git push --force-with-lease` and keep the original PR.

## Idempotency
- One PR per ecosystem per night.
- If a sibling night-shift/deps PR is already open for the same ecosystem, exit silently.
- If the audit reports clean — or only contains findings that fail the high-bar filter — exit silently. **Never** fabricate findings to justify a PR.
