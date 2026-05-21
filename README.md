# assistants

Monorepo containing skill libraries for multiple AI coding assistants.

## Structure

```
.github/
├── .agents/      # Skills for generic code agents (Codex, Claude Code, OpenCode)
├── .claude/      # Skills for Claude Desktop (symlink → .agents/skills)
├── .hermes/     # Skills for Hermes CLI agent
└── .pi/         # Skills for PI agent (placeholder)
```

## Installation

Choose your assistant below.

### Hermes Agent

```bash
# Option 1: Use full Hermes skills (includes shared + Hermes-only)
git clone git@github.com:leandrodpaula/assistants.git ~/.hermes-temp
cd ~/.hermes-temp
git checkout hermes
mv skills/* ~/.hermes/skills/

# Option 2: Use .agents skills as additional source
# In ~/.hermes/config.yaml, add:
# skills:
#   external_dirs:
#     - /Users/leo/.agents/skills
```

### Claude Desktop (claude.ai, Cursor, etc.)

```bash
# .claude/skills is a symlink to .agents/skills
git clone git@github.com:leandrodpaula/assistants.git ~/.claude-temp
cd ~/.claude-temp
git checkout claude
# Or simply link to .agents:
ln -sf ../.agents/skills ~/.claude/skills
```

### Codex CLI

```bash
git clone git@github.com:leandrodpaula/assistants.git ~/.agents-temp
cd ~/.agents-temp
git checkout agents
mkdir -p ~/.codex/skills
cp -r skills/* ~/.codex/skills/
# Or: export CODEX_EXTENSIONS_PATH=~/.agents/skills
```

### Claude Code (ccl)

```bash
git clone git@github.com:leandrodpaula/assistants.git ~/.agents-temp
cd ~/.agents-temp
git checkout agents
mkdir -p ~/.claude-code/skills
cp -r skills/* ~/.claude-code/skills/
```

### OpenCode

```bash
git clone git@github.com:leandrodpaula/assistants.git ~/.agents-temp
cd ~/.agents-temp
git checkout agents
mkdir -p ~/.opencode/skills
cp -r skills/* ~/.opencode/skills/
```

## Sync

To sync local changes back to the repo:

```bash
# Per-assistant branch
cd ~/.agents
git add skills/
git commit -m "Update skills"
git push origin agents

# For Hermes
cd ~/.hermes
git add skills/
git commit -m "Update skills"  
git push origin hermes

# Main (monorepo) — merges all assistants
# Note: Use git read-tree to merge branches into subdirectories
```

## Branches

- `main` — monorepo with all assistants as subdirectories
- `agents` — .agents skills only
- `hermes` — .hermes skills only
- `claude` — .claude skills (minimal, mostly symlinks)
- `pi` — .pi skills (placeholder)

See individual branches for precise per-assistant sync.