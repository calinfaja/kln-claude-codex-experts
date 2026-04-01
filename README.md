# Codex Experts - Expert Delegation for Claude Code

Run expert analysis (security audits, architecture reviews, code reviews) through OpenAI Codex **without bloating Claude's context**. Claude routes the task, Codex does the heavy lifting in its own 400k context window with full repo access, and only the structured summary comes back.

## Why this matters

- **Zero context cost**: Codex runs as a separate process. Its output stays in a Task subagent — your Claude conversation stays clean.
- **Full codebase access**: Codex reads your repo directly via its sandbox. No need to paste files or reference paths — it explores on its own.
- **Expert-tuned reasoning**: Each expert runs with calibrated reasoning effort (security=xhigh, architecture/adversarial=high, reviews=medium) so you're not overpaying for simple tasks.
- **Two model tiers**: `gpt-5.3-codex` for coding experts (implementer, code-reviewer, simplifier), `gpt-5.4` for analysis experts (architect, adversarial-reviewer, security-analyst, researcher, autoresearcher, scope-analyst, plan-reviewer). Override per-request with `gpt-5.4-mini`, `gpt-5.4-nano`, or `spark`.
- **Full Codex features**: `codex exec` reads your `~/.codex/config.toml` — MCP servers, feature flags, profiles, `AGENTS.md`, web search all work. If you've configured Codex with extra tools (DB explorers, Jira, Sentry, etc.), experts get them automatically.
- **Structured review output**: Review experts emit JSON conforming to `references/schemas/review-output.schema.json` alongside human-readable summaries.
- **Diff-aware reviews**: Review experts automatically receive the relevant `git diff` so Codex starts with the changes instead of discovering them.
- **Job tracking**: Background jobs are logged to `/tmp/codex-experts-jobs.json`. Check with "codex status", cancel with "codex cancel".

## Experts

| Expert | When To Use | Example Prompt |
|--------|------------|----------------|
| **Architect** | Evaluating design decisions, scaling strategies, component boundaries | `Ask the codex architect if splitting payments into a microservice is worth it` |
| **Code Reviewer** | PR reviews, code quality checks, catching bugs before merge | `Use codex to review the changes in src/api/ for correctness and performance` |
| **Security Analyst** | Auth flows, input validation, OWASP compliance, threat modeling | `Use codex to security review the auth module and session handling` |
| **Adversarial Reviewer** | Pressure-testing changes before shipping, devil's advocate, breaking confidence | `Pressure test the payment changes before we ship` |
| **Plan Reviewer** | Validating implementation plans before writing code | `Use codex to review my plan for migrating to PostgreSQL` |
| **Scope Analyst** | Clarifying ambiguous requirements before planning | `Use codex to analyze the scope of "add multi-tenant support"` |
| **Simplifier** | Finding unnecessary complexity, dead code, over-engineering | `Use codex to find what can be simplified in src/services/` |
| **Implementer** | Executing well-defined tasks: code, tests, commit-ready output | `Use codex to implement the caching layer from the plan` |
| **Researcher** | Quick codebase lookup: find a function, trace a call, map one flow | `Use codex to find how authenticate() is called and map the auth flow` |
| **Autoresearcher** | Deep investigation across multiple iterations or parallel topics | `Use codex to autoresearch these 3 topics in parallel: auth, db, errors` |

## Quick Install

Paste this into Claude Code:

> Fetch and follow the instructions from https://raw.githubusercontent.com/calinfaja/kln-claude-codex-experts/main/INSTALL.md

That's it. Claude will clone the repo and set up the skill.

### Manual Install

```bash
git clone --depth 1 https://github.com/calinfaja/kln-claude-codex-experts.git /tmp/skill-codex-experts && \
mkdir -p ~/.claude/skills && \
rm -rf ~/.claude/skills/codex-experts && \
cp -r /tmp/skill-codex-experts/ ~/.claude/skills/codex-experts && \
rm -rf /tmp/skill-codex-experts
```

### Prerequisites

- `codex` CLI installed and on `PATH` ([OpenAI Codex CLI](https://github.com/openai/codex))
- Codex configured with valid credentials
- Verify: `codex --version`

### Required: Bash Permissions

Add **both** permissions to your **global** `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(codex:*)",
      "Bash(python3:*)"
    ]
  }
}
```

- `Bash(codex:*)` -- enables single-expert execution via foreground Task agent
- `Bash(python3:*)` -- enables parallel multi-expert execution via Python subprocess wrapper

> **Why `settings.json` and not `settings.local.json`?** Due to a [known Claude Code issue](https://github.com/anthropics/claude-code/issues/18950), subagents don't inherit permissions from `settings.local.json`. Only `settings.json` propagates correctly.

> **Why `python3` for parallel?** Background Task agents (`run_in_background: true`) don't inherit `Bash(codex:*)` permissions. The parallel pattern uses `python3 subprocess.Popen` to launch multiple codex processes -- `Bash(python3:*)` is auto-approved even in background contexts.

> **Restart Claude Code** after installing the skill and updating permissions. Skills and permissions are loaded at startup.

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

# Routes to adversarial-reviewer (high reasoning, always read-only)
"Pressure test the auth changes before we ship"

# Routes to autoresearcher (deep iterative research, read-only)
"Use codex to autoresearch the authentication flow — map all entry points and session handling"

# Check running codex jobs
"codex status"

# Cancel a running job
"codex cancel the security review"

# Routes to autoresearcher x3 in parallel
"Use codex to autoresearch these 3 topics in parallel:
  1. Auth flow in src/auth/
  2. Database patterns in src/models/
  3. Error handling in src/api/"

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
                                       - Own 400k context window
                                       - Full repo access in sandbox
                                       - Expert prompt shapes analysis
                                       - Web search available
                                          |
                                     Returns structured findings
                                          |
Claude Code <── synthesized summary ──────┘
  (only the summary enters your context)
```

### Parallel Multi-Expert Execution

Run multiple experts simultaneously for comprehensive reviews:

```
# Routes to 3 experts in parallel
"Use 3 codex experts to review this PR: code-reviewer, architect, security"
```

Claude writes each expert's prompt to a file, then launches all codex processes in parallel via a Python subprocess wrapper. Results are collected from output files and presented as a consolidated report. A 3-expert parallel review typically completes in 5-6 minutes (wall clock), compared to 15-18 minutes sequential.

**How parallel works under the hood:**
1. Each expert prompt is written to `/tmp/codex-{role}-{TS}-prompt.txt` (timestamped to prevent collisions)
2. A single `python3` command launches all codex processes via `subprocess.Popen`
3. Each process reads its prompt from stdin and writes output to `/tmp/codex-{role}-{TS}-output.txt`
4. All processes run concurrently with full repo access via `--cd`

### Parallel Autoresearch

Run deep research on multiple topics simultaneously:

```
"Use codex to autoresearch these 3 topics in parallel, 10 iterations each:
  1. Authentication flow in src/auth/ — entry points, middleware, session handling
  2. Database access patterns in src/models/ — queries, transactions, pooling
  3. Error handling in src/api/ — error types, propagation, user-facing messages"
```

Each topic runs in its own Codex process (read-only, 400k context, full repo access). Each iteratively deepens its research — exploring more files, tracing more calls, refining findings each iteration. Results come back as per-topic technical reports with file paths, code snippets, and dependency maps.

Wall clock time: the slowest topic, not the sum of all three.

### Thinking Tokens

Thinking tokens (stderr) are suppressed by default with `2>/dev/null`. Ask Claude to show them if you need to debug Codex's reasoning.

### Second Opinion

When an expert recommends something with significant trade-offs, Claude will suggest a counterbalance expert. For example, if the architect recommends adding microservices, Claude offers the simplifier's take on whether the complexity is justified.

Natural pairings: architect/simplifier, implementer/code-reviewer, code-reviewer/security-analyst, adversarial-reviewer/implementer, scope-analyst/plan-reviewer, autoresearcher/architect. The second expert receives the first expert's key findings so it knows what it's evaluating.

### Critical Evaluation

Claude treats Codex output as a colleague's opinion, not an authority. If Codex produces something Claude knows is incorrect (stale APIs, wrong model names, outdated practices), Claude will push back, provide evidence, and let you decide.

### Session Resume

Say "codex resume" to continue the last session, or resume a specific session by ID for targeted follow-ups. Sessions are stored under `~/.codex/sessions/` and inherit their original model, reasoning, and sandbox settings.

### Switch Expert

Ask for a different expert's perspective on the same topic. Claude will start a new session with the new expert's prompt and carry forward relevant context.

## Credits

- [skill-codex](https://github.com/skills-directory/skill-codex) - Original Codex skill
- [claude-delegator](https://github.com/jarrodwatts/claude-delegator) by [@jarrodwatts](https://github.com/jarrodwatts) - Expert delegation patterns
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) by [@code-yeongyu](https://github.com/code-yeongyu) - Scope analyst and plan reviewer frameworks
- [claude-plugins-official](https://github.com/anthropics/claude-plugins-official) - Anthropic's code-simplifier agent (adapted as advisory-only simplifier)
- [superpowers](https://github.com/obra/superpowers) by [@obra](https://github.com/obra) - Implementer subagent pattern from subagent-driven-development
- [autoresearch](https://github.com/karpathy/autoresearch) by [@karpathy](https://github.com/karpathy) - Autonomous iteration principles (constraint + metric + iteration)
- [claude-autoresearch](https://github.com/uditgoenka/autoresearch) by [@uditgoenka](https://github.com/uditgoenka) - Generalized autoresearch skill for Claude Code

## License

MIT - see [LICENSE](LICENSE).
