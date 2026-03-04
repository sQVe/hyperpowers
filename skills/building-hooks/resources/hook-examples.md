# Complete hook examples

Production-ready hook implementations you can adapt.

## Example 1: File edit tracker (PostToolUse)

Tracks which files were edited for use by downstream hooks (build checker, formatter).

**File:** `~/.claude/hooks/post-tool-use/01-track-edits.sh`

```bash
#!/bin/bash

LOG_FILE="$HOME/.claude/edit-log.txt"
MAX_LOG_LINES=1000

touch "$LOG_FILE"

find_repo() {
    local dir=$(dirname "$1")
    while [ "$dir" != "/" ]; do
        if [ -d "$dir/.git" ]; then
            basename "$dir"
            return
        fi
        dir=$(dirname "$dir")
    done
    echo "unknown"
}

log_edit() {
    local file_path="$1"
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    local repo=$(find_repo "$file_path")
    echo "$timestamp | $repo | $file_path" >> "$LOG_FILE"
}

read -r tool_use_json
tool_name=$(echo "$tool_use_json" | jq -r '.tool.name')

case "$tool_name" in
    "Edit"|"Write")
        file_path=$(echo "$tool_use_json" | jq -r '.tool.input.file_path')
        ;;
    "MultiEdit")
        echo "$tool_use_json" | jq -r '.tool.input.edits[].file_path' | while read -r path; do
            log_edit "$path"
        done
        echo '{}'
        exit 0
        ;;
esac

if [ -n "$file_path" ] && [ "$file_path" != "null" ]; then
    log_edit "$file_path"
fi

line_count=$(wc -l < "$LOG_FILE")
if [ "$line_count" -gt "$MAX_LOG_LINES" ]; then
    tail -n "$MAX_LOG_LINES" "$LOG_FILE" > "$LOG_FILE.tmp"
    mv "$LOG_FILE.tmp" "$LOG_FILE"
fi

echo '{}'
```

**Configuration (`hooks.json`):**
```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "command": "~/.claude/hooks/post-tool-use/01-track-edits.sh",
      "blocking": false,
      "timeout": 1000
    }
  ]
}
```

## Example 2: Multi-repo build checker (Stop)

Runs builds on all repos modified since last check. Auto-detects build system.

**File:** `~/.claude/hooks/stop/20-build-checker.sh`

```bash
#!/bin/bash

LOG_FILE="$HOME/.claude/edit-log.txt"
PROJECT_ROOT="$HOME/git/myproject"
ERROR_THRESHOLD=5

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

get_modified_repos() {
    tail -50 "$LOG_FILE" 2>/dev/null | cut -d'|' -f2 | tr -d ' ' | sort -u | grep -v "unknown"
}

build_repo() {
    local repo_name="$1"
    local repo_path="$PROJECT_ROOT/$repo_name"

    [ -d "$repo_path" ] || return 0

    local build_cmd=""
    if [ -f "$repo_path/package.json" ]; then
        build_cmd="npm run build"
    elif [ -f "$repo_path/Cargo.toml" ]; then
        build_cmd="cargo build"
    elif [ -f "$repo_path/go.mod" ]; then
        build_cmd="go build ./..."
    else
        return 0
    fi

    echo "Building $repo_name..."
    cd "$repo_path"
    local output=$(eval "$build_cmd" 2>&1)
    local exit_code=$?

    if [ $exit_code -ne 0 ]; then
        local error_count=$(echo "$output" | grep -c "error" || echo "0")
        if [ "$error_count" -ge "$ERROR_THRESHOLD" ]; then
            echo -e "${YELLOW}⚠️  $repo_name: $error_count errors - consider auto-error-resolver agent${NC}"
        else
            echo -e "${RED}🔴 $repo_name: $error_count errors${NC}"
            echo "$output" | grep "error" | head -10
        fi
        return 1
    else
        echo -e "${GREEN}✅ $repo_name: Build passed${NC}"
    fi
}

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🔨 BUILD VERIFICATION"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

modified_repos=$(get_modified_repos)
[ -z "$modified_repos" ] && echo "No repos modified" && exit 0

build_failures=0
for repo in $modified_repos; do
    build_repo "$repo" || ((build_failures++))
    echo ""
done

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
if [ "$build_failures" -gt 0 ]; then
    echo -e "${RED}$build_failures repo(s) failed${NC}"
else
    echo -e "${GREEN}All builds passed${NC}"
fi

echo '{}'
```

## Example 3: TypeScript/JS formatter (Stop)

Auto-formats edited files using the nearest `.prettierrc`.

**File:** `~/.claude/hooks/stop/30-format-code.sh`

```bash
#!/bin/bash

LOG_FILE="$HOME/.claude/edit-log.txt"

get_edited_files() {
    tail -50 "$LOG_FILE" 2>/dev/null | cut -d'|' -f3 | tr -d ' ' | grep -E '\.(ts|tsx|js|jsx)$' | sort -u
}

format_file() {
    local file="$1"
    [ -f "$file" ] || return 0

    local dir=$(dirname "$file")
    local prettier_config=""

    while [ "$dir" != "/" ]; do
        if [ -f "$dir/.prettierrc" ] || [ -f "$dir/.prettierrc.json" ]; then
            prettier_config="$dir"
            break
        fi
        dir=$(dirname "$dir")
    done

    [ -z "$prettier_config" ] && return 0

    cd "$prettier_config"
    npx prettier --write "$file" 2>/dev/null && echo "✓ Formatted: $(basename "$file")"
}

echo "🎨 Formatting edited files..."

edited_files=$(get_edited_files)
[ -z "$edited_files" ] && echo "No files to format" && exit 0

formatted_count=0
for file in $edited_files; do
    format_file "$file" && ((formatted_count++))
done

echo "✅ Formatted $formatted_count file(s)"
echo '{}'
```

## Example 4: Error handling reminder (Stop)

Gentle self-check reminder when risky patterns (async, try/catch, database) are detected.

**File:** `~/.claude/hooks/stop/40-error-reminder.sh`

```bash
#!/bin/bash

LOG_FILE="$HOME/.claude/edit-log.txt"

get_edited_files() {
    tail -20 "$LOG_FILE" 2>/dev/null | cut -d'|' -f3 | tr -d ' ' | sort -u
}

risky_count=0
backend_files=0

for file in $(get_edited_files); do
    [ -f "$file" ] || continue
    if grep -q -E "try|catch|async|await|prisma|\.execute\(|fetch\(|axios\." "$file"; then
        ((risky_count++))
        echo "$file" | grep -q "backend\|server\|api" && ((backend_files++))
    fi
done

if [ "$risky_count" -gt 0 ]; then
    cat <<EOF

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 ERROR HANDLING SELF-CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  $risky_count file(s) with async/try-catch/database operations

   ❓ Did you add proper error handling?
   ❓ Are errors logged/captured appropriately?
   ❓ Are promises handled correctly?
EOF

    if [ "$backend_files" -gt 0 ]; then
        echo "   💡 Backend: all errors should be captured (Sentry/logging), DB ops need try-catch"
    fi

    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
fi

echo '{}'
```

## Example 5: Dangerous operation blocker (PreToolUse)

Blocks writes to protected production paths.

**File:** `~/.claude/hooks/pre-tool-use/dangerous-ops.sh`

```bash
#!/bin/bash

read -r tool_use_json

tool_name=$(echo "$tool_use_json" | jq -r '.tool.name')
file_path=$(echo "$tool_use_json" | jq -r '.tool.input.file_path // empty')

PROTECTED_PATHS=(
    "/production/"
    "/prod/"
    "/.env.production"
    "/config/production"
)

is_dangerous() {
    local path="$1"
    for protected in "${PROTECTED_PATHS[@]}"; do
        [[ "$path" == *"$protected"* ]] && return 0
    done
    return 1
}

if [ "$tool_name" == "Write" ] || [ "$tool_name" == "Edit" ]; then
    if is_dangerous "$file_path"; then
        cat <<EOF | jq -c '.'
{
  "decision": "block",
  "reason": "⛔ BLOCKED: Attempting to modify protected path\n\nFile: $file_path\n\nReview changes manually and confirm with teammate. To override, edit ~/.claude/hooks/pre-tool-use/dangerous-ops.sh"
}
EOF
        exit 0
    fi
fi

echo '{"decision": "allow"}'
```

## Testing these examples

```bash
# Test edit tracker
echo '{"tool": {"name": "Edit", "input": {"file_path": "/tmp/test.ts"}}}' | \
    bash ~/.claude/hooks/post-tool-use/01-track-edits.sh

# Test build checker (requires edit-log.txt with entries)
bash ~/.claude/hooks/stop/20-build-checker.sh

# Test formatter
bash ~/.claude/hooks/stop/30-format-code.sh

# Test skill activator (see skills-auto-activation resource)
```

**For skill auto-activation hook:** see `hyperpowers:skills-auto-activation` and its resources.
