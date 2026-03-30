---
name: epic-dev
description: 'Orchestrate full BMAD epic implementation using parallel agents. Reads epic definition, analyzes story dependency DAG, builds parallel execution tiers, then dispatches parallel agents for dev (bmad-dev-story) and review (bmad-code-review) per tier. Includes manual test plan generation and sprint status updates. Use when the user says "epic dev N", "implement epic N", or wants to execute an entire epic end-to-end. Stage 2 of the epic pipeline: epic-stories → epic-dev → epic-push → epic-feedback.'
---

# Epic Dev

Orchestrate an entire BMAD epic from start to finish using parallel agent tiers.

**Invocation:** `epic dev N` or `implement epic N`

Read and follow [references/workflow.md](references/workflow.md) step by step.
If the relative path does not resolve, glob for `**/epic-dev/references/workflow.md` and read the first match.

## Critical Rules

- **DO NOT** wait for user approval after presenting the plan — show the plan briefly, then execute immediately.
- **ALWAYS** create a task list at the start showing all phases, and update it as you progress.
- **ALWAYS** dispatch parallel agents for concurrent work (use the platform's agent/task dispatching mechanism).
- **NEVER** run stories from different tiers in parallel — tiers are sequential gates.
- **NEVER** skip code review — every story gets reviewed after dev.
- **NEVER** skip non-trivial review fixes — implement everything except Trivial/Deferred.
- **ALWAYS** save review output to a file, even if no issues found (include approval status).
- **Sprint status** — Update `{implementation_artifacts}/sprint-status.yaml` as stories transition.
- This skill is **project-agnostic** — it works with any BMAD project. Do not hardcode project names.

### Progress Updates

Long-running agent phases (dev, review, fix) can take many minutes. Keep the user informed so they know the process isn't hung.

- **After dispatching agents** — Immediately report: which agents were dispatched, what each is working on.
- **While agents are running** — Every ~2 minutes, poll agent status and report a brief summary:
  ```
  [3 min elapsed] Agent dev-9-1: implementing API endpoint. Agent dev-9-2: writing tests. Agent dev-9-3: building UI component.
  ```
- **When an agent completes** — Report immediately, don't wait for the others:
  ```
  Agent dev-9-1 complete (4 min). Waiting on dev-9-2, dev-9-3.
  ```
- **Phase transitions** — Always report when moving between dev → review → fix → next tier.
- **Keep it brief** — 1-2 lines per update. Don't waste tokens on verbose progress. The goal is a heartbeat, not a novel.

## Agent Dispatch Prompts

### Dev Agent Prompt Template

```
You are implementing a BMAD story.

Invoke the `bmad-dev-story` skill to implement this story:
  Story file: {implementation_artifacts}/{story-file}.md

Work autonomously until the story is fully implemented with all tests passing.
Follow the story spec exactly — implement ONLY what the story requires.
Update sprint-status.yaml: mark the story "in-progress" at start, "review" when done.
```

### Review Agent Prompt Template

```
You are reviewing code for a BMAD story.

Invoke the `bmad-code-review` skill.
Review uncommitted/staged changes for story {N.M} ({title}).
Story spec: {implementation_artifacts}/{story-file}.md

IMPORTANT: You MUST save a review file regardless of outcome.
Save to: {output_folder}/reviews/epic-{N}/review-{N}-{M}-{slug}.md

The review file MUST include:
1. Summary of what was reviewed
2. All findings (Critical, Major, Minor, Trivial) with descriptions
3. An **Approval Status** section at the end:
   - "APPROVED" if no blocking issues remain
   - "APPROVED WITH CONDITIONS" listing what must be done before merge
   - "CHANGES REQUIRED" listing blocking issues
4. If APPROVED WITH CONDITIONS, list specific verification steps

Even if no issues are found, save the file with "APPROVED - No issues found."
```

### Fix Agent Prompt Template

```
You are fixing code review findings.

Read the review: {output_folder}/reviews/epic-{N}/review-{N}-{M}-{slug}.md
Implement ALL Critical and Major findings.
Implement Minor findings unless explicitly trivial.
Skip items marked Trivial or Deferred.
Run tests after fixes to confirm nothing is broken.

After fixing, update the review file's Approval Status to reflect the current state.
```
