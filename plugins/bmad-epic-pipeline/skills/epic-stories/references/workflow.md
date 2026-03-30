# Epic Stories Workflow

Follow these phases sequentially. Each phase must complete before the next begins.

**Before starting**, create a task list to track progress:

```
Phase 1: Discovery              — pending
Phase 2: Show plan and execute  — pending
Phase 3: Create story files     — pending
Phase 4: Party mode review      — pending
Phase 5: Wrap up                — pending
```

Mark each phase as `in-progress` when starting and `done` when complete.

---

## Path Resolution

**Before any phase**, resolve file paths from the BMAD project config.

1. Read the BMAD config: `{project-root}/_bmad/bmm/config.yaml`
2. Extract these variables:
   - `planning_artifacts` — where epic overviews and epics index live
   - `implementation_artifacts` — where story files and sprint status live
3. `{project-root}` = the current working directory (workspace root)

### Path Reference

| Artifact | Path Pattern | Example (escola-divina) |
|----------|-------------|------------------------|
| BMAD config | `{project-root}/_bmad/bmm/config.yaml` | `_bmad/bmm/config.yaml` |
| Epic overview | `{planning_artifacts}/epics/epic-{NN}-*.md` | `_artifacts/planning-artifacts/epics/epic-09-feedback.md` |
| Epics index | `{planning_artifacts}/epics.md` | `_artifacts/planning-artifacts/epics.md` |
| Sprint status | `{implementation_artifacts}/sprint-status.yaml` | `_artifacts/implementation-artifacts/sprint-status.yaml` |
| Story file (output) | `{implementation_artifacts}/{N}-{M}-{slug}.md` | `_artifacts/implementation-artifacts/9-1-build-feedback-api-endpoint-with-email-notification.md` |

`{NN}` = zero-padded epic number (e.g., `09`). `{N}` = unpadded (e.g., `9`). `{M}` = story number. `{slug}` = kebab-case title from the epics file.

---

## Phase 1: Discovery

**Goal:** Identify the epic, its stories, and which ones need creation.

1. Parse the epic number from user input (e.g., "epic stories 9").
2. Determine the execution mode:
   - **Autonomous** (default): Agents resolve everything without user input.
   - **Interactive**: Only if the user explicitly said "ask me questions", "come back with feedback", "interactive", or similar.
3. Read the epic definition file:
   - `{planning_artifacts}/epics/epic-{NN}-*.md` (NN = zero-padded)
   - Extract the full list of stories (e.g., 9.1, 9.2, 9.3, 9.4)
4. Read current sprint status:
   - `{implementation_artifacts}/sprint-status.yaml`
   - For each story in the epic, check its status
5. Build the creation list:
   - Include stories with status `backlog` — these need story files created
   - Skip stories with status `ready-for-dev`, `in-progress`, `review`, or `done` — already created
6. If no stories are in `backlog`:
   - Report: "All stories for Epic {N} already have files created. Nothing to do."
   - HALT.

---

## Phase 2: Show Plan and Execute

**Goal:** Briefly show what will be created, then start immediately. Do NOT wait for approval.

Display the plan concisely:

```
## Epic Stories: Epic {N} — {title}
Stories to create: {count}
Mode: {Autonomous / Interactive}

Creating: {N.1} ({title}), {N.2} ({title}), {N.3} ({title}), ...
Already created: {list or "none"}

Dispatching {count} agents now...
```

Proceed directly to Phase 3.

---

## Phase 3: Create Story Files

**Goal:** Dispatch parallel agents to create all story files simultaneously.

1. For each story in the creation list, dispatch a parallel agent:
   - Use the **Story Creation Agent Prompt Template** from SKILL.md
   - Populate `{epic_num}` and `{story_num}` for each story
   - Name agents descriptively: `"story-{N}-{M}"` (e.g., `"story-9-1"`)
   - Dispatch ALL agents in a **single message** (parallel)
2. Wait for all agents to complete.
3. Verify each story file was created:
   - Glob: `{implementation_artifacts}/{N}-{M}-*.md` for each story
   - If any are missing, report the failure.
4. Verify sprint-status.yaml was updated:
   - Each created story should now be `ready-for-dev`
   - If any weren't updated, fix the status file.
5. Brief status update:
   ```
   Story creation complete: {list}. All files created and status updated. Moving to group review.
   ```

---

## Phase 4: Party Mode Review

**Goal:** Run a group discussion to review and refine all newly created story files.

### 4a: Prepare Context

1. Read all newly created story files to build the story list summary.
2. For each story, extract:
   - Title and scope
   - Key dependencies
   - Acceptance criteria count
3. Build the `{story_list}` string for the party mode prompt:
   ```
   - {N}.1: {title} — depends on {deps} — {AC_count} acceptance criteria
   - {N}.2: {title} — depends on {deps} — {AC_count} acceptance criteria
   ...
   ```

### 4b: Dispatch Party Mode

1. Select the appropriate prompt template based on execution mode:
   - **Autonomous** → Use "Party Mode Agent Prompt Template (Autonomous)" from SKILL.md
   - **Interactive** → Use "Party Mode Agent Prompt Template (Interactive)" from SKILL.md
2. Dispatch a single agent for party mode:
   - Use the selected prompt template
   - Populate `{N}`, `{story_list}`
   - Name: `"review-epic-{N}-stories"`
3. Wait for the agent to complete.
4. Read the agent's output to understand what was reviewed and changed.
5. Brief status update:
   ```
   Party mode review complete. {summary of changes or "No changes needed"}.
   ```

---

## Phase 5: Wrap Up

1. Read the final state of all story files (in case party mode made changes).
2. Verify sprint-status.yaml is accurate.
3. Present final summary:

```
## Epic {N} Stories Created

**Stories created:** {count}
**Mode:** {Autonomous / Interactive}
**Party mode changes:** {count or "None"}

### Story Files:
- {N}-{M}-{slug}.md — {title} [ready-for-dev]
- {N}-{M}-{slug}.md — {title} [ready-for-dev]
...

### Party Mode Summary:
{Brief summary of group review findings and any changes made}

### Suggested next:
- Review story files if desired: {implementation_artifacts}/{N}-*.md
- Run `epic dev {N}` to implement all stories
```
