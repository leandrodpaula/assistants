---
name: layered-client-monorepo
description: Scaffold and evolve a cross-platform (web + mobile) client monorepo with Clean Architecture layers (domain, data, view-models, ui-kit) in a pnpm/Turbo workspace.
trigger: When the user asks to create, restructure, or evolve a client-side monorepo with shared domain logic across web (Next.js) and mobile (Expo/React Native) apps, or when working on a multi-tenant whitelabel platform frontend.
---

# Layered Client Monorepo

## Context
Build a client monorepo where business logic is shared between web and mobile while UI is platform-specific. The architecture separates concerns into 5 packages:

| Package | Responsibility | Dependencies |
|---------|--------------|--------------|
| `domain` | Entities, interfaces, value objects | None |
| `data` | Repository implementations (HTTP, storage) | `domain` |
| `view-models` | React hooks exposing UI-ready state | `domain`, `data` |
| `ui-kit-core` | Design tokens (colors, spacing, shadows) | None |
| `ui-kit-web` / `ui-kit-native` | Platform components | `ui-kit-core` |

## Architecture Rules

1. **Domain never imports external libraries** (no axios, no react). Pure TypeScript interfaces and entities.
2. **Data depends only on Domain**. Repositories implement domain interfaces.
3. **View-models depend on Domain + Data**. Hooks consume repositories and expose state.
4. **Apps depend on View-models + UI-Kit**. Pages compose hooks and components.
5. **UI-Kit-Core is platform-agnostic**. Never import `react-native` here; use `typeof window` for platform detection.

## Directory Structure

```
packages/
  domain/src/
    entities/        # Tenant, User, Appointment, Segment, etc.
    interfaces/      # IAuthRepository, ITenantRepository, etc.
    index.ts         # Re-export everything
  data/src/
    api/             # httpClient, jwt, errorMapper
    auth/            # AuthRepository
    segments/        # SegmentRepository
    professionals/   # ProfessionalRepository
    services/        # ServiceRepository
    index.ts         # Re-export repositories + api helpers
  view-models/src/
    auth/            # useAuthViewModel
    tenant/          # useTenantContext, useWhitelabelViewModel
    index.ts         # Re-export hooks
  ui-kit-core/src/
    colors.ts, spacing.ts, typography.ts, shadows.ts, radius.ts
  ui-kit-web/src/
    components/      # Button, Card, Input, TenantSwitcher
  ui-kit-native/src/
    components/      # Same concepts, React Native implementation
```

## Next.js Admin Server Component Migration

When an admin dashboard built with Next.js App Router has list/detail pages using `"use client"` + `useEffect` for data fetching, migrate them to Server Components (SC) to eliminate client-side waterfalls, reduce bundle size, and improve SEO.

### Step 1: Create a server-safe API helper

Create `lib/serverApi.ts` that reads auth cookies server-side and forwards them as `Authorization` headers:

```ts
import { cookies } from "next/headers";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "";

export async function serverRequest<T>(
  method: string,
  path: string,
  body?: unknown
): Promise<T> {
  const cookieStore = await cookies();
  const token = cookieStore.get("admin_token")?.value;

  const headers: Record<string, string> = { "Content-Type": "application/json" };
  if (token) headers["Authorization"] = `Bearer ${token}`;

  const res = await fetch(`${API_BASE}${path}`, {
    method,
    headers,
    body: body ? JSON.stringify(body) : undefined,
    cache: "no-store",
  });

  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.detail || `HTTP ${res.status}`);
  }
  return res.json() as Promise<T>;
}

export const serverApi = {
  get: <T>(path: string) => serverRequest<T>("GET", path),
  post: <T>(path: string, body: unknown) => serverRequest<T>("POST", path, body),
  put: <T>(path: string, body: unknown) => serverRequest<T>("PUT", path, body),
  patch: <T>(path: string, body: unknown) => serverRequest<T>("PATCH", path, body),
  del: <T>(path: string) => serverRequest<T>("DELETE", path),
};
```

### Step 2: Convert list pages to async SC

Remove `"use client"`, `useState`, `useEffect`, and make the component `async`:

```tsx
// Before: Client Component
"use client";
import { useEffect, useState } from "react";
import { adminApi } from "@/lib/adminApi";

export default function CustomersPage() {
  const [items, setItems] = useState([]);
  useEffect(() => { adminApi.get("/admin/customers").then(setItems); }, []);
  // ...
}

// After: Server Component
import { serverApi } from "@/lib/serverApi";

export default async function CustomersPage() {
  const { items } = await serverApi.get("/admin/customers");
  return <DataTable data={items} />;
}
```

### Step 3: Convert detail pages to async SC

Dynamic route segments work the same way. Use `params` as a Promise in Next.js 15+:

```tsx
interface Props { params: Promise<{ id: string }>; }

export default async function CustomerDetailPage({ params }: Props) {
  const { id } = await params;
  const customer = await serverApi.get(`/admin/customers/${id}`);
  if (!customer) notFound();
  return <DetailView customer={customer} />;
}
```

### Step 4: Extract interactive UI into Client Components

If a page has charts (e.g., Recharts), forms, or client-side state, keep those parts as Client Components and import them into the SC page:

```tsx
// page.tsx (Server Component)
import { MrrChart } from "@/components/MrrChart"; // "use client" inside

export default async function MrrPage() {
  const { items } = await serverApi.get("/admin/mrr");
  return <MrrChart data={items} />;
}
```

### What stays as Client Component

- Pages with forms (POST/PUT/PATCH submissions with `onSubmit` handlers)
- Pages with charts requiring client-side rendering libraries (Recharts, D3)
- Login pages (no server-side auth cookie yet)
- Error boundaries (`error.tsx` must be `"use client"`)

### Common Pitfall: CORS + credentials on server fetch

`fetch` in Server Components does **not** send cookies automatically. You must read them via `cookies()` from `next/headers` and forward them as headers.

## Http Client Patterns

Create a single axios instance with interceptors:
- **Request**: inject `Authorization: Bearer <token>` and `X-Tenant-Id: <tenant>`
- **Response 401**: automatic token refresh via `/auth/refresh`, then retry original request
- **No token**: call `onUnauthorized()` callback so the app can redirect to login

```ts
// data/src/api/httpClient.ts
export function createHttpClient(
  baseURL: string,
  storage: TokenStorageAdapter,
  getTenantId?: () => string | null,
  onUnauthorized?: () => void,
): AxiosInstance
```

## Auth Patterns

### Domain Interface
```ts
export interface IAuthRepository {
  loginWithEmail(email: string, password: string): Promise<AuthResult>;
  loginWithGoogle(googleToken: string): Promise<AuthResult>;
  loginWithFacebook(facebookToken: string): Promise<AuthResult>;
  exchangeCode(dto: ExchangeCodeDTO): Promise<AuthResult>;
  me(): Promise<Customer>;
  logout(): Promise<void>;
  refreshToken(): Promise<string>;
}
```

### View-Model Hook
`useAuthViewModel` manages:
- Initial `checkAuth()` on mount (reads token, calls `me()`)
- `loginWithEmail` / `loginWithGoogle` / `loginWithFacebook` / `loginWithFacebookCode`
- `logout()` clears storage + state
- `isLoading`, `isAuthenticated`, `customer`, `error`

### App Provider
Each app wraps its root with an `AuthProvider` that instantiates the repository and passes to `useAuthViewModel`.

## Multi-Tenant Patterns

### Tenant Context
```tsx
<TenantProvider tenantRepo={tenantRepo}>
  {children}
</TenantProvider>
```

Exposes:
- `tenants`: list of user's tenants
- `currentTenant`: selected tenant (persisted in storage)
- `setCurrentTenant(tenantId)`: switch active tenant
- `isLoading`, `error`

### Whitelabel Tokens
`useWhitelabelViewModel(segmentBranding, tenantBrandingOverride)` merges:
1. Segment defaults (from platform config)
2. Tenant override (from user's settings)

Returns CSS variables or React Native style tokens.

## Platform Storage

| Platform | Access Token | Refresh Token | Tenant ID |
|----------|-------------|---------------|-----------|
| Web | `localStorage` | `localStorage` | `localStorage` |
| Mobile | `expo-secure-store` | `expo-secure-store` | `expo-secure-store` |

Use a `TokenStorageAdapter` interface so repositories are storage-agnostic.

## Common Pitfalls

1. **Snake_case API responses vs camelCase domain entities**
   Python FastAPI returns `snake_case` (PEP 8) while TypeScript domain uses `camelCase`.
   When a page consumes `Schedule.start_time` but the domain only exposes `date`,
   type-check fails with `Property 'start_time' does not exist on type 'Schedule'`.

   **Three resolution strategies** (pick one per project and stick to it):

   - **(A) Dual-field domain entities** (pragmatic, used in TeAjuda)
     ```ts
     export interface Schedule {
       id: string;
       tenant_id: string;
       customer_name: string;
       date: string;            // canonical domain field
       start_time?: string;     // optional alias for raw API response
       duration_minutes?: number;
       notes?: string;
       status: ScheduleStatus;
       price: number;
     }
     ```
     Pros: zero transform layer, works immediately.
     Cons: domain leaks backend convention; consumers must handle `undefined`.

   - **(B) Repository transform layer** (cleanest)
     ```ts
     // data/src/schedules/ScheduleRepository.ts
     private mapDtoToEntity(dto: ScheduleDto): Schedule {
       return {
         id: dto._id,
         tenantId: dto.tenant_id,
         customerName: dto.customer_name,
         startTime: dto.start_time,
         durationMinutes: dto.duration_minutes,
         ...
       };
     }
     ```
     Pros: domain stays pure camelCase.
     Cons: every repository needs a mapper; more boilerplate.

   - **(C) `as` casts at consumption** (quick fix, not recommended for scale)
     ```ts
     new Date(s.start_time as string)
     ```
     Pros: no structural changes.
     Cons: unsafe, loses type benefits.

   **Recommendation**: start with (A) for speed, migrate to (B) as the project matures.

2. **Exported return-type interfaces**
   If `useApi.ts` defines `interface ApiState<T>` and `useProfessionals.ts` returns
   `ApiState<ProfessionalsResponse>`, but `ApiState` is **not exported**, TypeScript
   reports:
   ```
   Return type of exported function has or is using name 'ApiState' from external module
   but cannot be named. ts(4058)
   ```
   Fix: export the interface:
   ```ts
   export interface ApiState<T> { ... }
   ```

3. **Platform imports in shared packages**
   - Never `import { Platform } from 'react-native'` in `ui-kit-core` or `domain`.
   - Use `const isWeb = typeof window !== 'undefined'`.
   - *Session example*: `ui-kit-core/src/shadows.ts` had `import { Platform } from 'react-native'` which broke web type-check. Fix: detect web with `typeof window` and return web-only shadow styles.

4. **Circular dependencies between domain and data**
   - Domain defines interfaces; data implements them. No data types leak into domain.

5. **Missing re-exports**
   - After adding a new entity, always export it from `domain/src/index.ts`.
   - Same for new repositories in `data/src/index.ts` and hooks in `view-models/src/index.ts`.

6. **Test mocks out of sync with interfaces**
   - When extending `IAuthRepository` (e.g. adding `loginWithGoogle`, `loginWithFacebook`, `loginWithEmail`), every mock object in tests must include the new methods or TypeScript will error with `missing properties`.
   - *Fix pattern*: update the mock factory:
     ```ts
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
   - Also rename removed methods in test assertions (e.g. `login` → `loginWithEmail`).

7. **Http client without tenant header**
   - The httpClient interceptor must read the current tenant from context/storage and inject `X-Tenant-Id`. Failing to do this causes 403 errors on multi-tenant endpoints.
   - Pass `getTenantId` (not `getAgentId`) to `createHttpClient`.

8. **Renaming workspace packages breaks dependents**
   - Renaming `@teajuda-cliente/universal` → `@teajuda-cliente/mobile` requires updating:
     - `package.json` `name` field
     - Every `package.json` that depends on it (`workspace:*` references)
     - `turbo.json` env vars if package-specific
     - `pnpm-workspace.yaml` if globs changed
   - Run `pnpm install --no-frozen-lockfile` after rename.

9. **Next.js App Router route groups need shared layout**
   - Use `(public)` and `(platform)` route groups so each group can have its own `layout.tsx` without affecting URL paths.
   - Wrap `(platform)/layout.tsx` with `AuthProvider`, `TenantProvider`, and `WhitelabelProvider`.

## App-Level Patterns

### ProtectedRoute (Next.js App Router)
Create a client component that redirects unauthenticated users:

```tsx
"use client";
import { useEffect } from "react";
import { useRouter, usePathname } from "next/navigation";
import { useAuth } from "@/providers/AuthProvider";

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth();
  const router = useRouter();
  const pathname = usePathname();

  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      router.push(`/login?redirect=${encodeURIComponent(pathname)}`);
    }
  }, [isLoading, isAuthenticated, router, pathname]);

  if (isLoading) return <LoadingSpinner />;
  if (!isAuthenticated) return null;
  return <>{children}</>;
}
```

Wrap the platform layout:
```tsx
export default function PlatformLayout({ children }) {
  return (
    <ProtectedRoute>
      <TenantProvider tenantRepo={tenantRepo}>
        <WhitelabelProvider>
          {children}
        </WhitelabelProvider>
      </TenantProvider>
    </ProtectedRoute>
  );
}
```

### WhitelabelProvider with Dynamic CSS Variables
Apply tokens to the document root so Tailwind `var(--primary)` picks them up:

```tsx
"use client";
import { createContext, useContext, useEffect } from "react";
import { useWhitelabelViewModel } from "@teajuda-cliente/view-models";
import { useCurrentTenant } from "@teajuda-cliente/view-models";

const WhitelabelContext = createContext(null);

export function WhitelabelProvider({ children }) {
  const { currentTenant } = useCurrentTenant();
  const vm = useWhitelabelViewModel(
    undefined,          // segmentBranding (fetch from backend)
    currentTenant?.branding_override
  );

  useEffect(() => {
    const root = document.documentElement;
    root.style.setProperty("--primary", vm.tokens.primary);
    root.style.setProperty("--secondary", vm.tokens.secondary);
    root.style.setProperty("--accent", vm.tokens.accent);
  }, [vm.tokens]);

  return (
    <WhitelabelContext.Provider value={vm}>
      {children}
    </WhitelabelContext.Provider>
  );
}
```

For React Native, pass tokens via context and consume in themed components instead of CSS variables.

## Http Client Interceptor (with Tenant Header)

```ts
// data/src/api/httpClient.ts
instance.interceptors.request.use(async (config) => {
  const token = await storage.getAccessToken();
  if (token) {
    config.headers.set('Authorization', `Bearer ${token}`);
  }
  const tenantId = getTenantId?.();
  if (tenantId) {
    config.headers.set('X-Tenant-Id', tenantId);
  }
  return config;
});
```

The `getTenantId` callback should read from the active TenantContext or storage. This is critical for multi-tenant endpoints.

## Provider Hierarchy

Apps must nest providers in this order:

```
AuthProvider          (login state, tokens)
  └─ TenantProvider   (tenant list, current tenant, X-Tenant-Id)
      └─ WhitelabelProvider (tokens, CSS variables)
          └─ AppContent
```

This ensures:
- Auth is checked first (redirect if not logged in)
- Tenant is loaded only after auth is confirmed
- Whitelabel tokens have access to the current tenant's branding override

- [ ] `pnpm run type-check` passes for **all** packages
- [ ] Domain package has zero runtime dependencies
- [ ] Data package only imports from domain + axios
- [ ] View-models only import from domain + data + react
- [ ] Apps only import from view-models + ui-kit
- [ ] `ui-kit-core` has no react-native imports
- [ ] Auth flow: login → token stored → `me()` → redirect to tenant selector
- [ ] Tenant switch: updates context → httpClient injects new `X-Tenant-Id` → refetch data

## Related
- `typescript-monorepo-evolution` — for evolving shared types while keeping type-check green
- `references/fastapi-di-factories.md` — FastAPI dependency-injection factory pattern for repository → service → router wiring
- `references/fastapi-typescript-field-mapping.md` — concrete snake_case ↔ camelCase mapping strategies and collision points from a FastAPI + TypeScript monorepo
