---
name: epic-stories
description: 'Create all dev story files for a BMAD epic in parallel. Reads the epic overview, identifies stories needing creation, dispatches parallel agents running bmad-create-story for each, then runs bmad-party-mode for group review. Fully autonomous by default. Use when the user says "epic stories N", "create epic N stories", or wants to generate all story files for an epic. Stage 1 of the epic pipeline: epic-stories → epic-dev → epic-push → epic-feedback.'
---

# Epic Stories

Create all dev story files for an epic using parallel agents, then refine via group discussion.

**Invocation:** `epic stories N` or `create epic N stories`

Read and follow [references/workflow.md](references/workflow.md) step by step.
If the relative path does not resolve, glob for `**/epic-stories/references/workflow.md` and read the first match.

## Critical Rules

- **DO NOT** wait for user approval after presenting the plan — show it briefly, then execute immediately.
- **ALWAYS** create a task list at the start showing all phases, and update it as you progress.
- **PARALLEL CREATION** — Dispatch one agent per story, all in a single message. Stories within the same epic are independent at the creation stage.
- **DEFAULT: AUTONOMOUS** — Unless the user explicitly says "ask me questions" or "come back with feedback", the entire process runs without user intervention. Agents resolve ambiguities between themselves during party mode.
- **INTERACTIVE MODE** — Only activated when the user explicitly requests it. In this mode, party mode presents questions/findings to the user before making changes.
- **ALWAYS** update `sprint-status.yaml` as stories transition from `backlog` → `ready-for-dev`.
- **ALWAYS** run party mode after story creation — this is not optional. The group review catches inconsistencies, missing context, and cross-story conflicts.
- **NEVER** skip stories that are in `backlog` status — create ALL of them.
- **NEVER** re-create stories that are already `ready-for-dev` or beyond — only `backlog` stories get created.
- This skill is **project-agnostic** — it works with any BMAD project. Do not hardcode project names.

## Agent Dispatch Prompts

### Story Creation Agent Prompt Template

```
You are creating a BMAD story file.

Invoke the `bmad-create-story` skill to create this story:
  Story identifier: {epic_num}-{story_num} (e.g., "9-1")

Work autonomously until the story file is fully created.
Follow the bmad-create-story workflow exactly.
The sprint-status.yaml should be updated: story moves from "backlog" to "ready-for-dev".
```

### Party Mode Agent Prompt Template (Autonomous)

```
You are running a BMAD party mode session to review newly created story files.

Invoke the `bmad-party-mode` skill.

Context: All story files for Epic {N} have just been created. The team needs to review
them as a group for:
- Cross-story consistency (shared components, APIs, data contracts)
- Missing context or guardrails that could cause dev agent mistakes
- Dependency accuracy (does each story correctly reference what it needs?)
- Acceptance criteria completeness
- Potential conflicts between stories

IMPORTANT: This is an AUTONOMOUS session. Do NOT ask the user any questions.
The agents must discuss, debate, and resolve all issues between themselves.
When the group reaches consensus on a change, implement it directly in the story files.

Topic to discuss:
"Review all story files for Epic {N}. The stories are:
{story_list}

Each agent should review from their expertise:
- Architect: technical feasibility, architecture compliance, dependency accuracy
- QA: acceptance criteria completeness, testability, edge cases
- PM: requirement coverage, business value alignment
- Dev: implementability, missing guardrails, practical concerns

Debate any disagreements. Make changes directly to story files when consensus is reached.
When satisfied, summarize what was reviewed and any changes made, then exit."
```

### Party Mode Agent Prompt Template (Interactive)

```
You are running a BMAD party mode session to review newly created story files.

Invoke the `bmad-party-mode` skill.

Context: All story files for Epic {N} have just been created. The team needs to review
them as a group for:
- Cross-story consistency (shared components, APIs, data contracts)
- Missing context or guardrails that could cause dev agent mistakes
- Dependency accuracy
- Acceptance criteria completeness
- Potential conflicts between stories

IMPORTANT: This is an INTERACTIVE session. The user wants to be consulted.
Present findings and questions to the user BEFORE making changes to story files.
Wait for user direction on any proposed changes.

Topic to discuss:
"Review all story files for Epic {N}. The stories are:
{story_list}

Each agent should review from their expertise and present findings to the user."
```
