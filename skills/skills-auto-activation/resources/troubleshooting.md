# Troubleshooting skills auto-activation

## Hook not running

Check `~/.claude/hooks.json` has the `UserPromptSubmit` entry, then test manually:

```bash
echo '{"text": "test backend endpoint"}' | \
  node ~/.claude/hooks/user-prompt-submit/skill-activator.js
```

If the file isn't executable: `chmod +x ~/.claude/hooks/user-prompt-submit/skill-activator.js`

Check logs at `~/.claude/logs/hooks.log`. Increase `timeout` in hooks.json if the hook is timing out (default 1000ms may be too low on slow machines).

## No skills activating

Enable debug mode to diagnose:

```bash
DEBUG=true echo '{"text": "create backend controller"}' | \
  node ~/.claude/hooks/user-prompt-submit/skill-activator.js 2>&1
```

"No rules loaded" means skill-rules.json isn't found — check `~/.claude/skill-rules.json` exists and is valid JSON (`cat ~/.claude/skill-rules.json | jq '.'`). "No skills activated" means keywords don't match — try adding keyword variations or broadening intent patterns.

## Wrong skills activating (false positives)

Debug shows which keyword/pattern matched. Tighten by using multi-word keywords (`"API endpoint"` instead of `"api"`), adding negative lookahead patterns (`(?!.*test).*backend`), or lowering priority so the skill gets deprioritized when others match. Reduce `maxSkills` in CONFIG to cap how many can fire at once.

## Hook is slow

Measure with `time echo '{"text": "test"}' | node ~/.claude/hooks/user-prompt-submit/skill-activator.js`. Target: <500ms total.

Fix slow patterns by anchoring regex (`(create|build).*(endpoint|route)` instead of `.*create.*backend.*endpoint.*`). Cache compiled regex by building a `Map` of `new RegExp(pattern, 'i')` at startup rather than inside the loop. Remove rarely-used low-priority rules.

## Skills activated but Claude ignores them

The hook is working — the problem is Claude ignoring the suggestion. Try strengthening the activation message:

```javascript
'⚠️ IMPORTANT: You MUST check these skills before responding. Use the Skill tool to load them.'
```

Also ensure skill descriptions in their YAML frontmatter are specific (name the exact files, patterns, or technologies they cover), and reference skills in the project's CLAUDE.md.

## Hook crashes Claude Code

Check `~/.claude/logs/hooks.log`. Common causes: unhandled promise rejection, missing `try/catch` around `main()`, or blocking I/O. The implementation in hook-implementation.md wraps everything in `try/catch` and always returns `{ decision: 'approve' }` on error — use that pattern. Add a self-timeout if needed:

```javascript
const timeout = setTimeout(() => {
    console.log(JSON.stringify({ decision: 'approve' }));
    process.exit(0);
}, 900);
```

## Context overload (too many activations)

Set `maxSkills: 1` in CONFIG, shorten the activation message to a single line (`🎯 Use: ${skills.map(s => s.skill).join(', ')}`), or remove low-priority rules from skill-rules.json.

## Inconsistent activation (same prompt, different results)

This is expected if prompt wording varies. Add keyword variations (`"back-end"`, `"server side"`) and bidirectional intent patterns (`"create.*backend"` and `"backend.*create"`). Log prompts to identify the pattern:

```javascript
fs.appendFileSync(
    path.join(process.env.HOME, '.claude/prompt-log.txt'),
    `${new Date().toISOString()} | ${prompt.text}\n`
);
```

## Checklist before asking for help

- [ ] Hook runs manually without errors
- [ ] skill-rules.json is valid JSON
- [ ] Node.js v18+ installed
- [ ] Debug mode shows expected behavior
- [ ] Tested with simplified single-rule configuration
- [ ] Checked `~/.claude/logs/hooks.log`
