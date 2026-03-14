# Project Index: skill-codex-experts

Generated: 2026-03-14

## Project Structure

```
skill-codex-experts/
├── README.md                 # Overview, benefits, quick install
├── SKILL.md                  # Full skill spec: routing, execution, examples
├── INSTALL.md                # 3-step setup guide
├── LICENSE                   # MIT (2026, Calin Faja)
└── references/
    └── experts/              # 9 expert role definitions
        ├── architect.md
        ├── autoresearcher.md
        ├── code-reviewer.md
        ├── implementer.md
        ├── plan-reviewer.md
        ├── researcher.md
        ├── scope-analyst.md
        ├── security-analyst.md
        └── simplifier.md
```

13 files | ~1,320 lines | No executable code -- prompt-only skill

## What This Is

A Claude Code skill that delegates analysis tasks to OpenAI Codex CLI via specialized expert agents. Each expert runs in its own 400k-context Codex process with full repo access. Only structured summaries return to Claude's context (zero context cost).

**Architecture**: Claude orchestrates (routing, synthesis) -> Codex executes (deep analysis in sandbox)

## Entry Points

- **Skill activation**: SKILL.md frontmatter triggers on keywords (codex, delegate, architect, security, autoresearch, etc.)
- **Expert routing**: SKILL.md routing table maps user intent to expert + reasoning level + sandbox mode
- **Expert prompts**: `references/experts/{role}.md` loaded on-demand per dispatch

## Expert Roster

| Expert | File | Lines | Reasoning | Sandbox | Trigger Keywords |
|--------|------|-------|-----------|---------|-----------------|
| architect | references/experts/architect.md | 71 | high | read-only | system design, architecture, tradeoffs, scaling |
| autoresearcher | references/experts/autoresearcher.md | 170 | high | read-only | autoresearch, iterate on, deep research, investigate |
| code-reviewer | references/experts/code-reviewer.md | 72 | medium | read-only | review code, PR review, code quality |
| implementer | references/experts/implementer.md | 86 | high | workspace-write | implement, build feature, execute plan |
| plan-reviewer | references/experts/plan-reviewer.md | 83 | medium | read-only | review plan, validate plan, RFC review |
| researcher | references/experts/researcher.md | 89 | high | read-only | explore codebase, trace function, map dependencies |
| scope-analyst | references/experts/scope-analyst.md | 83 | medium | read-only | scope, requirements, ambiguity, pre-planning |
| security-analyst | references/experts/security-analyst.md | 71 | xhigh | read-only | security, vulnerabilities, auth, OWASP |
| simplifier | references/experts/simplifier.md | 72 | medium | read-only | simplify, reduce complexity, over-engineered |

**Sandbox override**: "fix", "implement", "apply", "change" -> workspace-write (except simplifier and autoresearcher, always read-only)

## Expert Prompt Template

Each expert file follows this structure:
1. **Role declaration** -- who the expert is
2. **Context** -- when this expert is invoked
3. **Analysis Framework** -- structured evaluation method (checklists, scoring, etc.)
4. **Response Format** -- advisory mode and implementation mode output templates
5. **Checklist** -- pre-submission quality gates

## Execution Model

### Single Expert (foreground)
```
User prompt -> Claude routes -> Expert prompt loaded from references/experts/
-> Written to /tmp/codex-{expert}-{TS}-prompt.txt
-> Task tool: codex exec -m {model} --config model_reasoning_effort="{effort}" --sandbox {mode} "$(cat /tmp/codex-{expert}-{TS}-prompt.txt)"
-> Claude synthesizes summary
```

### Parallel Multi-Expert (background)
```
Each expert prompt -> /tmp/codex-{role}-{TS}-prompt.txt
-> python3 subprocess.Popen launches all codex processes concurrently
-> Output -> /tmp/codex-{role}-{TS}-output.txt
-> Claude reads outputs and presents consolidated report
```

Parallel 3-expert review: ~5-6 min wall clock (vs ~18 min sequential)

### Parallel Autoresearch
```
Each topic prompt -> /tmp/codex-autoresearcher-{topic}-{TS}-prompt.txt
-> python3 subprocess.Popen launches N codex processes (read-only)
-> Each iterates on its research independently
-> Output -> /tmp/codex-autoresearcher-{topic}-{TS}-output.txt
-> Claude presents consolidated research report
```

## Key Features

- **Auto-routing**: Keywords in user prompt select the expert automatically
- **Parallel autoresearch**: Multiple topics researched simultaneously in separate Codex processes
- **Second opinion pairing**: architect/simplifier, implementer/code-reviewer, code-reviewer/security-analyst, scope-analyst/plan-reviewer, autoresearcher/architect
- **Critical evaluation**: Claude treats Codex output as peer opinion, pushes back with evidence when wrong
- **Session resume**: `codex resume --last` or by session ID from `~/.codex/sessions/`
- **Timestamped output**: All prompt/output files include `{TS}` (YYYYMMDD-HHMMSS) to prevent collisions
- **Plain Codex fallback**: No expert match -> asks model/reasoning, runs vanilla codex exec

## Configuration

| What | Where | Purpose |
|------|-------|---------|
| Codex credentials | `~/.codex/config.toml` | API keys, feature flags, profiles |
| Bash permissions | `~/.claude/settings.json` | `Bash(codex:*)` + `Bash(python3:*)` |
| Codex sessions | `~/.codex/sessions/` | Resume previous expert sessions |

**Dependencies**: codex CLI, python3, bash

## Documentation Map

| File | Audience | Content |
|------|----------|---------|
| README.md (199 lines) | New users | Problem statement, expert table, install, usage examples |
| SKILL.md (327 lines) | Claude Code | Full skill spec: routing table, command builder, execution model, error handling |
| INSTALL.md (51 lines) | Installers | git clone, permissions setup, troubleshooting |

## Credits

Based on: skill-codex, claude-delegator, oh-my-opencode, claude-plugins-official, superpowers, karpathy/autoresearch, uditgoenka/autoresearch
