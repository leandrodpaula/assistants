# Environment Security Audit Checklist

> Session-derived checklist from teajuda.app.br architecture remediation (2026-05-21).
> Use when creating or reviewing plans that touch `.env`, secrets, or deployment configs.

## Why This Matters

During a multi-repo architectural review, this checklist uncovered:
- `.env` file **committed to git history** (3 commits deep)
- `.gitignore` **completely missing** from a Next.js app
- `CORS_ALLOW_ORIGINS=*` in a production `.env`
- Real secrets (MongoDB password, Mercado Pago token, Facebook App Secret, JWT key, Chroma API key) in a local `.env`

## The Checklist

Run this in **every repo** that has environment variables:

### 1. Git Hygiene
```bash
# Is .env tracked?
git log --all --full-history -- .env

# Is .env in .gitignore?
grep -n "^\.env$" .gitignore

# Does .gitignore exist at all?
test -f .gitignore || echo "MISSING .gitignore"

# Check for any .env.* files tracked
git ls-files | grep "\.env"
```

### 2. Secrets Rotation Status
```bash
# List all .env files (excluding node_modules, .venv, dist)
find . -name '.env*' -not -path '*/node_modules/*' -not -path '*/.venv/*' -not -path '*/dist/*'

# Scan for high-entropy strings that look like real secrets
grep -rE '(SECRET|TOKEN|KEY|PASSWORD)=[A-Za-z0-9_-]{20,}' .env 2>/dev/null
```

### 3. .env.example Completeness
```bash
# Does .env.example exist?
test -f .env.example || echo "MISSING .env.example"

# Does it have all keys from the real .env?
# (Manual: compare line-by-line, real .env should never be shown in output)
```

### 4. CORS / Security Headers in .env
```bash
# Dangerous patterns
grep -n "CORS_ALLOW_ORIGINS=\*" .env
grep -n "allow_credentials.*[Tt]rue" .env  # If paired with wildcard above, CRITICAL
```

## What To Do When You Find Issues

| Finding | Immediate Action |
|---------|------------------|
| `.env` tracked in git | `git filter-repo --path .env --invert-paths` or BFG Repo-Cleaner |
| `.gitignore` missing | Create it, add `.env`, `.env.local`, `.env.*.local` |
| Real secrets in `.env` | Rotate the secret in the provider dashboard FIRST, then update `.env` |
| `CORS_ALLOW_ORIGINS=*` | Change to explicit comma-separated origins, never `*` with credentials |
| `.env.example` outdated | Sync with all keys from real `.env`, use placeholder values |

## Related

- This checklist pairs with `writing-plans` when a plan includes environment setup or security tasks.
