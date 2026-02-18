# How Claude Code Intercepts and Collects Actions

This document explains how the Claude Code plugin system intercepts and monitors Claude's actions during execution.

## Table of Contents

- [Overview](#overview)
- [Hook System Architecture](#hook-system-architecture)
- [Hook Events](#hook-events)
- [How Hooks Intercept Actions](#how-hooks-intercept-actions)
- [Data Flow](#data-flow)
- [Implementation Examples](#implementation-examples)
- [Creating Custom Hooks](#creating-custom-hooks)

## Overview

Claude Code uses a **plugin hook system** to intercept and monitor Claude's actions. Hooks are event-driven scripts that execute in response to specific events during Claude's operation, such as before/after tool use, session start/stop, and user prompt submission.

**Key capabilities:**
- ✅ Intercept tool calls before they execute (validation, blocking, modification)
- ✅ Monitor tool results after execution (logging, analysis)
- ✅ Track user prompts and Claude's responses
- ✅ Enforce policies and standards
- ✅ Collect data for analysis and debugging
- ✅ Integrate external tools and workflows

## Hook System Architecture

### Core Concept

Hooks are **event listeners** that register for specific Claude Code events. When an event occurs, Claude Code:

1. **Serializes event data** → Converts the event into JSON format
2. **Invokes registered hooks** → Executes all hooks registered for that event type
3. **Passes data via stdin** → Sends JSON data to hook scripts via standard input
4. **Reads hook output** → Receives hook response via stdout
5. **Takes action** → Blocks, allows, or modifies the operation based on hook response

### Hook Configuration

Hooks are configured in `hooks/hooks.json` within plugin directories:

```json
{
  "description": "Hook configuration for plugin",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [...],
    "Stop": [...],
    "UserPromptSubmit": [...]
  }
}
```

**Key components:**
- **Event type**: Which event to listen for (PreToolUse, PostToolUse, Stop, etc.)
- **Matcher**: Optional filter for specific tools (e.g., "Bash", "Edit|Write")
- **Hook type**: "command" (shell script) or "prompt" (LLM-based)
- **Command**: Script to execute when event fires
- **Timeout**: Maximum execution time in seconds

## Hook Events

Claude Code provides several hook events that allow interception at different stages:

### 1. PreToolUse
**When**: Before Claude executes any tool (bash, file edit, etc.)

**Purpose**: Validate, block, or modify tool calls before execution

**Input data (via stdin as JSON)**:
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  },
  "session_id": "abc123"
}
```

**Output (to stdout as JSON)**:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny"
  },
  "systemMessage": "⚠️ Dangerous command blocked!"
}
```

**Exit codes**:
- `0` = Allow operation
- `2` = Block operation and show stderr to Claude

### 2. PostToolUse
**When**: After Claude executes a tool

**Purpose**: Monitor results, log actions, trigger follow-up automation

**Input data**:
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Edit",
  "tool_input": {...},
  "tool_output": "File edited successfully",
  "session_id": "abc123"
}
```

### 3. Stop
**When**: When Claude wants to end the session

**Purpose**: Verify completion criteria, enforce standards

**Input data**:
```json
{
  "hook_event_name": "Stop",
  "reason": "Task completed",
  "transcript_path": "/path/to/session/transcript.json",
  "session_id": "abc123"
}
```

**Output to block stopping**:
```json
{
  "decision": "block",
  "reason": "Tests not run",
  "systemMessage": "Please run tests before stopping"
}
```

### 4. UserPromptSubmit
**When**: When user submits a prompt

**Purpose**: Validate input, add context, enforce policies

**Input data**:
```json
{
  "hook_event_name": "UserPromptSubmit",
  "user_prompt": "Delete all the files",
  "session_id": "abc123"
}
```

### 5. SessionStart
**When**: At the start of a new session

**Purpose**: Load project context, configure environment

### 6. SessionEnd
**When**: At the end of a session

**Purpose**: Cleanup, logging, final validations

## How Hooks Intercept Actions

### Example: Intercepting Dangerous Commands

Let's trace how the `hookify` plugin intercepts a dangerous `rm -rf` command:

#### Step 1: User asks Claude to run a command
```
User: "Delete the /tmp/test directory"
```

#### Step 2: Claude decides to use the Bash tool
```
Claude thinking: I'll use the Bash tool with command "rm -rf /tmp/test"
```

#### Step 3: PreToolUse event is triggered

Before executing, Claude Code:
1. Creates JSON input:
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  },
  "session_id": "xyz789"
}
```

2. Invokes all registered PreToolUse hooks for "Bash" tool
3. Passes JSON via stdin to hook script

#### Step 4: Hook script analyzes the command

The `pretooluse.py` script:
```python
import json
import sys
from hookify.core.config_loader import load_rules
from hookify.core.rule_engine import RuleEngine

# Read input from stdin
input_data = json.load(sys.stdin)

# Load rules that match this event type
rules = load_rules(event='bash')

# Evaluate all rules against the input
engine = RuleEngine()
result = engine.evaluate_rules(rules, input_data)

# Output result to stdout
print(json.dumps(result), file=sys.stdout)
```

The rule engine checks loaded rules (from `.claude/hookify.*.local.md` files):
```python
def _rule_matches(self, rule: Rule, input_data: Dict) -> bool:
    tool_input = input_data.get('tool_input', {})
    command = tool_input.get('command', '')
    
    # Check if command matches the pattern (e.g., "rm\s+-rf")
    if re.search(rule.pattern, command):
        return True
    return False
```

#### Step 5: Hook returns decision

If rule matches and action is "block":
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny"
  },
  "systemMessage": "⚠️ **Dangerous rm command detected!**\n\nThis command could delete important files."
}
```

#### Step 6: Claude Code blocks the operation

Claude Code receives the hook response and:
1. Sees `permissionDecision: "deny"`
2. Blocks the Bash tool from executing
3. Shows the systemMessage to Claude
4. Claude sees the warning and chooses a safer approach

### Example: Intercepting File Edits for Security

The `security-guidance` plugin intercepts file edits:

#### Step 1: Claude tries to edit a file
```python
# Claude wants to write this code
eval(user_input)
```

#### Step 2: PreToolUse event fires for Edit tool
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/home/user/app.py",
    "old_string": "# TODO",
    "new_string": "eval(user_input)"
  }
}
```

#### Step 3: Security hook analyzes content

The `security_reminder_hook.py` checks patterns:
```python
SECURITY_PATTERNS = [
    {
        "ruleName": "eval_injection",
        "substrings": ["eval("],
        "reminder": "⚠️ Security Warning: eval() executes arbitrary code..."
    }
]

def check_patterns(file_path, content):
    for pattern in SECURITY_PATTERNS:
        if "substrings" in pattern:
            for substring in pattern["substrings"]:
                if substring in content:
                    return pattern["ruleName"], pattern["reminder"]
```

#### Step 4: Hook blocks and warns

Exit code 2 blocks the operation and shows warning to Claude:
```python
if rule_name and reminder:
    print(reminder, file=sys.stderr)
    sys.exit(2)  # Block tool execution
```

## Data Flow

### Visual Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Claude Code                             │
│                                                                 │
│  1. Claude decides to use a tool                                │
│     ↓                                                           │
│  2. Create JSON with tool info                                  │
│     ↓                                                           │
│  3. Trigger PreToolUse event                                    │
│     ↓                                                           │
│  ┌──────────────────────────────────────────┐                  │
│  │   For each registered hook:               │                  │
│  │   - Spawn hook process                   │                  │
│  │   - Send JSON via stdin                  │                  │
│  │   - Wait for response (with timeout)     │                  │
│  │   - Read stdout for decision             │                  │
│  └──────────────────────────────────────────┘                  │
│     ↓                                                           │
│  4. Aggregate all hook responses                                │
│     ↓                                                           │
│  5. Make decision: allow / deny / ask                           │
│     ↓                                                           │
│  6. If denied: show systemMessage to Claude                     │
│     If allowed: execute tool and trigger PostToolUse            │
└─────────────────────────────────────────────────────────────────┘
         │                                        ↑
         │ stdin (JSON)                           │ stdout (JSON)
         ↓                                        │
┌─────────────────────────────────────────────────────────────────┐
│                      Hook Script (Python/Bash)                  │
│                                                                 │
│  1. Read JSON from stdin                                        │
│  2. Parse event data (tool_name, tool_input, etc.)             │
│  3. Apply validation logic                                      │
│     - Pattern matching                                          │
│     - LLM evaluation                                            │
│     - External tool checks                                      │
│  4. Make decision: allow / deny / warn                          │
│  5. Output JSON response to stdout                              │
│  6. Exit with appropriate code                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Collection Points

Hooks can collect various data at different stages:

| Event | What Can Be Collected |
|-------|----------------------|
| **PreToolUse** | Tool name, tool parameters, command strings, file paths, edit content |
| **PostToolUse** | Tool results, stdout/stderr, success/failure status |
| **UserPromptSubmit** | User's raw input, prompt text |
| **Stop** | Stop reason, full session transcript, final state |
| **SessionStart** | Project context, environment variables, configuration |
| **SessionEnd** | Session duration, tools used, errors encountered |

## Implementation Examples

### Example 1: Logging All Tool Usage

Create a hook that logs every tool Claude uses:

**hooks/hooks.json:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/logger.py"
          }
        ]
      }
    ]
  }
}
```

**hooks/logger.py:**
```python
#!/usr/bin/env python3
import json
import sys
from datetime import datetime

def main():
    # Read input
    input_data = json.load(sys.stdin)
    
    # Extract data
    tool_name = input_data.get('tool_name', 'Unknown')
    tool_input = input_data.get('tool_input', {})
    
    # Log to file
    log_entry = {
        'timestamp': datetime.now().isoformat(),
        'tool': tool_name,
        'input': tool_input
    }
    
    with open('/tmp/claude_tool_log.json', 'a') as f:
        f.write(json.dumps(log_entry) + '\n')
    
    # Allow operation (exit 0, no output)
    sys.exit(0)

if __name__ == '__main__':
    main()
```

### Example 2: Pattern-Based Validation

The hookify plugin uses pattern matching to validate actions:

**.claude/hookify.dangerous-rm.local.md:**
```markdown
---
name: block-dangerous-rm
enabled: true
event: bash
pattern: rm\s+-rf
action: block
---

⚠️ **Dangerous rm command detected!**

This command could delete important files. Please verify the path.
```

**How it works:**
1. Rule is loaded from markdown file with YAML frontmatter
2. When Bash tool is used, PreToolUse hook fires
3. Hook loads all enabled rules for "bash" event
4. Pattern `rm\s+-rf` is matched against command string using regex
5. If match and action="block", hook returns deny decision
6. Command is blocked and message shown to Claude

### Example 3: Security Pattern Detection

The security-guidance plugin scans for security issues:

```python
SECURITY_PATTERNS = [
    {
        "ruleName": "eval_injection",
        "substrings": ["eval("],
        "reminder": "⚠️ eval() executes arbitrary code and is a security risk."
    },
    {
        "ruleName": "innerHTML_xss",
        "substrings": [".innerHTML ="],
        "reminder": "⚠️ innerHTML can lead to XSS vulnerabilities."
    }
]

def check_patterns(file_path, content):
    for pattern in SECURITY_PATTERNS:
        for substring in pattern["substrings"]:
            if substring in content:
                return pattern["ruleName"], pattern["reminder"]
    return None, None
```

## Creating Custom Hooks

To create your own hooks:

### 1. Create Plugin Structure
```
my-plugin/
├── .claude-plugin/
│   └── plugin.json
├── hooks/
│   ├── hooks.json
│   └── my_hook.py
└── README.md
```

### 2. Define Hook Events (hooks/hooks.json)
```json
{
  "description": "My custom hooks",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/my_hook.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### 3. Implement Hook Script (hooks/my_hook.py)
```python
#!/usr/bin/env python3
import json
import sys

def main():
    # Read event data from stdin
    input_data = json.load(sys.stdin)
    
    tool_name = input_data.get('tool_name', '')
    tool_input = input_data.get('tool_input', {})
    
    # Your validation logic here
    if should_block(tool_input):
        # Block the operation
        result = {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny"
            },
            "systemMessage": "Operation blocked: [reason]"
        }
        print(json.dumps(result), file=sys.stdout)
        sys.exit(0)
    
    # Allow operation (exit 0, no output)
    sys.exit(0)

if __name__ == '__main__':
    main()
```

### 4. Install and Use
```bash
# Install plugin
cc --plugin-dir /path/to/my-plugin

# Hook will automatically execute on matching events
```

## Key Takeaways

1. **Hooks are the interception mechanism** - They allow external scripts to intercept and validate Claude's actions before they execute

2. **JSON-based communication** - All data flows via stdin/stdout as JSON, making it language-agnostic

3. **Event-driven architecture** - Multiple hook events (PreToolUse, PostToolUse, Stop, etc.) cover the entire lifecycle

4. **Flexible validation** - Hooks can use pattern matching, LLM reasoning, external tools, or custom logic

5. **Non-invasive** - Hooks run in separate processes with timeouts, preventing blocking the main Claude Code process

6. **Composable** - Multiple plugins can register hooks for the same events, and all are evaluated

7. **Fail-safe** - Hook errors don't crash Claude Code; operations proceed if hooks fail

## Further Reading

- [Official Claude Code Plugin Documentation](https://docs.anthropic.com/en/docs/claude-code/plugins)
- [Hookify Plugin README](../plugins/hookify/README.md)
- [Security Guidance Plugin](../plugins/security-guidance/)
- [Hook Development Skill](../plugins/plugin-dev/skills/hook-development/SKILL.md)
- [Example Hooks](../examples/hooks/)
