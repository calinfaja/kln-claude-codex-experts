---
name: codex-experts
description: Use when the user asks to run Codex CLI, delegate to an expert (architect, security, reviewer, simplifier, implementer), or references OpenAI Codex for code analysis, refactoring, security review, simplification, or automated editing. Triggers on codex, delegate, ask architect, review security, analyze scope, review plan, simplify, implement.
---

# Codex Experts Skill

Extends Codex CLI with expert delegation. Routes tasks to specialized personas (architect, code-reviewer, security-analyst, plan-reviewer, scope-analyst, simplifier, implementer) that run as Codex sessions with tuned reasoning and structured output.

## Expert Routing Table

Match the user's task against these patterns. If no expert matches, run plain Codex (no expert prompt).

| Expert | Trigger Patterns | Reasoning | Default Sandbox |
|--------|-----------------|-----------|-----------------|
| `architect` | system design, architecture, tradeoffs, scaling, component boundaries, database schema | high | read-only |
| `code-reviewer` | review code, PR review, code quality, review changes, review diff | medium | read-only |
| `security-analyst` | security, vulnerabilities, auth, OWASP, threat model, secrets, injection | xhigh | read-only |
| `plan-reviewer` | review plan, validate plan, check plan, RFC review | medium | read-only |
| `scope-analyst` | scope, requirements, ambiguity, pre-planning, analyze request, what's missing | medium | read-only |
| `simplifier` | simplify, reduce complexity, clean up, redundant, over-engineered | medium | read-only |
| `implementer` | implement task, build feature, execute plan, write the code, do the work | high | workspace-write |

**Sandbox override**: If the user says "fix", "implement", "apply", or "change" -> use `workspace-write` instead of `read-only`. Exception: `simplifier` stays `read-only` (advisory only, never modifies code).

## Execution Model

This skill runs **inline** (not `context: fork`) so it can see conversation history for routing decisions. The actual `codex exec` call is dispatched via the **Task tool** to keep Codex output out of the main context.

### Why inline routing + Task execution

- **Inline routing**: Claude needs conversation context to pick the right expert ("review the auth module we discussed" requires knowing which module).
- **Task execution**: Codex output can be large (thousands of tokens). Running via Task tool keeps the main conversation clean — only the synthesized summary comes back.

## Command Builder

Follow these steps to build and execute a Codex command:

### Step 1: Determine Parameters

1. **Expert**: Match task against routing table. If ambiguous, ask using `AskUserQuestion` with the top 2 candidates.
2. **Model**: Ask the user which model (`gpt-5.2-codex` or `gpt-5`) unless already specified. `gpt-5.2-codex` is optimized for agentic coding tasks; `gpt-5` is the general-purpose model. Use a single `AskUserQuestion` prompt combining model + reasoning effort if the expert's defaults aren't overridden.
3. **Reasoning effort**: Use the expert's default from the routing table. Override only if user specifies.
4. **Sandbox**: Use routing table default. Override to `workspace-write` if implementation mode detected.

### Step 2: Load Expert Prompt (if expert matched)

Read the expert's reference file:
```
references/experts/{expert-name}.md
```
This file contains the persona, analysis framework, and response format. Use its full content as the system context for the Codex prompt.

### Step 3: Build the Combined Prompt

Construct the Codex input by combining:
1. The expert's full prompt (from the reference file)
2. A separator: `---`
3. The user's actual task/question
4. Any relevant context (current file, diff, error output)

### Step 4: Execute via Task Tool

Dispatch the codex command using the **Task tool** (`subagent_type: Bash`). This runs in a subagent so codex output stays isolated from the main conversation.

```
Task tool:
  subagent_type: Bash
  description: "Codex {expert-name}: {short task summary}"
  prompt: |
    Run this command and return the full output:
    echo "{combined_prompt}" | codex exec -m {model} \
      --config model_reasoning_effort="{effort}" \
      --sandbox {sandbox_mode} \
      --full-auto \
      --skip-git-repo-check 2>/dev/null
```

Always use `--skip-git-repo-check`. Always append `2>/dev/null` to suppress thinking tokens unless user requests them.

**Fallback**: If the Task tool is unavailable, run directly via Bash. The skill works either way — Task tool is preferred, not required.

### Step 5: Synthesize

After the Task agent returns:
1. Present the structured output (verdicts, ratings, findings) in Claude's voice
2. Preserve any tables, checklists, or severity ratings from the expert's response format
3. Inform the user: "You can resume this session with 'codex resume' or ask for a different expert's perspective."

## Plain Codex Mode (No Expert)

When no expert matches, fall back to original behavior:

1. Ask model + reasoning effort via `AskUserQuestion` (single prompt, two questions)
2. Select sandbox mode for the task (default: `read-only`)
3. Run: `codex exec -m {model} --config model_reasoning_effort="{effort}" --sandbox {mode} --full-auto --skip-git-repo-check "{user_prompt}" 2>/dev/null`
4. Summarize output and offer resume

## Session Management

### Resume
```bash
echo "{prompt}" | codex exec --skip-git-repo-check resume --last 2>/dev/null
```
No config flags on resume unless user explicitly specifies model or reasoning. The resumed session inherits its original settings.

### Switch Expert
If user wants a different expert's take on the same topic:
1. Start a new Codex session (don't resume)
2. Load the new expert's prompt
3. Include context from the previous expert's findings if relevant

## Examples

### 1. Security Review
**User**: "Use codex to review the auth module for security issues"
**Route**: `security-analyst` (triggers: "security", "auth")
**Action**: Read `references/experts/security-analyst.md`, combine with task, run with reasoning=xhigh, sandbox=read-only

### 2. Architecture Consultation
**User**: "Ask the architect about our database schema design"
**Route**: `architect` (triggers: "architect", "database schema")
**Action**: Read `references/experts/architect.md`, combine with task, run with reasoning=high, sandbox=read-only

### 3. Code Review with Fix
**User**: "Review and fix the performance issues in utils.py"
**Route**: `code-reviewer` (triggers: "review") + sandbox override ("fix" detected)
**Action**: Read `references/experts/code-reviewer.md`, combine with task, run with reasoning=medium, sandbox=workspace-write

### 4. Pre-Planning Analysis
**User**: "Analyze the scope of this feature request before we plan"
**Route**: `scope-analyst` (triggers: "scope", "pre-planning")
**Action**: Read `references/experts/scope-analyst.md`, combine with task, run with reasoning=medium, sandbox=read-only

### 5. Simplification Analysis
**User**: "Use codex to find what can be simplified in src/utils/"
**Route**: `simplifier` (triggers: "simplify")
**Action**: Read `references/experts/simplifier.md`, combine with task, run with reasoning=medium, sandbox=read-only. Returns proposals only, no changes.

### 6. Task Implementation
**User**: "Use codex to implement the caching layer from the plan"
**Route**: `implementer` (triggers: "implement", "plan")
**Action**: Read `references/experts/implementer.md`, combine with task, run with reasoning=high, sandbox=workspace-write

### 7. Plain Codex (No Expert)
**User**: "Use codex to refactor the logging module"
**Route**: No expert match (general refactoring)
**Action**: Ask model + reasoning, run plain codex exec with user prompt

## Error Handling

- **Non-zero exit**: Stop and report the error. Ask user before retrying.
- **Missing expert file**: Fall back to plain Codex mode. Inform user the expert reference wasn't found.
- **Permission gates**: Before using `--full-auto`, `--sandbox danger-full-access`, ask user permission via `AskUserQuestion` unless already granted.
- **Stderr warnings**: If output includes warnings or partial results, summarize and ask how to proceed.

## Boundaries

### This skill DOES
- Route tasks to expert personas via Codex CLI
- Load expert prompts on-demand (not at startup)
- Preserve structured output formats (verdicts, ratings, tables)
- Support session resume and expert switching

### This skill DOES NOT
- Run multiple Codex sessions in parallel
- Modify expert prompt files at runtime
- Make decisions without user confirmation on high-impact flags
- Use MCP servers or external infrastructure
