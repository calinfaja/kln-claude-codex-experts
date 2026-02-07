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

**IMPORTANT**: Do NOT use heredoc (`<<'EOF'`) or pipe (`cat ... |`) to pass prompts. These cause the `Bash(codex:*)` permission pattern to fail in background subagents because the permission matcher sees `cat` or the heredoc delimiter as the command, not `codex`.

Write the combined prompt to `/tmp/codex-prompt.txt` using the Write tool **before** dispatching the Task. This avoids quoting issues with expert prompts that contain apostrophes.

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
      "$(cat /tmp/codex-prompt.txt)" 2>/dev/null
```

`"$(cat /tmp/codex-prompt.txt)"` is a positional argument (command substitution), not a pipe, so it matches the `Bash(codex:*)` permission pattern.

Always use `--skip-git-repo-check`. Always append `2>/dev/null` to suppress thinking tokens unless user requests them.

**Fallback**: If the Task tool is unavailable, run directly via Bash. The skill works either way — Task tool is preferred, not required.

### Step 5: Synthesize

After the Task agent returns:
1. Present the structured output (verdicts, ratings, findings) in Claude's voice
2. Preserve any tables, checklists, or severity ratings from the expert's response format
3. Suggest a second opinion when the expert's recommendation involves a significant trade-off. Use this pairing table to pick the natural counterbalance:

| Expert used | Suggest second opinion from | When |
|-------------|----------------------------|------|
| `architect` | `simplifier` | Recommends new layers, services, or abstractions |
| `simplifier` | `architect` | Proposes removing something that may exist for structural reasons |
| `implementer` | `code-reviewer` | After implementation is done |
| `code-reviewer` | `security-analyst` | Findings touch auth, input handling, or data flow |
| `scope-analyst` | `plan-reviewer` | Scope looks large or ambiguous |
| `security-analyst` | `implementer` | After identifying fixes that need to be applied |

   Format: "The [expert] suggests [summary]. Want a second opinion from the [counterbalance] to [reason]?"

   If the user accepts, follow the Switch Expert flow (new session, not resume) and prepend the first expert's key findings to the second expert's prompt so it knows what it's evaluating. Example combined prompt:
   ```
   {counterbalance expert prompt from references/experts/}
   ---
   A previous [expert] analysis recommended: [key findings summary]
   ---
   Evaluate whether this recommendation is justified. {user's original task}
   ```

   Skip the suggestion when the output is straightforward, low-stakes, or the user already asked for a specific expert only.
4. Inform the user: "You can resume this session with 'codex resume' or ask for a different expert's perspective."

## Plain Codex Mode (No Expert)

When no expert matches, fall back to original behavior:

1. Ask model + reasoning effort via `AskUserQuestion` (single prompt, two questions)
2. Select sandbox mode for the task (default: `read-only`)
3. Write prompt to `/tmp/codex-prompt.txt`, then run: `codex exec -m {model} --config model_reasoning_effort="{effort}" --sandbox {mode} --full-auto --skip-git-repo-check "$(cat /tmp/codex-prompt.txt)" 2>/dev/null`
4. Summarize output and offer resume

## Session Management

### Resume
```bash
# Resume most recent session (from current working directory):
codex exec --skip-git-repo-check resume --last '{follow-up prompt}' 2>/dev/null

# Resume a specific session by ID:
codex exec --skip-git-repo-check resume {SESSION_ID} '{follow-up prompt}' 2>/dev/null
```
Use `--last` for quick follow-ups on the most recent session. Use a session ID when multiple sessions exist (e.g., after a second opinion). Session IDs are stored under `~/.codex/sessions/`. The resumed session inherits its original model, reasoning, and sandbox settings.

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

### 9. Second Opinion
**User**: "Ask the architect to review our API gateway design"
**Route**: `architect` (triggers: "architect", "design")
**Action**: Read `references/experts/architect.md`, combine with task, run with reasoning=high, sandbox=read-only
**Codex returns**: Recommends splitting into 3 microservices with an event bus
**Second opinion trigger**: Architect recommends new layers → suggest simplifier
**Claude says**: "The architect recommends splitting into 3 microservices with an event bus. Want a second opinion from the simplifier to check if that complexity is justified?"
**User**: "yes"
**Action**: Read `references/experts/simplifier.md`, prepend architect's key findings, run new session with the combined prompt

## Critical Evaluation of Codex Output

Codex is powered by OpenAI models with their own knowledge cutoffs and limitations. Treat Codex as a **colleague, not an authority**.

### Guidelines
- **Trust your own knowledge** when confident. If Codex claims something you know is incorrect, push back directly.
- **Research disagreements** using WebSearch or documentation before accepting Codex's claims.
- **Remember knowledge cutoffs** — Codex may not know about recent releases, APIs, or changes after its training data.
- **Don't defer blindly** — evaluate Codex suggestions critically, especially regarding:
  - Model names and capabilities
  - Recent library versions or API changes
  - Best practices that may have evolved

### When Codex is Wrong
1. State your disagreement clearly to the user
2. Provide evidence (your own knowledge, web search, docs)
3. Optionally resume the Codex session to discuss. Identify yourself as Claude so Codex knows it's a peer AI discussion:
   ```bash
   codex exec --skip-git-repo-check resume --last 'This is Claude following up. I disagree with [X] because [evidence]. What is your take on this?' 2>/dev/null
   ```
4. Frame disagreements as discussions, not corrections — either AI could be wrong
5. Let the user decide how to proceed if there is genuine ambiguity

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
