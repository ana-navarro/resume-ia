---
name: "speckit-complete"
description: "Run the local pipeline simulation (lint, tests, coverage gate) for the active validated task and push on success."
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

1. **Load the active task**: read `.specify/active-task.json`.
   - If it does not exist, or `status` is not `"validated"`: stop and tell the user this task has not
     been committed yet — run `/speckit-validate` first (and approve it) before `/speckit-complete`.

2. **Locate the pipeline entry point**: look for a `Makefile` (or equivalent script referenced by the
   project's `/run-pipeline` command) at the repository root with a `validate-pipeline` target.
   - **If none exists**: stop and tell the user plainly that no pipeline automation
     (`make validate-pipeline` or equivalent) exists yet in this repository, so this step cannot run
     for real. Do not fabricate a passing result. Suggest creating the Makefile/script (lint + pytest/
     Jest + coverage check ≥ 80%, excluding `@wip`, per Constitution Principle III) as a prerequisite,
     or ask the user how they want the pipeline invoked in this project.
   - **If it exists**: proceed.

3. **Run the pipeline** (e.g. `make validate-pipeline`), capturing full output. Parse and summarize:
   - Lint results (pass/fail, error count).
   - Unit test results (pass/fail, count).
   - Coverage percentage vs. the 80% gate (Constitution Principle III), noting `@wip`-excluded files.

4. **If the pipeline fails**:
   - Do **not** push.
   - Report the failure clearly (which stage, what errors).
   - Ask the user how to proceed: fix manually then re-run `/speckit-complete`, or go back to
     `/speckit-implement` to address it. Do not attempt silent autonomous fixes — that behavior
     belonged to the old single-command `/implement` workflow and is not part of this 4-step lifecycle.
   - Leave `.specify/active-task.json` `"status"` as `"validated"` (already committed, just not pushed).

5. **If the pipeline passes**:
   - Check for a configured git remote (`git remote -v`). If none exists, tell the user there is
     nowhere to push to and stop (do not invent a remote).
   - **Confirm with the user before pushing** — state the branch and remote you are about to push to,
     and wait for explicit confirmation. Pushing is a shared-state action and must not happen silently,
     regardless of pipeline success.
   - On confirmation, run `git push` to the current branch's upstream (never `--force`).
   - Update `.specify/active-task.json`: set `"status": "completed"`.
   - Update `<task_directory>/task.md`'s `**Status**:` line to `Completed`.

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
- Pipeline result summary (lint, tests, coverage %).
- Whether a push happened (and to which remote/branch), or why it did not.
- Final task status (`validated` if push pending/failed, `completed` if pushed).

## Done When

- [ ] Pipeline was actually executed (or the user was clearly told no pipeline exists yet — never a
      fabricated pass)
- [ ] On failure: no push occurred, failure reported, user given a clear path forward
- [ ] On success: user explicitly confirmed before any `git push`, then push executed without `--force`
- [ ] `.specify/active-task.json` and `task.md` status updated to reflect the real outcome
- [ ] Extension hooks dispatched or skipped according to the rules in Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with pipeline results and push outcome
