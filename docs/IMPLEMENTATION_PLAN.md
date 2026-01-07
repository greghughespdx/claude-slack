# Claude-Slack Integration: Implementation Plan

## Document Purpose

This document provides step-by-step instructions for fixing all identified issues. An implementation agent MUST be able to execute this plan without asking clarifying questions. Every change specifies exact file paths, line numbers, before/after code, and verification steps.

---

## Pre-Implementation Checklist

Before starting, verify these prerequisites:

```bash
# 1. Confirm working directory
cd /Users/greg/Dev/contrib/claude-slack

# 2. Verify git status is clean or changes are committed
git status

# 3. Confirm Python 3.8+ is available
python3 --version

# 4. Confirm Claude Code is installed
which claude
```

---

## Issue 1: Remove Hardcoded Developer Path (CRITICAL)

### Problem

Two files contain a hardcoded path to the original developer's machine:

```
/Users/danielbennett/codeNew/.claude/claude-slack
```

This path appears in the `get_exact_permission_options()` function and will break permission prompt text generation for any other user.

### Affected Files

| File | Line | Status |
|------|------|--------|
| `/Users/greg/Dev/contrib/claude-slack/hooks/on_notification.py` | 741 | Hardcoded |
| `/Users/greg/Dev/contrib/claude-slack/.claude/hooks/on_notification.py` | 741 | Hardcoded |

### Fix Instructions

**File 1: `/Users/greg/Dev/contrib/claude-slack/hooks/on_notification.py`**

Locate line 741 inside the `get_exact_permission_options()` function:

**BEFORE (line 741):**
```python
    # Get project directory (hardcoded based on analysis)
    project_dir = "/Users/danielbennett/codeNew/.claude/claude-slack"
```

**AFTER:**
```python
    # Get project directory dynamically from environment or cwd
    project_dir = os.environ.get('CLAUDE_PROJECT_DIR', os.getcwd())
```

**File 2: `/Users/greg/Dev/contrib/claude-slack/.claude/hooks/on_notification.py`**

Apply the identical change at line 741.

### Verification

```bash
# Verify no hardcoded paths remain
grep -rn "danielbennett\|/Users/" /Users/greg/Dev/contrib/claude-slack --include="*.py" --include="*.sh" 2>/dev/null | grep -v ".venv"

# Expected output: No matches (empty output)
```

---

## Issue 2: Remove Literal `~` Directory (HIGH)

### Problem

A literal directory named `~` exists in the repository root, containing files that should have been written to the user's home directory. This was caused by unquoted `~` in shell commands where shell expansion didn't occur.

### Affected Path

```
/Users/greg/Dev/contrib/claude-slack/~/
```

Contents:
- `~/.claude/slack/logs/wrapper_e3423585.log`
- `~/.claude/slack/registry.db`

### Fix Instructions

**Step 1: Remove the literal `~` directory**

```bash
rm -rf "/Users/greg/Dev/contrib/claude-slack/~"
```

**Step 2: Add `~` to `.gitignore`**

Edit `/Users/greg/Dev/contrib/claude-slack/.gitignore`. Add this line at the end:

**ADD to .gitignore:**
```
# Prevent literal tilde directory (path bugs)
~/
```

### Verification

```bash
# Verify directory is removed
ls -la /Users/greg/Dev/contrib/claude-slack/~ 2>&1

# Expected output: "No such file or directory"

# Verify .gitignore contains the entry
grep "^~/$" /Users/greg/Dev/contrib/claude-slack/.gitignore

# Expected output: ~/
```

---

## Issue 3: Fix Hook Path Architecture (CRITICAL)

### Problem

The `settings.local.json.template` references hooks via `$CLAUDE_PROJECT_DIR/.claude/hooks/`, but:
1. `$CLAUDE_PROJECT_DIR` is set to the user's current working directory by the wrapper
2. Hooks are installed at `~/.claude/claude-slack/hooks/`
3. The paths don't match unless user is running Claude from within the claude-slack directory

### Decision

Use absolute paths pointing to the installed hooks location at `$HOME/.claude/claude-slack/hooks/`.

### Affected File

`/Users/greg/Dev/contrib/claude-slack/hooks/settings.local.json.template`

### Fix Instructions

**BEFORE (lines 29-61):**
```json
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python3 $CLAUDE_PROJECT_DIR/.claude/hooks/on_pretooluse.py",
            "timeout": 5
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 $CLAUDE_PROJECT_DIR/.claude/hooks/on_stop.py"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 $CLAUDE_PROJECT_DIR/.claude/hooks/on_notification.py"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/session-start.sh"
          }
        ]
      }
    ]
  }
```

**AFTER:**
```json
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python3 $HOME/.claude/claude-slack/hooks/on_pretooluse.py",
            "timeout": 5
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 $HOME/.claude/claude-slack/hooks/on_stop.py"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 $HOME/.claude/claude-slack/hooks/on_notification.py"
          }
        ]
      }
    ]
  }
```

Note: The `SessionStart` hook is removed because it references a non-existent file (`$HOME/.claude/hooks/session-start.sh`) and is not part of the claude-slack integration.

### Verification

```bash
# Verify the template has correct paths
grep -c "CLAUDE_PROJECT_DIR" /Users/greg/Dev/contrib/claude-slack/hooks/settings.local.json.template

# Expected output: 0 (no occurrences)

grep -c '$HOME/.claude/claude-slack/hooks/' /Users/greg/Dev/contrib/claude-slack/hooks/settings.local.json.template

# Expected output: 3 (one for each hook)
```

---

## Issue 4: Add Claude Settings Integration to install.sh (HIGH)

### Problem

Users must manually configure Claude's global `settings.local.json` to register hooks. The install script does not handle this.

### Decision

The install script WILL:
1. Check if `~/.claude/settings.local.json` exists
2. If not, create it from the template
3. If exists, merge the hooks section (with backup)

### Affected File

`/Users/greg/Dev/contrib/claude-slack/install.sh`

### Fix Instructions

Add a new step after Step 7 (before "Step 8: Configure shell"). Insert these lines after line 119:

**ADD after line 119:**
```bash
# Step 7.5: Configure Claude Code settings
echo -e "${BLUE}Configuring Claude Code settings...${NC}"
CLAUDE_SETTINGS="$HOME/.claude/settings.local.json"
TEMPLATE_SETTINGS="$INSTALL_DIR/hooks/settings.local.json.template"

if [ ! -f "$CLAUDE_SETTINGS" ]; then
    # Create Claude settings directory if needed
    mkdir -p "$HOME/.claude"
    # Copy template as-is (it uses $HOME which Claude Code expands)
    cp "$TEMPLATE_SETTINGS" "$CLAUDE_SETTINGS"
    echo -e "${GREEN}  Created $CLAUDE_SETTINGS${NC}"
else
    # Backup existing settings
    BACKUP="$CLAUDE_SETTINGS.backup.$(date +%Y%m%d_%H%M%S)"
    cp "$CLAUDE_SETTINGS" "$BACKUP"
    echo -e "${YELLOW}  Backed up existing settings to $BACKUP${NC}"

    # Check if hooks already configured
    if grep -q "claude-slack" "$CLAUDE_SETTINGS" 2>/dev/null; then
        echo -e "${GREEN}  Claude Code settings already configured for claude-slack${NC}"
    else
        echo -e "${YELLOW}  Manual step required: Merge hooks from $TEMPLATE_SETTINGS into $CLAUDE_SETTINGS${NC}"
        echo -e "${YELLOW}  Or replace with: cp $TEMPLATE_SETTINGS $CLAUDE_SETTINGS${NC}"
    fi
fi
echo ""
```

### Verification

```bash
# Syntax check the install script
bash -n /Users/greg/Dev/contrib/claude-slack/install.sh

# Expected output: No output (no syntax errors)
```

---

## Issue 5: Fix Session ID Complexity (MEDIUM)

### Problem

The wrapper creates two session registrations:
1. An 8-character wrapper ID (e.g., `e3423585`)
2. A 36-character Claude UUID (e.g., `abc12345-6789-...`)

Both sessions are registered with the same Slack thread. When hooks fire, they use Claude's UUID, but the socket is owned by the wrapper session. The current "self-healing" logic attempts to match IDs but is fragile.

### Decision

KEEP both session registrations but FIX the lookup logic. The current architecture is actually correct:
- Wrapper session owns the socket
- Claude session is registered for hook lookups
- Self-healing bridges them when needed

The real fix is ensuring the `REGISTER_EXISTING` command properly copies all metadata.

### Affected File

`/Users/greg/Dev/contrib/claude-slack/core/session_registry.py` (if it exists) or the registry handling code.

### Fix Instructions

No code change required for this issue. The current implementation already handles dual registration correctly through:
1. `register_with_registry()` - Creates wrapper session with Slack thread
2. `register_claude_session()` - Registers Claude UUID with same Slack metadata

The self-healing in `on_stop.py` and `on_notification.py` correctly bridges the gap.

### Verification

```bash
# Start a session and verify both registrations
claudes

# In another terminal, check database
sqlite3 ~/.claude/slack/registry.db "SELECT session_id, LENGTH(session_id), slack_thread_ts FROM sessions ORDER BY created_at DESC LIMIT 2;"

# Expected: Two rows with same thread_ts - one 8 chars, one 36 chars
```

---

## Issue 6: Add PID File Locking for Listener (MEDIUM)

### Problem

Multiple `slack_listener.py` processes can run simultaneously due to race conditions between install.sh, claude-slack-ensure, and launchd.

### Decision

Add PID file locking to `slack_listener.py` to ensure only one instance runs.

### Affected File

`/Users/greg/Dev/contrib/claude-slack/core/slack_listener.py`

### Fix Instructions

Add PID file locking at the start of the `main()` function. Locate the `main()` function (starts at line 730).

**ADD after line 731 (`"""Start the Slack bot in Socket Mode"""`), before the print statement:**
```python
    # Ensure single instance via PID file
    import fcntl
    PID_FILE = os.path.expanduser("~/.claude/slack/slack_listener.pid")

    # Create directory if needed
    os.makedirs(os.path.dirname(PID_FILE), exist_ok=True)

    # Try to acquire exclusive lock
    pid_fp = open(PID_FILE, 'w')
    try:
        fcntl.flock(pid_fp, fcntl.LOCK_EX | fcntl.LOCK_NB)
        pid_fp.write(str(os.getpid()))
        pid_fp.flush()
    except IOError:
        print("❌ Another slack_listener instance is already running", file=sys.stderr)
        print(f"   PID file: {PID_FILE}", file=sys.stderr)
        sys.exit(1)

```

**ADD at the end of main(), before the final `sys.exit(0)`:**
```python
    finally:
        # Release PID lock
        try:
            fcntl.flock(pid_fp, fcntl.LOCK_UN)
            pid_fp.close()
            os.remove(PID_FILE)
        except:
            pass
```

Note: The `finally` block needs to wrap the `handler.start()` call. Restructure the try/except at line 769-772:

**BEFORE (lines 769-772):**
```python
    try:
        handler.start()
    except KeyboardInterrupt:
        print("\n👋 Slack bot stopped")
        sys.exit(0)
```

**AFTER:**
```python
    try:
        handler.start()
    except KeyboardInterrupt:
        print("\n👋 Slack bot stopped")
    finally:
        # Release PID lock
        try:
            fcntl.flock(pid_fp, fcntl.LOCK_UN)
            pid_fp.close()
            os.remove(PID_FILE)
        except:
            pass
        sys.exit(0)
```

### Verification

```bash
# Start listener manually
python3 /Users/greg/Dev/contrib/claude-slack/core/slack_listener.py &

# Try starting another instance
python3 /Users/greg/Dev/contrib/claude-slack/core/slack_listener.py

# Expected: "Another slack_listener instance is already running"

# Cleanup
pkill -f slack_listener.py
```

---

## Issue 7: Fix Mirroring State Type Mismatch (MEDIUM)

### Problem

The `slack_enabled` field has inconsistent type handling:
- Database stores as string: `'true'` or `'false'`
- Code sometimes checks as boolean: `if slack_enabled == False`
- `to_dict()` returns boolean but database operations use strings

### Decision

Standardize on string storage (`'true'`/`'false'`) but ensure all comparisons handle both types.

### Affected Files

1. `/Users/greg/Dev/contrib/claude-slack/hooks/on_stop.py` - Lines 397-401
2. `/Users/greg/Dev/contrib/claude-slack/hooks/on_notification.py` - Lines 1176-1179

### Fix Instructions

**File 1: `/Users/greg/Dev/contrib/claude-slack/hooks/on_stop.py`**

Locate lines 397-401:

**BEFORE:**
```python
        # Check if Slack mirroring is enabled for this session
        # Note: Database stores "true"/"false" as strings, not booleans
        slack_enabled = session.get("slack_enabled", "true")
        if slack_enabled == "false" or slack_enabled is False:
            log_info(f"Slack mirroring disabled for session {session_id[:8]}, skipping")
```

**AFTER:**
```python
        # Check if Slack mirroring is enabled for this session
        # Handle both string ('true'/'false') and boolean values
        slack_enabled = session.get("slack_enabled", True)
        if slack_enabled in ("false", False, "False", 0, "0"):
            log_info(f"Slack mirroring disabled for session {session_id[:8]}, skipping")
```

**File 2: `/Users/greg/Dev/contrib/claude-slack/hooks/on_notification.py`**

Locate lines 1176-1179:

**BEFORE:**
```python
        # Check if Slack mirroring is enabled for this session
        # Note: Database stores "true"/"false" as strings, not booleans
        slack_enabled = session.get("slack_enabled", "true")
        if slack_enabled == "false" or slack_enabled is False:
```

**AFTER:**
```python
        # Check if Slack mirroring is enabled for this session
        # Handle both string ('true'/'false') and boolean values
        slack_enabled = session.get("slack_enabled", True)
        if slack_enabled in ("false", False, "False", 0, "0"):
```

### Verification

```bash
# Test with string value
python3 -c "
slack_enabled = 'false'
if slack_enabled in ('false', False, 'False', 0, '0'):
    print('Correctly detected disabled')
"

# Expected: "Correctly detected disabled"
```

---

## Issue 8: Add Pre-flight Validation to install.sh (MEDIUM)

### Problem

The install script doesn't validate prerequisites before making changes.

### Decision

Add validation checks at the start of install.sh.

### Affected File

`/Users/greg/Dev/contrib/claude-slack/install.sh`

### Fix Instructions

Add validation after line 22 (`echo ""`), before Step 1:

**ADD after line 22:**
```bash
# Pre-flight checks
echo -e "${BLUE}Running pre-flight checks...${NC}"

# Check Python 3.8+
PYTHON_VERSION=$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")' 2>/dev/null)
if [ -z "$PYTHON_VERSION" ]; then
    echo -e "${RED}Error: Python 3 not found${NC}"
    exit 1
fi
PYTHON_MAJOR=$(echo "$PYTHON_VERSION" | cut -d. -f1)
PYTHON_MINOR=$(echo "$PYTHON_VERSION" | cut -d. -f2)
if [ "$PYTHON_MAJOR" -lt 3 ] || ([ "$PYTHON_MAJOR" -eq 3 ] && [ "$PYTHON_MINOR" -lt 8 ]); then
    echo -e "${RED}Error: Python 3.8+ required (found $PYTHON_VERSION)${NC}"
    exit 1
fi
echo -e "${GREEN}  Python $PYTHON_VERSION${NC}"

# Check Claude Code is installed
if ! command -v claude &> /dev/null; then
    echo -e "${YELLOW}Warning: Claude Code CLI not found in PATH${NC}"
    echo -e "${YELLOW}  Install from: https://claude.ai/code${NC}"
fi

# Check for conflicting processes
EXISTING_LISTENER=$(pgrep -f "slack_listener.py" || true)
if [ -n "$EXISTING_LISTENER" ]; then
    echo -e "${YELLOW}Warning: Existing slack_listener process found (PID: $EXISTING_LISTENER)${NC}"
    echo -e "${YELLOW}  Will be replaced after installation${NC}"
fi

echo -e "${GREEN}  Pre-flight checks passed${NC}"
echo ""
```

### Verification

```bash
# Syntax check
bash -n /Users/greg/Dev/contrib/claude-slack/install.sh

# Expected: No output (no syntax errors)
```

---

## Issue 9: Enhance claude-slack-diagnose (LOW)

### Problem

The diagnostic tool doesn't check hook registration or settings configuration.

### Decision

Add checks for Claude settings and hook configuration.

### Affected File

`/Users/greg/Dev/contrib/claude-slack/bin/claude-slack-diagnose`

### Fix Instructions

Add a new section after "5. LOG FILES" (after line 200), before "6. Overall Summary":

**ADD after line 200:**
```bash
# 5.5. Claude Code Settings
echo -e "${CYAN}5.5. CLAUDE CODE SETTINGS${NC}"
echo "─────────────────────────────────"

CLAUDE_SETTINGS="$HOME/.claude/settings.local.json"
echo -n "  settings.local.json: "
if [ -f "$CLAUDE_SETTINGS" ]; then
    echo -e "${GREEN}✓ Found${NC}"

    # Check for claude-slack hooks
    echo -n "    claude-slack hooks: "
    if grep -q "claude-slack" "$CLAUDE_SETTINGS" 2>/dev/null; then
        HOOK_COUNT=$(grep -c "claude-slack" "$CLAUDE_SETTINGS" 2>/dev/null || echo "0")
        echo -e "${GREEN}✓ Configured ($HOOK_COUNT references)${NC}"
    else
        echo -e "${RED}✗ Not configured${NC}"
        OVERALL_STATUS="BROKEN"
        ISSUES+=("Claude Code hooks not configured")
    fi

    # Check hook paths exist
    echo -n "    Hook files: "
    HOOKS_DIR="$HOME/.claude/claude-slack/hooks"
    if [ -d "$HOOKS_DIR" ]; then
        HOOK_FILES=$(ls -1 "$HOOKS_DIR"/*.py 2>/dev/null | wc -l | tr -d ' ')
        echo -e "${GREEN}✓ $HOOK_FILES Python hooks found${NC}"
    else
        echo -e "${RED}✗ Hooks directory not found${NC}"
        ISSUES+=("Hooks directory missing: $HOOKS_DIR")
    fi
else
    echo -e "${RED}✗ Not found${NC}"
    OVERALL_STATUS="BROKEN"
    ISSUES+=("Claude Code settings not configured")
fi

echo ""
```

### Verification

```bash
# Run the enhanced diagnostic
/Users/greg/Dev/contrib/claude-slack/bin/claude-slack-diagnose

# Expected: New section "5.5. CLAUDE CODE SETTINGS" appears in output
```

---

## Implementation Order

Execute fixes in this order to minimize risk and allow incremental verification:

| Order | Issue | Risk | Time |
|-------|-------|------|------|
| 1 | Issue 2: Remove `~` directory | None | 1 min |
| 2 | Issue 1: Remove hardcoded path | Low | 2 min |
| 3 | Issue 3: Fix hook paths | Low | 5 min |
| 4 | Issue 7: Fix type mismatch | Low | 3 min |
| 5 | Issue 4: Settings integration | Medium | 10 min |
| 6 | Issue 8: Pre-flight validation | Low | 5 min |
| 7 | Issue 6: PID file locking | Medium | 10 min |
| 8 | Issue 9: Diagnose enhancement | Low | 5 min |

---

## Post-Implementation Verification

After all fixes are applied, run this comprehensive test:

```bash
# 1. Clean uninstall
/Users/greg/Dev/contrib/claude-slack/uninstall.sh

# 2. Verify clean state
ls ~/.claude/claude-slack 2>&1  # Should not exist
pgrep -f "slack_listener.py"    # Should be empty

# 3. Fresh install
cd /Users/greg/Dev/contrib/claude-slack
./install.sh

# 4. Run diagnostics
claude-slack-diagnose

# 5. Start a session
claudes

# 6. Verify in Slack (manual check)
# - New thread should appear
# - Send a message from Slack
# - Claude should receive it
# - Claude's response should appear in thread

# 7. Test mirroring toggle (from Slack)
# - Send "!off" in the thread
# - Send a message to Claude from terminal
# - Response should NOT appear in Slack
# - Send "!on" in the thread
# - Send another message
# - Response SHOULD appear in Slack
```

---

## Files Changed Summary

| File | Change Type | Lines Modified |
|------|-------------|----------------|
| `hooks/on_notification.py` | Edit | 741, 1176-1179 |
| `.claude/hooks/on_notification.py` | Edit | 741 |
| `hooks/on_stop.py` | Edit | 397-401 |
| `hooks/settings.local.json.template` | Edit | 29-61 |
| `install.sh` | Edit | +23 lines (pre-flight), +20 lines (settings) |
| `core/slack_listener.py` | Edit | 730-772 |
| `bin/claude-slack-diagnose` | Edit | +30 lines |
| `.gitignore` | Edit | +2 lines |
| `~/` (directory) | Delete | Entire directory |

---

## Rollback Instructions

If issues arise, restore from git:

```bash
cd /Users/greg/Dev/contrib/claude-slack
git checkout -- .
git clean -fd
```

Then uninstall and reinstall:

```bash
./uninstall.sh
./install.sh
```
