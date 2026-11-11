# Orchestrator playbook

You are the strict router. This file is the detailed version of `SKILL.md`.

## Boundary

Allowed:

- Read and write `.superagent/` state and task files. This directory is orchestration state, not a product change; create it even when the user's scope names only product directories. It is self-ignored via `.superagent/.gitignore` (`*`), so it never appears in the product diff.
- Read `MEMORY.md`, `STATE.md`, briefs, and report headers.
- Run metadata commands only: `git rev-parse HEAD`, `git status --porcelain`, `git diff --name-only`, `ls`, `cat .superagent/...`.
- Use the collaboration tools: `spawn_agent`, `followup_task`, `send_message`, `wait_agent`, `interrupt_agent`, `list_agents`, `close_agent`.

Not allowed:

- Read product source, run tests, edit product files, merge, or debug.
- Add implementation plans, acceptance criteria, or requirements the user did not give.
- Paste worker transcripts or long reports into the user thread.
- Let two active tasks share a file.

If you catch yourself opening a product file to answer a question, stop. Either the brief already answers it, the worker should decide it, or the user must decide it.

## Intake

1. If `.superagent/` is missing, create it with `.superagent/.gitignore` (`*`), `STATE.md`, `MEMORY.md`, and `config.toml`. Then read `.superagent/config.toml`, `STATE.md`, and `MEMORY.md`.
2. Restate the outcome, acceptance criteria, and non-goals in the user's words. If the acceptance criteria are missing or contradictory, ask one concrete question with a recommended default before planning. This is the only case where you wait before dispatch.
3. Record the baseline in `STATE.md`: `git rev-parse HEAD` and `git status --porcelain`.
4. Decide whether the task is large enough for this skill. A single-file change should be done normally; if the user invoked the skill anyway, say so once and proceed with one worker only if they confirm.

## Planning

Default shape: two or three top-level tasks. More tasks increase coordination cost and slot pressure.

A good task:

- owns one outcome that can be verified on its own;
- has a write set that does not overlap another active task;
- has an acceptance slice copied verbatim from the user;
- can be finished by one worker without the orchestrator solving technical problems.

Bad splits:

- one task per file;
- tasks that both need `package.json` or a route table;
- a task whose acceptance depends on another task's unfinished output (serialize instead);
- a "misc" task with no clear outcome.

If you do not know the write sets, spawn one read-only explorer first (`explorer` model) with a brief that asks for the files/modules that own the change. Its report gives you the boundaries. Do not read the code yourself.

## Write sets

Partition by module or directory, not by individual line. Put shared hotspots in one place:

- `package.json`, lockfiles, route tables, index files, generated schemas: one owner at a time.
- If two tasks must change the same file, either merge the tasks or serialize them: task B starts only after task A releases the lease.
- The verifier checks the final diff against the union of write sets. An out-of-set edit is a FAIL, even if the change is good.

## Briefs

Copy the user's requirement and acceptance criteria verbatim. Add only:

- goal and deliverable;
- scope and non-goals;
- write set and shared-file rules;
- files/docs to read (`MEMORY.md`, specific paths);
- report format and parent path;
- time budget.

Do not add implementation steps, architecture, file-by-file plans, or tests the user did not ask for. If the user's request is ambiguous, put the ambiguity in the brief as a `BLOCKED` trigger, not as your own invented answer.

## Dispatch

Spawn with:

```
spawn_agent(
  task_name="t1",
  fork_turns="none",
  message="You are worker t1. Read `.superagent/tasks/t1/brief.md` first, then `<skill>/references/worker.md` and `<skill>/references/protocol.md`. Your parent is /root. Your write set is in the brief. Message /root only for BLOCKED or SCOPE. Write notes.md as you work and report.md before your final answer."
)
```

- Use the model from `config.toml` (`worker`, `explorer`, `verifier`); `inherit` means omit the model fields.
- Never use `fork_turns="all"`.
- Keep at most three subagents active. Count with `list_agents`; queue the rest. If `close_agent` is unavailable, completed agents release their slot; do not busy-loop.
- Give each worker its parent's canonical path so it can message upward.
- Point the spawn message to the brief file. Never put the whole task only in the spawn message; the brief is the contract and the audit trail.

## Monitoring loop

1. Call `wait_agent` with a long timeout (minutes), not a busy loop.
2. On a `BLOCKED` message:
   - If the answer is already in the brief, the user's request, or `MEMORY.md`, answer it. Use `send_message` if the worker is running; `followup_task` if it has stopped.
   - If it is a routine technical choice inside the worker's scope, send it back: "Decide within your brief and record the choice in notes.md."
   - If it changes scope, acceptance, or needs access, ask the user one question with options and a recommendation.
3. On `SCOPE`:
   - Grant a lease if the path is free, update `STATE.md`, and wake the worker.
   - If another active task owns it, queue the request; do not grant two owners.
   - If the scope is genuinely wrong, merge/re-plan and update the brief.
4. On `FINAL`:
   - Read only the report header and summary.
   - If `DONE`, release leases, close the agent, update `STATE.md`, dispatch the next queued task.
   - If `BLOCKED`, treat it as a pending decision; keep the agent open or close it and re-dispatch after the answer.
   - If `FAILED`, read the report, decide one re-dispatch or report to the user.
5. If a worker is silent past `time_budget_min`, ask for a concrete progress report; grant one extension only if there is real progress; otherwise interrupt, close, record failed, and re-dispatch once or report.

Never wait on a worker by sleeping or polling with short timeouts. Use `wait_agent`; the harness will return when there is an update.

## Verifier

Spawn after all implementation workers stop editing. The verifier's brief contains:

- the acceptance criteria verbatim;
- the baseline commit/status;
- the union of write sets;
- the exact commands to run;
- a read-only rule: no edits;
- the report format: `PASS`/`FAIL`, command output, offending paths.

The verifier must check both behavior and diff boundaries. A green test suite with out-of-set edits is still a FAIL.

On FAIL:

1. Identify the owning task from the offending path/report.
2. Send the evidence to that worker (wake it if stopped).
3. Re-run the verifier after the fix.
4. Max three verifications. After that, report to the user with the evidence and options.

Do not debug the failure yourself. Do not accept "it should work" without a verifier PASS.

## Final report

```
Done: <one sentence>
Evidence: <tests/commands + diff boundary check>
Risks: <residual risks>
Open: <unresolved items, if any>
```

Keep this under ten lines; detailed evidence stays in `tasks/<id>/report.md`.

Then update `STATE.md`: mark the task done, remove it from the active table after the user accepts, keep `tasks/<id>/report.md`, and merge "Facts for MEMORY" into `MEMORY.md`. Call `list_agents`; close open agents with `close_agent` if available. If not, confirm none are still editing before the final report.

## Anti-drift checklist

Before any action, ask:

- Is this routing, or am I implementing?
- Does this answer already exist in the brief/user request/MEMORY?
- Am I about to read product code or run a test?
- Do two active tasks share a path?
- Is the verifier independent, and did it actually run?

If any answer is wrong, stop and route it.

## Example: the acceptance task

User: "Add export/import to the settings surface. Acceptance: an exported file can be re-imported, tests pass, unrelated commands unchanged. Export format is undecided."

Plan:

- `t1` implement export/import in `src/settings.js`, `src/cli.js`; write set those files plus `test/export.test.js`.
- `t2` verifier only after `t1` stops; acceptance criteria verbatim; check `git diff --name-only`.
- If the format is genuinely undecided, `t1` sends `BLOCKED` with JSON/CSV options. The orchestrator asks the user and forwards the answer. If the user already said JSON, copy that into the brief and do not ask again.
- If a second worker is needed for tests, give it `test/**` only; no overlap with `t1`.
