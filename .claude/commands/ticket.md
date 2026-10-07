---
description: Intake a Jira ticket, classify its lane, and produce/update the spec up to /speckit-analyze (no code)
argument-hint: <JIRA_KEY-NNN>
---

Ticket: $ARGUMENTS

Follow `CLAUDE.md` sections 2–5. Do **not** write production code in this command.

## 1. Intake
1. Fetch the ticket via Atlassian MCP: summary, description, acceptance criteria, attachments,
   linked Figma URLs, linked tickets. If `$ARGUMENTS` is empty or not a valid key, ask for it.
2. If acceptance criteria are missing or vague: draft them marked `[PROPOSED]` and ask the user
   to confirm before continuing.
3. Resolve the target repo(s) under `workspace/` (`CLAUDE.md` §1). If unclear, ask the user.
   Then read `workspace/<repo>/CLAUDE.md` if it exists (precedence in `CLAUDE.md` §1).
   All following steps run inside `workspace/<repo>/`. Multi-repo ticket: repeat steps 3–4 per
   repo, one spec each, cross-linked.
4. Search `workspace/<repo>/specs/` for an existing spec covering the affected feature (by name,
   FR ids, routes, modules touched).

## 2. Classify
State the lane (0/1/2/3) and a one-line reason.
- **Lane 0 or 1:** stop here and tell the user to run `/fix $ARGUMENTS`.
- **Lane 2 or 3:** continue.

## 3. Spec
`cd workspace/<repo>` before any `/speckit-*` command so its scripts and paths resolve inside
the target repo (`CLAUDE.md` §2 "Spec Kit layout across repos").
### Lane 2 — change to existing feature
1. If no spec exists for the feature, create it once with `/speckit-specify`, giving it a
   description of the **current** behavior read from the code (not the ticket's change). This
   creates `specs/<NNN-feature>/` and points `.specify/feature.json` at it. Do not run any git
   hook it announces (`CLAUDE.md` §2). Then continue with step 2 on that spec.
   If it exists, write `.specify/feature.json` as `{ "feature_directory": "specs/<NNN-feature>" }`
   so the `/speckit-*` skills below act on this spec, not the last one specified.
2. Append `## Change: <KEY> — <title>` to the existing `spec.md` (format in `CLAUDE.md` §4) and
   update the affected requirements in place. Do **not** create a new spec folder.
3. Run `/speckit-clarify` if anything is ambiguous.
4. Update `plan.md` for the change (BE: and the contract notes in `contracts/`, same rules as
   Lane 3 step 4). Fill the `Contract change` and `Migrations` lines of the `Change:` section.
   Then:
   - `tasks.md` exists → do **not** run `/speckit-tasks` (it regenerates the file and drops
     ticked tasks and the `## Build log`). Append the new tasks by hand under a
     `## Change: <KEY>` phase, continuing the existing task IDs, same checklist format,
     tagged with the ticket key.
   - No `tasks.md` yet → run `/speckit-tasks`.

### Lane 3 — new feature
1. Optional: `superpowers:brainstorming` to shape the idea (only here, before specify). Stop once
   the idea is agreed: no design doc, no hand-off to `writing-plans` (`CLAUDE.md` §2). Feed the
   outcome into step 2.
2. `/speckit-specify` with the ticket content, AC, and Figma links.
3. `/speckit-clarify`.
4. `/speckit-plan`. BE repo: design the contract with `api-design-principles` (and
   `supabase-postgres-best-practices` for the data model on Postgres), and ask for the API
   contract as OpenAPI. If the repo already has an
   OpenAPI file, `contracts/` only notes the endpoints added/changed and links to it;
   otherwise `contracts/` holds the OpenAPI file (`CLAUDE.md` §5 "Backend work").
   FE repo of a fullstack ticket: `contracts/` must not define API endpoints; link to the BE
   contract instead.
5. `/speckit-tasks`.

### Both lanes — task list rules
- Every task: one verifiable outcome, ≤ ~5 files (`CLAUDE.md` §5).
- UI tasks include "verify with agent-browser". If the ticket links Figma, reference the exact
  Figma node URL and verify against its screenshot; otherwise verify against the AC and the
  existing design system (`CLAUDE.md` §5).
- BE tasks (`CLAUDE.md` §5 "Backend work"):
  - Contract first: a task that creates/updates the contract (repo OpenAPI file, or the spec's
    `contracts/` file if the repo has none) comes before the endpoint tasks that implement it.
    Lane 2 on a repo with no OpenAPI and a spec without `contracts/`: the first task documents
    the current endpoints as-is, then a second applies the change.
  - Each migration is its own task, followed by a `[manual]` task with the exact apply
    command for the user. Tasks that need the new schema on the dev server come after it.
  - Each BE task lists the checks it needs, per the table in `CLAUDE.md` §5 "BE verification"
    (e.g. endpoint task: integration test, endpoint calls per AC case, contract test; plus
    `/security-review` when it touches auth, input, queries, or secrets).
  - Breaking contract changes are flagged to the user before the tasks are written.
- Fullstack ticket: the FE spec links to the BE contract for every endpoint it uses and does
  not redefine payloads. Its API-dependent tasks note the BE spec they depend on and that UI
  checks run against the local BE dev server (`CLAUDE.md` §1).
- Non-automatable checks are separate tasks marked `[manual]` with concrete steps.
- Add an empty `## Build log` section at the end of `tasks.md` if missing.

## 4. Analyze
Run `/speckit-analyze`. Fix CRITICAL/HIGH findings in spec/plan/tasks and re-run until none remain.

## 5. Report and stop
Output:
- Lane and reason
- Target repo(s) and spec path(s) (`workspace/<repo>/specs/<feature>/`), created or updated
- AC status (confirmed / proposed-pending)
- Task count, `[manual]` tasks, open questions
- Breaking contract changes and migrations (BE), if any
- Next step: `/build workspace/<repo>/specs/<feature>` (in dependency order if multi-repo: BE first)

Do **not** run `/speckit-implement`, `writing-plans`, or start building.
