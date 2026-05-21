# Next.js + pnpm Standalone Setup — Session Transcript

## Environment
- macOS 15.7.7
- pnpm 11.1.2
- Node.js 22
- Next.js 16.2.6

## Problem

Running `npx create-next-app@latest` with `--use-pnpm` inside a new directory:

```
[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: sharp@0.34.5, unrs-resolver@1.12.2
Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.
Aborting installation.
  pnpm install has failed.
```

Attempting `pnpm approve-builds` interactively via terminal tool failed with:

```
Error [ERR_USE_AFTER_CLOSE]: readline was closed
```

The interactive multi-select prompt cannot be answered by an agent.

## Solution

Add `"pnpm": { "onlyBuiltDependencies": ["sharp", "unrs-resolver"] }` to `package.json`:

```json
{
  "name": "my-app",
  "version": "0.1.0",
  "private": true,
  "scripts": { "dev": "next dev", "build": "next build" },
  "dependencies": { "next": "16.2.6", "react": "19.2.4", "react-dom": "19.2.4" },
  "devDependencies": { "@tailwindcss/postcss": "^4", "tailwindcss": "^4", "typescript": "^5" },
  "pnpm": {
    "onlyBuiltDependencies": ["sharp", "unrs-resolver"]
  }
}
```

Then `pnpm install` succeeds without interactive prompts.

## Second Problem: Workspace Inheritance

After scaffolding `teajuda.app.br.portal`, the project inherited a `pnpm-workspace.yaml` from a prior state, causing `pnpm install` to resolve packages against the parent monorepo instead of locally.

## Fix

```bash
rm -f pnpm-workspace.yaml
```

This makes the project a true standalone pnpm project.

## Verification

```bash
cd my-app && npx next build
# ✓ Compiled successfully
# ✓ Generating static pages (7/7)
```
