# Install Codex Experts Skill

## Step 1: Install skill files

```bash
git clone --depth 1 https://github.com/calinfaja/kln-claude-codex-experts.git /tmp/skill-codex-experts && \
mkdir -p ~/.claude/skills && \
rm -rf ~/.claude/skills/codex-experts && \
cp -r /tmp/skill-codex-experts/ ~/.claude/skills/codex-experts && \
rm -rf /tmp/skill-codex-experts
```

## Step 2: Add Bash permissions for Codex

Background subagents need explicit permission to run `codex` commands. Add both `Bash(codex:*)` and `Bash(python3:*)` to your **global** `~/.claude/settings.json`:

```bash
# If you already have a settings.json, add both to permissions.allow manually.
# If you don't have one yet, create it:
cat > ~/.claude/settings.json << 'EOF'
{
  "permissions": {
    "allow": [
      "Bash(codex:*)",
      "Bash(python3:*)"
    ]
  }
}
EOF
```

- `Bash(codex:*)` — enables single-expert foreground execution via Task tool
- `Bash(python3:*)` — enables parallel multi-expert execution via Python subprocess wrapper

If you already have a `~/.claude/settings.json`, merge the permissions into your existing `permissions.allow` array. **Do not** put this in `settings.local.json` — subagents only inherit permissions from `settings.json` ([known issue](https://github.com/anthropics/claude-code/issues/18950)).

## Step 3: Verify

```bash
ls ~/.claude/skills/codex-experts/SKILL.md
codex --version
```

The skill is now ready. It activates automatically when you mention "codex", "delegate", "ask architect", "review security", or similar trigger phrases.

## Troubleshooting

**Codex commands fail with "Permission auto-denied" in background subagents:**
- Ensure `Bash(codex:*)` is in `~/.claude/settings.json` (not `.local.json`)
- Restart Claude Code after changing settings
- The permission must be in `settings.json` due to a [known Claude Code bug](https://github.com/anthropics/claude-code/issues/18950) where `settings.local.json` permissions don't propagate to subagents
