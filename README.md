# Codex Experts - Expert Delegation for Claude Code

Extends the [skill-codex](https://github.com/skills-directory/skill-codex) skill with expert delegation. Routes tasks to specialized personas that run as Codex sessions with tuned reasoning effort and structured output.

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

# No expert match - plain codex mode
"Use codex to refactor the logging module"
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
