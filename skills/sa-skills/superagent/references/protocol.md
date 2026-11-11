# Protocol

The schemas, messages, and tool semantics the orchestrator and workers share.

## Workspace and tools

Codex multi-agent facts this skill relies on:

- All agents share one directory and one filesystem. Edits are immediately visible to everyone. There is no workspace fork.
- The default concurrency is four slots including the root agent, so at most three subagents can be active at once. `limits.max_active_subagents` defaults to 3.
- `fork_turns` is `all` or `none`. Always pass `none`. `all` inherits the parent conversation and forbids model overrides.
- A child's final answer is delivered to its parent automatically. `send_message` delivers without starting a turn; `followup_task` starts a turn if the target is idle.

Tool map (names in current Codex builds):

| Purpose | Tool |
| --- | --- |
| Create a worker | `spawn_agent` |
| Wake an idle worker with a new task/answer | `followup_task` |
| Send to a running worker | `send_message` |
| Wait for updates | `wait_agent` |
| Stop a turn | `interrupt_agent` |
| List live agents | `list_agents` |
| Close an agent and its descendants, free a slot (if available) | `close_agent` |

If a build exposes `send_input` / `resume_agent` instead, map them to `send_message` / `followup_task`.

## Directory layout

```
.superagent/
├── .gitignore          # "*": the board is never a product change
├── STATE.md
├── MEMORY.md
├── config.toml
└── tasks/
    └── <id>/
        ├── brief.md
        ├── notes.md
        └── report.md
```

`.superagent/` is orchestration state, not a product change. A user instruction such as "only change `src/` and `test/`" applies to product files; it does not forbid the task board. Create the board anyway, with `.superagent/.gitignore` containing `*`, so it never appears in the product diff. If the user explicitly forbids `.superagent/`, stop and ask; do not silently fall back to messages-only coordination.

Task ids are short and stable: `t1`, `t2`, `t1.1`, `t1.2`. The canonical agent path is `/root/t1` for a top-level task and `/root/t1/t1.1` for a nested one.

## STATE.md

Overwrite the whole file on every change. Keep it short and tabular.

```markdown
# Superagent State
Updated: <ISO timestamp>
Project: <absolute cwd>
Baseline: <git HEAD sha> (<clean|dirty>)
Plan: <one or two sentences: current outcome and split>

## Tasks
| id | parent | status | owner | write_set | lease | brief | report |
| --- | --- | --- | --- | --- | --- | --- | --- |
| t1 | root | running | /root/t1 | src/settings.js, src/export.js | active | tasks/t1/brief.md | tasks/t1/report.md |
| t2 | root | queued | - | test/export.test.js | - | tasks/t2/brief.md | - |

Queue: t2, t3
Active: 1/3
Open decisions: <none | one line>
```

Status values: `queued`, `running`, `blocked`, `verifying`, `done`, `failed`.

`write_set` is comma-separated paths or directory prefixes. `lease` is `active`, `queued`, or `released`.

## MEMORY.md

Current facts only. Superseded entries are replaced, never appended as history.

```markdown
# Superagent Memory

## Facts
- [2026-11-11] Node 22 ESM; tests run with `node --test`. (source: t1)

## Decisions
- [2026-11-11] Export format is JSON for v1. (source: user)
```

Workers read this file. Only the orchestrator writes it, using "Facts for MEMORY" from reports.

## config.toml

Defaults when the file is absent:

```toml
[models]
worker = "inherit"
explorer = "gpt-6-luna"
verifier = "inherit"

[limits]
max_active_subagents = 3
max_depth = 3
time_budget_min = 30
extension_min = 30
max_verifications = 3
```

`inherit` is this skill's convention, not a Codex keyword: do not pass `model` or `reasoning_effort`; the child uses the parent's current model. Only `fork_turns="none"` allows overrides. If a configured model fails to spawn, retry with `inherit` and record it in `STATE.md`.

## brief.md

The orchestrator writes this before spawning. The worker reads it first. Every spawn message must point to this file; do not put the full task only in the spawn message.

```markdown
# Brief <id>
Role: worker
Parent: /root
Task name: <id>
Time budget: 30 minutes

## Goal
<one paragraph: the outcome this task owns>

## Scope
- ...

## Non-goals
- ...

## Acceptance
<verbatim user requirement and acceptance criteria for this slice>

## Write set
- <path or directory prefix> — you own this; do not edit outside it.
- Shared files (serialized): <paths, only if the lease grants them>

## Read first
- `.superagent/MEMORY.md`
- <files or docs the worker needs>

## Report
Write `tasks/<id>/report.md` and end with the same status in your final answer.
Use `send_message` to `<parent>` only for `BLOCKED` or `SCOPE`.
```

Briefs must copy the user's requirements and acceptance criteria verbatim. Do not paraphrase them into a plan. Add only scope boundaries, write sets, references, and routing facts.

## report.md

The worker writes this before its final answer.

```markdown
# Report <id>
Status: DONE | FAILED | BLOCKED
Summary: <two or three sentences>

Files changed:
- <path>

Verification:
- `<command>` -> <result>

Risks:
- ...

Open:
- ...

Facts for MEMORY:
- <durable fact, or "none">
```

The orchestrator reads the header and summary only. The verifier and future workers can read the details.

## Messages

### Worker -> parent

`BLOCKED` — send, then end the turn; wait for `followup_task` with the answer.

```
[<id>] BLOCKED: <one concrete question>
Options: A) ... B) ...
Recommend: A because ...
Impact: <what cannot proceed>
```

`SCOPE` — the brief is wrong, too large, overlaps another task, or needs a path outside the write set.

```
[<id>] SCOPE: <one sentence>
Evidence: <path or observation>
Proposed: <new scope or path>
```

`FINAL` — the final answer. It should start with the status and point to the report:

```
[<id>] DONE. Report: .superagent/tasks/<id>/report.md
```

### Parent -> worker

`ANSWER` — the decision and its reason, not an implementation plan.

```
[t1] ANSWER: Use JSON. Reason: the existing API is JSON and the user chose JSON. Keep the change inside your write set.
```

`STEER` — a new constraint or priority from the user or `MEMORY.md`.

`STOP` — stop work, leave the current files, and report status.

### Orchestrator -> user

Plan (once, at start):

```
Plan: t1 <outcome> (write set ...); t2 <outcome> (...). I will report at acceptance.
```

Decision (only when the user must decide):

```
Decision needed (<id>): <question>
Options: ...
Recommend: ...
```

Final:

```
Done: <outcome>
Evidence: <tests, diff boundary check>
Risks: ...
Open: ...
```

One screen maximum. Details stay in `.superagent/`.

## Write-set leases

- At plan time, assign every task an exclusive write set. No two active tasks may overlap.
- If two tasks need the same path, either merge them, serialize them (task B waits), or give the shared path to a final integration task.
- Shared hotspots (`package.json`, lockfiles, route tables, index files) are serialized by default. A worker that does not own a lockfile must not run a command that rewrites it.
- A worker that needs a path outside its write set sends `SCOPE`. The orchestrator either grants a lease (updating `STATE.md`) or queues it. Grant one owner at a time.
- On `FINAL`, the orchestrator releases the lease. The next owner reads the current file and continues on top of the previous changes; it must not revert them.
- The verifier checks that the final diff is inside the union of granted write sets. Out-of-set edits are a FAIL.

## Slots and nesting

- Count live subagents with `list_agents`. Keep at most `limits.max_active_subagents` active; the root does not count toward that number.
- On `FINAL`, `close_agent` immediately to free the slot; then dispatch the next queued task.
- A worker may spawn a subworker only if `list_agents` shows a free slot and the subtask is bounded and independent. Depth limit `limits.max_depth` (default 3).
- A subworker's write set must be a subset of its parent worker's write set. If it needs more, the parent asks the orchestrator.
- A worker waiting on a subworker still occupies a slot. Prefer breadth over depth when slots are tight.
- In builds with `close_agent`, completed agents remain open and count toward the limit until closed; close them before the final report. In current v2 builds `close_agent` may be absent; completed agents then release their slot, so use `list_agents` to confirm nothing is still running.

## Baseline and verification

Before dispatch, record in `STATE.md`:

- `git rev-parse HEAD`
- `git status --porcelain`

The verifier:

1. Reads its brief (acceptance criteria, write sets, baseline).
2. Runs the acceptance commands.
3. Runs `git diff --name-only HEAD` and `git status --porcelain`; if the baseline was dirty, subtract the baseline paths. Every remaining path must be inside the union of granted write sets.
4. Reports `PASS` or `FAIL` with the exact command output and the offending paths.

If the verifier fails, the orchestrator sends the evidence to the owning worker and re-verifies. Max `limits.max_verifications` (default 3). After that, report to the user with the evidence and options.

## Failure and fallback

- `spawn_agent` fails on the slot limit: queue; do not loop. If `close_agent` is unavailable, wait for a completion and retry once.
- `spawn_agent` fails on a model: retry with `inherit`; record it.
- A worker is silent past its budget: ask for a concrete progress report; grant one extension only if there is real progress; otherwise interrupt, close, and re-dispatch once or report.
- A worker reports `FAILED`: read the report, re-dispatch once with the failure evidence, then report.
- Approval/tool errors: report the exact error to the user; do not retry blindly.
- If `fork_turns` is unavailable: warn the user that context isolation is weaker and keep briefs self-contained.
