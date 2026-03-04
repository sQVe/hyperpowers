# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Hyperpowers is a Claude Code plugin that provides structured workflows, best practices, and specialized agents for software development. It's a plugin system that adds skills (reusable workflows), slash commands (quick access to workflows), specialized agents (domain-specific task handlers), and hooks (automatic behaviors).

Inspired by [obra/superpowers](https://github.com/obra/superpowers).

## CRITICAL: Understanding User Requests in This Repository

User reports from other sessions are requests to **improve the plugin**, not to debug that session. Improve skills, hooks, agents, commands to prevent the pattern from recurring. Never try to investigate past sessions — you cannot access them.

| What user says | What they actually want |
|----------------|------------------------|
| "Claude truncated the br task" | Create hook to block br truncation |
| "Claude edited pre-commit with sed" | Create hook to block pre-commit modifications |
| "The test-runner agent didn't activate" | Improve skill-rules.json triggers |
| "Claude ignored the skill" | Improve skill description or add hook |
| "This caused incomplete implementation" | Add blocking hook to prevent pattern |

## Plugin Structure

The repository is organized as follows:

- **skills/** - Reusable workflow definitions (each in its own directory with SKILL.md)
- **commands/** - Slash command definitions that invoke skills
- **agents/** - Specialized subagent prompts (code-reviewer, codebase-investigator, internet-researcher, test-runner)
- **hooks/** - Automatic behaviors triggered by events
- **.claude-plugin/** - Plugin metadata (plugin.json)

## Key Architecture Concepts

### Skills System

Skills are detailed workflow instructions stored in `skills/*/SKILL.md`. Each skill follows a specific pattern:

1. **Frontmatter** - YAML metadata (name, description)
2. **Overview** - Core principle and context
3. **The Process** - Step-by-step workflow with exact commands
4. **Common Rationalizations** - Mistakes to avoid
5. **Red Flags** - Anti-patterns to prevent
6. **Integration** - How this skill calls/is called by others

**Critical distinction:**
- Some skills are **rigid processes** (TDD, verification) - follow exactly, no adaptation
- Some skills are **flexible patterns** (architecture, naming) - adapt principles to context
- The skill itself tells you which type it is

### Skill Invocation Pattern

Skills are invoked through slash commands that expand to prompts. The flow is:

1. User types `/hyperpowers:write-plan`
2. Command file (`commands/write-plan.md`) expands with instruction: "Use the writing-plans skill exactly as written"
3. Claude uses the Skill tool to load `skills/writing-plans/SKILL.md`
4. Claude follows the skill's detailed instructions

### br (beads_rust) integration

Many skills integrate with `br` (a task management tool). The workflows expect:

- **Epics** - High-level features/initiatives (created by writing-plans)
- **Tasks** - Specific implementation steps (created by writing-plans, executed by executing-plans)
- **Dependencies** - Task relationships (blocking, parent-child)
- **Status tracking** - Open, in-progress, done, ready

**Note:** `br` is non-invasive and never executes git commands. After `br sync --flush-only`, you must manually run `git add .beads/ && git commit`.

Common br commands:
```bash
br list --type epic --status open       # Find open epics
br ready                                 # Show ready tasks
br show br-1                            # Show task details
br dep tree br-1                        # Show task tree
br update br-3 --status in_progress     # Update task status
```

### Agent System

Specialized agents run in separate contexts to handle specific tasks:

1. **test-runner** (uses Haiku) - Runs tests/hooks/commits, returns only summary + failures to keep context clean
2. **code-reviewer** - Reviews implementations against plans and coding standards
3. **codebase-investigator** - Explores codebase state and patterns when planning/designing
4. **internet-researcher** - Researches APIs, libraries, docs when planning/designing

**Critical pattern:** Agents keep verbose output (test results, formatting diffs) in their own context, returning only essential info to the main conversation.

### Common Patterns Location

`skills/common-patterns/br-commands.md` — standard br command examples, referenced by executing-plans, writing-plans, managing-br-tasks.

## Core Workflows

- Feature: brainstorming → writing-plans → executing-plans → review-implementation → finishing-branch
- Bug fix: debugging-with-tools → fixing-bugs → verification-before-completion
- Refactor: refactoring-safely (change→test→commit cycle)
- TDD: test-driven-development (RED-GREEN-REFACTOR, mandatory)
- Verify: verification-before-completion (mandatory before claiming done)

## Development Commands

This is a plugin repository with no build system - it's pure markdown files. There are no tests, linters, or build commands.

### Testing Skills

When creating or modifying skills, use the `writing-skills` skill which applies TDD to documentation:

1. Test skill with subagents BEFORE writing final version
2. Iterate until the skill is bulletproof against rationalization
3. Document what failure modes you tested

### Publishing

The plugin is published to the Claude Code marketplace:

```bash
# In the marketplace system (not in this repo)
claude plugin install hyperpowers@withzombies-hyper
```

Version is tracked in `.claude-plugin/plugin.json`.

## File Naming Conventions

- Skills: `skills/<skill-name>/SKILL.md` (frontmatter + content)
- Commands: `commands/<command-name>.md` (frontmatter + brief invocation)
- Agents: `agents/<agent-name>.md` (frontmatter + detailed prompt)
- Common patterns: `skills/common-patterns/<pattern-name>.md`

## Important Notes

- This plugin is loaded automatically when installed; there's no runtime execution
- Skills are documentation that Claude reads at runtime, not executable code
- Changes to skill files take effect immediately in new conversations
- The test-runner agent uses Haiku model for cost efficiency
- `sre-task-refinement` is a skill, not a subagent type
- If you see `Agent type 'hyperpowers:sre-task-refinement' not found`, use `/hyperpowers:sre-task-refinement` or invoke the skill directly
- The sre-task-refinement skill uses Opus 4.1 for deep analysis
- Most other operations use the default model (Sonnet)

## Contributing Guidelines

From writing-skills skill:

1. Test skills with subagents before finalizing
2. Iterate until bulletproof against rationalization
3. Follow the skill structure pattern (Overview, Process, Rationalizations, Red Flags, Integration)
4. Reference common-patterns instead of duplicating content
5. Be explicit about whether skill is rigid (must follow exactly) or flexible (adapt principles)
