# Epic Feedback Workflow

Follow these phases sequentially. Each phase must complete before the next begins.

---

## Path Resolution

**Before any phase**, resolve file paths from the BMAD project config.

1. Read the BMAD config: `{project-root}/_bmad/bmm/config.yaml`
2. Extract these variables:
   - `planning_artifacts` — where epic overviews live
   - `implementation_artifacts` — where story files, sprint status, and test plans live
   - `output_folder` — base output directory (for feedback reports)
3. `{project-root}` = the current working directory (workspace root)

### Path Reference

| Artifact | Path Pattern | Example (escola-divina) |
|----------|-------------|------------------------|
| BMAD config | `{project-root}/_bmad/bmm/config.yaml` | `_bmad/bmm/config.yaml` |
| Manual test plan (input) | `{implementation_artifacts}/epic-{N}-manual-test-plan.md` | `_artifacts/implementation-artifacts/epic-8-manual-test-plan.md` |
| Story files | `{implementation_artifacts}/{N}-{M}-{slug}.md` | `_artifacts/implementation-artifacts/8-3-implement-share-button-and-share-sheet.md` |
| Epic overview | `{planning_artifacts}/epics/epic-{NN}-*.md` | `_artifacts/planning-artifacts/epics/epic-08-sharing-deeplinks.md` |
| Sprint status | `{implementation_artifacts}/sprint-status.yaml` | `_artifacts/implementation-artifacts/sprint-status.yaml` |
| Feedback report (output) | `{output_folder}/reviews/epic-{N}/testing-feedback-{N}-r{R}.md` | `_artifacts/reviews/epic-8/testing-feedback-8-r1.md` |

`{NN}` = zero-padded epic number (e.g., `08`). `{N}` = unpadded (e.g., `8`). `{R}` = round number.
`{output_folder}` = the base output directory from config (e.g., `_artifacts`).

---

## Phase 1: Parse Input

**Goal:** Extract the epic number and the user's testing feedback.

1. Parse the epic number from user input (e.g., "epic feedback 8").
2. Extract the user's feedback items. These typically reference test plan item numbers:
   - `#3 — Share button doesn't appear on landscape`
   - `#7 — Deep link opens wrong hymn`
   - Free-form descriptions are also valid
3. Normalize each feedback item into a structured list:
   - **Test plan item #** (if referenced, otherwise `N/A`)
   - **Area / Screen** (inferred from context or test plan)
   - **Issue description** (what the user observed)
   - **Expected behavior** (what the user wants instead)

---

## Phase 2: Load Context

**Goal:** Understand the test plan, stories, and current code state.

1. Read the manual test plan:
   - `{implementation_artifacts}/epic-{N}-manual-test-plan.md`
2. Read ALL story files for the epic:
   - Glob: `{implementation_artifacts}/{N}-*.md`
3. Read the epic definition:
   - `{planning_artifacts}/epics/epic-{NN}-*.md` (NN = zero-padded)
4. Read current sprint status:
   - `{implementation_artifacts}/sprint-status.yaml`
5. For each feedback item that references a test plan #, map it to:
   - The test plan row (Screen/Area, Action, Expected Result, Stories)
   - The story file(s) that cover that functionality

---

## Phase 3: Codebase Recon

**Goal:** Locate the code that needs to change for each feedback item.

1. From the test plan mapping and feedback descriptions, identify:
   - Components, screens, hooks, services, routes involved
   - Test files that may need updating
2. Use Explore agents (or Glob/Grep) to find the relevant source files.
3. Group feedback items by code area to identify natural parallelization boundaries:
   - Items touching the same file(s) → same fix agent
   - Items in unrelated areas → separate parallel agents

---

## Phase 4: Show Plan and Execute

**Goal:** Briefly show the fix plan, then start executing immediately. Do NOT wait for approval.

Display the plan concisely:

```
## Testing Feedback: Epic {N}
Items: {count} | Fix groups: {count}

Group 1 ({area}): #{item}, #{item} — {brief description}
Group 2 ({area}): #{item} — {brief description}
Group 3 ({area}): #{item}, #{item} — {brief description}

Executing fixes now...
```

Proceed directly to Phase 5.

---

## Phase 5: Implement Fixes

**Goal:** Implement all fixes from the user's feedback.

### 5a: Dispatch Fix Agents

1. For each fix group, dispatch a parallel agent:
   - Use the **Fix Agent Prompt Template** from SKILL.md
   - Populate `{feedback_items}` with the full detail for that group
   - Include relevant file paths discovered in Phase 3
   - Name agents descriptively: `"fix-{N}-{area}"` (e.g., `"fix-8-share-sheet"`)
   - Dispatch ALL groups in a **single message** (parallel)
2. If there's only one group or the fixes are simple, implement directly without dispatching agents.

### 5b: Verify

1. Wait for all fix agents to complete.
2. Run the full test suite for affected areas.
3. If tests fail, diagnose and fix. Re-run until green.
4. Brief status:
   ```
   All {count} feedback items implemented. Tests passing.
   ```

---

## Phase 6: Spec Reconciliation

**Goal:** Update story and epic spec files so future BMAD runs don't revert manual testing fixes.

This is the **critical drift-prevention step**. Without it, a future `bmad-dev-story` or `bmad-epic-orchestrator` run could undo these changes because the specs don't reflect them.

### 6a: Identify Spec Drift

1. For each feedback item that was fixed, determine:
   - Which story file(s) originally specified the behavior that changed
   - Whether the fix contradicts, extends, or refines the original spec
2. Build a list of story files that need amendment.

### 6b: Amend Story Files

For each affected story file:

1. Read the current file.
2. Append a `## Manual Testing Amendments` section (or add to an existing one):

```markdown
## Manual Testing Amendments

### Amendment — {YYYY-MM-DD}

**Source:** Manual testing of Epic {N}, test plan items: #{item}, #{item}

**Changes:**
- {What was changed and why}
- {Updated expected behavior}

**Note to future agents:** The acceptance criteria above reflect the original spec.
This amendment reflects real-world testing corrections. When implementing this story,
follow the amended behavior described here. If there is a conflict between the original
criteria and this amendment, **this amendment takes precedence**.
```

3. If the fix is minor (e.g., styling tweak), a one-line note suffices.
4. If the fix changes fundamental behavior, be detailed.

### 6c: Amend Epic File (if needed)

If any fix changes a high-level feature description in the epic file, add a similar amendment note.

### 6d: Dispatch or Direct

- If amendments are simple and few, make them directly.
- If many stories need updating, dispatch a parallel Spec Reconciliation agent:
  - Use the **Spec Reconciliation Agent Prompt Template** from SKILL.md
  - Populate `{changes_summary}` and `{story_files}`

---

## Phase 7: Update Test Plan

**Goal:** Annotate the manual test plan with results.

1. Read `{implementation_artifacts}/epic-{N}-manual-test-plan.md`
2. Add a `## Test Results` section (or update existing) with:
   - Date of testing round
   - For each item the user tested: PASS / FAIL / FIXED
   - Notes on what was fixed
3. If new test items emerged from the feedback (things not originally in the plan), add them.

---

## Phase 8: Save Feedback Report

**Goal:** Create a record of the testing feedback and resolutions.

Save to: `{output_folder}/reviews/epic-{N}/testing-feedback-{N}-{round}.md`

Where `{round}` is `r1`, `r2`, etc. (check for existing files to determine round number).

**Report format:**

```markdown
# Epic {N} — Manual Testing Feedback (Round {R})

**Date:** {YYYY-MM-DD}
**Test plan:** {implementation_artifacts}/epic-{N}-manual-test-plan.md

## Feedback Items

| # | Test Plan Item | Issue | Resolution | Files Changed |
|---|---------------|-------|------------|---------------|
| 1 | #{item}       | {what was wrong} | {what was done} | {file list} |
| 2 | ...           | ...   | ...        | ...           |

## Spec Amendments

| Story File | Amendment Summary |
|-----------|-------------------|
| {N}-{M}-{slug}.md | {what was amended} |

## Test Results

- All automated tests: PASSING / {count} failures
- Items verified: {count}/{total}
```

---

## Phase 9: Wrap Up

1. Commit and push all changes to the PR branch:
   ```
   git add -A && git commit -m "fix: manual testing feedback round {R} for epic {N}"
   git push
   ```
2. Update `{implementation_artifacts}/sprint-status.yaml` if needed:
   - If stories were marked `done` but feedback revealed issues, keep them at `review` or `done` with a note
   - Generally, testing feedback doesn't change story status — it refines the implementation
3. Present final summary:

```
## Testing Feedback Complete: Epic {N} — Round {R}

**Items fixed:** {count}
**Specs amended:** {count} story files
**Tests:** All passing
**Changes pushed to PR**

### What was changed
- {brief description of fix 1}
- {brief description of fix 2}
- ...

### What to re-test
Focus on these areas (you don't need to re-run the full test plan):
- {specific area / test plan item affected by fix 1}
- {specific area / test plan item affected by fix 2}
- ...

### Files:
- Feedback report: {output_folder}/reviews/epic-{N}/testing-feedback-{N}-r{R}.md
- Updated test plan: {implementation_artifacts}/epic-{N}-manual-test-plan.md

### Suggested next:
- Re-test the items listed above
- If more issues found, run `epic feedback {N}` again (Round {R+1})
- When satisfied, review the PR and merge
- Run `bmad-retrospective` for Epic {N} to capture lessons learned
```
