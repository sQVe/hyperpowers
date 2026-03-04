# Hook patterns library

Reusable patterns not covered in the main SKILL.md examples.

## Rate limiting

Prevent hooks from running too frequently (useful for expensive operations).

```bash
#!/bin/bash

RATE_LIMIT_FILE="/tmp/hook-last-run"
MIN_INTERVAL=30  # seconds

should_run() {
    [ ! -f "$RATE_LIMIT_FILE" ] && return 0

    local last_run=$(cat "$RATE_LIMIT_FILE")
    local elapsed=$(( $(date +%s) - last_run ))

    if [ "$elapsed" -lt "$MIN_INTERVAL" ]; then
        echo "Skipping (ran ${elapsed}s ago)"
        return 1
    fi
}

mark_run() {
    date +%s > "$RATE_LIMIT_FILE"
}

if should_run; then
    perform_expensive_operation
    mark_run
fi

echo '{}'
```

## Smart caching

Cache results keyed by file modification time to skip redundant work.

```bash
#!/bin/bash

CACHE_DIR="$HOME/.claude/hook-cache"
mkdir -p "$CACHE_DIR"

cache_key() {
    local file="$1"
    echo -n "$file:$(stat -f %m "$file" 2>/dev/null || stat -c %Y "$file")" | md5sum | cut -d' ' -f1
}

check_cache() {
    local key=$(cache_key "$1")
    local cache_file="$CACHE_DIR/$key"
    [ -f "$cache_file" ] && cat "$cache_file" && return 0
    return 1
}

update_cache() {
    local key=$(cache_key "$1")
    echo "$2" > "$CACHE_DIR/$key"
    find "$CACHE_DIR" -type f -mtime +1 -delete 2>/dev/null
}

if cached=$(check_cache "$file_path"); then
    echo "Cache hit: $cached"
else
    result=$(expensive_operation "$file_path")
    update_cache "$file_path" "$result"
    echo "Computed: $result"
fi
```

## Parallel execution

Run multiple checks simultaneously to stay within time budget.

```bash
#!/bin/bash

run_parallel_checks() {
    local pids=()

    check_typescript &
    pids+=($!)

    check_eslint &
    pids+=($!)

    local exit_code=0
    for pid in "${pids[@]}"; do
        wait "$pid" || exit_code=1
    done

    return $exit_code
}

check_typescript() {
    npx tsc --noEmit > /tmp/tsc-output.txt 2>&1
}

check_eslint() {
    npx eslint . > /tmp/eslint-output.txt 2>&1
}

if run_parallel_checks; then
    echo "✅ All checks passed"
else
    echo "⚠️  Some checks failed"
    cat /tmp/tsc-output.txt
fi

echo '{}'
```

## Hook coordination

Share state between hooks via a temp file.

```bash
# Hook 1: write state
STATE_FILE="/tmp/hook-state.json"
jq -n \
    --arg timestamp "$(date +%s)" \
    --arg files "$files_edited" \
    '{lastRun: $timestamp, filesEdited: ($files | split(","))}' \
    > "$STATE_FILE"
echo '{}'
```

```bash
# Hook 2: read state
STATE_FILE="/tmp/hook-state.json"
if [ -f "$STATE_FILE" ]; then
    files=$(jq -r '.filesEdited[]' "$STATE_FILE")
    for file in $files; do
        process_file "$file"
    done
fi
echo '{}'
```

## Error accumulation

Collect multiple errors before reporting so output is unified.

```bash
#!/bin/bash

ERRORS=()

add_error() { ERRORS+=("$1"); }

report_errors() {
    if [ ${#ERRORS[@]} -eq 0 ]; then
        echo "✅ No errors found"
        return 0
    fi

    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "⚠️  Found ${#ERRORS[@]} issue(s):"
    local i=1
    for error in "${ERRORS[@]}"; do
        echo "$i. $error"
        ((i++))
    done
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    return 1
}

run_typescript_check || add_error "TypeScript compilation failed"
run_lint_check || add_error "Linting issues found"
report_errors

echo '{}'
```

## Conditional blocking

Block only on critical errors, warn on lesser ones.

```bash
#!/bin/bash

ERROR_LEVEL="none"

check_critical_issues() {
    grep -q "FIXME\|XXX\|TODO: CRITICAL" "$file_path" && ERROR_LEVEL="critical" && return 1
    return 0
}

check_warnings() {
    grep -q "console.log\|debugger" "$file_path" && ERROR_LEVEL="warning" && return 1
    return 0
}

check_critical_issues
check_warnings

case "$ERROR_LEVEL" in
    "critical")
        echo '{"decision": "block", "reason": "🚫 CRITICAL: Found critical TODOs/FIXMEs that must be addressed"}' | jq -c '.'
        ;;
    "warning")
        echo "⚠️  Warning: Found debug statements (console.log, debugger)"
        echo '{}'
        ;;
    *)
        echo '{}'
        ;;
esac
```

## Context injection (UserPromptSubmit)

Inject project state into Claude's prompt beyond skill activation.

```javascript
function injectContext(prompt) {
    const context = [];

    if (prompt.includes('API')) {
        context.push('📖 API Documentation: https://docs.example.com/api');
    }

    const recentFiles = getRecentlyEditedFiles();
    if (recentFiles.length > 0) {
        context.push(`📝 Recently edited: ${recentFiles.join(', ')}`);
    }

    const buildStatus = getLastBuildStatus();
    if (!buildStatus.passed) {
        context.push(`⚠️  Current build has ${buildStatus.errorCount} errors`);
    }

    if (context.length === 0) return { decision: 'approve' };

    return {
        decision: 'approve',
        additionalContext: `\n\n---\n${context.join('\n')}\n---\n`
    };
}
```

## Progressive output

Show progress for hooks that take multiple steps.

```bash
#!/bin/bash

show_progress() { echo -n "$1..."; }
complete_progress() { [ "$1" == "success" ] && echo " ✅" || echo " ❌"; }

show_progress "Running TypeScript compiler"
npx tsc --noEmit 2>/dev/null && complete_progress "success" || complete_progress "failure"

show_progress "Running linter"
npx eslint . 2>/dev/null && complete_progress "success" || complete_progress "failure"

echo '{}'
```

## Desktop notification

Notify user of important events (cross-platform).

```bash
#!/bin/bash

notify() {
    local message="$1"
    case "$OSTYPE" in
        darwin*) osascript -e "display notification \"$message\" with title \"Claude Code Hook\"" ;;
        linux*)  notify-send "Claude Code Hook" "$message" ;;
    esac
}

if [ "$error_count" -gt 10 ]; then
    notify "⚠️  Build has $error_count errors"
fi

echo '{}'
```
