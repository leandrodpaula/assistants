# Hermes external `.agents` skills

Session-derived pattern for consolidating assistant skill libraries without replacing Hermes' local catalog.

## Goal

Make Hermes read skills from `/Users/leo/.agents/skills` while keeping `/Users/leo/.hermes/skills` intact.

## Confirmed behavior

Hermes skill discovery uses:
1. local skills from `get_skills_dir()` (`$HERMES_HOME/skills`, normally `~/.hermes/skills`)
2. external skill directories from `skills.external_dirs` in `config.yaml`

So `.agents` can be added as an additional catalog. It is not a config-level replacement for `~/.hermes/skills`.

## Config snippet

In `/Users/leo/.hermes/config.yaml`:

```yaml
skills:
  external_dirs:
  - /Users/leo/.agents/skills
```

Preserve existing sibling keys under `skills`, such as `template_vars`, `inline_shell`, and curator-related settings.

## Verification

After editing, verify with Hermes' own CLI:

```bash
hermes config path
hermes config check
hermes skills list | grep -E 'gh-env-sync|impeccable'
```

A good verification is seeing skills that exist only in `.agents/skills` appear in `hermes skills list`.

## Backup pattern

Before editing config:

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak.skills-external-dirs.$(date +%Y%m%d_%H%M%S)
```

## Pitfalls

- Do not use `HERMES_HOME=/Users/leo/.agents` just to share skills; it moves sessions, config, memory, logs, auth, and other state.
- Do not merge same-name skills with different content hashes automatically.
- If verifying by importing Hermes modules directly, use the Python version Hermes expects. Prefer the `hermes` CLI for user-facing verification.
