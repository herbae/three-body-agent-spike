# Three-Body Agent — Architecture Analysis

> Analysis of the forked `leonardocardoso/three-body-agent` pipeline as it ships in this repo, produced before adapting it. All file:line citations refer to the current HEAD on `main`.

## 1. Overview

Three-Body Agent is an **autonomous issue-to-PR-to-merged-code pipeline** built entirely on GitHub Actions + the Claude Code CLI (headless). It has **no external runtime dependencies** — state lives on a GitHub Projects V2 board, milestones, labels, and branch names; orchestration is shell + `gh` + `jq` + `envsubst`.

Three agents do the work, each invoked by a dedicated workflow that shells out to `claude -p` with a prompt rendered from `.github/prompts/*.md`:

| Agent | Workflow | Model | Role |
|---|---|---|---|
| **Implementer** | `autoagent-implementer.yml` | `claude-opus-4-6` | Picks a `Todo` issue, implements, opens a PR |
| **Fixer** | `autoagent-fixer.yml` | `claude-sonnet-4-6` | Fixes CI failures / review comments / merge conflicts |
| **Merger** | `autoagent-merger.yml` | `claude-sonnet-4-6` (single-turn gate) | Merges PRs ready to ship |

Two support workflows keep the board and calendar consistent:

| Workflow | Purpose |
|---|---|
| `autoagent-board-sync.yml` | Moves the issue between Projects V2 columns on PR events |
| `auto-week-rollover.yml` | Creates this week's milestone/iteration, rolls unfinished work forward |

Plus one reusable worker:

| Workflow | Purpose |
|---|---|
| `telegram.yml` | Reusable `workflow_call` — sends a single Telegram message via the Bot API |

**There is no Planner agent today.** The Implementer jumps straight from a Todo issue to code — no design document, no risk classification, no human gate.

---

## 2. The state machine

All state is expressed as **Projects V2 `Status` column transitions** driven by branch name conventions and PR events.

```
                       +-----------+
       (issue created) |   Todo    |<-------------------------+
                       +-----+-----+                          |
                             |                                |
           Implementer picks | (priority + milestone filter)  | PR closed
           top issue, moves  v                                | without merge
                       +-----------+                          |
                       |In Progress|                          |
                       +-----+-----+                          |
                             |                                |
                Claude pushes| branch `autoagent/<num>-<slug>`|
                and opens PR v                                |
                       +-----------+                          |
                       |Ready For QA|-------------------------+
                       +-----+-----+
                             |
         All checks pass,    |  (Fixer loops here on failures,
         no CHANGES_REQUESTED|   no board transitions during fixes)
         & Claude gate says  v
         MERGE               |
                       +-----------+
                       |   Done    |
                       +-----------+
```

- **Todo → In Progress**: `autoagent-implementer.yml` (prepare job) when it picks the issue.
- **In Progress → Ready For QA**: `autoagent-board-sync.yml` on `pull_request.opened`, keyed off branch prefix `autoagent/<issue_num>-`.
- **Ready For QA → Done**: `autoagent-board-sync.yml` on `pull_request.closed && merged`.
- **→ Todo (back)**: `autoagent-board-sync.yml` on `pull_request.closed && !merged`.

**Column names are case-sensitive string matches** in GraphQL responses — `Todo`, `In Progress`, `Ready For QA`, `Done`.

---

## 3. Workflows in detail

### 3.0 `autoagent-planner.yml` (286 lines)

**Triggers**
- `issues` on `labeled` — fires for every label event; a gate step compares the label name to `$AUTOAGENT_LABEL_PLAN` (from `.github/autoagent-config.yml`) and writes `skip=true` to `GITHUB_OUTPUT` if they do not match, short-circuiting all subsequent steps.
- `workflow_dispatch` with inputs `issue_number` (required) and `base_branch` (default `main`).

**Concurrency**: `autoagent-planner-<issue_num>`, `cancel-in-progress: false`. The key mixes both event shapes (`github.event.issue.number` for label events, `inputs.issue_number` for dispatches).

**Permissions**: `contents: write`, `issues: write`, `pull-requests: write` — all three are needed because the high-risk path commits a file and opens a PR, while the low/medium path posts an issue comment.

**Jobs**
1. `plan` (30 min, single job) — resolves inputs, installs Claude Code, runs the Planner prompt, parses the risk tier, labels the issue, and routes to one of two downstream actions.
2. `notify` — calls `notify.yml` (reusable) after `plan` completes; fires even on failure if `issue_num` output is non-empty.

**`plan` step-by-step**
1. **Checkout** (`AGENT_PAT`, `fetch-depth: 0`) and **load config** via `.github/actions/autoagent-setup` (exports `AUTOAGENT_*` env vars for the rest of the job).
2. **Gate**: compare `github.event.label.name` to `$AUTOAGENT_LABEL_PLAN`; set `skip=true` if mismatch.
3. **Resolve inputs**: for `workflow_dispatch`, read `inputs.issue_number` + `inputs.base_branch`; for label events, read `github.event.issue.number` and default base to `main`. Fetch `title` and `body` from `gh issue view`.
4. **Install Claude Code**: `npm install -g @anthropic-ai/claude-code`.
5. **Run Planner**: render `.github/prompts/planner.md` with `envsubst '$PLAN_LABEL $ISSUE_NUM $ISSUE_TITLE $ISSUE_BODY $BASE'`, then invoke `claude -p --model $AUTOAGENT_PLANNER_MODEL --max-turns $AUTOAGENT_PLANNER_MAX_TURNS --permission-mode auto --output-format text`. Assert `/tmp/change-set.md` is non-empty; parse `risk` with `yq --front-matter=extract '.risk'`; validate it is one of `low | medium | high`.
6. **Apply risk label**: `gh issue edit --add-label $AUTOAGENT_LABEL_{LOW|MEDIUM|HIGH}_RISK` and remove the plan label.
7. **Route by risk**:
   - **low / medium** — post `/tmp/change-set.md` as an issue comment prefixed with the sentinel `<!-- AUTOAGENT_CHANGE_SET -->`, then `gh workflow run autoagent-implementer.yml -f issue_number=… -f base_branch=… -f model=$AUTOAGENT_IMPLEMENTER_MODEL`.
   - **high** — commit the change-set to `docs/change-sets/<N>.md` on a new branch `<BRANCH_PREFIX>plan-<N>-<slug>` branched off `origin/$BASE`, then open a spec PR with instructions for the human reviewer. The Implementer is **not** dispatched — the pipeline pauses until a human manually triggers it.

### 3.1 `autoagent-implementer.yml` (436 lines)

**Triggers**
- `schedule` (commented out — must be enabled by the operator): hourly `0 * * * *`.
- `workflow_dispatch` with inputs `issue_number`, `base_branch` (default `main`), `model` (`claude-opus-4-6` | `claude-sonnet-4-6`).

**Concurrency**: `autoagent-implementer`, `cancel-in-progress: false` (queues).

**Jobs**
1. `prepare` (5m, `contents/issues/actions: read`) — resolves the issue, outputs `{skip, issue_number, issue_title, base_branch, model, start_message}`.
2. `notify-start` — calls `telegram.yml` if `!skip`.
3. `implement` (90m, `contents/pull-requests/issues: write`) — runs Claude, outputs `after_message`.
4. `notify-done` (`always()`) — calls `telegram.yml` if message non-empty.

**`prepare` logic**
- **Manual dispatch path** (lines 63-76): reads inputs straight through; `model` is propagated from input.
- **Scheduled path**:
  1. Compute current milestone as `YY CW WW` (e.g. `26 CW 14`) via `date -u +%y` / `date -u +%V` (with `10#` to strip the leading zero, line 81).
  2. Paginate GraphQL `organization.projectV2.items(first:100, after:$cursor)` until `!pageInfo.hasNextPage` (lines 85-150) — fixed in commit `97830b2` for boards >100 items.
  3. Filter for `Status == "Todo"`, issue `state == OPEN`, `milestone.title == CURRENT_MILESTONE`.
  4. Sort by priority label (`p0`=0 … `p5`=5, otherwise 6), then by body length descending (longer == better-specified).
  5. Safety gates:
     - Skip if another implementer run is `in_progress` (`gh run list --workflow=autoagent-implementer.yml --status in_progress`, line ~165).
     - Skip if an open PR already exists for the branch `autoagent/<num>` (`gh pr list --search "head:autoagent/$N"`).
  6. **Dependency detection** (lines 207-217): regex `Depends on:*\s*#*\([0-9]\+\)` in the issue body. If found, re-base off the dependency PR's head branch rather than `main`.
  7. **Move to In Progress**: two GraphQL queries (project id, status field + option IDs) followed by `updateProjectV2ItemFieldValue` mutation.
  8. **Emit outputs** (lines 272-276). `model` for scheduled runs is hardcoded to `claude-opus-4-6` at line 275 — the manual dispatch `model` input is not used on the scheduled path (see §9.1).

**`implement` logic**
- Checkout `base_branch` with `fetch-depth: 0`, using `AGENT_PAT` so pushes can trigger downstream workflows.
- `npm install -g @anthropic-ai/claude-code` (line ~322) — fresh install per run.
- `gh issue view $N --json title,body,labels > /tmp/issue.json`.
- Render prompt: `git show origin/main:.github/prompts/implementer.md | envsubst '$ISSUE_NUM $ISSUE_TITLE $ISSUE_BODY $BASE'`. Note it reads the prompt from `origin/main`, not the working tree — edits to prompts must be on `main` to take effect.
- Invoke Claude: `claude -p --verbose --output-format stream-json --model "$MODEL" --max-turns 500 --permission-mode auto`.
- Post success/failure comment on the issue with the PR URL or run URL.
- Build `after_message` for Telegram (files changed, commit list).

**What Claude is expected to do** (see `.github/prompts/implementer.md`): create `autoagent/<num>-<slug>` branch, implement, add/update tests, commit, `git push -u origin`, `gh pr create --base $BASE` with `Closes #$N`. The workflow does **not** independently verify tests; it trusts the prompt + Claude's 500-turn budget.

### 3.2 `autoagent-fixer.yml` (398 lines)

**Triggers**
- `schedule` (commented out): every 30 min at `:15,:45`.
- `check_suite` on `completed` failures for `autoagent/*` branches (commented out).
- `workflow_dispatch` with `pr_number`, `fix_type` (`all|ci|review|conflict`), `model` (sonnet | opus).

**Jobs**
1. `prepare` — decide which PR to fix.
2. `notify-start`.
3. `fix` (60m) — gather context, run Fixer prompt, force a push if Claude made no commits.
4. `notify-done`.

**PR-selection logic** (auto-scan path)
- **Dedup**: query `gh api /repos/.../actions/workflows/autoagent-fixer.yml/runs?status=in_progress` + `queued`, collect `.inputs.pr_number` from each run to skip PRs already being worked on.
- For each open `autoagent/*` PR (newest first), check:
  - `gh pr checks --json state` — any failing/cancelled check?
  - `gh api /repos/.../pulls/$N/reviews` — any `CHANGES_REQUESTED`?
  - `gh pr view --json mergeable` — `CONFLICTING`?
- Pick the first PR that needs a fix and isn't already being fixed.
- Guard: branch must start with `autoagent/`.

**`fix` logic**
- Checkout the PR branch.
- Build `/tmp/failure-context.md` containing:
  - `gh pr checks $N` output.
  - Failed run logs: `gh run view --log-failed | tail -300`, with `::error::` markers stripped so they don't break the step.
  - Inline review comments (`/pulls/$N/comments`), formatted `### FILE:LINE\nBODY`.
  - Review verdicts (`/pulls/$N/reviews`) filtered to `CHANGES_REQUESTED`.
  - Mergeable status.
- Snapshot `pre_sha=$(git rev-parse HEAD)`.
- Render `.github/prompts/fixer.md` with `envsubst '$PR_NUM $BRANCH $BASE $CONTEXT'` and run Claude (same CLI flags as Implementer; model is whatever `prepare` decided).
- **CI re-trigger fallback** (lines ~273-303): if `post_sha == pre_sha` (Claude made no commits), the workflow tries `git merge origin/$BASE` and pushes. If still no change, it creates an empty commit `chore: re-trigger CI` and pushes.
- **Re-request reviewers**: iterate reviewers from `/pulls/$N/reviews` and call `POST /pulls/$N/requested_reviewers` for each.
- Post a PR comment summarising the fix run.

### 3.3 `autoagent-merger.yml` (308 lines)

**Triggers**
- `schedule` (commented out): every 2h during waking hours.
- `workflow_dispatch`.

**Concurrency**: `autoagent-merger`, `cancel-in-progress: false`.

**Jobs**
1. `prepare` — scan for merge-ready PRs, output base64-encoded `pr_list` (base64 because GH output can't preserve newlines cleanly).
2. `notify-start`.
3. `merge` (15m) — merge sequentially.
4. `notify-done`.

**Merge-ready filter** (prepare job)
- For each open `autoagent/*` PR:
  - Skip if `mergeable == CONFLICTING`.
  - Skip if any `gh pr checks` row is not `SUCCESS|SKIPPED`; skip if no checks reported yet.
  - Skip if any review state is `CHANGES_REQUESTED`.
  - **Fast keyword filter**: skip if any review comment body matches `\bCRITICAL\b|\bMEDIUM\b`. Only these two severities block the merge — `LOW`/informational do not.

**Merge loop** (merge job)
- Sparse-checkout limited to `.github/prompts/` (only the merger prompt is needed — no code checkout required).
- For each candidate PR:
  - Re-verify all three gates (conflicts, checks, reviews) in case upstream state changed while earlier PRs were being merged.
  - If the PR has any review comments or verdicts: fetch the last 10 commits and invoke Claude (`--model claude-sonnet-4-6 --max-turns 1 --output-format text`) with `merger.md`. Expected output: `MERGE` or `SKIP` on the first line, reason on the second. **If the CLI fails or times out, the workflow defaults to `MERGE`** (fail-open).
  - `gh pr merge $N --merge` (creates a merge commit — not squash, not rebase).
  - `sleep 10` between merges to give GitHub time to recompute merge base for the next PR.
- Emit per-PR result (`merged|#N|TITLE` or `skip|#N|TITLE|reason`) into `/tmp/merger-results.txt`, then summarise to Telegram + job summary.

### 3.4 `autoagent-board-sync.yml` (190 lines)

**Triggers**
- `pull_request` on `[opened, closed]` filtered to `main` (commented out — needs the operator to enable).
- `workflow_dispatch`.

Single `sync` job. Extracts the issue number from the PR branch name via regex `autoagent/(\d+)-` (line ~32), then:
- On `opened`: status → `Ready For QA`.
- On `closed && merged`: status → `Done`.
- On `closed && !merged`: status → `Todo`.

Each transition is an `updateProjectV2ItemFieldValue` mutation, preceded by a GraphQL query for `{projectId, statusFieldId, itemId, optionIds}`. **No cursor pagination here** — the query fetches `items(first: 100)` only, so boards with >100 items may miss the issue (see §9.4).

Also sends Telegram messages directly via `curl` (not through `telegram.yml`) — both `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` are checked and skipped silently if absent.

### 3.5 `auto-week-rollover.yml` (331 lines)

**Triggers**
- `schedule` (commented out): `0 6 * * 1` (Monday 06:00 UTC).
- `workflow_dispatch` with `mode` input: `milestones` (default) or `iterations`.

**Milestones mode**
1. Compute this/last week with `YY CW WW` format.
2. Due date = next Sunday; computed via `date -d` (Linux) with a `date -j` (macOS) fallback at lines ~75-76.
3. Upsert the milestone (`GET /repos/$REPO/milestones` → `POST` or `PATCH`).
4. Re-assign all open issues from last week's milestone to this week's.
5. Close last week's milestone if empty.

**Iterations mode**
1. Query the project's iteration field (`Calendar Week`), extract configured iterations sorted by `startDate`.
2. Compute next iteration start date, CW number, title.
3. Create next iteration via `updateProjectV2IterationFieldConfiguration` mutation (appending to the `iterations` array).
4. Move every non-Done OPEN item in the current CW to the next CW via `updateProjectV2ItemFieldValue` with `{iterationId: ...}`.

### 3.6 `telegram.yml` (25 lines)

Reusable workflow; single step that POSTs to `https://api.telegram.org/bot$TOKEN/sendMessage` with `chat_id` + `text`. Requires both secrets or the job fails — there is **no clean "notifications off" switch** today (see §9.9).

---

## 4. Prompts

### 4.0 `.github/prompts/planner.md` (82 lines)

- **Placeholders**: `${PLAN_LABEL}`, `${ISSUE_NUM}`, `${ISSUE_TITLE}`, `${ISSUE_BODY}`, `${BASE}`.
- **Role**: the Planner is explicitly told it is *not* a code-writing agent. Its only output is `/tmp/change-set.md` — a YAML-frontmatter document specifying scope, files to touch, API contract changes, a test plan, risk justification, and open questions. It must not create branches, commit, or open PRs.
- **Change-set format**: frontmatter keys `risk:` (one of `low | medium | high`), `issue:` (integer), `base:` (branch name), followed by freeform markdown sections. The frontmatter is machine-parsed by the workflow with `yq`.
- **Risk rubric** (from high to low):
  - **high**: database schema / migrations; auth, IAM, or secrets; public API surface; infrastructure-as-code (Terraform, Docker, Kubernetes, CI pipeline core control flow); new runtime dependencies.  *Carve-out*: editing prompts under `.github/prompts/` or non-critical helper workflows is `medium`, not `high`.
  - **medium**: substantial new business logic; refactor touching more than 3 call-sites; new first-class module.
  - **low**: bug fixes, docs, tests, copy changes, UI tweaks, internal renames, config values.
  - Tie-break rule: low vs. medium → pick `medium`; medium vs. high → pick `high`.
- **Guardrails**: must read `CLAUDE.md` and `ARCHITECTURE.md` before classifying; must not invent requirements not stated in the issue (file them under "Open questions" instead); frontmatter keys must be exact.

### 4.1 `.github/prompts/implementer.md` (62 lines)

- **Placeholders**: `${ISSUE_NUM}`, `${ISSUE_TITLE}`, `${ISSUE_BODY}`, `${BASE}`.
- **Contract**: create `autoagent/${ISSUE_NUM}-<slug>`, implement, add/update tests, commit with clear messages, `git push -u origin`, `gh pr create --base ${BASE}` with Conventional-Commits title (<70 chars) and `Closes #${ISSUE_NUM}` in the body.
- **Guardrails**: no unrelated refactors, no dep updates, no force-push, no merging the PR, no hypothetical abstractions, validate inputs at boundaries, no hardcoded secrets.
- **Inner loop**: the prompt tells Claude to run tests and fix failures *before* opening the PR, but doesn't specify a retry budget — the only hard limit is `--max-turns 500`. The workflow does not independently verify test status.

### 4.2 `.github/prompts/fixer.md` (41 lines)

- **Placeholders**: `${PR_NUM}`, `${BRANCH}`, `${BASE}`, `${CONTEXT}` (the failure-context markdown assembled by the workflow).
- **Contract**: analyse failures; for conflicts use `git fetch $BASE && git rebase origin/$BASE && <resolve> && git push --force-with-lease`; for CI failures fix root cause; for reviews, address only `CRITICAL` and `MEDIUM` issues (explicitly *not* `LOW`); separate commits per logical fix with descriptive messages.
- **Guardrails**: no unrelated refactors; `--force-with-lease` permitted only during rebase.

### 4.3 `.github/prompts/merger.md` (20 lines)

- **Placeholders**: `${PR_NUM}`, `${REVIEW_COMMENTS}`, `${REVIEW_VERDICTS}`, `${RECENT_COMMITS}`.
- **Contract**: exactly two lines of output — line 1 is `MERGE` or `SKIP`, line 2 is a one-sentence reason.
- **Decision rule**: `SKIP` if any `CRITICAL`/`MEDIUM` issue in the review comments is not visibly addressed in the recent commits; otherwise `MERGE`.
- Invoked with `--max-turns 1 --output-format text`.

---

## 5. Issue selection, dedup, and dependency handling

**Selection (Implementer)** — priority label first, then body length. There is no randomisation and no aging, so a lonely `p5` ticket with a short body can starve indefinitely behind better-specified `p4`s.

**Dedup (all agents)**
- Implementer: skip if another implementer run is `in_progress`; skip if a PR already exists for the branch.
- Fixer: query `in_progress|queued` fixer runs and extract `inputs.pr_number` to build a busy set.
- Merger: relies on concurrency group + re-verification before each merge — no dedup across runs.

**Dependencies**: issue body line `Depends on: #123` reroutes the branch base from `main` to the open PR for #123. Only the first match is honoured; the dependency PR must be open (a merged dep falls back to `main`). No transitive resolution.

---

## 6. Secrets and environment variables

**Required secrets**
| Secret | Used by | Scope notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | Implementer, Fixer, Merger | Claude Code CLI |
| `AGENT_PAT` | All agent workflows + board sync + week rollover | Fine-grained PAT with `repo`, `project`, and `workflow` permissions. Required because the default `GITHUB_TOKEN` can't access org-level Projects V2. |
| `TELEGRAM_BOT_TOKEN` | `telegram.yml`, board-sync | No good way to opt out today. |
| `TELEGRAM_CHAT_ID` | Same | Same. |

**Environment variables at the workflow level** — all hardcoded, duplicated across files:

| File | Line | Variable | Default | Purpose |
|---|---|---|---|---|
| autoagent-implementer.yml | 32 | `PROJECT_ORG` | `your-org` | GraphQL `organization(login: …)` |
| autoagent-implementer.yml | 33 | `REPO` | `your-org/your-repo` | Referenced in a handful of `gh api` calls |
| autoagent-implementer.yml | 35 | `PROJECT_NUMBER` | `1` | Board number |
| autoagent-board-sync.yml | 11 | `PROJECT_ORG` | `your-org` | |
| autoagent-board-sync.yml | 13 | `PROJECT_NUMBER` | `1` | |
| auto-week-rollover.yml | 19 | `REPO` | `your-org/your-repo` | |
| auto-week-rollover.yml | 20 | `PROJECT_ORG` | `your-org` | |
| auto-week-rollover.yml | 22 | `PROJECT_NUMBER` | `1` | |
| auto-week-rollover.yml | 24 | `ITERATION_FIELD` | `Calendar Week` | iterations mode only |
| auto-week-rollover.yml | 25 | `DONE_STATUS` | `Done` | iterations mode only |

---

## 7. Hardcoded values worth parameterising

Beyond the env-vars table above, these appear inline in workflows/prompts:

| Concern | File:line | Today | Should be configurable because |
|---|---|---|---|
| Milestone format | `autoagent-implementer.yml:82`, `auto-week-rollover.yml:~68` | `YY CW WW` | Teams without a week-based cadence won't use milestones at all |
| Status column names | `autoagent-implementer.yml:~108-110`, `autoagent-board-sync.yml:~75-76` | `Todo / In Progress / Ready For QA / Done` | Case-sensitive string matches, very common to customise |
| Branch prefix | `autoagent-board-sync.yml:~32`, `autoagent-implementer.yml` safety checks, `implementer.md` | `autoagent/` | Teams may want separate prefixes per agent team |
| Priority labels | `autoagent-implementer.yml` sort logic | `p0`…`p5` | Other projects use `P0`/`P1`, `prio/high`, etc. |
| Implementer model | `autoagent-implementer.yml:275` (scheduled path) | `claude-opus-4-6` hardcoded | Only manual dispatch respects the input |
| Fixer model | `autoagent-fixer.yml:~68` | `claude-sonnet-4-6` | Cost knob |
| Merger model | `autoagent-merger.yml:~206` | `claude-sonnet-4-6` | Cost knob |
| Merge method | `autoagent-merger.yml:~222` | `--merge` | Many teams prefer `--squash` |
| `max-turns` | implementer, fixer | `500` | Huge cost amplifier |
| Timeouts | implementer (90m), fixer (60m), merger (15m) | | Too low for large repos, too high for tiny ones |
| Install command | `autoagent-implementer.yml:~322`, fixer | `npm install -g @anthropic-ai/claude-code` | Runners may already have it |
| Severity keywords for merger gate | `autoagent-merger.yml:81`, `merger.md`, `fixer.md` | `CRITICAL`, `MEDIUM` | Team conventions vary |
| Pause between merges | `autoagent-merger.yml:~231` | `sleep 10` | |

---

## 8. Runner assumptions

- `runs-on: ubuntu-latest` throughout. The one macOS-sensitive line (`date -j` fallback in `auto-week-rollover.yml:~75`) means the workflows will still run correctly on a macOS self-hosted runner, but Linux is the default path.
- Runner must have `node` + `npm` available for the global `claude-code` install.
- Runner must have `gh`, `jq`, `envsubst`, `git`, `curl`. All standard on `ubuntu-latest` images.
- No Docker, no external DB, no cache dependency. Each run is stateless beyond what the repo/board/Telegram hold.

---

## 9. Gaps, rough edges, and outright bugs

1. **Scheduled-path model is not configurable.** `autoagent-implementer.yml:275` emits a literal `model=claude-opus-4-6` regardless of operator preference. Manual dispatch respects the input; scheduled runs don't. Low-risk to fix (read from a workflow-level env var or config file).
2. **Prompt is read from `origin/main`**, not the working tree (implementer + fixer). PR-based edits to prompts won't take effect until merged — easy gotcha when iterating.
3. **No independent test verification.** The Implementer trusts Claude's prompt to run tests. If Claude says "done" without running them, we only find out when CI fails downstream and the Fixer wakes up.
4. **Board-sync pagination**: `autoagent-board-sync.yml` fetches `items(first: 100)` only — boards with >100 items silently break on the sync path. Implementer pagination was fixed in `97830b2`; board-sync was missed.
5. **GraphQL option IDs are fetched every run.** Project ID / field ID / option IDs are stable but queried repeatedly. Cheap optimisation: cache in Actions variables.
6. **Merger fail-opens on Claude errors** — API outage, timeout, or empty output ⇒ `MERGE`. Arguably correct to avoid stalls, but undocumented and scary.
7. **Merger only gates on reviews.** If a PR has zero review comments/verdicts, the keyword filter never fires and Claude is never asked — the PR merges as long as checks pass. Fine for low-risk changes, risky for high-risk ones (part of the motivation for the Planner agent).
8. **No branch-name validation.** If Claude creates `feat/foo` instead of `autoagent/123-foo`, the board-sync regex misses it and the issue never moves. Silent failure.
9. **Telegram is not optional.** `telegram.yml` requires both secrets; if either is missing, the reusable job fails hard, cascading into agent failures. Any "notifications plugin" redesign has to address this.
10. **Dependency regex is shallow.** `Depends on: #N` only handles the first match; no transitive chains; merged deps silently fall back to `main`. Fine for small graphs, surprising for anything larger.
11. **Priority vs. body-length ordering can starve issues.** There's no aging/fairness term.

---

## 10. Dataflow at a glance

```
                 Issue in "Todo" (p0..p5, milestone = this week)
                                │
                                │ hourly cron / manual dispatch
                                ▼
   ┌────────────────────────────────────────────────────────────┐
   │ IMPLEMENTER                                                │
   │  1. scan board, rank, dedup, resolve `Depends on:`         │
   │  2. mutate Status → "In Progress"                          │
   │  3. checkout base; `npm i -g @anthropic-ai/claude-code`    │
   │  4. envsubst implementer.md | claude -p --max-turns 500   │
   │  5. Claude creates autoagent/<N>-<slug>, tests, pushes, PR │
   └────────────────────────┬───────────────────────────────────┘
                            │  PR opened on main
                            ▼
                  ┌──────────────────┐
                  │ BOARD-SYNC       │  Status → "Ready For QA"
                  └────────┬─────────┘
                           │
            ┌──────────────┴──────────────┐
            │                             │
   CI failure / review /           all checks green,
   conflict detected               no CHANGES_REQUESTED
            │                             │
            ▼                             ▼
   ┌────────────────┐         ┌──────────────────────┐
   │ FIXER (30 min) │         │ MERGER (every 2h)    │
   │  - dedup       │         │  - mergeable?        │
   │  - context     │         │  - checks pass?      │
   │  - claude      │         │  - no CHANGES_REQ?   │
   │  - push/retry  │         │  - no CRITICAL/MED?  │
   └────────┬───────┘         │  - claude gate       │
            │                 │  - gh pr merge       │
            │ pushes commits  └──────────┬───────────┘
            └────► back to CI ──────────►│
                                         │ PR merged
                                         ▼
                               ┌──────────────────┐
                               │ BOARD-SYNC       │  Status → "Done"
                               └──────────────────┘

   Weekly (Mon 06:00 UTC)
   ┌────────────────────────────────────────────────────┐
   │ WEEK ROLLOVER (milestones OR iterations)            │
   │  - upsert this week's milestone/iteration           │
   │  - move last week's unfinished → this week          │
   └────────────────────────────────────────────────────┘
```

---

## 11. What's missing for the AI Factory MVP

Mapped against `CLAUDE.md`:

- **2.1 Planner agent** — no equivalent today. Will need a new `autoagent-planner.yml` + `.github/prompts/planner.md`, triggered by a `plan` label or `workflow_dispatch`, producing a `change-set.md` and (for high risk) a spec PR that pauses the pipeline.
- **2.2 Risk gate** — no classification exists today. The Implementer runs unconditionally. The gate needs to live between Planner output and Implementer trigger.
- **2.3 Parameterisation** — duplicated env vars across 4 workflows; no config file. Notifications are hardwired to Telegram and can't be cleanly disabled.
- **2.4 Inner loop in Implementer** — the prompt *asks* for tests but the workflow doesn't verify. Need either an explicit retry budget in the prompt or a post-Claude CI-check step before PR creation (the latter is heavier — start with the prompt change).
- **2.5 README / setup** — the shipped README is detailed for the original project but won't match post-parameterisation. Issue template is not present.

These five items form the scope of the adaptation plan.
