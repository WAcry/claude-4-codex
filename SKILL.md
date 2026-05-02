---
name: claude-for-codex
description: Delegate subtasks to a local Claude Code instance. Suited for front-end UI implementation, documentation writing, rewriting investigation conclusions into human-readable formats, demo prototypes, and other tasks requiring aesthetics or human readability. Triggered when coordinating Claude Code to complete limited-scope work. This skill is for use only by agents powered by OpenAI GPT models.
---

# Claude Code Orchestrator

## Delegation Model

GPT and Claude have complementary strengths. GPT reasons much more accurately; Claude is less reliable and less intelligent. However, GPT's training favors precision and brevity—its output lacks design sense and reads as expert notes, not step-by-step pedagogical explanations. Claude's training favors patient, accessible teaching and polished visual design. These are model-level tendencies, not prompting problems—neither can replicate the other's strength by trying harder. GPT provides accuracy, Claude provides readability. Therefore GPT must not rephrase Claude's output.

### Tasks Suitable for Delegating to Claude Code

| Task Type | Claude's Strengths | Your Follow-up Action |
|-----------|-------------------|----------------------|
| Front-end UI implementation | Naturally good aesthetics, generates beautiful interfaces | Review and fix code bugs |
| Simple documentation (based on well-scoped code, PRs) | Clear, self-contained, human-friendly | Review and fix factual accuracy and logic |
| Complex document rewriting / polishing | Rewrites concise docs into thorough explanations with good formatting | Review and fix factual accuracy and logic |
| Implementation demos / prototypes | Quickly produces visually appealing prototypes | Review and fix factual accuracy and logic |

### Tasks Requiring GPT First, Claude Code Polish After

| Task Type | Why GPT Goes First | Workflow |
|-----------|-------------------|----------|
| System design documents | Requires broad search and deep reasoning, not limited to a single diff | GPT writes first draft → Claude Code polishes formatting → GPT reviews |
| Investigation analysis, answering user questions | Scope is unclear, requires deep codebase investigation, synthesizing information from multiple sources, requires high reasoning ability | GPT investigates and writes conclusions → Claude Code thoroughly restructures into a readable document at length → GPT reviews |
| Task implementation plans | Scope is unclear, requires deep investigation of feasible approaches, evaluating trade-offs, original design, requires high reasoning ability | GPT writes plan → Claude Code improves descriptions, adds readability, adds necessary examples → GPT reviews |

### Tasks Not Suitable for Independent Completion

- Complex bug investigation or deep investigation/reasoning—Claude often reaches wrong conclusions
- Writing bug-free code—Claude's code frequently has bugs
- Writing original design documents or task plans that require deep thinking—Claude's plans are usually not the best approach and may even contain errors

### Conclusion Rewriting Workflow

When a user question requires deep investigation and the answer needs to be human-readable:

1. **You investigate first**, reaching an accurate conclusion (your conclusion is accurate but concise; the user may have difficulty understanding it)
2. **Delegate to Claude Code**: hand it your conclusion + relevant files and context paths, asking it to rewrite your conclusion into a human-readable explanatory document, generate a markdown document, and provide the path.
3. **You review the document**: only check for factual deviations, make factual corrections, do not change style or length, do not compress, supplement, or restructure.
4. **Deliver**: provide the file path to the user with a brief summary.

## Common Paths for Documents, Artifacts, and Prompts

Unless otherwise specified, store them in `state_root`. Do not place them in `workdir` to avoid polluting the Git environment.

## Output Handling Rules

Claude's output is more readable for users. Your responsibility is to review accuracy, not rewrite style.

| Rule | Description |
|------|-------------|
| Fix only actual errors | Only correct factual errors and obvious bugs; do not modify wording or style |
| Do not add or remove content | Do not add to, remove from, or compress Claude's output |
| Do not paraphrase | Do not redescribe Claude's output in your own words |
| Use files for long content | Have Claude output to a markdown file; after review, provide the file path to the user |

**Self-check signal**: if you find yourself restructuring or condensing Claude's paragraphs—stop. Only fix errors, not content.

## Quick Start

This orchestrator intentionally runs Claude Code in fully non-interactive mode for Codex automation:

- no Claude permission approvals
- no Claude sandbox gating
- no need to pass extra flags at launch time

It assumes the outer environment already provides the real safety boundary, such as a VM, container, or Codex sandbox.

```bash
# Launch a task
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py launch \
  --prompt-file /abs/path/to/prompt.txt \
  --workdir /abs/path/to/repo \
  --model 'anthropic/claude-opus-4-7[1m]' --effort xhigh

# View all tasks
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py status \
  --state-id GENERATED_STATE_ID

# View a single task + log tail
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py status JOB_ID \
  --state-id GENERATED_STATE_ID --tail 30

# Resume the same session
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py resume JOB_ID \
  --state-id GENERATED_STATE_ID --message "Continue with the remaining work" \
  --model 'anthropic/claude-opus-4-7[1m]' --effort xhigh

# Interrupt a running attempt to revise instructions or recover from a no-output stall
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py interrupt JOB_ID \
  --state-id GENERATED_STATE_ID \
  --message "Stop the current command if it is stuck. Summarize the current state and blocker, then continue the original scope if possible." \
  --model 'anthropic/claude-opus-4-7[1m]' --effort xhigh

# Queue a follow-up that should run only after the current attempt finishes
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py queue JOB_ID \
  --state-id GENERATED_STATE_ID --message "After your current work finishes, do this next" \
  --model 'anthropic/claude-opus-4-7[1m]' --effort xhigh
```

Default to omitting `--state-id` on the first launch. The orchestrator auto-generates a fresh state id, which avoids accidental collisions when agents launch jobs by habit.

## Model Recommendation

Use `--model 'anthropic/claude-opus-4-7[1m]' --effort xhigh` by default for delegated Claude work when an explicit override is needed. If `--model` is omitted, the orchestrator inherits the machine's Claude Code default from `~/.claude/settings.json`.

## Permission Model

The orchestrator gives Claude Code full non-interactive permissions by default. Launch, resume, queue, and interrupt commands always use:

- `--permission-mode bypassPermissions`
- `--dangerously-skip-permissions`

## Key Concepts

| Term | Meaning |
|------|---------|
| **State** (`state_id`, `state_root`) | A single orchestration run, holding one or more jobs under an isolated temp directory. Each run gets its own `state_id`. |
| **Job/Task** (`job_id`) | A single Claude Code invocation within a state, with its own session, logs, and prompt. One job = one focused task. |
| **Attempt/Resume** | One interaction within a job. A job may have multiple attempts via `resume`. All attempts share the same Claude session, so Claude retains full history. |

## Running Job Control

Long Claude Code jobs can legitimately spend time reading, planning, testing, or waiting for commands to finish. Do not treat slow progress, verbose logs, or a period with no file writes as stuck by itself. First inspect the job status, log timestamps, active process, and worktree status:

```bash
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py status JOB_ID \
  --state-id STATE_ID --tail 50 --tail-both
```

The tail output is the raw Claude `stream-json` transcript. Use it to inspect the actual last events before acting. It can include `thinking` entries, tool calls such as `Bash`/`Skill`/`Agent`, local-agent lifecycle events such as `task_started` and `task_notification`, tool results with `agentId`/usage, and final `result` records. When diagnosing a suspected stall or forbidden subagent use, read the last raw lines rather than relying only on the job status label.

For most active-job interventions, use `interrupt` with a continuation prompt. This is appropriate when revising instructions for a running job, when Claude appears to be on the wrong path, or when the session tails have shown no changes for about 20 minutes. Before 20 minutes, wait patiently unless there is stronger evidence of drift, a dead child process, an infinite loop, or an explicit user request to intervene. Interrupting sends `SIGINT` to the active Claude CLI/tool process tree, waits for that attempt to stop, then resumes the same Claude session with the new message. Keep the original scope unless the user explicitly changes it:

```bash
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py interrupt JOB_ID \
  --state-id STATE_ID \
  --message "Stop the current command if it is stuck. Summarize the current state and blocker, then continue the original scope if possible."
```

Use `--interrupt-timeout SECONDS` if Claude needs more time to unwind a long tool call.

Use `queue` or `resume --wait` only to assign the next task that should run after the current attempt finishes. These commands are for sequential task handoff, not for stuck-job recovery or instruction revision. `queue` is a foreground wait in the caller process, not a durable background queue; if the caller is killed before the wait finishes, rerun the command. Use plain `resume` only when the job is already stopped.

## Handoff Prompt Template

```text
<role>
You are Claude Code working as a subordinate agent.
state-id: ......
state-root: ......
</role>

<task>
[Specific expected output. Clearly define what "done" looks like.]
</task>

<context>
- Working directory: [...]
- Relevant file paths: [...] (provide paths, not full content)
- Known constraints: [...]
</context>

<operating_rules>
- Read the repository and relevant files first; verify assumptions before acting.
- Stay within the requested scope; avoid unrelated work.
- Do not use subagents or the Agent tool; complete the task directly in this Claude session.
- For explanatory tasks, output to a markdown file inside state-root and return the path.
</operating_rules>

<deliverable>
- Describe what you changed or learned.
- List the files you touched.
- Explain how you verified the results.
- Note any uncertainties or follow-up work.
</deliverable>
```

If the prompt needs Claude to know generated paths up front, use placeholders instead of manually fixing `state_id`:

- `{{STATE_ID}}`
- `{{STATE_ROOT}}`
- `{{JOB_ID}}`
- `{{JOB_DIR}}`
- `{{WORKDIR}}`
- `{{SESSION_ID}}`

The orchestrator expands those placeholders before Claude starts.
It also automatically adds `state_root` to Claude's allowed directories. Use `--add-dir` only for extra paths outside `workdir` and `state_root`.

**Prompt writing tips**:
- Provide file paths and let Claude read them itself—it is an agent with investigation capabilities; you do not need to copy content into the prompt
- When incorrect output is costly, include verification instructions in the task
- When comparing multiple tasks, keep each prompt's scope independent

## Operating Rules

- **One task, one focus**: each launch corresponds to a limited-scope task. Unrelated work should be launched as separate tasks and tracked independently.
- **Review after completion**: wait for Claude Code to finish before reviewing; do not intervene midway.
- **Use one-pass prompts**: write prompts that Claude can finish without follow-up interaction.
- **No Claude subagents**: every Claude prompt must explicitly forbid subagents / the Agent tool because they make sessions more likely to stop updating for long periods.
- **Inspect before acting**: do not treat slowness, long logs, or quiet file status as failure; inspect status/logs/processes before deciding.
- **Interrupt for active-job correction**: use `interrupt` when revising instructions, correcting drift, or recovering from about 20 minutes with no session-tail changes; do not reduce scope by default.
- **Queue only next work**: use `queue` or `resume --wait` only when assigning the next task after the current attempt finishes.
- **Default to full permissions**: the orchestrator bypasses Claude's own approval layer by default. Treat the surrounding VM/container/sandbox as the safety boundary.

## Command Reference

### launch

```bash
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py launch \
  --prompt "task description" \   # or --prompt-file path, or stdin
  --workdir /abs/path/to/repo \
  --model 'anthropic/claude-opus-4-7[1m]' \  # recommended; use sonnet only for cheaper low-risk work
  --effort xhigh \
  --add-dir /extra/dir \          # additional accessible directory
  --replace \                     # replace a stopped task with the same id
  --dry-run                       # generate command only, do not execute
```

Add `--state-id` only when you intentionally want multiple launches to share one state.
Design prompts so Claude can finish in one pass.

After running, the following fields are printed:
```text
job_id: ...
state_id: ...
state_root: ...
status: ...
session_id: ...
command: ...
```

### status

```bash
# All tasks
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py status \
  --state-id STATE_ID

# Single task + view both stdout and stderr
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py status JOB_ID \
  --state-id STATE_ID --tail 30 --tail-both
```

### resume

```bash
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py resume JOB_ID \
  --state-id STATE_ID \
  --message "follow-up instructions" \
  --model 'anthropic/claude-opus-4-7[1m]' \  # optional override
  --effort xhigh         # optional override
```

Add `--wait` only when the message is the next task to run after the current attempt finishes.

### queue

```bash
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py queue JOB_ID \
  --state-id STATE_ID \
  --message "follow-up instructions after the current attempt finishes" \
  --model 'anthropic/claude-opus-4-7[1m]' \
  --effort xhigh \
  --wait-timeout 3600
```

`queue` waits for the running attempt to finish, then starts a normal resume attempt in the same Claude session. Use it only for next-task handoff, not for stuck-job recovery.

### interrupt

```bash
python ~/.codex/skills/claude-for-codex/scripts/claude_orchestrator.py interrupt JOB_ID \
  --state-id STATE_ID \
  --message "Stop the current command if it is stuck. Summarize the current state and blocker, then continue the original scope if possible." \
  --model 'anthropic/claude-opus-4-7[1m]' \
  --effort xhigh \
  --interrupt-timeout 30
```

Use this for most active-job interventions: revised instructions, suspected drift, stuck tool calls, dead/no-output child processes, infinite loops, explicit user requests, or about 20 minutes with no session-tail changes. Before 20 minutes, wait patiently unless there is stronger evidence of a real problem. `interrupt` stops the current Claude CLI/tool process tree and then sends the message via `--resume` in the same session.

All commands support `--json` for JSON-formatted output.

## Task Directory Structure

```
<system temp dir>/codex-claude-code-orchestrator-state-<state_id>/
├── registry.json              # Summary index of all tasks
└── jobs/JOB_ID/
    ├── job.json               # Task metadata and attempt history
    ├── settings.json          # Claude settings
    ├── prompt.txt             # Launch prompt
    ├── attempt-NNNN-input.txt # Input for resume attempts
    ├── stdout.log             # Claude stdout
    ├── stderr.log             # Claude stderr
    └── output_doc_v1.md       # Document written for the user
```
