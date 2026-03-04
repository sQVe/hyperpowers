---
name: managing-br-tasks
description: Use for advanced br operations - splitting tasks mid-flight, merging duplicates, changing dependencies, archiving epics, querying metrics, cross-epic dependencies
---

<skill_overview>
Advanced br operations for managing complex task structures; br is single source of truth, keep it accurate.
</skill_overview>

<rigidity_level>
HIGH FREEDOM - These are operational patterns, not rigid workflows. Adapt operations to your specific situation while following the core principles (keep br accurate, merge don't delete, document changes).
</rigidity_level>

<quick_reference>
| Operation | When | Key Command |
|-----------|------|-------------|
| Split task | Task too large mid-flight | Create subtasks, add deps, close parent |
| Merge duplicates | Found duplicate tasks | Combine designs, move deps, close with reference |
| Change dependencies | Dependencies wrong/changed | `br dep remove` then `br dep add` |
| Archive epic | Epic complete, hide from views | `br close br-X --reason "Archived"` |
| Query metrics | Need status/velocity data | `br list` + filters + `wc -l` |
| Cross-epic deps | Task depends on other epic | `br dep add` works across epics |
| Bulk updates | Multiple tasks need same change | Loop with careful review first |
| Recover mistakes | Accidentally closed/wrong dep | `br update --status` or `br dep remove` |

**Core principle:** Track all work in br, update as you go, never batch updates.
</quick_reference>

<when_to_use>
Use this skill for **advanced** br operations:
- Split task that's too large (discovered mid-implementation)
- Merge duplicate tasks
- Reorganize dependencies after work started
- Archive completed epics (hide from views, keep history)
- Query br for metrics (velocity, progress, bottlenecks)
- Manage cross-epic dependencies
- Bulk status updates
- Recover from br mistakes

**For basic operations:** See skills/common-patterns/br-commands.md (create, show, close, update)
</when_to_use>

<operations>
## Operation 1: Splitting Tasks Mid-Flight

**When:** Task in-progress but turns out too large.

**Example:** Started "Implement authentication" - realize it's 8+ hours of work across multiple areas.

**Process:**

### Step 1: Create subtasks for remaining work

```bash
# Original task br-5 is in-progress
# Already completed: Login form
# Remaining work gets split:

br create "Auth API endpoints" --type task --priority P1 --design "
POST /api/login and POST /api/logout endpoints.
## Success Criteria
- [ ] POST /api/login validates credentials, returns JWT
- [ ] POST /api/logout invalidates token
- [ ] Tests pass
"
# Returns br-12

br create "Session management" --type task --priority P1 --design "
JWT token tracking and validation.
## Success Criteria
- [ ] JWT generated on login
- [ ] Tokens validated on protected routes
- [ ] Token expiration handled
- [ ] Tests pass
"
# Returns br-13

br create "Password hashing" --type task --priority P1 --design "
Secure password hashing with bcrypt.
## Success Criteria
- [ ] Passwords hashed before storage
- [ ] Hash verification on login
- [ ] Tests pass
"
# Returns br-14
```

### Step 2: Set up dependencies

```bash
# Password hashing must be done first
# API endpoints depend on password hashing
br dep add br-12 br-14  # br-12 depends on br-14

# Session management depends on API endpoints
br dep add br-13 br-12  # br-13 depends on br-12

# View tree
br dep tree br-5
```

### Step 3: Update original task and close

```bash
br edit br-5 --design "
Implement user authentication.

## Status
✓ Login form completed (frontend)
✗ Remaining work split into subtasks:
  - br-14: Password hashing (do first)
  - br-12: Auth API endpoints (depends on br-14)
  - br-13: Session management (depends on br-12)

## Success Criteria
- [x] Login form renders
- [ ] See subtasks for remaining criteria
"

br close br-5 --reason "Split into br-12, br-13, br-14"
```

### Step 4: Work on subtasks in order

```bash
br ready  # Shows br-14 (no dependencies)
br update br-14 --status in_progress
# Complete br-14...
br close br-14

# Now br-12 is unblocked
br ready  # Shows br-12
```

---

## Operation 2: Merging Duplicate Tasks

**When:** Discovered two tasks are same thing.

**Example:**
```
br-7: "Add email validation"
br-9: "Validate user email addresses"
^ Duplicates
```

### Step 1: Choose which to keep

Based on:
- Which has more complete design?
- Which has more work done?
- Which has more dependencies?

**Example:** Keep br-7 (more complete)

### Step 2: Merge designs

```bash
br show br-7
br show br-9

# Combine into br-7
br edit br-7 --design "
Add email validation to user creation and update.

## Background
Originally tracked as br-7 and br-9 (now merged).

## Success Criteria
- [ ] Email validated on creation
- [ ] Email validated on update
- [ ] Rejects invalid formats
- [ ] Rejects empty strings
- [ ] Tests cover all cases

## Notes from br-9
Need validation on update, not just creation.
"
```

### Step 3: Move dependencies

```bash
# Check br-9 dependencies
br show br-9

# If br-10 depended on br-9, update to br-7
br dep remove br-10 br-9
br dep add br-10 br-7
```

### Step 4: Close duplicate with reference

```bash
br edit br-9 --design "DUPLICATE: Merged into br-7

This task was duplicate of br-7. All work tracked there."

br close br-9
```

---

## Operation 3: Changing Dependencies

**When:** Dependencies were wrong or requirements changed.

**Example:** br-10 depends on br-8 and br-9, but br-9 got merged and br-10 now also needs br-11.

```bash
# Remove obsolete dependency
br dep remove br-10 br-9

# Add new dependency
br dep add br-10 br-11

# Verify
br dep tree br-1  # If br-10 in epic br-1
br show br-10 | grep "Blocking"
```

**Common scenarios:**
- Discovered hidden dependency during implementation
- Requirements changed mid-flight
- Tasks reordered for better flow

---

## Operation 4: Archiving Completed Epics

**When:** Epic complete, want to hide from default views but keep history.

```bash
# Verify all tasks closed
br list --parent br-1 --status open
# Output: [empty] = all closed

# Archive epic
br close br-1 --reason "Archived - completed Oct 2025"

# Won't show in open listings
br list --status open  # br-1 won't appear

# Still accessible
br show br-1  # Still shows full epic
```

**Use archived for:** Completed epics, shipped features, historical reference
**Use open/in-progress for:** Active work
**Use closed with note for:** Cancelled work (explain why)

---

## Operation 5: Querying for Metrics

### Velocity

```bash
# Tasks closed this week
br list --status closed | grep "closed_at" | grep "2025-10-" | wc -l

# Tasks closed by epic
br list --parent br-1 --status closed | wc -l
```

### Blocked vs Ready

```bash
# Ready to work on
br ready
br ready | grep "^br-" | wc -l

# All open tasks
br list --status open | wc -l

# Blocked = open - ready
```

### Epic Progress

```bash
# Show tree
br dep tree br-1

# Total tasks in epic
br list --parent br-1 | grep "^br-" | wc -l

# Completed tasks
br list --parent br-1 --status closed | grep "^br-" | wc -l

# Percentage = (completed / total) * 100
```

**For detailed metrics guidance:** See [resources/metrics-guide.md](resources/metrics-guide.md)

---

## Operation 6: Cross-Epic Dependencies

**When:** Task in one epic depends on task in different epic.

**Example:**
```
Epic br-1: User Management
  - br-10: User CRUD API

Epic br-2: Order Management
  - br-20: Order creation (needs user API)
```

```bash
# Add cross-epic dependency
br dep add br-20 br-10
# br-20 (in br-2) depends on br-10 (in br-1)

# Check dependencies
br show br-20 | grep "Blocking"

# Check ready tasks
br ready
# Won't show br-20 until br-10 closed
```

**Best practices:**
- Document cross-epic dependencies clearly
- Consider if epics should be merged
- Coordinate if different people own epics

---

## Operation 7: Bulk Status Updates

**When:** Need to update multiple tasks.

**Example:** Mark all test tasks closed after suite complete.

```bash
# Get tasks
br list --parent br-1 --status open | grep "test:" > test-tasks.txt

# Review list
cat test-tasks.txt

# Update each
while read task_id; do
  br close "$task_id"
done < test-tasks.txt

# Verify
br list --parent br-1 --status open | grep "test:"
```

**Use bulk for:**
- Marking completed work closed
- Reopening related tasks
- Updating priorities

**Never bulk:**
- Thoughtless changes
- Hiding problems (closing unfinished tasks)

---

## Operation 8: Recovering from Mistakes

### Accidentally closed task

```bash
br update br-15 --status open
# Or if was in progress
br update br-15 --status in_progress
```

### Wrong dependency

```bash
br dep remove br-10 br-8  # Remove wrong
br dep add br-10 br-9     # Add correct
```

### Undo design changes

```bash
# br has no undo, restore from git
git log -p -- .beads/issues.jsonl | grep -A 50 "br-10"
# Find previous version, copy

br edit br-10 --design "[paste previous]"
```

### Epic structure wrong

1. Create new tasks with correct structure
2. Move work to new tasks
3. Close old tasks with reference
4. Don't delete (keep audit trail)
</operations>

<examples>
<example>
<scenario>Developer closes duplicate without merging information</scenario>

<code>
# Found duplicates
br-7: "Add email validation"
br-9: "Validate user email addresses"

# Developer just closes br-9
br close br-9

# Loses information from br-9's design
# br-9 mentioned validation on update (br-7 didn't)
# Now that requirement is lost
# Work on br-7 completes, but misses update validation
# Bug ships to production
</code>

<why_it_fails>
- Closed duplicate without reading its design
- Lost requirement mentioned only in duplicate
- Information not preserved
- Incomplete implementation ships
- br not accurate source of truth
</why_it_fails>

<correction>
**Correct process:**

```bash
# Read BOTH tasks
br show br-7  # Only mentions validation on creation
br show br-9  # Mentions validation on update too

# Merge information
br edit br-7 --design "
Email validation for user creation and update.

## Background
Merged from br-9.

## Success Criteria
- [ ] Validate on creation (from br-7)
- [ ] Validate on update (from br-9)  ← Preserved!
- [ ] Tests for both cases
"

# Then close duplicate with reference
br edit br-9 --design "DUPLICATE: Merged into br-7"
br close br-9
```

**What you gain:**
- All requirements preserved
- br remains accurate
- No information lost
- Complete implementation
- Audit trail clear
</correction>
</example>

<example>
<scenario>Developer doesn't split large task, struggles through</scenario>

<code>
br-15: "Implement payment processing" (started)

# 3 hours in, developer realizes:
# - Need Stripe API integration (4 hours)
# - Need payment validation (2 hours)
# - Need retry logic (3 hours)
# - Need receipt generation (2 hours)
# Total: 11 more hours!

# Developer thinks: "Too late to split, I'll power through"
# Works 14 hours straight
# Gets exhausted, makes mistakes
# Ships buggy code
# Has to fix in production
</code>

<why_it_fails>
- Didn't split when discovered size
- "Sunk cost" rationalization (already started)
- No clear stopping points
- Exhaustion leads to bugs
- Can't track progress granularly
- If interrupted, hard to resume
</why_it_fails>

<correction>
**Correct approach (split mid-flight):**

```bash
# 3 hours in, stop and split

br edit br-15 --design "
Implement payment processing.

## Status
✓ Completed: Payment form UI (3 hours)
✗ Split remaining work into subtasks:
  - br-20: Stripe API integration
  - br-21: Payment validation
  - br-22: Retry logic
  - br-23: Receipt generation
"

br close br-15 --reason "Split into br-20, br-21, br-22, br-23"

# Create subtasks with dependencies
br create "Stripe API integration" ...  # br-20
br create "Payment validation" ...      # br-21
br create "Retry logic" ...             # br-22
br create "Receipt generation" ...      # br-23

br dep add br-21 br-20  # Validation needs API
br dep add br-22 br-20  # Retry needs API
br dep add br-23 br-22  # Receipts after retry works

# Work on one at a time
br update br-20 --status in_progress
# Complete br-20 (4 hours)
br close br-20

# Take break
# Next day: br-21
```

**What you gain:**
- Clear stopping points (can pause between tasks)
- Track progress granularly
- No exhaustion (spread over days)
- Better quality (not rushed)
- If interrupted, easy to resume
- Each subtask gets proper focus
</correction>
</example>

<example>
<scenario>Developer adds dependency but doesn't update dependent task</scenario>

<code>
# Initial state
br-10: "Add user dashboard" (in progress)
br-15: "Add analytics to dashboard" (blocked on br-10)

# During br-10 implementation, discover need for new API
br create "Analytics API endpoints" ...  # Creates br-20

# Add dependency
br dep add br-15 br-20  # br-15 now depends on br-20 too

# But br-10 completes, closes
br close br-10

# br-15 shows as ready (br-10 closed)
br ready  # Shows br-15

# Developer starts br-15
br update br-15 --status in_progress

# Immediately blocked - needs br-20!
# br-20 not done yet
# Have to stop work on br-15
# Time wasted
</code>

<why_it_fails>
- Added dependency but didn't document in br-15
- br-15's design doesn't mention br-20 requirement
- Appears ready when not actually ready
- Wastes time starting work that's blocked
- Dependencies not obvious from task design
</why_it_fails>

<correction>
**Correct approach:**

```bash
# Create new API task
br create "Analytics API endpoints" ...  # br-20

# Add dependency
br dep add br-15 br-20

# UPDATE br-15 to document new requirement
br edit br-15 --design "
Add analytics to dashboard.

## Dependencies
- br-10: User dashboard (completed)
- br-20: Analytics API endpoints (NEW - discovered during br-10)

## Success Criteria
- [ ] Integrate with analytics API (br-20)
- [ ] Display charts on dashboard
- [ ] Tests pass
"

# Close br-10
br close br-10

# Check ready
br ready  # Does NOT show br-15 (blocked on br-20)

# Work on br-20 first
br update br-20 --status in_progress
# Complete br-20
br close br-20

# NOW br-15 is truly ready
br ready  # Shows br-15
```

**What you gain:**
- Dependencies documented in task design
- Clear why task is blocked
- No false "ready" signals
- Work proceeds in correct order
- No wasted time starting blocked work
</correction>
</example>
</examples>

<critical_rules>
## Rules That Have No Exceptions

1. **Keep br accurate** → Single source of truth for all work
2. **Merge duplicates, don't just close** → Preserve information from both
3. **Split large tasks when discovered** → Not after struggling through
4. **Document dependency changes** → Update task designs when deps change
5. **Update as you go** → Never batch updates "for later"

## Common Excuses

All of these mean: **STOP. Follow the operation properly.**

- "Task too complex to split" (Every task can be broken down)
- "Just close duplicate" (Merge first, preserve information)
- "Won't track this in br" (All work tracked, no exceptions)
- "br is out of date, update later" (Later never comes, update now)
- "This dependency doesn't matter" (Dependencies prevent blocking, they matter)
- "Too much overhead to split" (More overhead to fail huge task)
</critical_rules>

<br_best_practices>
**For detailed guidance on:**
- Task naming conventions
- Priority guidelines (P0-P4)
- Task granularity
- Success criteria
- Dependency management

**See:** [resources/task-naming-guide.md](resources/task-naming-guide.md)
</br_best_practices>

<red_flags>
Watch for these patterns:

- **Multiple in-progress tasks** → Focus on one
- **Tasks stuck in-progress for days** → Blocked? Split it?
- **Many open tasks, no dependencies** → Prioritize!
- **Epics with 20+ tasks** → Too large, split epic
- **Closed tasks, incomplete criteria** → Not done, reopen
</red_flags>

<verification_checklist>
After advanced br operations:

- [ ] br still accurate (reflects reality)
- [ ] Dependencies correct (nothing blocked incorrectly)
- [ ] Duplicate information merged (not lost)
- [ ] Changes documented in task designs
- [ ] Ready tasks are actually unblocked
- [ ] Metrics queries return sensible numbers
- [ ] No orphaned tasks (all part of epics)

**Can't check all boxes?** Review operation and fix issues.
</verification_checklist>

<integration>
**This skill covers:** Advanced br operations

**For basic operations:**
- skills/common-patterns/br-commands.md

**Related skills:**
- hyperpowers:writing-plans (creating epics and tasks)
- hyperpowers:executing-plans (working through tasks)
- hyperpowers:verification-before-completion (closing tasks properly)

**CRITICAL:** Use br CLI commands, never read `.beads/issues.jsonl` directly.
</integration>

<resources>
**Detailed guides:**
- [Metrics guide (cycle time, WIP limits)](resources/metrics-guide.md)
- [Task naming conventions](resources/task-naming-guide.md)

**When stuck:**
- Task seems unsplittable → Ask user how to break it down
- Duplicates complex → Merge designs carefully, don't rush
- Dependencies tangled → Draw diagram, untangle systematically
- br out of sync → Stop everything, update br first
</resources>
