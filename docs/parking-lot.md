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

## Authenticate the change-set comment (filter by author)

**Surfaced by:** Task 8 code review on commit `1f139c6`.

The Implementer's fetch step (`autoagent-implementer.yml:364-365`) picks the latest issue comment whose body starts with `<!-- AUTOAGENT_CHANGE_SET -->` — **without verifying who posted it**. Any collaborator with issue-comment permission could post a comment with that marker after the Planner runs and cause the Implementer to use *their* change-set instead.

Worst case is prompt injection into Claude (the content is envsubst'd into the prompt, not exec'd by shell — existing hardening covers shell safety). Still, a real trust hole.

Mitigations to consider:
- Add an author filter to the jq: `select(.author.login == "<expected-login>" and (.body | startswith("<!-- AUTOAGENT_CHANGE_SET -->")))`. Today AGENT_PAT is user `herbae`'s token, so the comment's author is `herbae`. If we migrate to a GitHub App or bot account, switch the filter to that login.
- Or: have the Planner sign the marker with a rotating nonce stored on the project board / as a workflow artifact, which the Implementer cross-checks before trusting the comment.

Not blocking while the org is single-contributor, but should land before we open the repo to external collaborators.

## Implementer `Comment on success` race condition

**Surfaced by:** 2026-04-20 Planner smoke-test (issue #4 / run `24680644485`).

The Implementer's `Comment on success` step (`autoagent-implementer.yml:~414`) calls `gh pr list --search "head:${AUTOAGENT_BRANCH_PREFIX}${ISSUE_NUMBER}" --state open` to find the just-opened PR URL and comment on the issue. GitHub's PR search index has a few-second lag — the search ran ~6s after `gh pr create` and returned empty. The `|| echo ""` fallback swallows the race, so the step exits 0 but no comment is posted.

Fix options:
- Swap the search for a direct `gh api` call to `/repos/{owner}/{repo}/pulls?head={owner}:{branch}&state=open` (no search index in the loop).
- Or add a short retry: `for i in 1 2 3; do PR_URL=$(gh pr list ...); [ -n "$PR_URL" ] && break; sleep 3; done`.

Not critical — the PR still opens and the board still transitions; the issue just doesn't get the convenience comment. First-class priority the next time we touch the Implementer workflow.

## Merger: skip plan branches

**Surfaced by:** 2026-04-21 high-risk flow on `herbae-org/foreman-todo-api` issue #1 (while fixing board-sync's plan-branch bug).

The Merger's PR scan (`autoagent-merger.yml`) filters by `startsWith(.headRefName, "autoagent/")`. That set includes spec PRs on `autoagent/plan-*` branches. If the Merger cron (or a manual dispatch) fires while a spec PR is open, it would run the Claude merge-analysis and potentially auto-merge the spec — **bypassing the human review gate that high-risk is supposed to impose**.

Same fix as `fix(board-sync): skip plan branches` (commit `3ea68fa`): add `!startsWith(.headRefName, "autoagent/plan-")` to whatever filter the Merger uses. Needs a quick read of `autoagent-merger.yml` to confirm the exact site.

Not blocking today because the Merger runs on a slow cron and we're dispatching it manually; but it's a trap for anyone who tightens the cron or turns on scheduled execution.

## Pipeline replication: foreman ↔ foreman-todo-api drift

**Surfaced by:** 2026-04-21 board-sync fix had to be applied in BOTH repos.

The AI Factory pipeline lives in `.github/` of every repo that uses it. Bug fixes in the canonical source (foreman) need to be copy-pasted to every instance (foreman-todo-api today; future forks). No mechanism to sync.

Possible mitigations when this becomes painful:
- A `sync-pipeline.yml` cron in each instance repo that pulls `.github/` from foreman and opens a PR with the diff.
- Or: publish the pipeline as a composite action or reusable workflow hosted in foreman; instance repos only ship `.github/autoagent-config.yml` + thin wrappers. More invasive but clean.

For now, manual sync is fine.

## Upgrade `actions/checkout@v4 → @v5`

GitHub CI annotates that `actions/checkout@v4` uses Node.js 20 — deprecated June 2026. Bump to `@v5` across all workflows when convenient.
