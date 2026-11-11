# Context and recovery

Codex compacts long threads automatically and does not ask before doing it. Treat compaction as normal. The task board, not your memory of the chat, is the source of truth.

`.superagent/` is orchestration state, not a product change. It is self-ignored (`.superagent/.gitignore` contains `*`), so it never appears in the product diff. Never skip it because the user's scope names only product directories.

## On every resume or suspected compaction

1. Read `.superagent/STATE.md`.
2. Read `.superagent/MEMORY.md`.
3. List `.superagent/tasks/` and read the short headers of `report.md` for active tasks.
4. Rebuild: what is done, what is running, what is queued, which leases are active, what waits on the user.
5. Only then act.

Do not reconstruct from the conversation summary alone. If `STATE.md` and the summary disagree, trust `STATE.md` and the report files, then update `STATE.md` if it is stale.

## The chain of recall

Codex has no object graph or recall tool. Use paths:

- A message carries a task id and a path, never a long body.
- From a task id, read `tasks/<id>/brief.md`, then `tasks/<id>/report.md`.
- From a report, follow the paths it names.
- From `MEMORY.md`, follow the source labels if a fact needs checking.

This is the replacement for SA's chained recall: every hop is an explicit file path, and every message stays short.

## What to persist

Persist:

- the current plan and task table;
- write-set leases and slot queue;
- durable facts and decisions with date and source;
- the baseline commit/status;
- open decisions.

Do not persist:

- worker transcripts or reasoning;
- long tool output;
- superseded plans;
- praise, progress narration, or duplicate summaries.

`STATE.md` is overwritten on every change. `MEMORY.md` keeps only current entries; when a fact or decision is superseded, replace it. Do not build a history log.

## Briefs and summaries

- Copy the user's requirement and acceptance criteria verbatim into the brief. Do not summarize them into your own words; summaries drift.
- A worker's context is `fork_turns="none"` plus its brief, `MEMORY.md`, and the files it reads. It does not inherit your conversation.
- Keep the user's thread short: report outcomes and evidence, not the work.

## When the user changes requirements mid-task

1. Update `STATE.md` and, if durable, `MEMORY.md`.
2. Send `STEER` to the affected running workers; wake stopped workers with `followup_task`.
3. If the change alters the split or write sets, re-plan and update briefs before the workers continue.
4. If the change invalidates completed work, say so and re-dispatch; do not hide it.

## After a crash or restart

- `list_agents` shows what is still live. Dead workers are gone; their partial edits may remain in the shared workspace.
- Re-dispatch from the brief. Tell the new worker: "A previous attempt may have left changes. Inspect `git diff` and continue or revert as the brief requires."
- Do not assume a task is done because its worker disappeared. Check `report.md` and the verifier.

## Memory hygiene

- One fact per line, with a date and source.
- Replace stale facts; do not append a second version.
- A newer user instruction wins over an older `MEMORY.md` entry; update the entry.
- Keep `MEMORY.md` short enough to read in one screen. If it grows, move details to a task report and keep the pointer.
- Never put credentials, tokens, or private keys in `MEMORY.md`, briefs, or reports.
