# Night Shift — wall dashboard

A tiny static dashboard for a wall screen. Two numbers per project:

| Project | Hours last night | Hours total |
|---|---|---|
| frontkom/night-shift | 43m | 3.2h |
| perandre/ns | 29m | 1.0h |
| … | … | … |

## How to view it

- **Live, deployed:** https://perandre.github.io/ns/dashboard/ (auto-deploys on push to the mirror — see `CLAUDE.md`).
- **Local server:** `python3 -m http.server 8000` from the repo root, then open <http://localhost:8000/dashboard/>.
- **Directly from disk (`file://`):** also works — the page renders inline seed data and shows a small "preview" badge top-right. To see live data, switch to one of the two options above.

## How data gets in

The dashboard reads **one file**: `dashboard/runs.jsonl`. Each line is one
schema-conformant event (see `../reporting/schema.json`). The only field
the dashboard cares about is `duration_seconds` (wall-clock from env
start to idle — what the workflow already records).

You have three options for getting events into that file.

### Option 1 — Workflow auto-append (recommended for production)

Set two secrets in each repo that runs Night Shift:

```
NIGHTSHIFT_DASHBOARD_REPO   = "frontkom/night-shift"   # where dashboard/runs.jsonl lives
NIGHTSHIFT_DASHBOARD_TOKEN  = <PAT with contents:write on that repo>
```

Pass them through in the caller workflow (uncomment the two lines in
`github-actions/caller-template.yml`). The reusable workflow will then
append one JSONL line per run, with conflict-retry for concurrent runs
across repos. The dashboard picks the new line up on its next
60-second refresh.

### Option 2 — Manual append (for ad-hoc testing)

Append a line yourself, commit, and push. Example for a 27-minute run:

```bash
jq -nc \
  --arg repo "frontkom/my-app" \
  --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --arg run "https://github.com/frontkom/my-app/actions/runs/12345" \
  --argjson dur 1620 \
  '{
    repo: $repo, timestamp: $ts, run_url: $run,
    backend: "github-actions",
    bundles: {
      plans:        { status: "ok",      prs: [], commits: 0 },
      docs:         { status: "silent",  prs: [], commits: 0 },
      "code-fixes": { status: "skipped", prs: [], commits: 0 },
      audits:       { status: "skipped", prs: [], commits: 0 }
    },
    duration_seconds: $dur,
    model: "claude-opus-4-7"
  }' >> dashboard/runs.jsonl

git commit -am "dashboard: log run" && git push origin main && git push mirror main
```

### Option 3 — Demo only

Do nothing. The file already has 15 seed events (4 example projects),
so the dashboard looks populated out of the box. Useful while you wire
up Option 1.

## What "last night" means

The most recent UTC calendar date with any activity, globally across
all projects. A project that didn't run on that date shows `0h` so it's
obvious at a glance which projects are quiet.

## Status badge (top right)

| Badge | Meaning |
|---|---|
| `live` (teal) | Dashboard fetched `runs.jsonl` over HTTP successfully. |
| `preview` (amber) | Showing inline seed data — typically because the page was opened from `file://`, which blocks `fetch()`. |
| `error` (coral) | Couldn't load `runs.jsonl` *and* there was no inline seed. |
