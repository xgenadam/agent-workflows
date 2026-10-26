# Working strategy for agent projects

Strategy v2 (2026-10-25). Projects keep a copy of this file; this line says which version a copy is. Moving
an existing project to this version: `WORKING_STRATEGY_MIGRATION.md`, published next to this file.

How a multi-session agent project plans, runs, records and merges its work, so that any model can pick up a
task cold, a crash loses at most one step, tasks that don't collide can run in parallel, and the reasons
things worked or failed are kept. This file is generic; each project's `AGENTS.md` and `PLAN.md` fill in the
project-specific parts (see "What the project fills in").

## Principles

- One task per session (or per subagent). Every task leaves a journal entry, killed ones included; the
  "why" is what the next model needs.
- Write as you go. The entry and the task's run log are updated after every step that costs real time (a
  long test run, a benchmark, a build), not at the end. Sessions crash, run out of context or get stopped.
- Each fact lives in one place. The entry is the narrative, the ledger holds the numbers, the index holds
  status, locks and one-line results, the README gets the final write-up. Everything else points there.
- Every number cites its source: a ledger row, a log path, a result file.
- Shared files have one writer at a time (see "Who writes what").
- Times are UTC with a `Z` (`2026-10-25 01:50Z`) everywhere.

## Layout

| File | Holds | Written by |
|---|---|---|
| `WORKING_STRATEGY.md` | This file, copied in unchanged | Updated only from the source |
| `AGENTS.md` | Read order, bounds, how to run things, verification rules, the project's fill-ins | When a rule changes |
| `PLAN.md` | Context and the task board (columns below) | The planner / orchestrator |
| `journal/INDEX.md` | Board status, locks and queue, inbox, decisions and asks for the user | Claim and close only |
| `journal/TEMPLATE.md` | The entry template (below) | Rarely |
| `journal/NN-<slug>.md` | One entry per task | That task only |
| `journal/runs/T<n>.md` | The task's own run log: one row per run (`T<n>-R1`, `T<n>-R2`, ...), failures included | That task only |
| Ledger (e.g. `results/ledger.md`) | Every task's runs, merged at close; the scoreboard before/after | Close only |

Entry names are pre-assigned on the board (`NN-<slug>.md`); the number is an identifier, not the execution
order. Run IDs carry the task (`T7-R3`), so parallel tasks never race for a number.

Status values, used everywhere: `todo`, `ready`, `in progress (model, since)`, `waiting (resource)`,
`done`, `killed`, `parked`.

## The task board (`PLAN.md`)

| Column | Meaning |
|---|---|
| Task | `T<n>` and a title |
| Depends on | Tasks that must be `done` (or `killed`, if the board says that's enough) first |
| Resources | Locks the task needs, per phase: e.g. `prep: none; run: port:3000`. See below |
| Touches | Code areas the task changes (the areas are defined in `AGENTS.md`) |
| Model / effort | What to run it with, and why if it isn't obvious |
| Planning stage | yes / no / short |
| Entry | `NN-<slug>.md` |

A task is **ready** when its dependencies are met. Two ready tasks may run at the same time when their
Touches don't overlap; their Resources may overlap, since locks are taken per phase (below). Tasks that touch
the same area run one after the other; the board says which goes first.

## Resources and locks

Anything two tasks can't use at the same time without failing or changing each other's results is a lock,
listed in `INDEX.md`:

| Resource | Holder | Since | Queue |
|---|---|---|---|
| `port:3000` | T7 (run phase) | 2026-10-25 09:10Z | T9 |
| `db:dev` | - | | |

Typical resources (the project lists its own in `AGENTS.md`):

- A port, or a dev server that a task starts or restarts.
- A local database or test fixture that a task resets, seeds or migrates.
- A shared environment: staging, a test account, an external API with rate limits.
- `machine`: the whole computer, for phases whose result is a measurement (benchmarks, timing), since
  other work running at the same time skews the numbers.

Rules:

- Lock only the phase that needs it. A task's prep (design, code, scripts, tests that need nothing) runs
  without locks; only its run phase waits. A task in the queue keeps doing prep, or stops with status
  `waiting (<resource>)`; it doesn't hold anything while it waits.
- Take and release locks with a commit to `INDEX.md` on the main branch (see "Who writes what"). The queue
  is first come, first served; the user can reorder it.
- A lock whose holder's entry has no `Last update` for over 2 hours, with nothing of it running, is stale
  (see the workflow).

## Branches and worktrees

Each running task works in its own git worktree, on its own branch:

```bash
git worktree add .worktrees/T7 -b task/T7      # from the main checkout, on main
```

- `.worktrees/` is inside the project (so anything that sees only the project folder sees the worktrees
  too; a worktree's `.git` points back into the main repo's `.git`) and listed in `.gitignore`.
- A branch can be checked out in only one worktree at a time, which matches one task, one branch, one folder.
- A new worktree has only committed files. `AGENTS.md` lists what to link in from the main checkout
  (dependencies such as `node_modules` or `.venv`, untracked config such as `.env`, large data); link,
  don't copy.
- The task commits only to `task/T<n>`, and writes only its own entry, its run log and the code it touches.
- When the task closes, its branch is merged into main and the worktree removed (`git worktree remove
  .worktrees/T7`, then `git branch -d task/T7`).

A project that runs one task at a time may skip worktrees and work on one task branch in the main checkout;
everything else here still applies.

## Who writes what

Shared files (`INDEX.md`, the ledger, `PLAN.md`) are written only at claim and at close, on main:

- **With an orchestrator** (recommended for parallel work): one session plans, starts each task as a
  subagent in its worktree (Claude Code: `isolation: "worktree"`), and is the only writer of shared files:
  it takes locks, merges branches and adds run logs to the ledger. Subagents write their entry, run log and
  code, and hand back their Hand-over block.
- **With sessions you start yourself**: each session claims and closes with its own small commit on main.
  If the commit doesn't apply because another session's landed first, it re-reads `INDEX.md` and tries again;
  a claim that lost the race picks another task.

## Merging and conflicts

1. Bookkeeping can't conflict: each task writes only its own entry and run log; shared files are written
   only at claim and close by one writer. Append-only files (the ledger) can use `merge=union` in
   `.gitattributes`.
2. Most code conflicts are avoided at planning, by the Touches column.
3. First to finish merges. A later task merges main into its branch before closing, resolves any conflict in
   its own worktree (its session knows its change best), and re-runs verification: what is kept is the
   combined code, not the change alone. Use `git merge`, not rebase, so the history shows what was verified.
4. A conflict about behaviour, not text (two changes to the same function's logic, two kept changes that
   only work apart), goes to the user as an ask; the task waits.

## Read order for a session

1. `AGENTS.md`, then `PLAN.md`, then `journal/INDEX.md` (locks, queue and inbox included).
2. The entry of the task you take, in full, if it exists (a stale claim, or a design stage done earlier).
3. For each task it depends on: the entry's **Hand-over** block. Read the full entry only where you build on
   its code or measurements.

## Per-task workflow

These rules apply to every session or subagent that works in the project, including one started from
another project (a client report, say); its entry names that session.

1. **Read** the files above. Switch to the model and effort the board recommends; if you are a different
   model, say so in the entry.
2. **Check for stale claims and locks.** A task `in progress` whose entry's `Last update` is over 2 hours
   old, with nothing of it running, is stale. Say so in `INDEX.md`, release its locks, then either finish
   that entry first from what it left (its worktree, run log, Hand-over, Result) or ask the user.
3. **Claim** a ready task: create its worktree and branch, copy `journal/TEMPLATE.md` to the entry's name
   there, fill in the header, and commit (`WIP: T<n> claimed`). On main, set the board status to
   `in progress` (claim commit).
4. **Design first** if the board says the task has a planning stage: approach, alternatives considered and
   why they lost, kill test, test plan. Re-read it against the kill test before writing code.
5. **Prep** without locks: code, scripts, tests that need no locked resource.
6. **Run phase**: take the locks the board lists (or join the queue and keep prepping). After every costly
   step, before the next: a row in `journal/runs/T<n>.md`, the command and output files in What I did, the
   numbers in Measurements, `Last update` bumped, a commit (`WIP: T<n>-R<k>`). Release locks as soon as the
   run phase ends.
7. **Check the inbox** in `INDEX.md` at each step: findings from other tasks that affect this one.
8. **Finish** the entry: Hand-over, Result, Verification, Side findings, Open questions, Next step, wall
   time. A finding another running task needs goes in the inbox now, not only in the entry.
9. **Close**: merge main into the branch, resolve, re-verify (see Merging); then on main: merge the branch,
   add the run log to the ledger, update the board row (status, one-line result) and the scoreboard, release
   any locks left, remove the worktree. Free shared resources unless the board says the next task reuses
   them.

Escalate one model up when a task's first real step doesn't explain what you see; never escalate for
journal writing.

## Keeping the index small

- A one-line result is one line: kept or killed, the flag or file that carries it, one headline number.
- Notes for a later task go in the entry's Next step (and its Hand-over), not in the index header.
- The inbox holds only findings for tasks running now, dated and addressed (`to T9:`); the receiver deletes
  the line once it has acted on it.
- Decisions and asks for the user are appended, dated, in one section, so the user can answer them in a batch.

## What the project fills in

The project's `AGENTS.md` says "work follows `WORKING_STRATEGY.md` (v2)" and adds:

- Bounds, how to run things, and the verification the project uses (tests, a quality gate).
- Its resources: which ports, servers, databases and environments need a lock, and in which phases.
- Its code areas for the Touches column (directories or modules).
- What a new worktree links in from the main checkout.
- Orchestrator or user-started sessions, and the stale-claim time if not 2 hours.
- Anything in this file the project doesn't use (worktrees for one-at-a-time work, Verification for pure
  research).

## Entry template (`journal/TEMPLATE.md`)

Drop the sections a project doesn't use; keep the order.

```markdown
# NN: <task title> (T<n>)

Status: in progress | waiting (<resource>) | done | killed · Model: <model> · Effort: <level> · Session: <project / session id if not this project's own> · Branch: task/T<n> · Started: <YYYY-MM-DD HH:MMZ> · Last update: <YYYY-MM-DD HH:MMZ> · Wall time: <h:mm>

## Hand-over

At most 10 lines, written when the task ends (killed included): result, headline numbers with their run
IDs, the flag or file that carries the change, traps, next step. Later tasks read this block first.

## Goal

One or two sentences: what this task should change and how success is measured.

## Hypothesis and kill test

The mechanism we expect, the ceiling we computed, and the cheapest observation that would kill the idea.

## Setup

- Branch / base commit: `task/T<n>` from `<main sha>`
- Configuration: commands, flags, environment
- Locks: <per phase, as taken>
- Run IDs: T<n>-R1 (+ R2 ...)
- State before: resources free (yes/no), anything else that changes results

## Design (only for tasks with a planning stage)

Approach, alternatives considered and why they lost, data layout, failure modes, test plan.

## What I did

Exact commands in order, with their arguments and output files, enough to reproduce without guessing.
Append each step when it finishes.

## Measurements

Tables with the project's metric names; every number cites its run ID. Add each row when its run finishes.

| Metric | Baseline (<run>) | This run (T<n>-R<k>) | Delta |
|---|---:|---:|---:|

## Result

**Kept** or **killed**, and why, in two or three sentences. If killed: what the kill test showed, and
whether a variant is worth a later look.

## Verification

The project's correctness check on the candidate, and again on the combined code after merging main.

## Side findings

Anything a later task should know: measurements made in passing, code pointers (file:line), traps.

## Open questions

## Next step

Which task follows and what it should start with. Say what was left running.
```
