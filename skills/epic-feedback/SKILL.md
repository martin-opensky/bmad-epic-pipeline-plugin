---
name: epic-feedback
description: 'Process manual testing feedback for a BMAD epic. Reads the manual test plan, maps user feedback to test plan items, implements all fixes using parallel agents, then reconciles story/epic spec files to prevent drift on future runs. Use when the user says "epic feedback N", "testing feedback epic N", or provides feedback referencing a manual test plan. Stage 4 of the epic pipeline: epic-stories → epic-dev → epic-push → epic-feedback.'
---

# Epic Feedback

Implement fixes from manual testing feedback, reconcile specs, and guide re-testing.

**Invocation:** `epic feedback N` or `testing feedback epic N`

Read and follow [references/workflow.md](references/workflow.md) step by step.
If the relative path does not resolve, glob for `**/epic-feedback/references/workflow.md` and read the first match.

## Critical Rules

- **DO NOT** wait for user approval after presenting the fix plan — show it briefly, then execute immediately.
- **FREE-FORM IMPLEMENTATION** — Fixes are driven by the user's feedback, NOT constrained by story specs. The user is the authority at this stage.
- **ALWAYS** reconcile specs after fixing — update story and epic files so future BMAD runs don't revert the changes.
- **ALWAYS** run tests after implementing fixes.
- **ALWAYS** save a feedback resolution report, even if all items are trivial.
- **ALWAYS** push fixes to the PR branch after implementing — the PR is already open at this stage.
- **ALWAYS** track the round number — check for existing feedback reports to determine the current round.
- **NEVER** ignore user feedback because it contradicts a story spec — the user's observation is correct, the spec is outdated.
- **Status updates** — After each phase, give a 2-3 line summary. No verbose progress reports.
- **Sprint status** — Update `{implementation_artifacts}/sprint-status.yaml` if story statuses change.
- This skill is **project-agnostic** — it works with any BMAD project. Do not hardcode project names.

## Agent Dispatch Prompts

### Fix Agent Prompt Template

```
You are implementing manual testing fixes for Epic {N}.

Context:
- Manual test plan: {implementation_artifacts}/epic-{N}-manual-test-plan.md
- Feedback items assigned to you (below)

IMPORTANT: You are NOT constrained by story spec files. The user's feedback from manual
testing is the source of truth. Implement exactly what the user describes.

Feedback items to fix:
{feedback_items}

For each item:
1. Understand the user's observation and expected behavior
2. Find the relevant code
3. Implement the fix
4. Verify with any existing tests — add/update tests if the fix changes observable behavior

After fixing all items, run the relevant test suites to confirm nothing is broken.
```

### Spec Reconciliation Agent Prompt Template

```
You are reconciling BMAD spec files after manual testing fixes for Epic {N}.

Changes that were made (from feedback resolution):
{changes_summary}

Story files to update:
{story_files}

For each story file that is affected by the changes:
1. Read the current story file
2. Identify acceptance criteria or implementation details that no longer match the current code
3. Add a "## Manual Testing Amendments" section at the end (or append to existing one) with:
   - Date of amendment
   - What changed and why (reference the test plan item #)
   - Updated expected behavior
4. Do NOT delete or rewrite existing acceptance criteria — append the amendment so the
   history is preserved and future agents see both the original intent and the correction.

If the epic definition file has high-level descriptions that conflict with the fixes,
add a similar amendment note there too.
```
