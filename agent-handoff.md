---
name: agent-handoff
description: Triggers when the user wants to handoff the current task to another agent (Codex, Claude, Antigravity) or resume a task from another agent. Handles both exporting current state and parsing the previous agent's logs to ensure seamless context switching.
---

# Agent Handoff Protocol

When the user asks to switch agents, hand off a task, or resume a task from another agent, follow these procedures.

## SCENARIO A: Handing OFF to another agent (Exporting State)
If the user says "I am switching to Claude" or "Please prepare a handoff":
1. Gather all current context: what you have completed, what tests pass, what files were edited, and exactly what the remaining next steps are.
2. Write this detailed summary to a file named `handoff.md` in the root of the project workspace.
3. Tell the user the handoff is ready and they can now boot up the other agent.

## Resuming FROM another agent (Importing State)
If the user says "Pick up where Claude/Codex left off":
1. **Locate and Parse the Logs (Primary)**: You must parse the raw agent logs to seamlessly import their exact state.
   - **Codex**: Logs are in `~/.codex/sessions/` (Grouped by year/month/day).
   - **Claude**: Logs are in `~/.claude/projects/` (Grouped by project path, e.g., `-mnt-e-dev-myproject`).
   - **Antigravity**: Logs are in `~/.gemini/antigravity-cli/brain/` (Nested in conversation ID folders).
2. **Parse**: Extract the conversation by running the Python script below.
3. **Analyze & Summarize**: Read the last few exchanges, determine exactly what was completed vs blocked, and summarize it for the user.
4. **Fallback - Check for `handoff.md`**: If the logs cannot be found or parsed, look in the project root for `handoff.md` as a fallback.
5. **Resume**: Ask the user to confirm the next steps and proceed.

### Parsing Script
Use the terminal to execute this Python snippet to parse the raw JSONL logs (handles all 3 formats):

```python
import json
import sys
import os

fname = '<path-to-session.jsonl>'
if not os.path.exists(fname):
    print(f"File not found: {fname}")
    sys.exit(1)

with open(fname) as f:
    for line in f:
        line = line.strip()
        if not line: continue
        obj = json.loads(line)
        
        # Codex/Antigravity format
        payload = obj.get('payload')
        if isinstance(payload, dict):
            ptype = payload.get('type', '')
            ts = obj.get('timestamp', '')
            if ptype in ('user_message', 'USER_INPUT'):
                msg = payload.get('message', payload.get('content', ''))
                if msg: print(f'\n=== USER [{ts}] ===\n{msg[:1000]}')
            elif ptype in ('message', 'agent_message', 'PLANNER_RESPONSE'):
                text = payload.get('text', payload.get('content', ''))
                if text and not text.startswith('<model_switch>') and not text.startswith('<environment_context>'):
                    print(f'\n=== ASSISTANT [{ts}] ===\n{text[:1500]}')
        
        # Claude format
        else:
            typ = obj.get('type')
            if typ == 'user':
                msg = obj.get('message', {}).get('content', '')
                if msg and '<local-command-caveat>' not in msg and '<command-name>' not in msg:
                    print(f'\n=== USER ===\n{msg[:1000]}')
            elif typ == 'assistant':
                content = obj.get('message', {}).get('content', [])
                text = ''
                if isinstance(content, list):
                    for c in content:
                        if c.get('type') == 'text':
                            text += c.get('text', '')
                if text:
                    print(f'\n=== ASSISTANT ===\n{text[:1500]}')
```
