---
description: Execute tasks.md for a spec with Superpowers (subagents, TDD, review), tick tasks, then sync docs
argument-hint: <docs/repo/specs/NNN-feature>
---

Spec folder: $ARGUMENTS

Follow `CLAUDE.md` sections 2–6. `tasks.md` is the only plan.
**Forbidden:** `/speckit-implement`, `superpowers:writing-plans`, `superpowers:executing-plans`,
`superpowers:brainstorming`.

## 1. Preconditions
1. `$ARGUMENTS` is inside `docs/<repo>/specs/`. The code repo is `workspace/<repo>` (same
   `<repo>` name): code edits, verify commands, and code git operations happen inside it. Spec
   Kit and `tasks.md`/`spec.md` edits happen in `docs/<repo>` (`CLAUDE.md` §1). Read
   `workspace/<repo>/CLAUDE.md` if it exists (precedence in `CLAUDE.md` §1).
2. `$ARGUMENTS/spec.md`, `plan.md`, `tasks.md` exist. If not, stop and point to `/ticket`.
3. Write `docs/<repo>/.specify/feature.json` as
   `{ "feature_directory": "specs/<NNN-feature>" }` (this spec), then confirm `/speckit-analyze`
   (run from inside `docs/<repo>`) has no CRITICAL findings (re-run it if spec/tasks changed since).
4. Work on a feature branch or worktree of the code repo `workspace/<repo>`
   (`superpowers:using-git-worktrees`), never its default branch. Docs stay in `docs/<repo>`
   at the root; do not copy them into the branch or worktree.

## 2. Baseline
BE repo: first run the "Before the first test run" checks (`CLAUDE.md` §5 "Backend work":
container runtime available, integration tests use a throwaway DB). Stop and ask if either fails.
Then run the target repo's verify commands from `CLAUDE.md` §6 once. Record results (especially pre-existing
failures) under `## Build log` → `Baseline` in `tasks.md`. Baseline failures are not regressions.

## 3. Execute tasks
Use `superpowers:subagent-driven-development`, taking tasks **in order from `tasks.md`**
(respecting `[P]` parallel markers and dependencies). For each unchecked task:

1. **Implement** — dispatch a fresh implementer subagent with: the task text, relevant spec/plan
   sections, the constitution, the repo `CLAUDE.md` (if any), and the rules below. It follows `superpowers:test-driven-development`.
   - Task too large → split it in `tasks.md` into `T0XXa`, `T0XXb`… under the parent, then do them in order.
   - UI task with Figma → use Figma MCP (`get_design_context`, `get_variable_defs`,
     `get_screenshot`); do **not** use `frontend-design`. Without Figma, follow the existing design
     system; `frontend-design` only when a new visual direction is needed (`CLAUDE.md` §5).
   - UI task → check the result in the running app with the `agent-browser` skill only
     (`next-dev-loop` on Next.js 16.3+ with Turbopack; `CLAUDE.md` §5 "UI verification").
   - UI task in a fullstack ticket → the FE dev server points to the local BE dev server on
     the BE feature branch (`CLAUDE.md` §1, §5 "UI verification"). FE-only ticket → the FE's
     configured environment. Credentials per `CLAUDE.md` §5 "Test credentials and secrets".
   - BE task → follow `CLAUDE.md` §5 "Backend work": update the contract (repo OpenAPI file,
     or the spec's `contracts/` file if the repo has none) in the same task as the code; write
     migrations but never apply them to a persistent DB (integration tests may use a throwaway
     test DB); integration tests run against a real DB, not mocks; do not start a dev server
     that auto-applies a migration the user has not applied yet. Use `api-design-principles`
     for endpoint/contract work and `supabase-postgres-best-practices` for schema, migration,
     and query work on Postgres (`CLAUDE.md` §5 "Backend work" → Skills).
   - Migration apply task (`[manual]`) → hand the exact command to the user and wait for
     confirmation before tasks that need the new schema on the dev server.
   - `[manual]` task → perform the steps if possible and record the result; otherwise hand the
     steps to the user and leave it unticked.
2. **Review** — spec-compliance review, then code-quality review (`superpowers:requesting-code-review`).
   Reviewer subagents also get the constitution and the repo `CLAUDE.md`.
   UI tasks also require an `agent-browser` check of the running app: screenshots of every
   affected state, compared against the Figma frame when the task links one, otherwise against
   the AC. Attach the result to the review. The code-quality review of UI tasks also runs
   `web-design-guidelines` on the changed UI files.
   BE tasks also require, attached to the review, the evidence for each check the task lists
   (`CLAUDE.md` §5 "BE verification" table): integration test output, recorded endpoint calls
   on the local dev server for every AC case (request + status + relevant body, tokens masked),
   contract test output against the contract, and `/security-review` results. The code-quality
   review of BE tasks also checks endpoints against `api-design-principles` and schema/query
   changes against `supabase-postgres-best-practices` (Postgres repos).
3. **On pass** — immediately tick the task `[x]` in `docs/<repo>/specs/<feature>/tasks.md`, then
   commit the code in `workspace/<repo>`: `<type>(<scope>): <KEY> <task id> <summary>` (use the
   `git-commit` skill). The code commit never includes `docs/` files. Never batch ticks.
4. **On failure** — fix and re-review. After **2** failed attempts on the same task, stop: write a
   blocker note in `## Build log` (what was tried, errors, hypotheses), run
   `superpowers:systematic-debugging` once, or ask the user. Never skip/weaken tests.

A task passes only with no new failures versus baseline.

## 4. Final verification
`superpowers:verification-before-completion`: run all §6 verify commands and compare to baseline.
Confirm every acceptance criterion in `spec.md` maps to a ticked task with evidence.

## 5. Docs sync
Update `$ARGUMENTS/spec.md`:
- Correct any requirement text that drifted from what was built.
- Add/update `## Implementation notes`: key files/modules, deviations from `plan.md` with reasons,
  accepted Figma deviations, known limitations, manual verification results.
- BE: list migrations written (and whether the user applied them), contract changes (breaking
  or not), and accepted security-review risks.
- Lane 2: make sure `## Change: <KEY>` matches the final behavior.
Commit in the **root** repo, staging only `docs/<repo>/`:
`docs(<repo>): <KEY> sync spec with implementation` (`CLAUDE.md` §5 "General").
If `/build` stops early (blocker, user stops), commit the `docs/<repo>/` changes the same way
(`docs(<repo>): <KEY> update tasks progress`) so ticks and the build log are not lost.

## 6. Finish
Use `superpowers:finishing-a-development-branch`. Report: tasks done/remaining, verify results vs
baseline, manual checks pending for the user, deviations, and any blockers.
