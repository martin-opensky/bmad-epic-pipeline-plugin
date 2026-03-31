# BMAD Epic Pipeline Plugin

Claude Code plugin packaging the four-stage BMAD epic orchestration pipeline.

## Project Structure

```
plugins/bmad-epic-pipeline/
  .claude-plugin/plugin.json   ← plugin metadata + version
  skills/
    epic-stories/              ← Stage 1: create story files
    epic-dev/                  ← Stage 2: implement stories in parallel tiers
    epic-push/                 ← Stage 3: simplify, lint, PR, review, fix, push
    epic-feedback/             ← Stage 4: process manual testing feedback
```

Each skill has a `SKILL.md` (entry point) and `references/workflow.md` (detailed steps).

## Version Management

**Always bump the version in `plugins/bmad-epic-pipeline/.claude-plugin/plugin.json` when pushing any change.**

- Patch bump (1.1.0 → 1.1.1) for bug fixes, wording tweaks, small behavioral changes.
- Minor bump (1.1.0 → 1.2.0) for new features or significant workflow changes.
- Major bump (1.0.0 → 2.0.0) for breaking changes to skill invocation or behavior.

Consumers update via `/plugin marketplace update` + `/plugin install bmad-epic-pipeline@bmad-epic-pipeline`, and the version in plugin.json is how they confirm they're on the latest.

## Commit Conventions

Follow conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`.

Keep commit messages concise — first line summarizes the change, optional body with bullets for detail.

## Testing Changes Locally

```bash
claude --plugin-dir ./bmad-epic-pipeline-plugin
```

## Key Design Decisions

- Skills are project-agnostic — they read paths from `_bmad/bmm/config.yaml` at runtime.
- epic-push enforces lint + type-check as a hard gate at every checkpoint (pre-flight, post-simplification, post-review-fixes, final push).
- epic-dev auto-commits at the end of Phase 7 with a descriptive message referencing the epic number.
- epic-push does NOT auto-merge — the human always does the final merge.
