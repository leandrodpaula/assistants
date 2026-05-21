---
name: nextjs-standalone-setup
description: |
  Scaffold a new Next.js project outside a monorepo (standalone) using pnpm,
  handling common pnpm pitfalls with `approve-builds`, workspace leakage,
  and sharp/unrs-resolver native builds.
trigger:
  - "create a new Next.js project"
  - "scaffold Next.js with pnpm"
  - "npx create-next-app fails with pnpm"
  - "sharp build error Next.js"
  - "unrs-resolver build error"
  - "standalone landing page Next.js"
  - "approve-builds pnpm Next.js"
---

# Next.js Standalone Setup with pnpm

## When to use

Creating a new Next.js app **outside** a pnpm workspace (e.g., a standalone landing page, marketing site, or docs site) when the parent directory already contains a `pnpm-workspace.yaml` or when pnpm's `approve-builds` gate blocks installation.

## Steps

### Replacing an existing project (preserve assets)

When scaffolding over an existing directory that contains static assets you must keep:

```bash
# 1. Move assets to temp
mkdir -p /tmp/project-bak
mv assets /tmp/project-bak/

# 2. Clean everything except what create-next-app will recreate
rm -rf node_modules package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc

# 3. Scaffold
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --no-import-alias --use-pnpm

# 4. Restore assets
cp -r /tmp/project-bak/assets ./
```

Then apply Steps 2 and 3 below (onlyBuiltDependencies + workspace cleanup).

### 2. Handle pnpm `onlyBuiltDependencies`

**Do NOT** run `pnpm approve-builds` interactively inside an agent session — it requires TTY selection and will fail.

Instead, add to `package.json`:

```json
{
  "pnpm": {
    "onlyBuiltDependencies": ["sharp", "unrs-resolver"]
  }
}
```

Then run `pnpm install` again.

### 3. Prevent workspace inheritance

If the parent directory (or any ancestor) has a `pnpm-workspace.yaml`, the new project may be incorrectly treated as a workspace member, causing resolution conflicts.

**Fix:** Remove any inherited `pnpm-workspace.yaml` from the new project root:

```bash
rm -f pnpm-workspace.yaml
```

Also remove any `.npmrc` with workspace settings if present.

### 4. Verify build

```bash
npx next build
```

Expected output: static pages prerendered successfully with no TypeScript errors.

## Common errors and fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `[ERR_PNPM_IGNORED_BUILDS]` | pnpm blocks postinstall scripts | Add `onlyBuiltDependencies` to `package.json` |
| `pnpm install has failed` after `approve-builds` | TTY interactive prompt failed in agent | Do not use `approve-builds`; use `onlyBuiltDependencies` config |
| Workspace resolution conflicts | Inherited `pnpm-workspace.yaml` | `rm -f pnpm-workspace.yaml` in project root |
| `next: command not found` | Node_modules incomplete | Run `pnpm install` after fixing step 2 |

## References

- [references/nextjs-pnpm-workaround.md](references/nextjs-pnpm-workaround.md) — Session transcript showing the exact error and fix path
