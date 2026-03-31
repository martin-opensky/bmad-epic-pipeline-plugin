# Epic Dev Workflow

Follow these phases sequentially. Each phase must complete before the next begins.

**Before starting**, create a task list to track progress:

```
Phase 1: Discovery              — pending
Phase 2: Codebase recon         — pending
Phase 3: Tier planning          — pending
Phase 4: Execute tiers          — pending
Phase 5: Manual test plan       — pending
Phase 6: Wrap up                — pending
```

Mark each phase as `in-progress` when starting and `done` when complete.

---

## Path Resolution

**Before any phase**, resolve file paths from the BMAD project config.

1. Read the BMAD config: `{project-root}/_bmad/bmm/config.yaml`
2. Extract these variables:
   - `planning_artifacts` — where epic overviews and epics index live
   - `implementation_artifacts` — where story files and sprint status live
   - `output_folder` — base output directory (for reviews)
3. `{project-root}` = the current working directory (workspace root)

### Path Reference

| Artifact | Path Pattern | Example (escola-divina) |
|----------|-------------|------------------------|
| BMAD config | `{project-root}/_bmad/bmm/config.yaml` | `_bmad/bmm/config.yaml` |
| Epic overview | `{planning_artifacts}/epics/epic-{NN}-*.md` | `_artifacts/planning-artifacts/epics/epic-08-sharing-deeplinks.md` |
| Epics index | `{planning_artifacts}/epics.md` | `_artifacts/planning-artifacts/epics.md` |
| Sprint status | `{implementation_artifacts}/sprint-status.yaml` | `_artifacts/implementation-artifacts/sprint-status.yaml` |
| Story files | `{implementation_artifacts}/{N}-{M}-{slug}.md` | `_artifacts/implementation-artifacts/8-1-build-og-metadata-route-for-link-previews.md` |
| Review output | `{output_folder}/reviews/epic-{N}/review-{N}-{M}-{slug}.md` | `_artifacts/reviews/epic-8/review-8-1-og-metadata.md` |
| Manual test plan (output) | `{implementation_artifacts}/epic-{N}-manual-test-plan.md` | `_artifacts/implementation-artifacts/epic-8-manual-test-plan.md` |

`{NN}` = zero-padded epic number (e.g., `08`). `{N}` = unpadded (e.g., `8`). `{M}` = story number. `{slug}` = kebab-case title.
`{output_folder}` = the base output directory from config (e.g., `_artifacts`).

---

## Phase 1: Discovery

**Goal:** Understand the epic scope, stories, and dependencies.

1. Parse the epic number from user input.
2. Read the epic definition file:
   - `{planning_artifacts}/epics/epic-{NN}-*.md` (NN = zero-padded)
3. Read the centralized epic index for the dependency DAG:
   - `{planning_artifacts}/epics.md` — find the parallelization section for this epic
4. Read ALL story files for the epic:
   - Glob: `{implementation_artifacts}/{N}-*.md`
   - Read each to understand scope, acceptance criteria, and `Depends on:` fields
5. Read current sprint status:
   - `{implementation_artifacts}/sprint-status.yaml`
   - Verify stories are `ready-for-dev` (note any already `done`/`in-progress`)
6. Read `references/bmad-current-status.md` for context (if exists).

---

## Phase 2: Codebase Recon

**Goal:** Understand the existing codebase in areas the epic will touch.

1. From the story specs, identify key files/directories that will be created or modified.
2. Use the Explore agent (or Glob/Grep) to scan these areas:
   - Existing services, hooks, components, routes, schemas
   - Test patterns in use
   - Imports/exports that stories depend on
3. Note patterns, conventions, and shared utilities stories should follow.

---

## Phase 3: Tier Planning

**Goal:** Group stories into parallel execution tiers from the dependency DAG.

1. From the epic's parallelization section AND each story's `Depends on:` field, build the dependency graph.
2. Group stories into tiers:
   - **Tier 1:** Stories with no intra-epic dependencies (foundation layer)
   - **Tier 2:** Stories that depend only on Tier 1
   - **Tier 3:** Stories that depend on Tier 1 or 2
   - Continue until all stories are assigned
3. Within each tier, stories run in parallel.
4. A tier's full cycle (dev + review + fix) must complete before the next tier starts.

---

## Phase 4: Show Plan and Execute

**Goal:** Briefly show the plan, then start executing immediately. Do NOT wait for approval.

Display the plan concisely:

```
## Epic {N}: {title}
Stories: {count} | Tiers: {count}

Tier 1: {N.1} ({scope}), {N.2} ({scope})
Tier 2: {N.3} ({scope}), {N.4} ({scope})
Tier 3: {N.5} ({scope})

Executing now...
```

Proceed directly to Phase 5.

---

## Phase 5: Execute Tiers

For each tier, execute three sub-phases:

### 5a: Dev Phase

1. For each story in the tier, dispatch a parallel agent:
   - Use the **Dev Agent Prompt Template** from SKILL.md
   - Name agents descriptively: `"dev-{N}-{M}"` (e.g., `"dev-8-1"`)
   - Dispatch ALL tier agents in a **single message** (parallel)
2. Immediately report what was dispatched:
   ```
   Tier {T} dev started: {agent list with what each is working on}.
   ```
3. While waiting, poll agent status every ~2 minutes. Report completions immediately and summarize what's still running. See **Progress Updates** in SKILL.md.
4. Brief status update when all finish:
   ```
   Tier {T} dev complete: {story list}. Moving to review.
   ```

### 5b: Review Phase

1. Ensure review directory exists: `{output_folder}/reviews/epic-{N}/`
2. For each story in the tier, dispatch a parallel agent:
   - Use the **Review Agent Prompt Template** from SKILL.md
   - Name: `"review-{N}-{M}"`
   - Dispatch ALL in a **single message**
3. Wait for all reviews to complete.
4. Verify each review was saved to file. If any agent failed to save, write a stub review file.
5. Read each review to assess findings and approval status.
6. Brief status update:
   ```
   Tier {T} reviews: {X} critical, {Y} major, {Z} minor across {count} stories. {approved/needs fixes}
   ```

### 5c: Fix Phase

1. Assess all review findings across the tier.
2. If findings are simple, fix directly. If complex/multi-story, dispatch parallel fix agents:
   - Use the **Fix Agent Prompt Template** from SKILL.md
   - One agent per story with non-trivial findings
3. Run tests after all fixes.
4. Update review files' Approval Status after fixes are applied.
5. Brief status update:
   ```
   Tier {T} fixes applied. Tests passing. Moving to Tier {T+1}.
   ```
6. Update sprint-status.yaml: mark tier stories as `done`.

### Tier Transition

After 5c completes, proceed to next tier at 5a. Repeat until all tiers done.

---

## Phase 6: Manual Test Plan

**Goal:** Generate a concise, human-friendly manual test plan for the entire epic.

The test plan is for a human tester sitting with a device. Keep it **lean — aim for ~15-25 tests** that cover all user-facing functionality without redundancy. A human should be able to run through the whole plan in one sitting.

**CRITICAL FORMAT RULE:** The test plan MUST be a single markdown **table** — one row per test. Do **NOT** use separate sections per test, do **NOT** use `## Test N` headings, do **NOT** use Steps/Expected subsection blocks. The entire plan should fit on roughly one page. If your output exceeds ~80 lines, you are being too verbose.

### How to build the plan

1. Read all story acceptance criteria across the epic.
2. Build the consolidated test table:
   - **Group rows by screen / feature area** — order the table so related tests are adjacent (e.g., all "Player" tests together, all "API" tests together). No section headers — one continuous table.
   - **Sequential numbering** — from 1 to the end, never restarting.
   - **Cover every story** — every story must appear in at least one row.
   - **Deduplicate aggressively** — if testing the share flow inherently validates OG metadata output, don't add a separate OG test. One row can cover multiple stories.
   - **Each row = one clear action → one observable result** — keep it brief and human-readable.
3. If there are useful **copy-pasteable commands** (curl snippets, simulator commands, etc.), add a short helper block above the table. Keep it brief — just the commands the tester will actually need to copy-paste. Not every epic needs this.
4. If some tests genuinely **cannot be run locally** (require a deployed domain, real device, production infra), add a short "Requires Deployment" note after the table listing prerequisites and what they unlock. Most epics won't need this — only include it when relevant.
5. **Save to file:** `{implementation_artifacts}/epic-{N}-manual-test-plan.md`
6. **Output the full plan in the chat** so the user can read it without opening the file.

### Test plan format (follow exactly)

```markdown
# Epic {N}: {title} — Manual Test Plan

> **Generated:** {YYYY-MM-DD} | **Stories:** {N}.1–{N}.{last}

{Optional: copy-pasteable helper commands if useful for this epic, e.g.:}

Test API locally (`bun run dev` in apps/api):
\```
curl http://localhost:8787/api/some/endpoint
\```

| #  | Test Case | How to Test | Expected Result | Status |
|----|-----------|-------------|-----------------|--------|
| 1  | {short name} | {what to do — brief, direct} | {what should happen — one sentence} | |
| 2  | ...       | ...         | ...             | |
```

### Column guide

- **Test Case**: Short descriptive name (e.g., "Share from player", "Deep link — cold start")
- **How to Test**: What the tester physically does — brief and direct. Include the actual command if it's a CLI/curl test.
- **Expected Result**: Observable outcome in one sentence
- **Status**: Left blank — all tests are assumed passing. The user only reports failures when running `epic feedback`

---

## Phase 7: Wrap Up

1. Update `{implementation_artifacts}/sprint-status.yaml`:
   - Mark the epic as `done`
2. Update `references/bmad-current-status.md` with completion info.
3. **Commit all changes** — stage and commit everything produced during this epic run:
   - Run `git add -A`
   - Build a commit message:
     - First line: `epic {N}: {short summary of what was built}` (e.g., `epic 10: implement offline sync with queue and retry`)
     - Body (optional, 2-4 bullets): list the key deliverables or stories completed
   - Commit with that message
   - Do **NOT** push — `epic push` handles that
4. Present final summary:

```
## Epic {N} Complete

**Stories:** {count} implemented
**Tiers:** {count} executed
**Review findings:** {count} fixed
**Tests:** All passing

### Reviews:
- {N.1}: {APPROVED/APPROVED WITH CONDITIONS/etc.}
- {N.2}: ...

### Files:
- Reviews: {output_folder}/reviews/epic-{N}/
- Test plan: {implementation_artifacts}/epic-{N}-manual-test-plan.md

### Suggested next:
- Run `epic push {N}` to simplify, review, and create PR
- Then manually test using the test plan above
- Run `epic feedback {N}` with anything that's not right
```
