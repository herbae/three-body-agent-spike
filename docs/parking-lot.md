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

## Planner: high-risk re-run idempotency

**Surfaced by:** Task 7 code review on commit `21a4262`.

If a high-risk issue is re-planned (second `plan` label add, or manual re-dispatch) after a previous successful Planner run, the current flow breaks:

- Slug is deterministic, so `BRANCH` name repeats.
- `git push -u origin "$BRANCH"` fails non-fast-forward if the remote has diverged.
- `gh pr create` fails if a spec PR already targets that head.
- If the previous spec PR was merged, `docs/change-sets/${N}.md` already exists on main and `git commit` errors with "nothing to commit" under `set -e`.

Mitigations to consider:
- Detect an open spec PR on the same head and no-op with an issue comment explaining.
- Append a short run-id suffix to the branch name to allow fresh spec PRs.
- Detect `git commit --allow-empty` is needed and branch accordingly.

Not critical — no known user-facing regression until we start re-planning issues, which the normal cadence shouldn't do.

## Upgrade `actions/checkout@v4 → @v5`

GitHub CI annotates that `actions/checkout@v4` uses Node.js 20 — deprecated June 2026. Bump to `@v5` across all workflows when convenient.
