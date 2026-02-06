---
name: codex-experts
description: Use when the user asks to run Codex CLI, delegate to an expert (architect, security, reviewer, simplifier, implementer, researcher), or references OpenAI Codex for code analysis, codebase exploration, refactoring, security review, simplification, or automated editing. Triggers on codex, delegate, ask architect, review security, analyze scope, review plan, simplify, implement, explore codebase, find how X is used.
---

# Codex Experts Skill

Extends Codex CLI with expert delegation. Routes tasks to specialized personas (architect, code-reviewer, security-analyst, plan-reviewer, scope-analyst, simplifier, implementer, researcher) that run as Codex sessions with tuned reasoning and structured output.

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
| `researcher` | explore codebase, find files, trace function, map dependencies, gather context, how is X used | high | read-only |

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
2. **Model**: Default to `gpt-5.3-codex` unless the user specifies a different model. Available models: `gpt-5.3-codex` (latest, recommended), `gpt-5.2-codex`, `gpt-5`. Only ask the user if they haven't specified a preference.
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

**IMPORTANT**: Pass the prompt as a positional argument (single-quoted), NOT via heredoc (`<<'EOF'`) or pipe (`cat ... |`). Heredoc and pipe syntax cause the `Bash(codex:*)` permission pattern to fail in background subagents because the permission matcher sees `cat` or the multi-line heredoc delimiter as the command, not `codex`.

```
Task tool:
  subagent_type: Bash
  description: "Codex {expert-name}: {short task summary}"
  prompt: |
    Run this command and return the full output:
    codex exec -m {model} \
      --config model_reasoning_effort="{effort}" \
      --sandbox {sandbox_mode} \
      --full-auto \
      --skip-git-repo-check \
      '{combined_prompt}' 2>/dev/null
```

If the prompt contains single quotes, escape them as `'\''` or write the prompt to a temp file first, then use `"$(cat /tmp/codex-prompt.txt)"` as the argument.

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
3. Run: `codex exec -m {model} --config model_reasoning_effort="{effort}" --sandbox {mode} --full-auto --skip-git-repo-check '{prompt}' 2>/dev/null` with the prompt as a positional argument
4. Summarize output and offer resume

## Session Management

### Resume
```bash
codex exec --skip-git-repo-check resume --last '{prompt}' 2>/dev/null
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

### 7. Codebase Research
**User**: "Use codex to find how authenticate() is called across the codebase, what files import it, and map the auth flow"
**Route**: `researcher` (triggers: "find", "how is X used", "map")
**Action**: Read `references/experts/researcher.md`, combine with task, run with reasoning=high, sandbox=read-only. Returns structured report with file paths, line numbers, signatures, and call graph.

### 8. Plain Codex (No Expert)
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
