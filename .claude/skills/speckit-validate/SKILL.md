---
name: "speckit-validate"
description: "Reconcile the active lightweight task's checklist and commit the implementation directly — no file-by-file review, no per-run approval prompt."
argument-hint: "Optional notes"
compatibility: "Requires spec-kit project structure with .specify/ directory and an active tasks/<slug>/ task"
metadata:
  author: "project-constitution"
  source: "constitution Principle V (Ciclo Speckit) - step 3"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

Consider the user input before proceeding (if not empty). This command is step 3 of the Constitution's
4-step implementation lifecycle (Principle V): `/speckit-task` → `/speckit-implement` →
`/speckit-validate` → `/speckit-complete`.

## Pre-Execution Checks

**Check for extension hooks (before validation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_validate` key
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
   - If it does not exist, or the referenced `<task_directory>/task.md` is missing: stop and tell the
     user "No active lightweight task found. Run `/speckit-task` first." Do not touch the
     `specs/<feature>/` pipeline — this command only operates on `tasks/<slug>/`.
   - If `status` is already `"validated"` or `"completed"`, tell the user this task was already
     validated/completed and ask whether they want to re-review it anyway before continuing.

2. **Check git availability**: run `git rev-parse --git-dir`. If it fails (not a git repository), tell
   the user this task cannot be committed without version control and ask whether to run `git init`
   before continuing. Do not commit without a repository.

3. **Show the checklist status**: read `<task_directory>/task.md` and print its Implementation
   Checklist as-is (checked/unchecked items, and any justification lines already recorded). This is a
   quick status print, not a gate — do not wait for user acknowledgement before continuing.

4. **Determine the changed files** for this task: run `git status --porcelain` and `git diff --stat`
   scoped to the Affected Service(s) listed in `task.md` (do not use a blanket `git add -A` mentality —
   only files plausibly related to this task's affected service directories count). This list is used
   internally for staging in step 6 — do not dump full diffs/content to the user file-by-file; that
   interactive walkthrough is no longer part of this command (Constitution Principle V, step 3, as
   amended).

5. **Reconcile the checklist**: for every checklist item still unchecked in `task.md` that has no
   justification note, add one directly (infer from context — what was skipped and why, e.g. "out of
   scope", "superseded by X") rather than asking the user. This command does not stop for approval.

6. **Commit directly** — no approval question, no per-file review:
   - Before staging, verify no secret-looking files (`.env`, credentials, keys) are among the changed
     files identified in step 4. If one is present, stop and tell the user instead of committing it.
   - Stage only the specific files identified in step 4 (`git add <file> <file> ...`), never a blanket
     `git add -A` / `git add .`.
   - Generate a standardized, English, Conventional-Commits-style commit message summarizing the task
     (reference the task title from `task.md`).
   - Run `git commit` (never `--no-verify`, never skip hooks).
   - Update `.specify/active-task.json`: set `"status": "validated"`.
   - Update `task.md`'s `**Status**:` line to `Validated - Committed`.

If the user reacts afterward with something to change (they may, in normal conversation, not through a
formal reject flow), treat it as a new adjustment: log it under `task.md`'s `## Adjustment Requests`
with a timestamp, leave/reset `.specify/active-task.json` `"status"` to `"implementing"`, make the fix,
then re-run this command — there is no interactive "approve y/n" loop built into this command anymore.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_validate`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_validate` key.
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
- The commit hash/message for what was just committed.
- Remaining unchecked checklist items and their justifications, if any.
- Suggested next command: `/speckit-complete`.

## Done When

- [ ] Checklist status shown, all unchecked items carry a justification
- [ ] No secret-looking file was staged or committed
- [ ] Only the task-scoped files were staged (never a blanket `git add -A`/`.`) and committed with a
      standardized English message
- [ ] `.specify/active-task.json` and `task.md` status fields are `"validated"` / `Validated - Committed`
- [ ] Extension hooks dispatched or skipped according to the rules in Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with the commit hash and next command
