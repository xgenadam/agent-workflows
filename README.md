# Agent workflows

Working practices for projects that AI coding agents (Claude Code, Codex, Cursor and the like) work on
across many sessions. They say how work is planned, claimed, run, recorded and merged, so that:

- any model can pick up a task cold, from the files alone;
- a crashed or interrupted session loses at most one step;
- tasks that don't touch the same code can run in parallel;
- the reasons things worked or failed are kept, not just the code that survived.

Everything is plain Markdown in the project's own repo. There is nothing to install.

## What's here

| File | For |
|---|---|
| [`WORKING_STRATEGY.md`](WORKING_STRATEGY.md) | The strategy itself: principles, file layout, task board, resource locks, branches and worktrees, who writes shared files, merging, the per-task workflow, the journal and its entry template |
| [`WORKING_STRATEGY_MIGRATION.md`](WORKING_STRATEGY_MIGRATION.md) | A guide for an agent moving an existing project onto the strategy without changing its code, results or history |

## Quick start

### New project

1. Copy `WORKING_STRATEGY.md` into the project root, unchanged.
2. Ask the agent to set the project up from it, for example:

   > Read WORKING_STRATEGY.md. Set this project up to follow it: write AGENTS.md with the fill-ins it
   > lists, a PLAN.md with a task board for <your goal>, and journal/INDEX.md and journal/TEMPLATE.md.
   > Ask me about anything you'd be guessing.

3. Review `AGENTS.md` and the board, then start work:

   > Read AGENTS.md and take the next ready task.

### Existing project

Copy both files into the project root and ask:

> Read WORKING_STRATEGY.md and WORKING_STRATEGY_MIGRATION.md, then migrate this project.

The migration is a task of its own: one session, one journal entry, one commit. It changes how work is
planned and recorded, not the code.

### Making sure the agent reads it

Point the file your tool loads automatically at `AGENTS.md`. For Claude Code that is `CLAUDE.md`; one line
is enough:

```markdown
Read AGENTS.md first. Work follows WORKING_STRATEGY.md.
```

### Running tasks in parallel

With an orchestrator (recommended), one session plans and starts each task as a subagent in its own git
worktree, and is the only one that writes the shared files:

> Act as the orchestrator described in WORKING_STRATEGY.md: start the ready tasks whose Touches don't
> overlap as subagents in their own worktrees, and merge each one when it closes.

You can also start sessions yourself; each one claims and closes its task with a small commit on main.

## The ideas in brief

- **One task per session**, each with a journal entry, killed tasks included. The entry's short
  **Hand-over** block is what later tasks read first.
- **Write as you go.** The entry and the task's run log are updated after every slow step, not at the end.
- **Each fact lives in one place**, and every number cites its source (a run ID, a log, a result file).
- **A task board** (`PLAN.md`) lists each task's dependencies, the code areas it touches, the shared
  resources it needs and the model to run it with. Tasks with no overlap in what they touch can run at the
  same time.
- **Locks** for things that can't be shared (a port, a database a task resets, a benchmark run), taken only
  for the phase that needs them.
- **One branch and worktree per task**, merged into main when the task closes, with verification re-run on
  the merged code.
- **Asks for the user are batched** in one section of the index, so you can answer them together.

## Project layout

```text
WORKING_STRATEGY.md      this strategy, copied in unchanged
AGENTS.md                read order, rules, how to run and verify things, the project's fill-ins
PLAN.md                  context and the task board
journal/
  INDEX.md               status, locks and queue, inbox, decisions and asks for the user
  TEMPLATE.md            the entry template
  NN-<slug>.md           one entry per task
  runs/T<n>.md           each task's run log
results/ledger.md        every run, merged in when its task closes (optional)
.worktrees/              one git worktree per running task (in .gitignore)
```

Projects that don't need all of it drop parts (worktrees for one-task-at-a-time work, Measurements for
tasks that measure nothing) and say so in `AGENTS.md`.

## Versions

The line under the title of `WORKING_STRATEGY.md` names its version (currently v2). Projects keep their
own copy, and that line tells you which version a copy is. Project-specific rules belong in the project's
`AGENTS.md`, never in the copy.

## Licence

[CC0 1.0 Universal](LICENSE): dedicated to the public domain. Copy, change and use these files for any
purpose, with no need to credit or ask.
