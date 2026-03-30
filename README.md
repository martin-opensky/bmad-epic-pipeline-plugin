# BMAD Epic Pipeline Plugin

A Claude Code plugin that packages the four-stage BMAD epic orchestration pipeline as installable, updatable skills.

## Skills

| Skill | Stage | Description |
|-------|-------|-------------|
| **epic-stories** | 1 | Create all dev story files for an epic in parallel, then refine via group discussion (party mode) |
| **epic-dev** | 2 | Orchestrate full epic implementation — dependency-aware tier execution with parallel dev, review, and fix cycles |
| **epic-push** | 3 | Quality gate — code simplification, PR creation, code review (confidence threshold 40), fix, and push |
| **epic-feedback** | 4 | Process manual testing feedback, implement fixes, reconcile specs to prevent drift |

## Pipeline Flow

```
epic stories N → epic dev N → epic push N → epic feedback N
                                                    ↓
                                              (re-test, repeat)
```

## Installation

### Add the marketplace

```
/plugin marketplace add github:martin-opensky/bmad-epic-pipeline-plugin
```

### Install the plugin

```
/plugin install bmad-epic-pipeline@bmad-epic-pipeline
```

## Usage

Skills are invoked via natural language or namespaced commands:

```
epic stories 9        # Create all story files for Epic 9
epic dev 9            # Implement all stories with parallel tiers
epic push 9           # Simplify, PR, review, fix, push
epic feedback 9       # Process manual testing feedback
```

Or with explicit namespacing:

```
/bmad-epic-pipeline:epic-stories 9
/bmad-epic-pipeline:epic-dev 9
/bmad-epic-pipeline:epic-push 9
/bmad-epic-pipeline:epic-feedback 9
```

## Local Development

Test local changes without installing from the marketplace:

```bash
claude --plugin-dir ./bmad-epic-pipeline-plugin
```

## Updating

After pulling changes to the repo:

```
/plugin marketplace update
/plugin install bmad-epic-pipeline@bmad-epic-pipeline
```

## Migration from Standalone Skills

If you previously had these skills installed at `~/.claude/skills/epic-*`, both the standalone and plugin versions will be active simultaneously. To avoid duplicates, delete the standalone copies once you've verified the plugin works:

```bash
rm -rf ~/.claude/skills/epic-stories
rm -rf ~/.claude/skills/epic-dev
rm -rf ~/.claude/skills/epic-feedback
rm -rf ~/.claude/skills/epic-push
```

## Required Anthropic Plugins

The **epic-push** skill depends on two official Anthropic plugins for code simplification and code review. Install them first:

```
/plugin marketplace add github:anthropics/claude-plugins-official
/plugin install code-review@claude-plugins-official
/plugin install code-simplifier@claude-plugins-official
```

These provide:

| Plugin | Used By | Purpose |
|--------|---------|---------|
| [code-simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier) | epic-push (Phase 2) | Simplifies and refines code for clarity, consistency, and maintainability |
| [code-review](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-review) | epic-push (Phase 4) | Multi-agent code review with confidence-scored findings posted as PR comments |

## Other Prerequisites

These skills orchestrate BMAD workflows and expect:

- A BMAD-configured project with `_bmad/bmm/config.yaml`
- Planning and implementation artifact directories as defined in the BMAD config
- The `bmad-create-story`, `bmad-party-mode`, `bmad-dev-story`, and `bmad-code-review` skills available in the environment

## License

MIT
