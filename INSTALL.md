# Install Codex Experts Skill

Run these commands to install:

```bash
git clone --depth 1 https://github.com/calinfaja/kln-claude-codex-experts.git /tmp/skill-codex-experts && \
mkdir -p ~/.claude/skills && \
rm -rf ~/.claude/skills/codex-experts && \
cp -r /tmp/skill-codex-experts/ ~/.claude/skills/codex-experts && \
rm -rf /tmp/skill-codex-experts
```

After installing, verify by checking the skill directory exists:

```bash
ls ~/.claude/skills/codex-experts/SKILL.md
```

The skill is now ready. It activates automatically when you mention "codex", "delegate", "ask architect", "review security", or similar trigger phrases.
