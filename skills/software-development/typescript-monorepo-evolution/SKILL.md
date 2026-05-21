---
name: typescript-monorepo-evolution
description: Evolving TypeScript interfaces and types across a pnpm/Turbo monorepo while keeping type-check green.
trigger: When changing shared types, entities, or interfaces that affect multiple packages in a monorepo.
---

# TypeScript Monorepo Evolution

## Context
When evolving shared types (e.g., adding a required field to an entity) in a pnpm-workspace + Turbo monorepo, the change ripples through domain, data, view-models, and apps. Type-check failures are expected and should be resolved systematically.

## Workflow

1. **Update the source of truth first**
   - Edit the entity/interface in the domain package.
   - Export new types from `packages/domain/src/index.ts`.

2. **Let dependent packages fail**
   - Run `pnpm run type-check` at root (or per package with `--filter`).
   - Collect all errors across packages.

3. **Fix in dependency order**
   - domain → data → view-models → apps
   - For each package, run `tsc --noEmit` locally before moving up.

4. **Common fixes**
   - **Missing required field in test mocks**: add the new field to mock objects.
   - **Interface expanded → mock missing new methods**: when adding methods to an interface (e.g. `IAuthRepository` gains `loginWithGoogle`), every `vi.fn()` mock object must include the new method or TS will error with `Type '{...}' is missing properties from type 'IAuthRepository'`.
     ```ts
     // Before (broken after interface expansion)
     const authRepo: IAuthRepository = { exchangeCode: vi.fn(), me: vi.fn() };
     // After (fixed)
     const authRepo: IAuthRepository = {
       loginWithGoogle: vi.fn(),
       loginWithFacebook: vi.fn(),
       loginWithEmail: vi.fn(),
       exchangeCode: vi.fn(),
       me: vi.fn(),
       logout: vi.fn(),
       refreshToken: vi.fn(),
     };
     ```
   - **Import from domain missing**: add `export type { ... }` to domain/index.ts.
   - **Type `{}` has no property**: annotate the destructured object with `Partial<TheType>`.
   - **"File is not a module"**: remove broken `export *` lines pointing to empty/missing files.
   - **Unused import**: remove imports that are no longer referenced after refactoring.
   - **Renamed workspace package breaks dependents**: after renaming a package (e.g. `@teajuda-cliente/universal` → `@teajuda-cliente/mobile`), update all `workspace:*` references in dependent package.json files and re-run `pnpm install --no-frozen-lockfile`.

5. **Cross-package naming**
   - Use `export type { ... }` to avoid runtime imports when only types are needed.
   - Re-export from packages/data/index.ts and packages/view-models/index.ts so consumers import from one place.

## Platform-agnostic packages
If a shared package (e.g., `ui-kit-core`) references `react-native` types but must work in web packages that don't install `react-native`:
- Replace `import { Platform } from 'react-native'` with `const isWeb = typeof window !== 'undefined'`.
- Keep the same API surface (ShadowStyle, etc.) so native consumers still work.

## Verification
Always end with root-level `pnpm run type-check` confirming **all packages pass**.
