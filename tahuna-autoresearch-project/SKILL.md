---
name: tahuna-autoresearch-project
description: User-facing Tahuna autoresearch workflow. Use when an agent is helping a Tahuna user run Karpathy-style autonomous research loops for a local ML project by reading tahuna.research.toml or program.md, safely editing allowed files, creating candidate commits, launching detached Tahuna runs, polling logs/metrics, accepting or reverting candidate commits, and recording a local experiment ledger while using Tahuna only for orchestration.
---

# Tahuna Autoresearch Project

## Goal

Run an autonomous experiment loop against the user's local Tahuna project while Tahuna remains only the remote orchestration layer. Keep planning, file edits, git commits, scoring, and accept/reject decisions local to the user's repository.

Before launching paid compute, show the research objective, budget, editable files, run command shape, selected metric, and cancellation command, then ask the user to confirm unless they already gave explicit budget approval.

## Preflight

Start from the project root:

```bash
pwd
test -f tahuna.toml && sed -n '1,220p' tahuna.toml
test -f .tahuna/environment_id && cat .tahuna/environment_id
test -f tahuna.research.toml && sed -n '1,260p' tahuna.research.toml
test -f program.md && sed -n '1,260p' program.md
git status --short
tahuna run list
```

If `tahuna.toml` or `.tahuna/environment_id` is missing, use `$tahuna-setup-hardware` first. If neither `tahuna.research.toml` nor `program.md` exists, ask the user to provide an objective and create the smallest local research spec before running paid compute.

Stop before editing if the worktree has unrelated uncommitted changes. Ask whether to commit, stash, or exclude them from the autoresearch session.

## Research Spec

Prefer `tahuna.research.toml` when present. Use `program.md` for natural-language goals, constraints, and ideas. If both exist, `tahuna.research.toml` controls machine-readable settings and `program.md` supplies the research brief.

Expected `tahuna.research.toml` shape:

```toml
[objective]
metric = "val_loss"
direction = "minimize"
aggregation = "last"

[budget]
max_trials = 10
max_parallel_runs = 1
max_failed_runs = 3

[scope]
editable = ["train.py", "configs/**/*.toml"]
readonly = ["data/**", "tahuna.toml", "pyproject.toml", "uv.lock"]

[tahuna]
run_name_prefix = "autoresearch"
gpu_type = ""
gpu_count = 1
volume_gb = 80

[git]
branch_prefix = "autoresearch"
ledger_path = ".tahuna/research/ledger.jsonl"
```

Require an objective metric, direction, trial limit, and editable scope before starting the loop. Treat missing optional fields conservatively:

- `aggregation`: use `last`
- `max_parallel_runs`: use `1`
- `run_name_prefix`: use `autoresearch`
- `branch_prefix`: use `autoresearch`
- `ledger_path`: use `.tahuna/research/ledger.jsonl`

For the MVP workflow, run trials serially. If `max_parallel_runs` is greater than `1`, explain that parallel orchestration is not part of this skill yet and proceed with one active Tahuna run at a time unless the user explicitly approves manual parallel handling.

Require training logs to emit parseable Tahuna metrics such as:

```text
val_loss=1.234
val_bpb=0.987
accuracy=0.62
```

If the training code prints `metric: value` instead of `metric=value`, update the training code only with user approval, or ask the user to make the metric parseable. Tahuna's current metric fallback expects `name=value` style log lines.

## Branch And Ledger

Create or reuse a dedicated research branch. Do not run the loop on the user's main branch.

```bash
git switch -c autoresearch/<short-objective>
mkdir -p .tahuna/research
```

If the branch already exists, switch to it only after confirming it belongs to the current autoresearch session. Keep a local JSONL ledger at the configured `ledger_path`. Append one record for the baseline and one record per trial, including at least:

- timestamp
- trial number
- run ID
- run name
- commit hash
- parent or previous best commit
- metric name
- score
- status: `baseline`, `accepted`, `rejected`, `failed`, or `cancelled`
- summary of changes

Do not store secrets in the ledger. Keep `.tahuna/research/` as local state unless the user explicitly asks to version it.

## Baseline

Run a baseline before making candidate edits. Use the configured `run_name_prefix` instead of hardcoding `autoresearch` when it is provided:

```bash
tahuna run create --name autoresearch-baseline --detached
```

Use configured compute overrides when present:

```bash
tahuna run create --name autoresearch-baseline --gpu-type "<gpu>" --gpu-count 1 --volume-gb 80 --detached
```

Poll status and logs until terminal:

```bash
tahuna run show <run_id>
tahuna run logs <run_id>
tahuna run metrics <run_id> --metric <metric> --verbose
```

If the baseline fails, inspect logs and fix setup only with the user's approval. Do not start candidate trials until there is a valid baseline score or the user explicitly allows comparing against no baseline.

## Candidate Loop

For each trial:

1. Re-read the research brief, current best score, recent failures, and allowed edit scope.
2. Propose one narrow hypothesis before editing.
3. Edit only files matched by `scope.editable`.
4. Run a fast local check when safe:

```bash
uv run python -m py_compile train.py
```

5. Verify the diff scope:

```bash
git diff --name-only
```

If any changed file is outside `scope.editable`, stop and revert that change or ask the user to expand the scope. Treat `scope.readonly` as hard blocked.

Commit the candidate before launching Tahuna:

```bash
git add <allowed-files>
git commit -m "research: trial <n> <short hypothesis>"
git rev-parse HEAD
```

Launch a detached Tahuna run with the same compute override policy used for the baseline:

```bash
tahuna run create --name autoresearch-<n> --detached
# or, when compute overrides are configured:
tahuna run create --name autoresearch-<n> --gpu-type "<gpu>" --gpu-count 1 --volume-gb 80 --detached
```

Poll until terminal:

```bash
tahuna run show <run_id>
tahuna run logs <run_id>
tahuna run metrics <run_id> --metric <metric> --verbose
```

Score with the configured aggregation:

- `last`: last reported metric value
- `min`: minimum reported value
- `max`: maximum reported value
- `mean_last_5`: mean of the last five reported values

Accept the candidate only if it improves the current best according to `direction`. Otherwise reject it.

## Accept Or Reject

Accepted trial:

- keep the candidate commit
- update the current best commit and score
- append an `accepted` ledger record
- summarize why the result improved

Rejected trial:

- append a `rejected` ledger record with the score and reason
- return the working tree to the current best
- prefer `git revert --no-edit <candidate_commit>` so history stays auditable
- use history rewriting such as `git reset --hard` only if the user explicitly authorized it for this research branch

Failed run:

- inspect logs
- append a `failed` ledger record
- count it against `max_failed_runs`
- revert the candidate commit unless the failure itself points to a useful fix and the user approves another attempt

Always leave the repository in a clear state: either at the accepted best commit or with an explicit revert commit after rejected work.

## Budget And Safety

Stop the loop when any configured limit is reached:

- `max_trials`
- `max_failed_runs`
- user-provided wall-clock or spend limit
- repeated no-capacity responses
- missing metric for two consecutive runs
- active run cancellation requested by the user

Before starting each new run, check for active work:

```bash
tahuna run list
```

If a run is stuck or no longer needed:

```bash
tahuna run cancel <run_id>
tahuna run cancel <run_id> --force
```

Use force only when graceful cancellation is stuck or the user accepts that artifacts may not be saved.

Do not change provider credentials, runtime secrets, `tahuna.toml`, `pyproject.toml`, or lockfiles unless the user explicitly expands the scope. Use `tahuna env_vars` for runtime secrets and never print secret values.

## Handoff

End with:

- best commit hash
- best run ID and score
- baseline run ID and score
- trial count, accepted count, rejected count, failed count
- ledger path
- active run IDs, if any
- exact cancel command for active runs
- remaining recommended hypotheses
