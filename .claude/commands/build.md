---
description: Execute tasks.md for a spec with Superpowers (subagents, TDD, review), tick tasks, then sync docs
argument-hint: <workspace/repo/specs/NNN-feature>
---

Spec folder: $ARGUMENTS

Follow `CLAUDE.md` sections 2–6. `tasks.md` is the only plan.
**Forbidden:** `/speckit-implement`, `superpowers:writing-plans`, `superpowers:executing-plans`,
`superpowers:brainstorming`.

## 1. Preconditions
1. `$ARGUMENTS` is inside `workspace/<repo>/specs/`. The repo root is `workspace/<repo>`; all
   commands, git operations, and file edits happen inside it. Read `workspace/<repo>/CLAUDE.md`
   if it exists (precedence in `CLAUDE.md` §1).
2. `$ARGUMENTS/spec.md`, `plan.md`, `tasks.md` exist. If not, stop and point to `/ticket`.
3. Write `workspace/<repo>/.specify/feature.json` as
   `{ "feature_directory": "specs/<NNN-feature>" }` (this spec), then confirm `/speckit-analyze`
   (run from inside `workspace/<repo>`) has no CRITICAL findings (re-run it if spec/tasks changed since).
4. Work on a feature branch or worktree of that repo (`superpowers:using-git-worktrees`), never
   its default branch.

## 2. Baseline
Run the target repo's verify commands from `CLAUDE.md` §6 once. Record results (especially pre-existing
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
   - BE task → follow `CLAUDE.md` §5 "Backend work": update the `contracts/` OpenAPI file in
     the same task as the code; write migrations but never apply them to a persistent DB
     (integration tests may use a throwaway test DB); integration tests run against a real DB,
     not mocks.
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
   BE tasks also require, attached to the review: integration test output, recorded endpoint
   calls on the local dev server for every AC case (request + status + relevant body), contract
   test output against `contracts/`, and `/security-review` results when the task touches auth,
   input handling, queries, file/network access, or secrets.
3. **On pass** — immediately tick the task `[x]` in `tasks.md`, then commit:
   `<type>(<scope>): <KEY> <task id> <summary>` (use the `git-commit` skill). Never batch ticks.
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
Commit: `docs(<scope>): <KEY> sync spec with implementation`.

## 6. Finish
Use `superpowers:finishing-a-development-branch`. Report: tasks done/remaining, verify results vs
baseline, manual checks pending for the user, deviations, and any blockers.
