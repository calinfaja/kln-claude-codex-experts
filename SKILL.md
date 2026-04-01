---
name: codex-experts
description: Use when the user asks to run Codex CLI, delegate to an expert (architect, security, reviewer, adversarial reviewer, simplifier, implementer, researcher, autoresearcher), or references OpenAI Codex for code analysis, codebase exploration, refactoring, security review, adversarial review, simplification, automated editing, or autonomous research. Triggers on codex, delegate, ask architect, review security, adversarial review, challenge this, pressure test, devil's advocate, analyze scope, review plan, simplify, implement, explore codebase, find how X is used, autoresearch, iterate on, deep research, investigate, research topic, autonomous research, codex status, codex cancel.
---

# Codex Experts Skill

Extends Codex CLI with expert delegation. Routes tasks to specialized personas (architect, code-reviewer, adversarial-reviewer, security-analyst, plan-reviewer, scope-analyst, simplifier, implementer, researcher, autoresearcher) that run as Codex sessions with tuned reasoning and structured output.

## Expert Routing Table

Match the user's task against these patterns. If no expert matches, run plain Codex (no expert prompt).

| Expert | Trigger Patterns | Reasoning | Default Sandbox |
|--------|-----------------|-----------|-----------------|
| `architect` | system design, architecture, tradeoffs, scaling, component boundaries, database schema | high | read-only |
| `code-reviewer` | review code, PR review, code quality, review changes, review diff | medium | read-only |
| `adversarial-reviewer` | adversarial review, challenge this, pressure test, devil's advocate, break confidence, ship or no-ship | high | read-only |
| `security-analyst` | security, vulnerabilities, auth, OWASP, threat model, secrets, injection | xhigh | read-only |
| `plan-reviewer` | review plan, validate plan, check plan, RFC review | medium | read-only |
| `scope-analyst` | scope, requirements, ambiguity, pre-planning, analyze request, what's missing | medium | read-only |
| `simplifier` | simplify, reduce complexity, clean up, redundant, over-engineered | medium | read-only |
| `implementer` | implement task, build feature, execute plan, write the code, do the work | high | workspace-write |
| `researcher` | explore codebase, find files, trace function, map dependencies, gather context, how is X used | high | read-only |
| `autoresearcher` | autoresearch, iterate on, deep research, investigate, research topic, autonomous research | high | read-only |

**Sandbox override**: If the user says "fix", "implement", "apply", or "change" -> use `workspace-write` instead of `read-only`. Exception: `simplifier`, `adversarial-reviewer`, and `autoresearcher` stay `read-only` (advisory/research only, never modify code).

**Autoresearcher note**: Always `read-only`. Iterates on its own research depth, not on code changes. User specifies: research topic, file scope, and optionally iteration count (default 10) or a research goal for early stopping. Output is a comprehensive technical report.

**Adversarial-reviewer note**: Always `read-only`. Breaks confidence in changes rather than validating them. Outputs structured JSON findings alongside human-readable summary. Use `code-reviewer` for balanced assessment, `adversarial-reviewer` to pressure-test before shipping.

## Execution Model

This skill runs **inline** (not `context: fork`) so it can see conversation history for routing decisions. The actual `codex exec` call is dispatched via the **Task tool** to keep Codex output out of the main context.

### Why inline routing + Task execution

- **Inline routing**: Claude needs conversation context to pick the right expert ("review the auth module we discussed" requires knowing which module).
- **Task execution**: Codex output can be large (thousands of tokens). Running via Task tool keeps the main conversation clean — only the synthesized summary comes back.

## Command Builder

Follow these steps to build and execute a Codex command:

### Step 1: Determine Parameters

1. **Expert**: Match task against routing table. If ambiguous, ask using `AskUserQuestion` with the top 2 candidates.
2. **Model**: Default to `gpt-5.3-codex` for coding experts (implementer, code-reviewer, simplifier) and `gpt-5.4` for all other experts (autoresearcher, architect, adversarial-reviewer, researcher, scope-analyst, plan-reviewer, security-analyst). `gpt-5.3-codex` is the top coding model; `gpt-5.4` is the flagship for reasoning, research, and general-purpose analysis (1M context, computer use, tool search). If user specifies a model (e.g. `gpt-5.4-mini`, `gpt-5.4-nano`, `spark`), use that instead. Map `spark` to `gpt-5.3-codex-spark`.
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

#### Diff-Aware Review Context

For review experts (`code-reviewer`, `adversarial-reviewer`, `security-analyst`), automatically compute and append the relevant diff before dispatching. This gives Codex the changes upfront instead of making it discover them:

- **User mentions a base branch** (e.g. "review against main"): `git diff {base}...HEAD`
- **User mentions staged changes**: `git diff --cached`
- **User mentions a specific PR**: `git diff {base}...HEAD` for the PR's base branch
- **Default** (no scope specified): `git diff HEAD` (all uncommitted changes)

Append the diff to the combined prompt after the task:
```
{expert prompt}
---
{user's task}
---
## Changes to Review
{diff output}
```

If the diff is larger than 80KB, truncate with a note: `[Diff truncated at 80KB — Codex will explore remaining files directly]`. If the diff is empty, note that and let Codex explore the working tree itself.

### Step 4: Execute via Task Tool

Dispatch the codex command using the **Task tool** (`subagent_type: Bash`). This runs in a subagent so codex output stays isolated from the main conversation.

**IMPORTANT**: Do NOT use heredoc (`<<'EOF'`) or pipe (`cat ... |`) to pass prompts. These cause the `Bash(codex:*)` permission pattern to fail because the permission matcher sees `cat` or the heredoc delimiter as the command, not `codex`.

Generate a timestamp: `TS=$(date +%Y%m%d-%H%M%S)`. Write the combined prompt to `/tmp/codex-{expert}-{TS}-prompt.txt` using the Write tool **before** dispatching the Task. This avoids quoting issues with expert prompts that contain apostrophes and prevents collisions across runs.

#### Single Expert (foreground Task agent)

For prompts under ~100KB, use command substitution as a positional argument:

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
      "$(cat /tmp/codex-{expert}-{TS}-prompt.txt)" 2>/dev/null

    If the command fails with "Argument list too long", use stdin redirection instead:
    codex exec -m {model} \
      --config model_reasoning_effort="{effort}" \
      --sandbox {sandbox_mode} \
      --full-auto \
      --skip-git-repo-check \
      - < /tmp/codex-{expert}-{TS}-prompt.txt 2>/dev/null
```

`"$(cat /tmp/codex-{expert}-{TS}-prompt.txt)"` is a positional argument (command substitution), not a pipe, so it matches the `Bash(codex:*)` permission pattern. For large prompts (>100KB, e.g. full PR diffs), the shell may reject the expansion with "Argument list too long" — the stdin fallback (`-` flag) handles this.

#### Multiple Experts in Parallel (Python subprocess)

Background Task agents (`run_in_background: true`) do **not** inherit `Bash(codex:*)` permissions. To run multiple experts in parallel, use a Python subprocess wrapper via `Bash(run_in_background: true)`. The `Bash(python3:*)` permission is auto-approved and bypasses the background limitation.

1. Generate a timestamp for this run: `TS=$(date +%Y%m%d-%H%M%S)`
2. Write each expert's combined prompt to a separate file: `/tmp/codex-{expert}-{TS}-prompt.txt`
3. Run a single background Bash command with Python:

```
Bash tool (run_in_background: true, timeout: 600000):
  python3 -c "
  import subprocess, time
  ts = '{TS}'  # same timestamp Claude used when writing prompt files
  start = time.time()
  procs = []
  configs = [
      ('{expert1}', '{effort1}'),
      ('{expert2}', '{effort2}'),
      ('{expert3}', '{effort3}'),
  ]
  for role, effort in configs:
      out_file = f'/tmp/codex-{role}-{ts}-output.txt'
      p = subprocess.Popen(
          ['codex', 'exec', '-m', '{model}',
           '--config', 'model_reasoning_effort=' + effort,
           '--sandbox', '{sandbox_mode}', '--full-auto', '--skip-git-repo-check',
           '--cd', '{working_dir}',
           '-o', out_file,
           '-'],
          stdin=open(f'/tmp/codex-{role}-{ts}-prompt.txt'),
          stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
      )
      procs.append((role, p, out_file))
      print(f'{role}: launched PID={p.pid} -> {out_file}')
  print(f'All launched in {time.time()-start:.1f}s, waiting...')
  for role, p, out_file in procs:
      rc = p.wait()
      print(f'{role}: done exit={rc} at {time.time()-start:.0f}s -> {out_file}')
  print(f'ALL DONE in {time.time()-start:.0f}s')
  "
```

4. Read output files (`/tmp/codex-{expert}-{TS}-output.txt`) when the background command completes.

**Why this works**: `subprocess.Popen` launches codex as child processes that read prompts from stdin (file objects). Each runs in its own process with full repo access via `--cd`. The `-o` flag writes the final response to a file for easy retrieval.

Always use `--skip-git-repo-check`. Always append `2>/dev/null` (single expert) or `stderr=subprocess.DEVNULL` (parallel) to suppress thinking tokens unless user requests them.

**Fallback**: If the Task tool is unavailable, run directly via Bash. The skill works either way — Task tool is preferred, not required.

### Step 4b: Job Registry

After launching each Codex process, record it in `/tmp/codex-experts-jobs.json` (one JSON object per line, append-only):

```bash
echo '{"id":"'$TS'","expert":"'$ROLE'","pid":'$PID',"started":"'$(date -Iseconds)'","status":"running","output_file":"/tmp/codex-'$ROLE'-'$TS'-output.txt","prompt_file":"/tmp/codex-'$ROLE'-'$TS'-prompt.txt"}' >> /tmp/codex-experts-jobs.json
```

For the parallel Python wrapper, write one line per expert after launching.

After a process completes, update its status by rewriting the line (or let the status check infer it from PID liveness).

#### Job Status ("codex status")

When the user asks "codex status" or "what's running":

1. Read `/tmp/codex-experts-jobs.json`
2. For each entry with `"status":"running"`, check if PID is alive: `kill -0 $PID 2>/dev/null`
3. Present a table:

```
| Expert | Started | Status | Output |
|--------|---------|--------|--------|
| security-analyst | 14:32:05 | running (PID 12345) | /tmp/codex-security-analyst-...-output.txt |
| code-reviewer | 14:32:05 | completed | /tmp/codex-code-reviewer-...-output.txt |
```

4. For completed jobs, offer to read the output file.

#### Job Cancel ("codex cancel")

When the user asks "codex cancel" or "kill codex":

1. Read `/tmp/codex-experts-jobs.json` for running jobs
2. If multiple running, ask which to cancel via `AskUserQuestion`
3. Kill the process: `kill $PID`
4. Inform the user

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
| `adversarial-reviewer` | `implementer` | After identifying issues that need fixing |
| `scope-analyst` | `plan-reviewer` | Scope looks large or ambiguous |
| `security-analyst` | `implementer` | After identifying fixes that need to be applied |
| `autoresearcher` | `architect` | Research reveals structural patterns worth evaluating for design quality |

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
3. Write prompt to `/tmp/codex-plain-{TS}-prompt.txt`, then run: `codex exec -m {model} --config model_reasoning_effort="{effort}" --sandbox {mode} --full-auto --skip-git-repo-check "$(cat /tmp/codex-plain-{TS}-prompt.txt)" 2>/dev/null`
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

### 8. Adversarial Review
**User**: "Pressure test the payment changes before we ship"
**Route**: `adversarial-reviewer` (triggers: "pressure test")
**Action**: Read `references/experts/adversarial-reviewer.md`, compute diff via `git diff HEAD`, combine with task and diff, run with reasoning=high, sandbox=read-only. Returns structured JSON findings + human-readable summary.

### 9. Job Status
**User**: "codex status"
**Route**: Job status check
**Action**: Read `/tmp/codex-experts-jobs.json`, check PID liveness for running jobs, present table of all recent jobs with status.

### 10. Job Cancel
**User**: "codex cancel the security review"
**Route**: Job cancel
**Action**: Read `/tmp/codex-experts-jobs.json`, find matching running job, `kill $PID`, inform user.

### 11. Plain Codex (No Expert)
**User**: "Use codex to refactor the logging module"
**Route**: No expert match (general refactoring)
**Action**: Ask model + reasoning, run plain codex exec with user prompt

### 12. Parallel Multi-Expert Review
**User**: "Use 3 codex experts to review this PR: code-reviewer, architect, security"
**Route**: Multiple experts detected
**Action**: Write each expert prompt to `/tmp/codex-{role}-{TS}-prompt.txt`, launch Python subprocess wrapper via `Bash(run_in_background: true)` with all 3 codex processes. Read `/tmp/codex-{role}-{TS}-output.txt` files when complete. Present consolidated findings.

### 13. Autonomous Research (Single Topic)
**User**: "Use codex to autoresearch the authentication flow — map all entry points, middleware, and session handling"
**Route**: `autoresearcher` (triggers: "autoresearch")
**Action**: Read `references/experts/autoresearcher.md`, combine with topic and scope, run with reasoning=high, sandbox=read-only. Codex iteratively explores the codebase — each iteration digs deeper, follows new leads, refines findings. Returns a comprehensive technical report with file paths, code snippets, call graphs, and dependency maps.

### 14. Parallel Autoresearch (Multiple Topics)
**User**: "Use codex to autoresearch these 3 topics in parallel, 10 iterations each:
1. Authentication flow in src/auth/ — entry points, middleware, session handling
2. Database access patterns in src/models/ — queries, transactions, connection pooling
3. Error handling in src/api/ — error types, propagation, user-facing messages"
**Route**: `autoresearcher` x3 (parallel)
**Action**: Write 3 autoresearcher prompts to `/tmp/codex-autoresearcher-{topic}-{TS}-prompt.txt`, each with its own research topic and scope. Launch via Python subprocess wrapper (same pattern as parallel multi-expert). Each Codex process runs its research loop independently in read-only mode. Read `/tmp/codex-autoresearcher-{topic}-{TS}-output.txt` files when complete. Present consolidated research report across all topics.

### 15. Guided Autoresearch (Claude-Steered)
**User**: "Use codex to autoresearch the payment integration, guide each step"
**Route**: `autoresearcher` in guided mode (trigger: "guide each step")
**Action**: Claude dispatches Codex for ONE research iteration at a time. Codex explores, returns current findings and remaining gaps. Claude reviews, suggests where to dig next, dispatches next iteration. Repeat until research is comprehensive or user stops.

### 16. Second Opinion
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
- Modify expert prompt files at runtime
- Make decisions without user confirmation on high-impact flags
- Use MCP servers or external infrastructure
