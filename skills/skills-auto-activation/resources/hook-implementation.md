# Complete Hook Implementation for Skills Auto-Activation

## Complete skill-activator.js

**Location:** `~/.claude/hooks/user-prompt-submit/skill-activator.js`

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

const CONFIG = {
    rulesPath: process.env.SKILL_RULES || path.join(process.env.HOME, '.claude/skill-rules.json'),
    maxSkills: 3,
    debugMode: process.env.DEBUG === 'true'
};

function loadRules() {
    try {
        return JSON.parse(fs.readFileSync(CONFIG.rulesPath, 'utf8'));
    } catch (error) {
        if (CONFIG.debugMode) console.error('Failed to load skill rules:', error.message);
        return {};
    }
}

function readPrompt() {
    return new Promise((resolve) => {
        let data = '';
        process.stdin.on('data', chunk => data += chunk);
        process.stdin.on('end', () => {
            try {
                resolve(JSON.parse(data));
            } catch (error) {
                resolve({ text: '' });
            }
        });
    });
}

function analyzePrompt(promptText, rules) {
    const lowerText = promptText.toLowerCase();
    const activated = [];

    for (const [skillName, config] of Object.entries(rules)) {
        let matched = false;
        let matchReason = '';

        if (config.promptTriggers?.keywords) {
            for (const keyword of config.promptTriggers.keywords) {
                if (lowerText.includes(keyword.toLowerCase())) {
                    matched = true;
                    matchReason = `keyword: "${keyword}"`;
                    break;
                }
            }
        }

        if (!matched && config.promptTriggers?.intentPatterns) {
            for (const pattern of config.promptTriggers.intentPatterns) {
                try {
                    if (new RegExp(pattern, 'i').test(promptText)) {
                        matched = true;
                        matchReason = `intent pattern: "${pattern}"`;
                        break;
                    }
                } catch (error) {
                    if (CONFIG.debugMode) console.error(`Invalid pattern "${pattern}":`, error.message);
                }
            }
        }

        if (matched) {
            activated.push({
                skill: skillName,
                priority: config.priority || 'medium',
                reason: matchReason,
                type: config.type || 'general'
            });
        }
    }

    const priorityOrder = { high: 0, medium: 1, low: 2 };
    activated.sort((a, b) => {
        const priorityDiff = priorityOrder[a.priority] - priorityOrder[b.priority];
        if (priorityDiff !== 0) return priorityDiff;
        const typeOrder = { process: 0, domain: 1, general: 2 };
        return (typeOrder[a.type] || 2) - (typeOrder[b.type] || 2);
    });

    return activated.slice(0, CONFIG.maxSkills);
}

function generateContext(skills) {
    if (skills.length === 0) return null;

    const lines = [
        '',
        '━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━',
        '🎯 SKILL ACTIVATION CHECK',
        '━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━',
        '',
        'Relevant skills for this prompt:',
        ''
    ];

    for (const skill of skills) {
        const emoji = skill.priority === 'high' ? '⭐' : skill.priority === 'medium' ? '📌' : '💡';
        lines.push(`${emoji} **${skill.skill}** (${skill.priority} priority)`);
        if (CONFIG.debugMode) lines.push(`   Matched: ${skill.reason}`);
    }

    lines.push('');
    lines.push('Before responding, check if any of these skills should be used.');
    lines.push('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');
    lines.push('');

    return lines.join('\n');
}

async function main() {
    try {
        const rules = loadRules();

        if (Object.keys(rules).length === 0) {
            console.log(JSON.stringify({ decision: 'approve' }));
            return;
        }

        const prompt = await readPrompt();

        if (!prompt.text || prompt.text.trim() === '') {
            console.log(JSON.stringify({ decision: 'approve' }));
            return;
        }

        const activatedSkills = analyzePrompt(prompt.text, rules);

        if (activatedSkills.length > 0) {
            const context = generateContext(activatedSkills);
            if (CONFIG.debugMode) console.error('Activated skills:', activatedSkills.map(s => s.skill).join(', '));
            console.log(JSON.stringify({ decision: 'approve', additionalContext: context }));
        } else {
            console.log(JSON.stringify({ decision: 'approve' }));
        }
    } catch (error) {
        if (CONFIG.debugMode) console.error('Hook error:', error.message, error.stack);
        console.log(JSON.stringify({ decision: 'approve' }));
    }
}

main();
```

Make it executable: `chmod +x ~/.claude/hooks/user-prompt-submit/skill-activator.js`

## File-based triggers extension

To check file paths alongside prompt text, extend the hook with these functions:

```javascript
function getRecentFiles(prompt) {
    return prompt.files || [];
}

function checkFileTriggers(files, config) {
    if (!files || files.length === 0 || !config.fileTriggers) return false;

    if (config.fileTriggers.pathPatterns) {
        for (const file of files) {
            for (const pattern of config.fileTriggers.pathPatterns) {
                if (globToRegex(pattern).test(file)) return true;
            }
        }
    }

    // Content patterns are better checked in a PostToolUse hook for performance
    return false;
}

function globToRegex(glob) {
    const regex = glob
        .replace(/\*\*/g, '___DOUBLE_STAR___')
        .replace(/\*/g, '[^/]*')
        .replace(/___DOUBLE_STAR___/g, '.*')
        .replace(/\?/g, '.');
    return new RegExp(`^${regex}$`);
}
```

Call `checkFileTriggers(getRecentFiles(prompt), config)` alongside `analyzePrompt` and merge results.

## Multi-hook configuration

The skill activator works alongside other hooks. Use numeric prefixes to control order:

```json
{
  "hooks": [
    {
      "event": "UserPromptSubmit",
      "command": "~/.claude/hooks/user-prompt-submit/00-log-prompt.sh",
      "description": "Log prompts for analysis",
      "blocking": false
    },
    {
      "event": "UserPromptSubmit",
      "command": "~/.claude/hooks/user-prompt-submit/10-skill-activator.js",
      "description": "Activate relevant skills",
      "blocking": false,
      "timeout": 1000
    }
  ]
}
```

## Testing

```bash
# Keyword match
echo '{"text": "How do I create a new API endpoint?"}' | \
    node ~/.claude/hooks/user-prompt-submit/skill-activator.js

# Multiple skills
echo '{"text": "Write a test for the API endpoint"}' | \
    node ~/.claude/hooks/user-prompt-submit/skill-activator.js

# Debug mode
DEBUG=true echo '{"text": "How do I create a component?"}' | \
    node ~/.claude/hooks/user-prompt-submit/skill-activator.js 2>&1
```

## Maintenance

```bash
# Check activation frequency
grep "Activated skills" ~/.claude/hooks/debug.log | sort | uniq -c

# Track rules in git
cd ~/.claude && git add skill-rules.json hooks/ && git commit -m "Update skill rules"
```

**Performance targets:** keyword matching <50ms, intent patterns <200ms, total <500ms.

When adding new skills: add to skill-rules.json, test with sample prompts, observe for false positives, refine.
