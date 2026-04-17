# Deferred Questions

This skill uses a per-job `PreToolUse` hook to intercept `AskUserQuestion` and `ExitPlanMode` during `claude -p` runs.

## Why This Exists

Non-interactive Claude Code runs are good for automation, but some tools require user input. Claude Code hooks can inspect tool calls and return a permission decision before the tool runs. For `AskUserQuestion` and `ExitPlanMode`, the hook can return `permissionDecision: "allow"` together with `updatedInput` so the tool can proceed with externally supplied input. Hooks receive JSON on stdin, and settings files can register command hooks for `PreToolUse`. The built-in `AskUserQuestion` tool is the standard way Claude asks clarifying multiple-choice questions.

## Repository Flow

1. `launch` or `resume` generates `settings.json` for the job, passes it to Claude with `--settings`, and feeds any prompt or follow-up message over stdin rather than embedding it in the command line.
2. If Claude hits `AskUserQuestion` or `ExitPlanMode`, the hook stores the tool payload in `pending-tool.json`.
3. If an operator answer already exists in `pending-answer.json`, the hook returns `permissionDecision: "allow"` with the stored `updatedInput`.
4. Otherwise, the hook returns `permissionDecision: "defer"` so the run can stop cleanly while preserving the session for a later `resume`.
5. Codex inspects `pending-tool.json`, prepares the correct `updatedInput` payload, writes it with `answer`, and resumes the same job.

## Important Limits

- The exact `updatedInput` shape depends on the tool payload. Inspect `pending-tool.json` before answering.
- `AskUserQuestion` usually expects a structured selection payload, not a free-form note.
- The live deferred flow depends on Claude Code support for deferred tool handling in `-p` mode.
- In this repository, the flow is tested with deterministic fixtures because the current workspace cannot complete real Claude API calls.

## Safe Usage Pattern

1. Run `status JOB_ID --tail 30 --tail-both` and inspect `pending-tool.json`.
2. Build the `updatedInput` object that matches the captured tool payload.
3. Use `answer JOB_ID --updated-input-json ... --resume-now`.
4. Run `status JOB_ID --tail 30` to confirm the resumed attempt progressed.
