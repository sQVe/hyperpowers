---
name: using-hyper
description: Use when starting any conversation - establishes mandatory workflows for finding and using skills
---

<EXTREMELY_IMPORTANT>
If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST read the skill.

**IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT.**

This is not negotiable. This is not optional. You cannot rationalize your way out of this.
</EXTREMELY_IMPORTANT>

<skill_overview>
Skills are proven workflows; if one exists for your task, using it is mandatory, not optional.
</skill_overview>

<rigidity_level>
HIGH FREEDOM - The meta-process (check for skills, use Skill tool, announce usage) is rigid, but each individual skill defines its own rigidity level.
</rigidity_level>

<quick_reference>
**Before responding to ANY user message:**

1. List available skills mentally
2. Ask: "Does ANY skill match this request?"
3. If yes → Use Skill tool to load the skill file
4. Announce which skill you're using
5. Follow the skill exactly as written

**Skill has checklist?** Create TodoWrite for every item.

**Finding a relevant skill = mandatory to use it.**
</quick_reference>

<when_to_use>
Applies to every task in every conversation. If any skill might apply, use it.
</when_to_use>

<the_process>
## 1. MANDATORY FIRST RESPONSE PROTOCOL

Before responding to ANY user message, complete this checklist:

1. ☐ List available skills in your mind
2. ☐ Ask yourself: "Does ANY skill match this request?"
3. ☐ If yes → Use the Skill tool to read and run the skill file
4. ☐ Announce which skill you're using
5. ☐ Follow the skill exactly

**Responding WITHOUT completing this checklist = automatic failure.**

---

## 2. Execute Skills with the Skill Tool

**Always use the Skill tool to load skills.** Never rely on memory.

```
Skill tool: "hyperpowers:test-driven-development"
```

**Why:**
- Skills evolve - you need the current version
- Using the tool ensures you get the full skill content
- Confirms to user you're following the skill

---

## 3. Announce Skill Usage

Before using a skill, announce it:

**Format:** "I'm using [Skill Name] to [what you're doing]."

**Examples:**
- "I'm using hyperpowers:brainstorming to refine your idea into a design."
- "I'm using hyperpowers:test-driven-development to implement this feature."
- "I'm using hyperpowers:debugging-with-tools to investigate this error."

**Why:** Transparency helps user understand your process and catch errors early. Confirms you actually read the skill.

---

## 4. Follow Mandatory Workflows

**Before writing ANY code:**
- Use hyperpowers:brainstorming to refine requirements
- Use hyperpowers:writing-plans to create detailed plan
- **STOP** — do NOT start executing. User must explicitly run `/hyperpowers:execute-plan`

**When user explicitly requests implementation:**
- Use hyperpowers:executing-plans to implement iteratively

**When implementing:**
- Use hyperpowers:test-driven-development (RED-GREEN-REFACTOR cycle)
- Use hyperpowers:verification-before-completion before claiming done

**When debugging:**
- Use hyperpowers:debugging-with-tools (tools first, fixes second)
- Use hyperpowers:fixing-bugs (complete workflow from discovery to closure)

**User instructions describe WHAT to do, not HOW.** "Add X" means use brainstorming, TDD, verification. Not permission to skip workflows.

---

## 5. Create TodoWrite for Skill Checklists

If a skill has a checklist, YOU MUST create TodoWrite todos for EACH item.

**Don't:**
- Work through checklist mentally
- Skip creating todos "to save time"
- Batch multiple items into one todo
- Mark complete without doing them

**Why:** Checklists without TodoWrite tracking = steps get skipped. Every time. The overhead is tiny compared to missing steps.

**Example:**

```
Skill has verification checklist:
- [ ] All tests pass
- [ ] No linter warnings
- [ ] br task updated

TodoWrite todos:
1. Run all tests and verify they pass
2. Run linter and verify no warnings
3. Update br task with completion status
```
</the_process>

<examples>
<example>
<scenario>User asks to implement a new feature</scenario>

<code>
User: "Add a user profile page with avatar upload"

Claude (without using-hyper):
"Sure! Let me start implementing..."
[Writes code without brainstorming or planning]
</code>

<why_it_fails>
- Skipped brainstorming (requirements unclear, edge cases missed)
- Skipped writing-plans (no plan for user to review)
- Skipped TDD (tests written after, if at all)
</why_it_fails>

<correction>
**Correct approach:**

Claude: "I'm using hyperpowers:brainstorming to refine your requirements."
[Asks Socratic questions about avatar size limits, formats, storage]
[Creates refined requirements]

Claude: "Now I'm using hyperpowers:writing-plans to create an implementation plan."
[Creates br epic with tasks]

Claude: "Plan complete. Run `/hyperpowers:execute-plan` to begin implementation."
[STOPS — waits for explicit user command before writing any code]
</correction>
</example>
</examples>

<critical_rules>
## Rules That Have No Exceptions

1. **Check for relevant skills BEFORE any task** → If skill exists, use it (not optional)
2. **Use Skill tool to load skills** → Never rely on memory (skills evolve)
3. **Announce skill usage** → Transparency helps catch errors early
4. **Follow mandatory workflows** → brainstorming before coding, TDD for implementation, verification before claiming done
5. **Create TodoWrite for checklists** → Mental tracking = skipped steps

## Common Rationalizations

All of these mean: **STOP. Check for and use the relevant skill.**

- "This is just a simple question" (Questions are tasks. Check for skills.)
- "I can check git/files quickly" (Files lack context. Check for skills.)
- "Let me gather information first" (Skills tell you HOW to gather. Check for skills.)
- "This doesn't need a formal skill" (If skill exists, use it. Not optional.)
- "I remember this skill" (Skills evolve. Use Skill tool to load current version.)
- "This doesn't count as a task" (Taking action = task. Check for skills.)
- "The skill is overkill for this" (Skills exist because "simple" becomes complex.)
- "I'll just do this one thing first" (Check for skills BEFORE doing anything.)
- "Instruction was specific so I can skip brainstorming" (Specific instructions = WHAT, not HOW. Use workflows.)
</critical_rules>

<understanding_rigidity>
## Rigid Skills (Follow Exactly)

These have LOW FREEDOM - follow the exact process:

- hyperpowers:test-driven-development (RED-GREEN-REFACTOR cycle)
- hyperpowers:verification-before-completion (evidence before claims)
- hyperpowers:executing-plans (continuous execution, substep tracking)

## Flexible Skills (Adapt Principles)

These have HIGH FREEDOM - adapt core principles to context:

- hyperpowers:brainstorming (Socratic method, but questions vary)
- hyperpowers:managing-br-tasks (operations adapt to project)
- hyperpowers:sre-task-refinement (corner case analysis, but depth varies)

**The skill itself tells you its rigidity level.** Check `<rigidity_level>` section.
</understanding_rigidity>



<integration>
**This skill calls:**
- ALL other skills (meta-skill that triggers appropriate skill usage)

**Critical workflows this establishes:**
- hyperpowers:brainstorming (before writing code)
- hyperpowers:test-driven-development (during implementation)
- hyperpowers:verification-before-completion (before claiming done)
</integration>

<resources>
**Available skills:**
- See skill descriptions in Skill tool's "Available Commands" section
- Each skill's description shows when to use it

**When unsure if skill applies:**
- If there's even 1% chance it applies → use it
- Better to load and decide "not needed" than to skip and fail
- Skills are optimized, loading them is cheap
</resources>
