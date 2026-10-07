---
description: Quick Lane 0/1 fix (typo, config, small bug) without Spec Kit
argument-hint: <JIRA_KEY-NNN | short description>
---

Input: $ARGUMENTS

Follow `CLAUDE.md` sections 4–6. No Spec Kit, no plan files.

## 1. Intake
- If the input is a Jira key, fetch the ticket via Atlassian MCP (summary, description, AC,
  steps to reproduce, attachments). Otherwise treat the input as the description.
- Resolve the target repo under `workspace/` (`CLAUDE.md` §1); ask if unclear. All edits,
  commands, and git operations run inside `workspace/<repo>/`.
- Read `workspace/<repo>/CLAUDE.md` if it exists (precedence in `CLAUDE.md` §1).
- If expected behavior is unclear or AC are missing, propose them marked `[PROPOSED]` and ask
  the user to confirm before changing behavior.

## 2. Confirm lane
- **Lane 0** (no behavior change) or **Lane 1** (local bug, ≤ ~3 files, no new requirement) → continue.
- Anything bigger (new requirement, many modules, design change, a database migration, or an
  API contract change) → stop and recommend
  `/ticket <KEY>`. Tell the user why. Exception: a production hotfix that needs a migration or
  contract change stays here under the limits in `CLAUDE.md` §5 "Hotfix".

## 3. Baseline (Lane 1)
BE repo: first run the "Before the first test run" checks (`CLAUDE.md` §5 "Backend work");
stop and ask if either fails. Then run the relevant tests once and note pre-existing failures.
They are not regressions.

## 4. Fix
**Lane 0:** make the edit → run lint/typecheck/build from `CLAUDE.md` §6.

**Lane 1:**
1. `superpowers:systematic-debugging` — reproduce and find the root cause (not the symptom).
2. Write a failing regression test that reproduces the bug (`superpowers:test-driven-development`).
   If it truly cannot be automated, write `[manual]` reproduction/verification steps instead.
3. Minimal fix within scope. Do not refactor unrelated code.
4. Verify: the new test passes, and no new failures versus baseline (§6 commands).
   BE repo: also run the checks from the `CLAUDE.md` §5 "BE verification" table that match
   what the fix touches (integration tests, calling the fixed endpoint on the local dev
   server, contract test, `/security-review`), with its migration and dev-server limits.

After **2** failed fix attempts, stop: report what was tried, the errors, and hypotheses, and ask the user.
Never skip/weaken tests to get green.

## 5. Spec impact
- If the fix changes behavior documented in an existing `workspace/<repo>/specs/*/spec.md`, update that requirement
  and add a short `## Change: <KEY>` entry. Otherwise no spec edit.
- Hotfix: if docs sync is deferred, say so explicitly in the report so it is done in a follow-up.

## 6. Commit and report
Commit (use the `git-commit` skill): `<type>(<scope>): <KEY> <summary>`, e.g. `fix(cart): ABC-45 handle empty items`
(or `<type>(<scope>): <summary>` without a key).
Report: lane, root cause, files changed, test evidence, baseline failures (if any),
manual steps pending, spec updated or not.
