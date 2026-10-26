# Migrating a project to the working strategy (v2)

For the agent working in an existing project. You know the project's context; this guide says what to
change and what to leave alone. Read `WORKING_STRATEGY.md` (v2) in full first; copy it into the project
root unchanged if it isn't there yet (it replaces any older `JOURNAL_STRATEGY.md`).

Treat the migration as a task of its own: one session, its own journal entry (`journal/NN-migration.md`, or
the project's equivalent), committed on its own. Nothing in it should change code, results or how the
project's work behaves; it changes how the work is planned, recorded and run.

## Ground rules

- **The project's own rules win.** Bounds, user instructions, "never edit" files, branch names, "never
  push": keep them. Where they conflict with the strategy, keep the project's rule and record the deviation
  in `AGENTS.md` ("Deviations from the strategy").
- **Don't rewrite history.** Finished entries, ledger rows and commits stay as they are. Old run IDs (`R13`)
  keep their names; new runs use `T<n>-R<k>`. No renumbering, no reformatting of old entries beyond what
  step 5 asks.
- **Don't delete documents.** Superseded text is moved or replaced by a pointer, not dropped silently.
- **"main" in the strategy is the project's integration branch**, whatever it's called (`develop`,
  `trunk`, `master`). Task branches come off it.
- **Ask the user, batched, when you'd be guessing**: which tasks may run in parallel when it isn't clear,
  orchestrator or user-started sessions, anything about resources you can't measure. Put the questions in
  `INDEX.md`'s asks section and in your report; don't stop the migration for them, mark the affected
  board cells `?`.
- **Leave the project working the old way until the migration commit lands.** If a task is running right
  now (status `in progress` with a recent `Last update`, or a lock held), don't migrate:
  report that and stop.

## Steps

### 1. Inventory

List what the project has and where each piece of the strategy currently lives:

- Plan and task board (`PLAN.md`, a TODO, issues), the index or status file, journal entries, the
  template, the numbers file or ledger, notes files, decision logs.
- Branches and how work is committed; any running servers or claimed resources.
- Rules about the journal in `AGENTS.md`, `CLAUDE.md` or elsewhere.

Write the inventory as a table in the migration entry: old file or section, what it holds, where it goes.

### 2. Resolve open work

- A task `in progress` that is stale (no update for over 2 hours, nothing running): finish its entry from
  what it left (result files, logs, commits) or mark it stale with a note, and ask the user which.
- Uncommitted work in the tree: don't commit it as part of the migration. Report it and ask.

### 3. The task board

Add the strategy's columns to the existing board, keeping its rows, order and wording:

- **Depends on**: usually already there; check it against what the entries say.
- **Resources**, per phase (`prep: none; run: port:3000`): from what each task actually runs. A phase that
  starts a server, resets a database or measures performance needs the matching lock. Pure code, design or
  analysis needs `none`.
- **Touches**: define the project's code areas first (step 6), then fill one cell per task.
- **Model / effort, Planning stage, Entry**: keep the existing values.
- **Status**: map to the strategy's values (`todo`, `ready`, `in progress`, `waiting`, `done`, `killed`,
  `parked`); `ready` for every `todo` whose dependencies are met.

Then say, in one line under the board, which ready tasks could run in parallel now (no Touches overlap),
or that none can.

### 4. The index

Restructure `journal/INDEX.md` (or its equivalent) to the strategy's sections, moving content rather than
rewriting it:

- Board status: one line per task; a one-line result longer than a line keeps its first sentence and the
  rest moves into that task's entry Hand-over (step 5) if it isn't already in the entry.
- Locks and queue: a table; any old "who is using X" line becomes its rows.
- Inbox: empty, unless a running task needs something.
- Decisions and asks for the user: kept as they are, with your migration questions appended.
- Notes for the next task in the index header move to the previous entry's Next step, or stay in a short
  "next up" line if they have no home.

### 5. Entries

- Copy the strategy's entry template into `journal/TEMPLATE.md`, adjusted for the project (drop sections
  it never uses; keep project-specific ones, e.g. a named quality gate under Verification).
- Add a **Hand-over** block to finished entries that a not-yet-done task depends on: at most 10 lines,
  taken from the entry's own Result, Measurements and Next step, nothing new. Other finished entries can
  stay without one.
- Don't touch anything else in finished entries.

### 6. AGENTS.md fill-ins

Add a section (or update the journal section) that says "work follows `WORKING_STRATEGY.md` (v2)" and fills
in, from what you know of the project:

- Resources: which ports, servers, databases and environments need a lock, and in which phases.
- Code areas for the Touches column.
- What a new worktree links in from the main checkout (dependencies such as `node_modules` or `.venv`,
  untracked config such as `.env`, large data).
- Orchestrator or user-started sessions (ask if unsure; default: orchestrator for parallel work).
- Stale-claim time if not 2 hours.
- Deviations from the strategy, each with its reason.

Replace the old journal rules in `AGENTS.md` with a pointer to the strategy plus these fill-ins; keep any
rule that is project-specific (measurement rules, verification commands).

### 7. Files and git

- Create `journal/runs/` (new runs only; old runs stay in the existing ledger).
- Add `.worktrees/` to `.gitignore`.
- For an append-only ledger, consider `merge=union` in `.gitattributes`.
- Commit everything as one migration commit on the integration branch, with the migration entry. No push
  unless the project allows it.

### 8. Check and report

- Re-read `AGENTS.md`'s read order as a fresh model would: can the next task be picked up from it alone?
- Check that every ready task has Resources and Touches filled in (or `?` with an ask).
- Report to the user in a few lines: what moved where, which tasks can now run in parallel, the open
  questions, and the commit.
