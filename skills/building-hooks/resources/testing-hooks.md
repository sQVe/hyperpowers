# Testing hooks

Strategies beyond what the main SKILL.md covers (isolation testing, mock data, debug flags).

## Unit testing hook functions

Extract testable functions from hook scripts and test them separately.

**Bash:**

```bash
# hook-functions.sh
validate_file_path() {
    local path="$1"
    [ -z "$path" ] || [ "$path" == "null" ] && return 1
    [[ ! "$path" =~ ^/ ]] && return 1
    [ ! -f "$path" ] && return 1
    return 0
}
```

```bash
# test-functions.sh
source ./hook-functions.sh

touch /tmp/test-file.txt
validate_file_path "/tmp/test-file.txt" && echo "✅ valid path" || echo "❌ valid path"
validate_file_path "" && echo "❌ empty path" || echo "✅ empty path"
validate_file_path "relative/path.txt" && echo "❌ relative path" || echo "✅ relative path"
rm /tmp/test-file.txt
```

**JavaScript:**

```javascript
// test.js
const assert = require('assert');

const testRules = {
    'backend-dev': {
        priority: 'high',
        promptTriggers: { keywords: ['backend', 'API', 'endpoint'] }
    }
};

function testKeywordMatching() {
    const result = analyzePrompt('How do I create a backend endpoint?', testRules);
    assert.equal(result.length, 1);
    assert.equal(result[0].skill, 'backend-dev');
    console.log('✅ Keyword matching');
}

function testNoMatch() {
    const result = analyzePrompt('How do I write Python?', testRules);
    assert.equal(result.length, 0);
    console.log('✅ No match');
}

function testCaseInsensitive() {
    const result = analyzePrompt('BACKEND endpoint', testRules);
    assert.equal(result.length, 1);
    console.log('✅ Case insensitive');
}

testKeywordMatching();
testNoMatch();
testCaseInsensitive();
```

## Integration testing with mock events

Mock event shapes for each hook type:

```json
// PostToolUse
{
  "event": "PostToolUse",
  "tool": {
    "name": "Edit",
    "input": { "file_path": "/Users/test/project/src/file.ts" }
  },
  "result": { "success": true }
}
```

```json
// UserPromptSubmit
{ "event": "UserPromptSubmit", "text": "How do I create a new API endpoint?" }
```

```json
// Stop
{ "event": "Stop", "sessionId": "abc123", "messageCount": 10 }
```

Testing a hook with mock events:

```bash
# test-hook.sh
test_edit_tracker() {
    export LOG_FILE="/tmp/test-edit-log.txt"
    rm -f "$LOG_FILE"

    echo '{"tool": {"name": "Edit", "input": {"file_path": "/tmp/test-file.ts"}}}' | \
        bash hooks/post-tool-use/01-track-edits.sh

    grep -q "test-file.ts" "$LOG_FILE" && echo "✅ Edit tracker" || echo "❌ Edit tracker"
}

test_edit_tracker
```

Testing JavaScript hooks via subprocess:

```javascript
const { execSync } = require('child_process');

function runSkillActivator(prompt) {
    const result = execSync(
        'node hooks/user-prompt-submit/skill-activator.js',
        {
            input: JSON.stringify({ text: prompt }),
            encoding: 'utf8',
            env: { ...process.env, SKILL_RULES: './test-skill-rules.json' }
        }
    );
    return JSON.parse(result);
}

const result = runSkillActivator('How do I create a backend endpoint?');
result.additionalContext?.includes('backend')
    ? console.log('✅ Backend activation')
    : console.log('❌ Backend activation');
```

## Regression suite

```bash
#!/bin/bash
# test/regression-test.sh

TEST_DIR="/tmp/hook-tests"

setup() {
    mkdir -p "$TEST_DIR"
    export LOG_FILE="$TEST_DIR/edit-log.txt"
    export PROJECT_ROOT="$TEST_DIR/projects"
    mkdir -p "$PROJECT_ROOT"
}

teardown() {
    rm -rf "$TEST_DIR"
}

test_edit_tracker_logs() {
    echo '{"tool": {"name": "Edit", "input": {"file_path": "/test/file.ts"}}}' | \
        bash hooks/post-tool-use/01-track-edits.sh

    grep -q "file.ts" "$LOG_FILE" && echo "✅ Edit tracker" && return 0
    echo "❌ Edit tracker" && return 1
}

test_formatter_missing_prettier() {
    mkdir -p "$PROJECT_ROOT/no-prettier"
    echo 'const x=1' > "$PROJECT_ROOT/no-prettier/file.js"
    echo "2025-01-15 10:00:00 | no-prettier | $PROJECT_ROOT/no-prettier/file.js" > "$LOG_FILE"

    bash hooks/stop/30-format-code.sh 2>&1 && echo "✅ Formatter handles missing prettier" && return 0
    echo "❌ Formatter handles missing prettier" && return 1
}

run_all_tests() {
    setup
    local failed=0

    test_edit_tracker_logs || ((failed++))
    test_formatter_missing_prettier || ((failed++))

    teardown

    [ $failed -eq 0 ] && echo "✅ All tests passed" && return 0
    echo "❌ $failed test(s) failed" && return 1
}

run_all_tests
```

## Performance benchmarking

```bash
#!/bin/bash
# test/benchmark-hook.sh

ITERATIONS=10
HOOK_PATH="hooks/stop/build-checker.sh"
total_time=0

for i in $(seq 1 $ITERATIONS); do
    start=$(date +%s%N)
    bash "$HOOK_PATH" > /dev/null 2>&1
    end=$(date +%s%N)
    elapsed=$(( (end - start) / 1000000 ))
    total_time=$(( total_time + elapsed ))
    echo "Iteration $i: ${elapsed}ms"
done

average=$(( total_time / ITERATIONS ))
echo "Average: ${average}ms"

[ $average -gt 2000 ] && echo "⚠️  Hook is slow (>2s)" && exit 1
echo "✅ Performance acceptable"
```

**Targets:** non-blocking <2s, blocking <5s, UserPromptSubmit <1s, PostToolUse <500ms.

## CI/CD integration

```yaml
# .github/workflows/test-hooks.yml
name: Test Claude Code Hooks

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '18'

      - name: Run hook tests
        run: bash test/regression-test.sh

      - name: Run performance tests
        run: bash test/benchmark-hook.sh
```

Add pre-commit gate to catch regressions before they ship:

```bash
#!/bin/bash
# .git/hooks/pre-commit

bash test/regression-test.sh && exit 0
echo "❌ Hook tests failed — fix before committing"
exit 1
```
