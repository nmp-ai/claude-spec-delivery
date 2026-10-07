# Claude Spec Delivery

A Claude Code workspace that takes Jira tickets to merged, documented code using
**Spec Kit** (spec → plan → tasks) and **Superpowers** (subagents, TDD, review).
It covers fullstack work: frontend (FE) and backend (BE) repos side by side, with the API
contract owned by the BE side.

- `CLAUDE.md` is the rulebook: lanes, workflow, guardrails, and per-repo commands.
- `.claude/commands/` adds three shortcuts: `/ticket`, `/build`, and `/fix`.
- `workspace/` holds the code repos (code only). Each repo keeps its own git history.
- `docs/` holds the Spec Kit artifacts per repo (constitution, specs, plans, tasks), committed
  in this root repo, so specs never end up in a code repo.

---

## Structure

```
.
├── CLAUDE.md                 # process rules (read by Claude Code automatically)
├── README.md
├── .claude/settings.json     # denies speckit-implement/converge/taskstoissues, superpowers writing/executing-plans, overlapping engineering:* skills, and global skills not shipped here
├── .claude/skills/speckit-*  # Spec Kit skills (copied from a repo, run inside the repo)
├── .claude/skills/git-commit # Conventional Commits skill (from github/awesome-copilot)
├── .claude/skills/agent-browser # UI verification skill (from vercel-labs/agent-browser)
├── .claude/skills/frontend-design # new visual direction, no Figma only (from anthropics/skills)
├── .claude/skills/web-design-guidelines # UI review (from vercel-labs/agent-skills)
├── .claude/skills/next-dev-loop # Next.js runtime verification via agent-browser (from vercel/next.js)
├── .claude/skills/vitest     # Vitest API reference (from antfu/skills)
├── .claude/skills/vercel-react-best-practices # React/Next performance rules (from vercel-labs/agent-skills)
├── .claude/skills/vercel-composition-patterns # React composition patterns (from vercel-labs/agent-skills)
├── .claude/skills/frontend-design-system # spacing, tokens, component reuse (team skill)
├── .claude/skills/api-design-principles # BE: REST/GraphQL API design reference (from wshobson/agents)
├── .claude/skills/supabase-postgres-best-practices # BE: Postgres schema/query/migration rules (from supabase/agent-skills)
├── .claude/commands/
│   ├── ticket.md             # /ticket <KEY>
│   ├── build.md              # /build <docs/repo/specs/feature>
│   └── fix.md                # /fix <KEY | description>
├── docs/                     # Spec Kit artifacts per code repo (committed here)
│   └── <repo>/
│       ├── .specify/         # Spec Kit scripts, templates, constitution
│       ├── .claude/          # skills created by `specify init` (git-ignored; copied to the root)
│       ├── specs/<NNN-feature>/{spec,plan,tasks}.md
│       └── specs/<NNN-feature>/contracts/  # BE: OpenAPI contract, or notes linking to the repo's own OpenAPI file
└── workspace/                # code repos, code only (git-ignored here)
    └── <repo>/
        ├── CLAUDE.md         # optional: repo-specific instructions
        └── ...code
```

---

## Prerequisites

| Tool | Purpose | Install |
|---|---|---|
| Claude Code | Runs everything | https://docs.claude.com/claude-code |
| Spec Kit (`specify` CLI) | Spec / plan / tasks | `uv tool install specify-cli --from git+https://github.com/github/spec-kit.git` |
| Superpowers plugin | Subagents, TDD, review skills | In Claude Code: `/plugin marketplace add obra/superpowers-marketplace`, then `/plugin install superpowers@superpowers-marketplace` |
| Atlassian MCP | Read Jira tickets | Connect Atlassian in Claude Code (`/mcp`) or in your claude.ai connectors |
| Figma MCP | Design context + screenshots (optional) | Connect Figma in Claude Code (`/mcp`) or in your claude.ai connectors |
| `agent-browser` CLI | UI verification (used by the `agent-browser` and `next-dev-loop` skills; `next-dev-loop` needs ≥ 0.31.1) | `npm i -g agent-browser && agent-browser install` |
| Docker (or the container runtime your BE tests use) | BE integration tests against a throwaway database | https://docs.docker.com/get-docker/ |

> Install commands change over time. Check each tool's own docs if one fails.

---

## Setup (once per project)

1. **Copy the template** to a new folder and open it in Claude Code.
2. **Fill `CLAUDE.md` §1**: project name, Jira key, stack summary, default branch.
3. **Add each code repo:**
   ```bash
   git clone <repo-url> workspace/<repo>
   ```
   - Add a row for it to the repo table in `CLAUDE.md` §1, with its type (FE or BE).
   - Add its lint, typecheck, test, and build commands to `CLAUDE.md` §6. If the repo already
     has its own `CLAUDE.md` listing them, just point to it. BE repos also need integration
     test, contract test, dev server, and migration create/apply commands.
   - A repo `CLAUDE.md` is read automatically once a ticket targets that repo. The repo file
     wins on technical rules and the root file wins on process; conflicts are flagged to you
     (`CLAUDE.md` §1). Don't `@import` repo files into the root `CLAUDE.md`.
4. **Initialize Spec Kit for each repo in `docs/<repo>`** (never inside `workspace/<repo>`):
   ```bash
   mkdir -p docs/<repo> && cd docs/<repo> && specify init --here --force --non-interactive --integration claude
   ```
   Use the same `<repo>` name as the folder in `workspace/`. (Spec Kit ≥ 1.x uses
   `--integration claude`; older versions used `--ai claude`.) If a code repo already has its
   own `.specify/` or `specs/` from before, move them into `docs/<repo>/` and remove them from
   the code repo in a normal code commit.

   Then block the Spec Kit skills this workflow forbids. Run this in `docs/<repo>`, and re-run
   it after every Spec Kit upgrade there:
   ```bash
   sed -i '' -e 's/^disable-model-invocation: false$/disable-model-invocation: true/' -e 's/^user-invocable: true$/user-invocable: false/' .claude/skills/speckit-implement/SKILL.md && for s in speckit-converge speckit-taskstoissues; do sed -i '' 's/^disable-model-invocation: false$/disable-model-invocation: true/' .claude/skills/$s/SKILL.md; done
   ```
   (`sed -i ''` is macOS syntax; on Linux use `sed -i`.) After this, Claude can't call
   `speckit-implement` and typing `/speckit-implement` does nothing. `speckit-converge` and
   `speckit-taskstoissues` stay available only when you type them yourself. The root
   `.claude/settings.json` also denies all three for Claude, which covers folders you haven't
   patched yet.
   If `.specify/extensions.yml` exists, set `enabled: false` on every `speckit.git.*` hook
   (branch creation, auto-commit). `/build` creates the branch and commits follow `CLAUDE.md` §5.
   Then run `/speckit-constitution` in Claude Code to record that repo's stack, coding
   standards, and test requirements. Technical rules go in the constitution, not in `CLAUDE.md`.
5. **Spec Kit skills at the root.** Claude Code started at the root does not see skills inside
   `docs/<repo>/.claude/`. The template already ships them in `.claude/skills/speckit-*`
   (Spec Kit 1.1.1). After upgrading Spec Kit, re-copy them from any initialized docs folder,
   leaving out the ones this workflow doesn't use:
   ```bash
   for d in docs/<repo>/.claude/skills/speckit-*; do case "$(basename "$d")" in speckit-implement|speckit-converge|speckit-taskstoissues) ;; *) rm -rf ".claude/skills/$(basename "$d")" && cp -R "$d" .claude/skills/ ;; esac; done
   ```
   Only the skills are copied. Scripts, templates, and the constitution stay in
   `docs/<repo>/.specify/`, and Claude `cd`s into `docs/<repo>` before running them.

   The template also ships the `git-commit` skill from
   [github/awesome-copilot](https://www.skills.sh/github/awesome-copilot/git-commit), used for
   every commit. To update it:
   ```bash
   curl -sfL https://raw.githubusercontent.com/github/awesome-copilot/main/skills/git-commit/SKILL.md -o .claude/skills/git-commit/SKILL.md
   ```
   Read the new version before keeping it. The other third-party skills are updated the same
   way from their sources:

   | Skill | Source file(s) |
   |---|---|
   | `agent-browser` | `vercel-labs/agent-browser` → `skills/agent-browser/SKILL.md` |
   | `frontend-design` | `anthropics/skills` → `skills/frontend-design/SKILL.md`, `LICENSE.txt` |
   | `web-design-guidelines` | `vercel-labs/agent-skills` → `skills/web-design-guidelines/SKILL.md` |
   | `next-dev-loop` | `vercel/next.js` (branch `canary`) → `skills/next-dev-loop/SKILL.md` |
   | `vitest` | `antfu/skills` → `skills/vitest/` (`SKILL.md`, `GENERATION.md`, `references/*.md`) |
   | `vercel-react-best-practices` | `vercel-labs/agent-skills` → `skills/react-best-practices/` (whole folder) |
   | `vercel-composition-patterns` | `vercel-labs/agent-skills` → `skills/composition-patterns/` (whole folder) |
   | `api-design-principles` | `wshobson/agents` → `plugins/backend-development/skills/api-design-principles/` (whole folder), root `LICENSE` → `LICENSE.txt` |
   | `supabase-postgres-best-practices` | `supabase/agent-skills` → `skills/supabase-postgres-best-practices/` (whole folder), root `LICENSE` → `LICENSE.txt` |

   `frontend-design-system` is maintained in this template; edit it here.

   **Global skills.** The project uses only the skills in `.claude/skills/`.
   `.claude/settings.json` denies every other skill from `~/.claude/skills/` by name. When you
   install a new global skill, add `Skill(<name>)` and `Skill(<name> *)` there too. A global
   skill with the **same name** as one shipped here can't be denied without also blocking the
   template copy, and Claude Code prefers the global copy. Remove or rename the global folder
   if you want the template version on your machine.

   **Global rules and plugins.** `CLAUDE.md` §2 tells Claude that this template's process
   overrides your `~/.claude/CLAUDE.md` and plugins: no oh-my-claudecode (OMC) skills, modes, or
   agents, and no per-task approval prompts during `/build` (your review after `/ticket` is the
   approval). Plugin skills that overlap Superpowers (`engineering:code-review`, `debug`,
   `testing-strategy`, `documentation`, `architecture`, `system-design`) are denied in
   `.claude/settings.json`; add others the same way if a new plugin competes. If a global rule
   still gets in the way, disable that plugin for this project.
6. **Check the Spec Kit names.** Spec Kit 1.x uses `/speckit-specify`, `/speckit-plan`, and so on
   (hyphen). Dotted names (`/speckit.specify`) don't work as commands in Claude Code. If your
   version differs, update `CLAUDE.md` and `.claude/commands/`.
7. **Smoke test:** run `/ticket <KEY>` on a small, real ticket and check that it picks the right
   lane and repo.

---

## Daily usage

### 1. Pick the path

| Lane | Ticket type | Commands |
|---|---|---|
| 0 | Typo, copy, config, no behavior change | `/fix <KEY>` |
| 1 | Small, local bug | `/fix <KEY>` |
| 2 | Change to an existing feature | `/ticket <KEY>` → `/build <spec-path>` |
| 3 | New feature | `/ticket <KEY>` → `/build <spec-path>` |

If you're not sure, start with `/ticket <KEY>`. It classifies the ticket and tells you to use
`/fix` if the ticket is Lane 0 or 1.

### 2. `/ticket <KEY>`: spec only, no code

```
/ticket MRP-123
```

1. Reads the Jira ticket, works out the target repo, and picks a lane.
2. If acceptance criteria are missing, it drafts them marked `[PROPOSED]` and waits for you to confirm.
3. Builds the spec, depending on the lane:
   - **Lane 2:** points Spec Kit at the feature's **existing** spec (`.specify/feature.json`),
     appends `## Change: MRP-123` to it, and updates the plan. New tasks are appended by hand
     to `tasks.md` under a `## Change: MRP-123` phase. `/speckit-tasks` is not re-run, because
     it would regenerate the file and drop ticked tasks and the build log.
   - **Lane 3:** runs brainstorming (optional), then specify → clarify → plan → tasks.
4. Runs `/speckit-analyze` until it reports no critical issues, then stops and prints the next command.

**Review the spec and `tasks.md` before building.** This is the cheapest point to change direction.

### 3. `/build <spec-path>`: implement

```
/build docs/web-app/specs/012-order-filter
```

1. Creates a feature branch or worktree in the code repo (`workspace/web-app`).
2. Records which tests already fail (the baseline) in `tasks.md` → `## Build log`.
3. For each task:
   1. A subagent implements it with TDD.
   2. It gets a spec-compliance review, then a code-quality review.
   3. Once both pass, the task is ticked `[x]` and committed.
4. Runs the repo's full lint, typecheck, test, and build commands, and compares the results to the baseline.
5. Syncs the docs by updating `spec.md` and its `## Implementation notes`.
6. Finishes the branch (Superpowers asks how: merge, PR, or keep it).

You can stop at any point. Running `/build` again continues from the first unticked task.

### 4. `/fix <KEY | description>`: quick fix

```
/fix MRP-456
/fix "Total price shows NaN when cart is empty"
```

1. Reproduces the bug and writes a failing test.
2. Makes a minimal fix and verifies it.
3. Commits.

If the fix turns out bigger than Lane 1, it stops and suggests `/ticket` instead.

---

## Rules worth knowing

- **One plan only:** `tasks.md`. Don't run `/speckit-implement` or Superpowers `writing-plans`
  (both are denied in `.claude/settings.json`). Lane 3 brainstorming stops once the idea is
  agreed, without writing its own design doc.
- **One spec per feature.** Changes to an existing feature update its spec with a `Change:` section.
- **Current feature:** Spec Kit picks the feature from `docs/<repo>/.specify/feature.json`,
  which only `/speckit-specify` writes. `/ticket` (Lane 2) and `/build` set it to the target
  spec first. Do the same if you run a `/speckit-*` skill yourself on an existing spec.
- **Lane 2 tasks:** never re-run `/speckit-tasks` on a spec that already has `tasks.md`; new
  tasks are appended by hand.
- **This template wins over global rules:** your `~/.claude/CLAUDE.md` and plugins (e.g. OMC)
  don't change the process here.
- **Specs are the docs.** They are synced with the code after every build.
- **Figma:**
  - If a ticket links Figma, Claude uses Figma MCP and compares screenshots to verify the UI.
  - Without Figma, Claude follows the existing design system. `frontend-design` is used only
    when there's no Figma and a new visual direction is needed.
  - Every UI task is reviewed with `web-design-guidelines`. `ui-ux-pro-max` is not used.
- **UI verification:** Claude checks UI in the running app with `agent-browser` only, compared
  against Figma when linked, otherwise against the AC. The skill
  ([vercel-labs/agent-browser](https://www.skills.sh/vercel-labs/agent-browser/agent-browser))
  is a stub that loads its guide from the CLI, so the CLI must be installed. On Next.js 16.3+
  with Turbopack, Claude verifies through `next-dev-loop`, which also reads `/_next/mcp`.
- **Backend:**
  - The API contract is the BE repo's own OpenAPI file if it has one; otherwise the OpenAPI
    file in the BE spec's `contracts/`. It changes in the same task as the code. Breaking
    changes are flagged to you first.
  - The checks for a throwaway test DB and Docker run before the baseline, not after.
  - Claude writes migrations but never applies them to a real database (dev, staging, prod).
    Applying is a `[manual]` task for you. Integration tests may use a throwaway test DB.
  - It checks that integration tests use a throwaway DB, and doesn't start a dev server that
    would auto-apply a migration you haven't applied. If Docker isn't running, it asks you.
  - BE tasks are verified by what they touch (`CLAUDE.md` §5 table): integration tests on a
    real DB, endpoint calls on the local dev server for each AC case, contract tests, and
    `/security-review` for auth, input, queries, or secrets.
  - A fix that needs a migration or a contract change is Lane 2, not `/fix`, except a
    production hotfix.
  - These safety rules win over a repo's own `CLAUDE.md`; conflicts are flagged to you.
- **Which API the FE uses:** a fullstack ticket points the FE dev server at the local BE on the
  feature branch. An FE-only ticket uses the FE's own configured env (which may be staging);
  there Claude uses test accounts only and asks before changing shared data.
- **Credentials:** only test users/tokens from seed or fixture files (or ones you give).
  Secrets are never written to specs, `tasks.md`, reviews, commits, or chat.
- **Stuck after 2 attempts:** Claude stops, writes a blocker note, and asks you. It doesn't keep guessing.
- **`[manual]` tasks:** tasks that can't be tested automatically have written steps. They stay unticked until verified.
- **Commits:** Claude uses the `git-commit` skill. The format is Conventional Commits with the
  ticket key: `<type>(<scope>): <KEY> <task id> <summary>`, e.g.
  `feat(orders): ABC-123 T012 add status filter`. `/fix` leaves out `<task id>`. The global
  `frontend-commit-messages` skill is denied in this project.
- **Hotfix:** use `/fix` on the release branch. The docs sync can come in a follow-up, but it can't be skipped.

The full rules are in [`CLAUDE.md`](CLAUDE.md).

---

## Multi-repo tickets

If a ticket touches more than one repo (for example, the API and the web app):

1. `/ticket` creates **one spec per repo**, cross-linked with the same ticket key.
2. The API contract lives only on the BE side (repo OpenAPI file or the BE spec's
   `contracts/`). The FE spec links to it instead of redefining endpoints.
3. Run `/build` for each spec in dependency order: BE first, then FE. If the BE build wrote a
   migration, apply it (the `[manual]` task) before building the FE.
4. While building the FE, its dev server points to the local BE dev server on the BE feature
   branch, not to staging.

---

## FAQ

**A repo has no spec for the feature I'm changing.**
`/ticket` writes a spec of the current behavior from the code once, then adds the `Change:` section.

**The tests already fail before I start.**
That's expected. They're recorded as the baseline. A task only fails if it adds *new* failures.

**I want to edit a task before building.**
Edit `tasks.md` directly. Then, inside `docs/<repo>`, make sure `.specify/feature.json`
points at that spec (`{ "feature_directory": "specs/<NNN-feature>" }`) and re-run
`/speckit-analyze`.

**Where do I commit?**
- **Code:** inside `workspace/<repo>`. Everything under `workspace/` is git-ignored at the root.
- **Specs and docs:** at the root, only `docs/<repo>/...` (e.g. `docs(web-app): ABC-123 add spec`).
  `/ticket`, `/build`, and `/fix` do this for you.
- **Template changes** (`CLAUDE.md`, commands, README): at the root, in their own commits.

The message format is under [Rules worth knowing](#rules-worth-knowing).
