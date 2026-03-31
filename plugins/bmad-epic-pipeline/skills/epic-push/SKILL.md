---
name: epic-push
description: 'Quality gate for a BMAD epic. Runs code simplification, creates a PR, runs code review (confidence threshold 40), implements all flagged fixes, and pushes. Use when the user says "epic push N", "epic ship N", or wants to finalize and PR an epic. Stage 3 of the epic pipeline: epic-stories → epic-dev → epic-push → epic-feedback.'
---

# Epic Push

Quality gate: simplify, PR, review, fix, and push.

**Invocation:** `epic push N`, `push epic N`, `epic ship N`, or `ship epic N`

Read and follow [references/workflow.md](references/workflow.md) step by step.
If the relative path does not resolve, glob for `**/epic-push/references/workflow.md` and read the first match.

## Critical Rules

- **DO NOT** wait for user approval after presenting the pre-flight results — show them briefly, then execute immediately.
- **ALWAYS** create a task list at the start showing all phases, and update it as you progress.
- **ALWAYS** verify all tests pass before creating the PR.
- **ALWAYS** lint and type-check the **entire codebase** before pushing — not just changed files. Fix all errors, including pre-existing ones. Zero lint/type errors is a hard gate.
- **NEVER** push code that has lint or type errors — this is as important as passing tests.
- **NEVER** skip the code review phase — it is the final quality gate.
- **NEVER** merge the PR automatically — the human does the final merge.
- **PR target defaults to `main`** unless the user explicitly specifies a different branch.
- **Review confidence threshold is 40** (not the default 80) — we want to catch more issues at this final gate.
- **Sprint status** — Update `{implementation_artifacts}/sprint-status.yaml` to mark the epic as `pr-open`.
- This skill is **project-agnostic** — it works with any BMAD project. Do not hardcode project names.

### Progress Updates

Long-running phases (simplification, review) can take many minutes. Keep the user informed:

- **Poll agents every ~2 minutes** while they're running and report a brief status.
- **Report immediately** when an agent completes — don't wait for the others.
- **After each phase**, output a summary of what was done (not just "done" — say what changed).
- **Phase transitions** — always announce when moving to the next phase.

## Plugins Used

| Plugin | Identifier | Invocation | Phase |
|--------|-----------|------------|-------|
| Code Simplifier | `code-simplifier@claude-plugins-official` | Agent: `code-simplifier:code-simplifier` | Phase 2 |
| Code Review | `code-review@claude-plugins-official` | Skill: `code-review:code-review` | Phase 4 |

### Token Recovery

The code-review plugin posts findings as **PR review comments** on GitHub. If the session runs out of tokens after Phase 4, findings are not lost — a new session can read them from the PR via `gh api`. Phase 5 handles both in-session and recovery scenarios.
