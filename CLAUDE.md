# AI Factory MVP — Fork & Adapt the Three-Body Agent

## Context

I'm forking `leonardocardoso/three-body-agent` (github.com/LeonardoCardoso/three-body-agent) — an autonomous agent system that uses the Claude Code CLI in headless mode plus GitHub Actions to pick up issues and deliver PRs. It has 3 agents (Implementer, Fixer, Merger) in ~800 lines of shell/YAML, with no dependencies beyond `gh` and `jq`.

The goal is to adapt this system to my own use case, adding a few improvements identified while studying the original architecture and discussing it with the community.

## Step 1: Study the original repo

Before any changes, read ALL the code in the forked repo. Understand:
- How each workflow works (triggers, jobs, steps)
- How the prompts are structured (`.github/prompts/`)
- How state management works (GitHub Projects V2, GraphQL mutations)
- How issue selection works (priority, dedup, dependencies)
- How notifications work (Telegram)
- What auxiliary scripts exist and what each does

Document what you learned in `ARCHITECTURE.md` before moving on.

## Step 2: Adaptations to make

### 2.1 Add a Planner agent (new)
The original Three-Body Agent jumps straight to implementation. Add an earlier agent:
- Trigger: `plan` label on the issue OR manual `workflow_dispatch`
- Reads the issue plus repo context (docs, CLAUDE.md, structure)
- Produces a `change-set.md` on the branch containing: exact scope, files to change/create, API contracts where applicable, risk classification (low/medium/high)
- If **high risk** → opens a spec PR for human review, pauses the pipeline
- If **low/medium** → dispatches the Implementer automatically
- Prompt lives in `.github/prompts/planner.md`

### 2.2 Risk classification at the human gate
Instead of fixed human gates per stage, the Planner classifies each change-set:
- **High risk**: schema migrations, IAM/permission changes, public API changes, infra changes (Terraform/Docker), new external dependencies
- **Medium risk**: substantial new business logic, significant refactoring
- **Low risk**: bug fixes, tests, docs, UI tweaks

Only high risk pauses for human approval. Everything else flows through automatically.

### 2.3 Adapt configuration
- Parameterise everything project-specific from the original repo (PROJECT_ID, board columns, labels, etc.) via env vars or a config file (`.github/autoagent-config.yml`)
- Make notifications pluggable (Telegram OR Slack OR none)
- Document self-hosted runner setup (Linux, not just macOS)

### 2.4 Inner loop in the Implementer
Verify whether the original Implementer already has a self-test loop. If not, add one:
- After implementing, run the tests
- On failure, attempt to fix (up to 3 retries)
- Only open the PR when tests pass
- This reduces load on the Fixer

### 2.5 README and setup guide
- Full `README.md` covering: what it is, how it works, how to configure (secrets, runner, Projects V2), how to create the first test issue
- An example issue template

## Step 3: What NOT to change

- Keep the shell + `gh` + `jq` philosophy (no frameworks)
- Keep GitHub Actions as the orchestrator (no Step Functions / ECS)
- Cap the system at 3+1 agents (Planner + Implementer + Fixer + Merger)
- Keep prompts as separate `.md` files rendered with `envsubst`
- Do not add an external database — GitHub Projects V2 is the state store
- Respect the original directory structure as much as possible

## Deliverables

1. `ARCHITECTURE.md` — analysis of the original system
2. New workflow `autoagent-planner.yml` + prompt `planner.md`
3. Adaptations to the existing workflows to integrate with the Planner
4. `.github/autoagent-config.yml` with parameterised configuration
5. Updated `README.md` with a full setup guide
6. Sample issue template for testing the pipeline

## Reference

- Original post: leocardz.com/2026/04/08/orchestrating-agents-with-github-actions
- Prior post (agent factory): leocardz.com/2026/04/01/how-i-built-an-agent-factory-that-ships-code-while-i-sleep
