# CLAUDE.md — Delivery Workflow

This file defines **process only**: how work moves from a ticket to merged, documented code.
Technical conventions (stack, coding standards, test requirements) live in each repo's Spec Kit
constitution (`workspace/<repo>/.specify/memory/constitution.md`, managed via
`/speckit-constitution`). Do not duplicate them here.

---

## 1. Project

- **Project:** `<PROJECT_NAME>`
- **Jira project key:** `<JIRA_KEY>` (tickets look like `<JIRA_KEY>-123`)
- **Stack (summary):** `<STACK>` — full details in the constitution.
- **Default branch:** `<main>`
- **Design source:** Figma (via Figma MCP) when the ticket links a Figma file.

### Workspace layout

```
<root>/
├── CLAUDE.md              # process (this file)
├── .claude/commands/      # /ticket, /build, /fix
└── workspace/             # code repos, each cloned here with its own git
    ├── <repo-a>/
    │   ├── CLAUDE.md      # optional: repo-specific instructions (see below)
    │   ├── .specify/      # Spec Kit config + constitution for this repo
    │   ├── specs/         # specs for features of this repo
    │   └── ...code
    └── <repo-b>/
```

| Repo | Path | Type | Purpose |
|---|---|---|---|
| `<repo-a>` | `workspace/<repo-a>` | FE | `<e.g. web app>` |
| `<repo-b>` | `workspace/<repo-b>` | BE | `<e.g. REST API>` |

The **Type** column (FE / BE) decides which guardrails in §5 apply: "UI work" and
"UI verification" for FE, "Backend work" for BE.

- Add a repo: `git clone <url> workspace/<repo>`, then add a row above and its commands in §6.
  If it has no Spec Kit yet, run `specify init --here` inside it, then `/speckit-constitution`.
- `workspace/*` is git-ignored by this root folder. Every git operation (branch, commit, push)
  runs **inside the target repo**, never at the root.
- Each ticket is resolved to a target repo first. Spec Kit commands, specs, branches, and
  verify commands all run inside `workspace/<repo>/`.
- A ticket spanning several repos: one spec per repo (in each repo's `specs/`), each linking to
  the other(s) and the same ticket key. Build them in dependency order (e.g. API before UI).
- Fullstack ticket (BE + FE): the API contract is defined only on the BE side (§5 "Backend
  work"). The FE spec links to it instead of redefining endpoints or payloads. Build the BE
  spec first.
- Which API the FE uses while verifying (§5 "UI verification"):
  - Fullstack ticket → the FE dev server points to the **local BE** dev server running the BE
    feature branch. BE dev-server rules apply (§5 "Backend work").
  - FE-only ticket → the FE dev server uses its own configured environment (`.env` of the FE
    repo), even if that is a shared/staging API.

### Repo-level `CLAUDE.md`

A repo may have its own `workspace/<repo>/CLAUDE.md`. Claude Code only loads it lazily (when a
file in that repo is read), so it is **not** guaranteed to be in context. Therefore:

- As soon as the target repo is resolved, **read `workspace/<repo>/CLAUDE.md` if it exists**,
  before classifying deeper, writing specs, or touching code.
- Do **not** `@import` repo files here — that would load every repo's rules into every session.
- Pass the repo `CLAUDE.md` (with the constitution) to every subagent working in that repo.

**Precedence**

| Topic | Wins |
|---|---|
| Process: lanes, `tasks.md` as the only plan, forbidden commands, ticking, docs sync, stuck rule | This root `CLAUDE.md` |
| Safety: never applying migrations to a persistent DB, BE checks never calling staging/production, no DB mocks in integration tests, test credentials and secrets (§5 "Backend work", "Test credentials and secrets") | This root `CLAUDE.md` |
| Technical: conventions, architecture, commands, test setup, repo-specific do/don't | Repo `CLAUDE.md` (and its constitution) |

If the repo `CLAUDE.md` contradicts a **process** or **safety** rule here (e.g. tells Claude to
run `/speckit-implement`, write its own plan, or run the migrate command after creating a
migration), do not pick one silently: point out the conflict and ask the user.

---

## 2. Tools and sources of truth

| Concern | Tool | Source of truth |
|---|---|---|
| Ticket / requirements | Jira (Atlassian MCP) | Jira ticket |
| Spec, plan, task list | Spec Kit | `workspace/<repo>/specs/<NNN-feature>/spec.md`, `plan.md`, `tasks.md` |
| Execution (TDD, subagents, review) | Superpowers | `tasks.md` checkboxes |
| UI design | Figma MCP | Figma frame linked in the ticket/spec |
| UI verification | `agent-browser` skill + CLI | Screenshots of the running app |
| API contract | BE repo / Spec Kit (`/speckit-plan`) | The BE repo's own OpenAPI file if it has one; otherwise the BE spec's `contracts/` (§5) |
| BE verification | Repo test runner + dev server | Integration/contract test output, recorded endpoint calls |
| Feature documentation | Spec Kit spec | `spec.md` + its `Implementation notes` section |

**Hard rules**

- **Global rules:** in this project, this file and `.claude/commands/` override user-level
  rules (`~/.claude/CLAUDE.md`) and plugins on process. Do not use oh-my-claudecode (OMC)
  skills, modes, or agents (e.g. `plan`, `ralplan`, `ralph`, `autopilot`, `team`, `executor`);
  Superpowers drives execution. Plugin or built-in skills that overlap Superpowers (debugging,
  code review, test strategy, docs, design), e.g. `engineering:*`, `/code-review`, `/simplify`,
  are not part of this workflow: use the Superpowers skill named in the command instead. The
  overlapping `engineering:*` skills are denied in `.claude/settings.json`. Exception: the
  built-in `/security-review` is part of BE verification (§5 "Backend work"). The user's review
  of `spec.md`/`tasks.md` after `/ticket` is the plan approval for `/build`: subagents do not
  stop to ask for approval before each task.
- `tasks.md` is the **only** plan. Never create a second plan.
  - **Forbidden:** `/speckit-implement`, `superpowers:writing-plans`, `superpowers:executing-plans`.
  - Execution of `tasks.md` uses `superpowers:subagent-driven-development` only (via `/build`).
- `superpowers:brainstorming` is allowed **only in Lane 3**, before `/speckit-specify`, and only
  to shape the idea in conversation. Stop it once the idea is agreed: do not let it write its own
  design doc or hand off to `writing-plans`. Its outcome goes into `/speckit-specify` as input.
- Root `.claude/settings.json` denies `Skill(superpowers:writing-plans)` and
  `Skill(superpowers:executing-plans)`. Do not remove these blocks or work around them.
- One feature = one spec folder. Changing an existing feature **updates its existing spec**;
  never create a duplicate spec for it.
- Spec Kit names: Spec Kit ≥ 1.x installs Claude **skills** named `/speckit-<cmd>` (hyphen),
  e.g. `/speckit-specify`. Use the hyphen form; dotted names like `/speckit.specify` are not
  recognized as commands by Claude Code and are sent as plain text.
- Spec Kit skills not used in this workflow: `/speckit-implement` (forbidden),
  `/speckit-converge` (appends tasks for `implement`; use `/speckit-tasks` + `/build` instead),
  `/speckit-taskstoissues` (creates GitHub issues — only when the user explicitly asks).
  They are blocked mechanically:
  - Root `.claude/settings.json` denies `Skill(speckit-implement)`, `Skill(speckit-converge)`,
    `Skill(speckit-taskstoissues)` for Claude, including when they load on demand from a repo.
  - In each repo their `SKILL.md` has `disable-model-invocation: true`; `speckit-implement` also
    has `user-invocable: false` so `/speckit-implement` does nothing (README → Setup step 4).
  Do not remove these blocks or work around them (e.g. by reading the SKILL.md and following it).

### Spec Kit layout across repos

Claude Code started at this root does **not** discover `.claude/commands/` or `.claude/skills/`
inside `workspace/<repo>/` at startup (verified: they return `Unknown command`). So:

- The Spec Kit skills live **once** at the root: `.claude/skills/speckit-*/`, copied from a repo
  (excluding `speckit-implement`, `speckit-converge`, `speckit-taskstoissues`).
- Scripts, templates, constitution, and specs live **per repo**: `workspace/<repo>/.specify/`
  and `workspace/<repo>/specs/`.
- **Before running any `/speckit-*` skill, `cd workspace/<repo>`.** Spec Kit scripts locate the
  project by walking up from the current directory to the nearest `.specify/` (or use
  `SPECIFY_INIT_DIR=workspace/<repo>`). All relative paths in the skills (`.specify/...`,
  `specs/...`) and "project root" mean the target repo, never this root. Never create
  `.specify/` or `specs/` at this root.
- Spec Kit finds the current feature through `workspace/<repo>/.specify/feature.json`, which
  only `/speckit-specify` writes. Before running any other `/speckit-*` skill on an existing
  spec (Lane 2, `/build`, re-runs), write it to point at the target spec:
  `{ "feature_directory": "specs/<NNN-feature>" }`. Otherwise the skills act on whichever
  feature was specified last.
- Spec Kit core does **not** create git branches. `/build` creates the feature branch.
- Spec Kit extension hooks for git (`speckit.git.*` in `workspace/<repo>/.specify/extensions.yml`,
  e.g. a `before_specify` branch hook or auto-commit hooks) must stay disabled
  (`enabled: false`): branches come from `/build`, commits follow §5. If a `/speckit-*` skill
  announces a git hook, do not run it; tell the user to disable it (README → Setup step 4).
- When Spec Kit is upgraded, re-copy the skills to the root (README → Setup).

---

## 3. Workflow

### Step 1 — Intake (ticket → spec → code → docs)

1. Fetch the ticket from Jira (`<JIRA_KEY>-NNN`) via Atlassian MCP: summary, description,
   acceptance criteria (AC), attachments, linked Figma URLs, linked tickets.
2. Resolve the target repo(s) in `workspace/` (from the ticket's component/labels/description,
   or the code). If unclear, ask the user.
3. Classify the ticket into a lane (section 4). State the lane and the reason in one line.
4. Search `workspace/<repo>/specs/` for an existing spec covering the affected feature.
5. Continue with the lane's path:
   - Lane 0/1 → `/fix <KEY>`
   - Lane 2/3 → `/ticket <KEY>` (stops after analyze), then `/build workspace/<repo>/specs/<feature>`
6. After code is done: sync docs (Step 4).

### Step 2 — Spec (Lane 2/3 only, via `/ticket`)

Ends at `/speckit-analyze` with no CRITICAL findings. No code is written in this step.

### Step 3 — Build (via `/build`)

Executes unchecked tasks in `tasks.md` with subagents, TDD, and two-stage review.
`tasks.md` is ticked **immediately** after each task's review passes — never batched.

### Step 4 — Docs sync

Before finishing the branch, update `spec.md` so it describes what was actually built:

- Fix any requirement text that drifted from the implementation.
- Add/update an `## Implementation notes` section: key files/modules, decisions and
  deviations from `plan.md` (with reasons), known limitations, manual verification results.
- For Lane 2, the `## Change: <KEY>` section must reflect the final behavior.

The spec is the feature documentation. There is no separate doc to maintain.

---

## 4. Lanes

| Lane | What | Examples | Path |
|---|---|---|---|
| **0 — Trivial** | No behavior change | typo, copy text, comment, config value, dependency patch bump | `/fix`: edit → lint/build → done. No Spec Kit. |
| **1 — Small bug** | Wrong behavior, local cause, ≤ ~3 files, no new requirement | off-by-one, wrong condition, missing null check | `/fix`: reproduce → failing test → fix → verify. No Spec Kit. |
| **2 — Change to existing feature** | New/changed requirement on a feature that already has (or should have) a spec | add a filter to an existing list, change a validation rule | `/ticket`: append `## Change: <KEY>` to the existing spec → update plan/tasks → analyze → `/build`. |
| **3 — New feature** | New capability, new screen/module, cross-cutting change | new page, new integration | `/ticket`: (brainstorming) → specify → clarify → plan → tasks → analyze → `/build`. |

**Escalation:** if work in a lane turns out bigger than the lane allows (e.g. a "bug" needs a
new requirement or touches many modules), stop and re-classify. Tell the user before switching.
A fix that needs a database migration or changes an API contract is never Lane 0/1: it is
Lane 2 on the BE feature's spec. Only exception: a production hotfix (§5 "Hotfix").

**Existing feature without a spec (Lane 2):** create the spec for the feature once, describing
current behavior as-is (from code), then add the `## Change: <KEY>` section on top.

**Lane 2 tasks:** `/speckit-tasks` regenerates `tasks.md` from scratch, which would drop
ticked tasks and the `## Build log`. If the spec already has a `tasks.md`, do **not** run
`/speckit-tasks`; append the new tasks by hand under a `## Change: <KEY>` phase, continuing
the existing task IDs and following its checklist format. Run `/speckit-tasks` only when the
spec has no `tasks.md` yet.

### Lane 2 — `Change:` section format

```markdown
## Change: <KEY> — <short title>

- **Reason:** <why, from ticket>
- **Before:** <current behavior>
- **After:** <new behavior>
- **Acceptance criteria:** <list>
- **Affected requirements:** <FR-xxx ids updated/added>
- **Contract change (BE):** <endpoints changed, breaking yes/no, FE impact — or "none">
- **Migrations (BE):** <migrations to write, destructive yes/no — or "none">
```

Update the affected requirements in place as well; the `Change:` section is the changelog,
the requirement list is always the current truth.

---

## 5. Guardrails and special cases

### Task size
- One task = one verifiable outcome, ≤ ~5 files, completable by one subagent in one pass.
- If a task is too large, the implementer subagent splits it **inside `tasks.md`** as
  sub-tasks (`T012a`, `T012b`, …) under the parent, then executes them in order. The parent
  is ticked only when all sub-tasks are ticked. Do not create a separate plan for this.

### UI work and Figma
- If the ticket/spec links Figma:
  - Use Figma MCP (`get_design_context`, `get_variable_defs`, `get_screenshot`) as the
    design source. Map to existing components and tokens.
  - **Forbidden:** the `frontend-design` skill (it invents its own visual direction).
  - Verify: screenshot the implemented UI and compare against the Figma `get_screenshot`
    output (layout, spacing, typography, color, states). List any differences and fix them
    or record them as accepted deviations in `Implementation notes`.
- If there is **no** Figma:
  - Screen inside an existing product → follow the existing design system and components
    (`frontend-design-system`). Do not use `frontend-design`.
  - New visual direction needed (new product/area with no design system) → `frontend-design`
    is allowed, constrained by the constitution. Record the chosen direction in
    `Implementation notes`.
- Review every UI task with `web-design-guidelines` on the changed UI files. Fix its findings
  or record them as accepted deviations in `Implementation notes`.
- Do not use `ui-ux-pro-max` (it generates its own design system, competing with Figma and
  existing tokens).

### UI verification (`agent-browser`)
- `agent-browser` is the **only** tool for checking implemented UI: open pages, interact,
  check states, take screenshots. Do not use the built-in browser, Claude in Chrome,
  Playwright MCP, or other browser tools for this.
- Load its workflow first with `agent-browser skills get core`, as the skill says.
- Next.js 16.3+ repo on Turbopack: verify through the `next-dev-loop` skill, which drives
  `agent-browser` and cross-checks `/_next/mcp` (compile, runtime, and React-level errors).
  If the repo is below 16.3 or on webpack, use `agent-browser` directly; do not upgrade Next.js
  or switch bundlers unless the ticket covers it. Opening the page in the built-in browser only
  so the user can watch is allowed; the check itself stays in `agent-browser`.
- Run against the repo's dev server (§6). Check every state the task touches (loading, empty,
  error, success) and the responsive breakpoints in the spec.
- API behind the FE (§1 "Which API the FE uses"): fullstack ticket → local BE dev server on
  the BE feature branch, with the BE's migrations applied by the user; start it under the BE
  dev-server rules (§5 "Backend work") or ask the user to start it. FE-only ticket → the FE's
  configured environment. On a shared/staging API, use test accounts only (see "Test
  credentials and secrets") and ask the user before any action that creates, changes, or
  deletes shared data beyond what the AC check needs.
- With Figma: compare the `agent-browser` screenshot against Figma `get_screenshot`.
  Without Figma: check against the spec's AC and the existing design system.
- Scope is the local app under development only. Do not use its Slack, Electron, cloud, or
  other features unless the user asks.
- If the `agent-browser` CLI is not installed, stop and ask the user to install it
  (README → Prerequisites). Do not install it yourself or fall back to another browser tool.

### Backend work
Applies to repos of type BE (§1). Stack-specific commands and tools come from the repo
`CLAUDE.md`, its constitution, and §6.

**API contract**
- **Source of truth:**
  - BE repo already has an OpenAPI file (checked in, or generated from code, e.g. springdoc,
    NestJS Swagger) → that file is the contract. Record its path in the repo `CLAUDE.md` or
    §6. The spec's `contracts/` holds only a short note of the endpoints this feature adds or
    changes, linking to that file; it never copies the full schema.
  - BE repo has no OpenAPI → the contract is the OpenAPI file in the BE spec's `contracts/`
    (e.g. `contracts/openapi.yaml`), one per feature.
- `/speckit-plan` picks a contract format per project type, so when running it for a BE repo,
  ask explicitly for OpenAPI and tell it which of the two cases above applies.
- FE specs never define API contracts. When running `/speckit-plan` for an FE repo, its
  `contracts/` (if any) holds only UI/component contracts and links to the BE contract for
  every endpoint used.
- Changing an endpoint, payload, status code, or error shape means updating the contract (the
  repo's OpenAPI file, or regenerating it, or the spec's file) in the same task as the code.
- Breaking change (removed/renamed field, new required input, changed type or status code):
  tell the user before writing it, and record it in the spec's `## Change: <KEY>` section
  with the FE impact. If an FE repo consumes the endpoint, its spec needs a matching change.

**Database migrations**
- Claude **writes** migrations but never **applies** them to a persistent database: not the
  local dev DB, not staging, not production. Do not run the repo's migrate/upgrade/rollback
  commands against them.
- Exception: integration tests may apply migrations to a throwaway test database that the
  test run creates and destroys itself (e.g. Testcontainers or a test-only container).
  Before the first integration test run in a repo, check the test config: if it points to a
  fixed database (e.g. `localhost:5432/app_test`, a shared or remote DB) instead of one the
  run creates, stop and ask the user before running it.
- Auto-migration on startup: some stacks apply pending migrations when the app starts
  (e.g. Flyway/Liquibase in Spring Boot, Hibernate `ddl-auto`, framework auto-migrate
  settings). Check the repo config before starting its dev server. If it auto-migrates and a
  migration from this work is not yet applied by the user, do not start the dev server: hand
  over the `[manual]` apply step and wait, or ask the user to start the server themselves.
- Every migration is reversible (a down/rollback step) or, if it truly cannot be, says why in
  a comment and in `Implementation notes`. Destructive changes (drop column/table, data
  rewrite) need user confirmation before they are written.
- Each migration is its own task, followed by a `[manual]` task with the exact command for the
  user to apply it locally. Tasks that depend on the new schema in the dev DB stay after that.

**BE verification** (in addition to unit tests; apply each check by what the task touches)

| Task touches | Required checks |
|---|---|
| Database access, queries, migrations | 1 |
| An endpoint (new or changed) | 1, 2, 3 |
| Auth, input handling, queries, file/network access, secrets | 4, plus the rows above that apply |
| Other logic (services, jobs) without DB or endpoint | Unit tests; 1 if it has DB side effects |

A task's verification list in `tasks.md` names the checks it needs; reviewers check that list.

1. **Integration tests against a real database** in a throwaway container. Do not mock the
   database or the ORM in these tests. If the container runtime (e.g. Docker) is not
   installed or not running, stop and ask the user to start/install it (README →
   Prerequisites). Do not install it yourself, and do not fall back to mocks or skip the tests.
2. **Endpoint calls on the local dev server** (§6): call each endpoint the task touches for
   every AC case (success, validation error, unauthenticated/forbidden, not found, and any
   case the spec lists). Record request and response (status, relevant body) in the review.
   Never call staging or production. If the dev server needs a migration the user has not
   applied yet, or would auto-apply it on startup (see "Database migrations"), hand over the
   `[manual]` step and wait.
3. **Contract tests**: responses match the contract (the repo's OpenAPI file, or the spec's
   `contracts/` file when the repo has none). Use the repo's contract-test command in §6, or
   schema validation of the recorded responses if the repo has none.
4. **Security review**: run `/security-review` on tasks that touch authentication or
   authorization, input handling, database queries, file or network access, or secrets.
   Fix findings or record accepted risks in `Implementation notes`.

**Before the first test run** in a BE repo (baseline included, in `/build` and `/fix`):
check that the container runtime is available and that integration tests use a throwaway
database (see "Database migrations" and check 1 above). Do this before running the baseline,
not only before a task.

### Test credentials and secrets
Applies to endpoint calls (BE) and UI checks (FE).
- Use only test users, tokens, and API keys from the repo's seed, fixture, or example-config
  files, or ones the user gives for this purpose. Never real user accounts or production
  credentials.
- If a check needs credentials that don't exist (e.g. a role with no seeded user), ask the
  user; do not create accounts on a shared environment.
- Read `.env` files only to find which env/host the app uses. Never copy secrets into
  `tasks.md`, specs, review notes, commit messages, or chat; mask tokens in recorded
  requests (e.g. `Authorization: Bearer ***`).

### Ticket missing acceptance criteria
- Do not invent AC silently. Draft proposed AC from the description, mark them
  `[PROPOSED]`, and ask the user to confirm before `/speckit-specify` (Lane 2/3) or before
  fixing (Lane 1). Optionally post them as a Jira comment **only if the user asks**.

### Baseline has failing tests
- BE repo: run the "Before the first test run" checks (§5 "Backend work") first.
- Before building, run the test suite once and record failures as the **baseline** in the
  `## Build log` section of `tasks.md` (Lane 2/3) or in the fix summary (Lane 1).
- Baseline failures are not regressions and are not fixed unless the ticket covers them.
- A task passes only if it introduces **no new** failures compared to the baseline.

### Things that can't be tested automatically
- Mark the task `[manual]` in `tasks.md` and write concrete manual verification steps
  (preconditions, steps, expected result).
- Record the result in `Implementation notes`. If the agent cannot perform the check itself
  (e.g. needs a device, a third-party account), hand the steps to the user and leave the
  task unticked until the user confirms.

### Hotfix (production incident)
- Use Lane 1 via `/fix`, branch from the release/production branch per team rules.
- Minimum: reproduce, fix, regression test (unless truly impossible — then `[manual]` steps).
- A hotfix that needs a DB migration or a contract change stays on `/fix` (not Lane 2) to ship
  fast, with these limits: the user confirms the hotfix scope first; the migration is still
  written only, never applied by Claude, and its apply/rollback commands are handed to the
  user; breaking contract changes are flagged to the user. The follow-up docs sync adds a
  `## Change: <KEY>` section to the affected BE (and FE) spec.
- Docs sync may follow after the hotfix ships, but must be done in a follow-up commit/PR
  referencing the same ticket. Do not skip it.

### Agent stuck
- If the same task/test fails after **2** fix attempts (or the subagent reports BLOCKED
  twice), **stop**. Do not try a third variation.
- Write a short blocker note: what was tried, the error output, hypotheses. Then use
  `superpowers:systematic-debugging` once on the root cause, or ask the user.
- Never weaken, skip (`.skip`/`.only`), or delete tests to get green.

### General
- Never mark a task done without fresh evidence (test/lint/build output).
- Do not modify code outside the task's scope. Note unrelated issues instead of fixing them.
- Commit per task using Conventional Commits with the ticket key (via the `git-commit` skill):
  `<type>(<scope>): <KEY> <task id> <summary>`, e.g. `feat(orders): ABC-123 T012 add status filter`.
  Omit `<task id>` outside `/build`. Do not use the global `frontend-commit-messages` skill.

---

## 6. Commands

Run from inside the repo (`cd workspace/<repo>`). One table per repo. If a repo's own
`CLAUDE.md` already lists its commands, write `See workspace/<repo>/CLAUDE.md` instead of
duplicating them — the repo file is the source of truth.

### `<repo-a>` (FE)

| Purpose | Command |
|---|---|
| Install | `<install command>` |
| Lint | `<lint command>` |
| Type check | `<typecheck command>` |
| Unit tests | `<test command>` |
| Single test file | `<test command> <path>` |
| E2E tests | `<e2e command>` |
| Build | `<build command>` |
| Run dev server | `<dev command>` |

### `<repo-b>` (BE)

| Purpose | Command |
|---|---|
| Install | `<install command>` |
| Lint | `<lint command>` |
| Type check | `<typecheck command, if any>` |
| Unit tests | `<test command>` |
| Single test file | `<test command> <path>` |
| Integration tests (throwaway DB) | `<integration test command>` |
| Contract tests | `<contract test command, if any>` |
| OpenAPI file (contract) | `<path, or generate command; "none" if the repo has none>` |
| Build | `<build command>` |
| Run dev server | `<dev command>` |
| Create migration (write only) | `<migration create command>` |
| Apply migration (**user only**, §5) | `<migration apply command>` |

"Verify" in this file means: lint + type check + tests + build all pass in the target repo
(relative to baseline). For BE repos, tests include integration and contract tests.

---

## 7. Shortcuts (`.claude/commands/`)

- `/ticket <KEY>` — intake, classify lane, produce/update spec up to `/speckit-analyze`. No code.
- `/build <workspace/repo/specs/feature>` — execute `tasks.md` with Superpowers, tick tasks, sync docs.
- `/fix <KEY or description>` — Lane 0/1 quick fix without Spec Kit.
