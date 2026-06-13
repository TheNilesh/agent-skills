# Agent Skills

These are skills maintained for my personal AI agents.

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # Skill definition and instructions
    agents/           # Agent-specific configurations
    references/       # Reference documents used by the skill
```

## Skills

| Skill | Description |
|-------|-------------|
| [casaos](skills/casaos/SKILL.md) | Skills for managing CasaOS home server applications |

## Installation

Replace `<skill-name>` with the skill directory name (e.g. `casaos`).

### Claude Code

```bash
npx skills add TheNilesh/agent-skills/skills/<skill-name> -g
```

Or manually:

```bash
cp -r skills/<skill-name> ~/.claude/skills/
```

### Hermes

```bash
hermes skills install TheNilesh/agent-skills/skills/<skill-name>
```

### Zeroclaw

```bash
# From a local clone
zeroclaw skills install /path/to/agent-skills/skills/<skill-name>

# From GitHub
zeroclaw skills install https://raw.githubusercontent.com/TheNilesh/agent-skills/main/skills/<skill-name>/SKILL.md
```

### Codex

Add to `~/.codex/config.toml`:

```toml
[[skills.config]]
path = "/path/to/agent-skills/skills/<skill-name>/SKILL.md"
enabled = true
```
