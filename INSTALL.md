# Install Codex Experts Skill

## Step 1: Install skill files

```bash
git clone --depth 1 https://github.com/calinfaja/kln-claude-codex-experts.git /tmp/skill-codex-experts && \
mkdir -p ~/.claude/skills && \
rm -rf ~/.claude/skills/codex-experts && \
cp -r /tmp/skill-codex-experts/ ~/.claude/skills/codex-experts && \
rm -rf /tmp/skill-codex-experts
```

## Step 2: Add Bash permission for Codex

Background subagents need explicit permission to run `codex` commands. Add `Bash(codex:*)` to your **global** `~/.claude/settings.json`:

```bash
# If you already have a settings.json, add "Bash(codex:*)" to permissions.allow manually.
# If you don't have one yet, create it:
cat > ~/.claude/settings.json << 'EOF'
{
  "permissions": {
    "allow": [
      "Bash(codex:*)"
    ]
  }
}
EOF
```

If you already have a `~/.claude/settings.json`, merge the permission into your existing `permissions.allow` array. **Do not** put this in `settings.local.json` — subagents only inherit permissions from `settings.json` ([known issue](https://github.com/anthropics/claude-code/issues/18950)).

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
