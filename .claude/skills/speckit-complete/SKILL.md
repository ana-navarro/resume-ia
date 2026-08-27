---
name: "speckit-complete"
description: "Push every repository with pending commits for the active validated task, wait for their remote CI pipelines to pass, then commit and push the task's completion."
argument-hint: "Optional notes"
compatibility: "Requires spec-kit project structure with .specify/ directory and a validated tasks/<slug>/ task"
metadata:
  author: "project-constitution"
  source: "constitution Principle V (Ciclo Speckit) - step 4"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

Consider the user input before proceeding (if not empty). This command is step 4 (final) of the
Constitution's 4-step implementation lifecycle (Principle V): `/speckit-task` → `/speckit-implement` →
`/speckit-validate` → `/speckit-complete`.

## Pre-Execution Checks

**Check for extension hooks (before completion)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_complete` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing command invocations from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to finish before continuing. Run it the same way you would run the command yourself in this agent/session (the invocation may differ from the literal `{command}` id shown above, e.g. a skills-mode agent runs it as `/skill:speckit-...` or `$speckit-...`). Emitting the block alone does not run the hook.
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

This project's working tree is a monorepo-of-repos: the top-level project directory and several
subdirectories (everything under `/services/*`, plus `/apps/resume-app` and `/resume-server`) are each
their own independent Git repository with their own GitHub remote, checked out side by side — not git
submodules. A single task's changes are typically spread across more than one of these repos. This
command MUST account for that; do not assume a single repo/single `git push`.

1. **Load the active task**: read `.specify/active-task.json`.
   - If it does not exist, or `status` is not `"validated"`: stop and tell the user this task has not
     been committed yet — run `/speckit-validate` first before `/speckit-complete`.

2. **Discover every repository with pending commits for this task**:
   - Enumerate every Git repository in the working tree (the top-level project repo, plus every nested
     one — look for `.git` directories a few levels deep, e.g. under `services/*`, `apps/*`, and
     top-level infra directories like `resume-server`).
   - For each, determine its current branch and run `git log origin/<branch>..HEAD --oneline`. Any repo
     with at least one commit ahead of its remote is in scope for this run.
   - If none are ahead, tell the user there is nothing to push and stop — nothing else in this command
     applies.

3. **Push every repo found in step 2** to `origin/<branch>` (never `--force`; if a push is rejected as
   non-fast-forward, stop and tell the user rather than force-pushing). This step is intentionally not
   gated behind a local pipeline run or a confirmation prompt — per Constitution Principle V step 4 (as
   amended), pushing task commits here is standing, pre-authorized behavior for this command. If a repo
   has a local pipeline (`Makefile` with a `validate-pipeline` target), you MAY still run it first as a
   fast local sanity check and mention the result in the report, but do not block the push on it — this
   dev environment may be missing tooling (e.g. Docker, shellcheck) that the local simulation needs, so
   it is not the authoritative gate here (that's what step 4 is for).

4. **Wait for remote CI on each pushed repo**:
   - For each repo just pushed, check if it has a workflow file that would trigger on this push (e.g.
     `.github/workflows/*.yml` with `on: push: branches: [<branch>]`). If it has none, there is nothing
     to wait for in that repo — move on.
   - Otherwise, determine `<owner>/<repo>` from `git remote get-url origin`, then poll
     `https://api.github.com/repos/<owner>/<repo>/actions/runs` (public REST API — no `gh` CLI or token
     needed for a public repo; use `curl`) for the run matching the pushed commit SHA, until its
     `status` is `"completed"`. Poll at a sensible interval (e.g. every 15–20s); if nothing shows up
     within about a minute of pushing, or a run is still not completed after roughly 10 minutes, stop
     waiting on that one, tell the user it's taking unusually long, and ask whether to keep waiting or
     proceed without it.

5. **If any polled run's `conclusion` is not `"success"`** (e.g. `failure`, `cancelled`, `timed_out`):
   - Do **not** create the final completion commit, and do not push anything further.
   - Report exactly which repo/run failed, with its `html_url` so the user can open it directly.
   - Leave `.specify/active-task.json` `"status"` as `"validated"` — the code is pushed, but not
     confirmed passing, so the task isn't complete. Tell the user to fix the failure (in that repo,
     following its own change process) and re-run `/speckit-complete` once it's pushed again.

6. **If every polled run succeeded** (or there was nothing to wait on):
   - Update `<task_directory>/task.md`'s `**Status**:` line to `Completed`.
   - Update `.specify/active-task.json`: set `"status": "completed"`.
   - Commit that status update in whichever repo tracks `tasks/<slug>/task.md` (normally the top-level
     project repo), with a standardized message like `chore(task): mark <slug> completed`.
   - Push that commit too (never `--force`).

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_complete`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_complete` key.
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue to the Completion Report.
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing command invocations from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Mandatory hook** (`optional: false`) — **You MUST emit `EXECUTE_COMMAND:` for each mandatory hook**:
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to finish before continuing. Run it the same way you would run the command yourself in this agent/session (the invocation may differ from the literal `{command}` id shown above, e.g. a skills-mode agent runs it as `/skill:speckit-...` or `$speckit-...`). Emitting the block alone does not run the hook.
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```

## Completion Report

Report:
- Every repo that was pushed, and to which remote/branch.
- Every remote CI run that was waited on, its conclusion, and a link — or that a repo had nothing to
  wait on.
- Whether the final completion commit was created and pushed, or why not.
- Final task status (`validated` if any pipeline failed or is still pending, `completed` if everything
  passed and the completion commit was pushed).

## Done When

- [ ] Every repo with pending commits for this task was identified and pushed (never `--force`)
- [ ] Every repo with a triggered workflow was actually polled via the GitHub Actions API — never a
      fabricated or assumed pass
- [ ] On any CI failure: no completion commit was created, the failing run was reported with a link,
      `.specify/active-task.json` left as `"validated"`
- [ ] On success: `.specify/active-task.json` and `task.md` updated to `"completed"` / `Completed`, that
      commit created and pushed too
- [ ] Extension hooks dispatched or skipped according to the rules in Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with the push/CI summary and final status
