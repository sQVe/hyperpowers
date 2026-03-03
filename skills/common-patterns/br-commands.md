# br (beads_rust) Command Reference

**Note:** `br` is non-invasive and never executes git commands. After `br sync --flush-only`, you must manually run `git add .beads/ && git commit`.

Common br commands used across multiple skills. Reference this instead of duplicating.

## Reading Issues

```bash
# Show single issue with full design
br show br-3

# List all open issues
br list --status open

# List closed issues
br list --status closed

# Show dependency tree for an epic
br dep tree br-1

# Find tasks ready to work on (no blocking dependencies)
br ready

# List tasks in a specific epic
br list --parent br-1
```

## Creating Issues

```bash
# Create epic
br create "Epic: Feature Name" \
  --type epic \
  --priority [0-4] \
  --design "## Goal
[Epic description]

## Success Criteria
- [ ] All phases complete
..."

# Create feature/phase
br create "Phase 1: Phase Name" \
  --type feature \
  --priority [0-4] \
  --design "[Phase design]"

# Create task
br create "Task Name" \
  --type task \
  --priority [0-4] \
  --design "[Task design]"
```

## Updating Issues

```bash
# Update issue design (detailed description)
br update br-3 --design "$(cat <<'EOF'
[Complete updated design]
EOF
)"
```

**IMPORTANT**: Use `--design` for the full detailed description, NOT `--description` (which is title only).

## Managing Status

```bash
# Start working on task
br update br-3 --status in_progress

# Complete task
br close br-3

# Reopen task
br update br-3 --status open
```

**Common Mistakes:**
```bash
# ❌ WRONG - br status shows database overview, doesn't change status
br status br-3 --status in_progress

# ✅ CORRECT - use br update to change status
br update br-3 --status in_progress

# ❌ WRONG - using hyphens in status values
br update br-3 --status in-progress

# ✅ CORRECT - use underscores in status values
br update br-3 --status in_progress

# ❌ WRONG - 'done' is not a valid status
br update br-3 --status done

# ✅ CORRECT - use br close to complete
br close br-3
```

**Valid status values:** `open`, `in_progress`, `blocked`, `closed`

## Managing Dependencies

```bash
# Add blocking dependency (LATER depends on EARLIER)
# Syntax: br dep add <dependent> <dependency>
br dep add br-3 br-2  # br-3 depends on br-2 (do br-2 first)

# Add parent-child relationship
# Syntax: br dep add <child> <parent> --type parent-child
br dep add br-3 br-1 --type parent-child  # br-3 is child of br-1

# View dependency tree
br dep tree br-1
```

## Syncing

```bash
br sync --flush-only
git add .beads/
git commit -m "sync beads"
```

## Commit Message Format

Reference br task IDs in commits (use hyperpowers:test-runner agent):

```bash
# Use test-runner agent to avoid pre-commit hook pollution
Dispatch hyperpowers:test-runner agent: "Run: git add <files> && git commit -m 'feat(br-3): implement feature

Implements step 1 of br-3: Task Name
'"
```

## Common Queries

```bash
# Check if all tasks in epic are closed
br list --status open --parent br-1
# Output: [empty] = all closed

# See what's blocking current work
br ready  # Shows only unblocked tasks

# Find all in-progress work
br list --status in_progress
```
