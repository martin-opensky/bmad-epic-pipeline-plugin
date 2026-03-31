# Epic Push Workflow

Follow these phases sequentially. Each phase must complete before the next begins.

**Before starting**, create a task list to track progress:

```
Phase 1: Pre-flight checks         — pending
Phase 2: Code simplification       — pending
Phase 3: Create PR                 — pending
Phase 4: Code review               — pending
Phase 5: Implement review fixes    — pending
Phase 6: Final verification & push — pending
```

Mark each phase as `in-progress` when starting and `done` when complete.

---

## Phase 0: Path Resolution

**Before any phase**, resolve file paths from the BMAD project config.

1. Read the BMAD config: `{project-root}/_bmad/bmm/config.yaml`
2. Extract these variables:
   - `planning_artifacts` — where epic overviews live
   - `implementation_artifacts` — where story files and sprint status live
   - `output_folder` — base output directory
3. `{project-root}` = the current working directory (workspace root)

### Path Reference

| Artifact | Path Pattern |
|----------|-------------|
| BMAD config | `{project-root}/_bmad/bmm/config.yaml` |
| Sprint status | `{implementation_artifacts}/sprint-status.yaml` |
| Epic overview | `{planning_artifacts}/epics/epic-{NN}-*.md` |
| Manual test plan | `{implementation_artifacts}/epic-{N}-manual-test-plan.md` |
| Simplification summary | `{output_folder}/reviews/epic-{N}/simplification-{N}.md` |

`{NN}` = zero-padded epic number (e.g., `08`). `{N}` = unpadded (e.g., `8`).

---

## Phase 1: Pre-flight Checks

**Goal:** Verify the epic is ready to push, committing any outstanding work first.

1. Parse the epic number from user input.
2. **Ensure the working directory is the project root** — verify by checking for the presence of `_bmad/` or `package.json`. If not at the root, navigate there.
3. Check the current git branch:
   - Expect `feature/epic-{N}` or similar.
   - If on `main`, stop — the user should be on a feature branch.
4. **Commit all uncommitted changes:**
   - Run `git status` to check for staged, unstaged, and untracked files.
   - If there are uncommitted changes, stage and commit them:
     ```
     git add -A && git commit -m "feat: epic {N} implementation"
     ```
   - This is essential — `epic-dev` may leave work uncommitted, and the diff/simplification phases need everything in the commit history.
5. Run the full test suite — all tests must pass.
   - Run tests from the project root (e.g., `bun test` or the project's test command).
   - If tests use workspaces, run per-workspace: `bun test --filter apps/mobile`, `bun test --filter apps/api`, etc.
6. **Lint and type-check the entire codebase** — this is a hard gate, not just the changed files:
   - Detect the project's lint and type-check commands from `package.json` scripts (e.g., `lint`, `typecheck`, `tsc`).
   - Run them across all workspaces / the full project.
   - If there are **any** lint or type errors — even pre-existing ones not introduced by this epic — **fix them all**.
   - Re-run lint + type-check after fixes to confirm zero errors.
   - Commit lint fixes separately: `git add -A && git commit -m "fix: resolve lint and type errors"`
   - The codebase must be fully clean before proceeding.
7. Read `{implementation_artifacts}/sprint-status.yaml`:
   - Verify all stories for this epic are `done`.
   - If any are not `done`, stop and report which stories are blocking.
8. Determine PR target:
   - **Default:** `main`
   - **Override:** If the user specifies a different branch (e.g., "PR to develop"), use that.
   - Detection: Check `git log --merges --oneline -10` to confirm the project's merge pattern.
9. Get the list of all files changed during the epic:
   - `git diff {target}...HEAD --name-only`
   - Store this list for Phase 2.
10. Brief status:
    ```
    Pre-flight passed. Tests passing. Lint clean. {count} stories done. Branch: {branch} → {target}. {count} files changed.
    ```

---

## Phase 2: Code Simplification

**Goal:** Run Anthropic's code-simplifier across all epic changes.

1. Report to the user:
   ```
   Starting code simplification across {count} files...
   ```
2. Dispatch a **code-simplifier:code-simplifier** agent:
   - Agent type: `code-simplifier:code-simplifier`
   - Scope: all files from the `git diff` list in Phase 1
   - The agent reviews for reuse, clarity, consistency, and maintainability
3. **Monitor progress** — while the agent is running:
   - Poll every ~2 minutes
   - Report a brief status each time:
     ```
     [~2 min] Code simplification in progress — agent is running ({X} tool calls so far)
     ```
   - When the agent completes, report immediately
4. Run the full test suite to confirm nothing broke.
5. Run lint + type-check to confirm simplification didn't introduce errors. Fix any issues found.
6. If tests or lint fail, revert the problematic simplification and re-run.
7. Commit simplification changes:
   ```
   git add -A && git commit -m "refactor: code simplification for epic {N}"
   ```
8. **Save and output a summary of what was simplified:**
   - Save to: `{output_folder}/reviews/epic-{N}/simplification-{N}.md`
   - Output the same summary to the user:
   ```
   ## Code Simplification Summary
   - {file}: {what was changed — e.g., "extracted shared helper", "removed duplication"}
   - {file}: {what was changed}
   - ...
   Total: {count} files simplified. Tests passing.
   Saved to: {output_folder}/reviews/epic-{N}/simplification-{N}.md
   ```
   Keep it concise — one line per changed file, not a full diff.

---

## Phase 3: Create PR

**Goal:** Push the branch and create a pull request.

1. Push the branch:
   ```
   git push -u origin HEAD
   ```
2. Build the PR body from:
   - Epic title and summary (from epic overview file)
   - List of all stories implemented (from sprint status)
   - Key changes per story (1-2 lines each)
   - Reference to the manual test plan file
3. Create the PR using a HEREDOC for the body:
   ```
   gh pr create --title "feat: Epic {N} — {title}" --body "$(cat <<'EOF'
   ## Summary
   {epic summary}

   ## Stories
   {story list with brief descriptions}

   ## Test Plan
   See `{implementation_artifacts}/epic-{N}-manual-test-plan.md`

   EOF
   )" --base {target}
   ```
4. Report the PR URL.
5. Brief status:
   ```
   PR created: {url}
   ```

---

## Phase 4: Code Review

**Goal:** Run the code-review plugin against the PR.

The plugin posts its findings as **review comments directly on the PR**. These persist on GitHub, so no local file saving is needed — if the session runs out of tokens, a new session can read the PR comments.

1. Report to the user:
   ```
   Starting code review against the PR...
   ```
2. Dispatch the code review (invoked as `code-review:code-review`):
   - It launches parallel review agents internally (CLAUDE.md compliance, bug scanner, history analyzer)
   - Each finding is scored 0-100 for confidence
   - Use a **confidence threshold of 40** (not the default 80)
3. **Monitor progress** — poll every ~2 minutes and report status.
4. Once the review completes, output a findings summary:
   ```
   ## Code Review Summary
   Findings: {count} total — {count} must-fix (≥40), {count} skipped (<40)
   Review comments posted on PR: {url}

   Must-fix:
   - [{confidence}] {file}: {issue description}
   - [{confidence}] {file}: {issue description}
   ```

---

## Phase 5: Implement Review Fixes

**Goal:** Fix all findings that meet the confidence threshold.

1. If no findings >= 40, skip to Phase 6.
2. Read the review findings. Two sources (try in order):
   - **Same session:** Use the findings collected in Phase 4.
   - **New session (token recovery):** Read PR review comments via `gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews` and `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments`.
3. For each finding with confidence >= 40:
   - Understand the issue and suggested fix
   - Implement the fix
   - If the fix is complex or the suggestion is wrong, use best judgment
4. Run the full test suite after all fixes.
5. Run lint + type-check. Fix any issues introduced by the review fixes. Re-run until clean.
6. Commit fixes (do NOT push yet — Phase 6 is the final gate):
   ```
   git add -A && git commit -m "fix: address code review findings for epic {N}"
   ```
7. Brief status:
   ```
   {count} review findings fixed. Tests passing. Lint clean.
   ```

---

## Phase 6: Final Verification & Push

**Goal:** One last check that everything is clean, then push.

1. Run the full test suite one final time.
2. Run lint + type-check one final time across the entire codebase.
3. If **anything** fails — tests or lint — fix it, commit, and re-run both until fully green. Do NOT proceed until both pass.
4. Push all commits:
   ```
   git push
   ```
5. Verify the PR is up to date with all commits pushed.
6. Update `{implementation_artifacts}/sprint-status.yaml`:
   - Mark the epic status as `pr-open`
7. Present the final report:

```
## Epic {N} — PR Open

**PR:** {url}
**Branch:** {branch} → {target}
**Simplifications:** {count} applied
**Review findings:** {count} fixed (threshold: 40)
**Tests:** All passing
**Lint:** Clean (zero errors)

### What to do now
1. **Manually test** — use the test plan at `{implementation_artifacts}/epic-{N}-manual-test-plan.md`
2. **Report issues** — run `epic feedback {N}` with anything that's not right
3. **Repeat** — re-test after fixes, run `epic feedback {N}` again if needed
4. **When happy** — review the PR at {url}, merge, and delete the branch

You can also move on to the next epic while this one is open:
- `epic stories {N+1}` → `epic dev {N+1}` → `epic push {N+1}`
```
