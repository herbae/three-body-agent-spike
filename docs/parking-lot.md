# Parking lot

Follow-ups surfaced during execution that are deliberately **not** on the project board, so the Implementer won't pick them up. Promote an entry to a real issue when we want to actually work it.

## 4.5 — Owner-type-agnostic GraphQL

**Surfaced by:** 2026-04-20 baseline smoke-test (`docs/smoke-tests/2026-04-20-baseline.md`).

Every GraphQL query that reads/mutates the Projects V2 board hardcodes `organization(login: "...")`. If someone clones the fork and owns the project as a **User** (not an Org), every board query returns `null` and the pipeline fails silently. Forced us to create a GitHub org during the baseline test.

Possible shapes:
- Add `github.owner_type: user | organization` (default `organization`) to `autoagent-config.yml`; export `AUTOAGENT_OWNER_TYPE`; the five GraphQL sites branch on it.
- OR detect owner type at loader time via `gh api users/<ORG>` and set the flag automatically.

Affects: `autoagent-implementer.yml` (x2), `autoagent-board-sync.yml` (x1), `auto-week-rollover.yml` (x2).

## Board-sync: handle `reopened`

The `pull_request` trigger only listens for `[opened, closed]`. If a PR is reopened, the board-sync doesn't fire and the card stays wherever it was (likely `Todo`, since `closed` moves it there). Add `reopened` to the types and either treat it like `opened` or branch on `action == reopened` to move to `Ready For QA` explicitly.

## Re-validate `Comment on success` post-`f735bfb`

`f735bfb` fixed the PR search in the Implementer's `Comment on success` step but the fix hasn't been exercised on a live run. Next dispatch will tell. If the comment still doesn't post, dig deeper.

## Upgrade `actions/checkout@v4 → @v5`

GitHub CI annotates that `actions/checkout@v4` uses Node.js 20 — deprecated June 2026. Bump to `@v5` across all workflows when convenient.
