# Codex Experts - Expert Delegation for Claude Code

Run expert analysis (security audits, architecture reviews, code reviews) through OpenAI Codex **without bloating Claude's context**. Claude routes the task, Codex does the heavy lifting in its own 192k context window with full repo access, and only the structured summary comes back.

### Why this matters

- **Zero context cost**: Codex runs as a separate process. Its output stays in a Task subagent — your Claude conversation stays clean.
- **Full codebase access**: Codex reads your repo directly via its sandbox. No need to paste files or reference paths — it explores on its own.
- **Expert-tuned reasoning**: Each expert runs with calibrated reasoning effort (security=xhigh, architecture=high, reviews=medium) so you're not overpaying for simple tasks.
- **Two models, best of both**: Claude orchestrates (conversation-aware routing), Codex executes (deep analysis with its own tool use and web search).
- **Full Codex features**: `codex exec` reads your `~/.codex/config.toml` — MCP servers, feature flags, profiles, `AGENTS.md`, web search all work. If you've configured Codex with extra tools (DB explorers, Jira, Sentry, etc.), experts get them automatically.

## Experts

| Expert | What It Does | Reasoning |
|--------|-------------|-----------|
| **Architect** | System design, tradeoff analysis, component boundaries | high |
| **Code Reviewer** | Code quality review: Correctness > Security > Performance > Maintainability | medium |
| **Security Analyst** | 8-point OWASP checklist, threat modeling, vulnerability assessment | xhigh |
| **Plan Reviewer** | Validates implementation plans before code is written | medium |
| **Scope Analyst** | Pre-planning ambiguity detection, requirement decomposition | medium |
| **Simplifier** | Proposes simplifications without making changes (advisory only) | medium |
| **Implementer** | Executes well-defined tasks: code, tests, verification, commit-ready | high |
| **Researcher** | Deep codebase exploration: file paths, signatures, call graphs, dependency maps | high |

## Quick Install

Paste this into Claude Code:

> Fetch and follow the instructions from https://raw.githubusercontent.com/calinfaja/kln-claude-codex-experts/main/INSTALL.md

That's it. Claude will clone the repo and set up the skill.

### Manual Install

```bash
git clone --depth 1 https://github.com/calinfaja/kln-claude-codex-experts.git /tmp/skill-codex-experts && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/skill-codex-experts/ ~/.claude/skills/codex-experts && \
rm -rf /tmp/skill-codex-experts
```

### Prerequisites

- `codex` CLI installed and on `PATH` ([OpenAI Codex CLI](https://github.com/openai/codex))
- Codex configured with valid credentials
- Verify: `codex --version`

### Required: Bash Permission

Codex experts run as background subagents via the Task tool. Background subagents auto-deny any Bash command not explicitly permitted. Add this to your **global** `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(codex:*)"
    ]
  }
}
```

> **Why `settings.json` and not `settings.local.json`?** Due to a [known Claude Code issue](https://github.com/anthropics/claude-code/issues/18950), subagents don't inherit permissions from `settings.local.json`. Only `settings.json` propagates correctly.

## Usage

Expert routing is automatic based on your prompt:

```
# Routes to security-analyst (xhigh reasoning)
"Use codex to review the auth module for security issues"

# Routes to architect (high reasoning)
"Ask the architect about our database schema design"

# Routes to code-reviewer with implementation mode
"Review and fix the performance issues in utils.py"

# Routes to scope-analyst
"Analyze the scope of this feature request before we plan"

# Routes to simplifier (advisory only, no changes)
"Use codex to find what can be simplified in src/utils/"

# Routes to implementer (workspace-write)
"Use codex to implement the caching layer from the plan"

# Routes to researcher (read-only, returns structured report)
"Use codex to find how authenticate() is called and map the auth flow"

# No expert match - plain codex mode
"Use codex to refactor the logging module"
```

### How It Works

```
You ── "review auth for security" ──> Claude Code (sees your conversation)
                                          |
                                     Routes to security-analyst
                                     Reads expert prompt (~700 tokens)
                                          |
                                     Dispatches via Task tool (subagent)
                                          |
                                     codex exec (separate process)
                                       - Own 192k context window
                                       - Full repo access in sandbox
                                       - Expert prompt shapes analysis
                                       - Web search available
                                          |
                                     Returns structured findings
                                          |
Claude Code <── synthesized summary ──────┘
  (only the summary enters your context)
```

### Thinking Tokens

Thinking tokens (stderr) are suppressed by default with `2>/dev/null`. Ask Claude to show them if you need to debug Codex's reasoning.

### Session Resume

Say "codex resume" to continue the last session. The resumed session inherits its original model, reasoning, and sandbox settings.

### Switch Expert

Ask for a different expert's perspective on the same topic. Claude will start a new session with the new expert's prompt and carry forward relevant context.

## Credits

- [skill-codex](https://github.com/skills-directory/skill-codex) - Original Codex skill
- [claude-delegator](https://github.com/jarrodwatts/claude-delegator) by [@jarrodwatts](https://github.com/jarrodwatts) - Expert delegation patterns
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) by [@code-yeongyu](https://github.com/code-yeongyu) - Scope analyst and plan reviewer frameworks
- [claude-plugins-official](https://github.com/anthropics/claude-plugins-official) - Anthropic's code-simplifier agent (adapted as advisory-only simplifier)
- [superpowers](https://github.com/obra/superpowers) by [@obra](https://github.com/obra) - Implementer subagent pattern from subagent-driven-development

## License

MIT - see [LICENSE](LICENSE).
