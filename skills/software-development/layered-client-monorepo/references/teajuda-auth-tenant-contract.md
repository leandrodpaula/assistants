# TeAjuda Session Reference — Auth, Tenant & Whitelabel Contracts

> Session: 2026-05-20. Fase 2 (Client Monorepo) — Grupos 2.1–2.3.

## Entities Created

| Entity | File | Key Fields |
|--------|------|-----------|
| Segment | `domain/src/entities/Segment.ts` | `id, slug, name, branding: {primaryColor, secondaryColor, accentColor, logoUrl}, modules: SegmentModule[]` |
| Professional | `domain/src/entities/Professional.ts` | `id, tenantId, name, email, phone, bio, avatarUrl, schedule: ScheduleSettings, isActive` |
| Service | `domain/src/entities/Service.ts` | `id, tenantId, name, description, durationMinutes, price, color, isActive` |
| Plan | `domain/src/entities/Plan.ts` | `id, name, title, description, price, interval, features[], limits{}` |
| FinancialRecord | `domain/src/entities/FinancialRecord.ts` | `id, tenantId, type, status, amount, description, method, date` |
| Tenant (evolved) | `domain/src/entities/Tenant.ts` | `id, name, slug, agent_id, business_type, segment_id, branding_override, modules_enabled[], settings{}` |

## Interfaces Created

| Interface | File | Methods |
|-----------|------|---------|
| ISegmentRepository | `domain/src/interfaces/ISegmentRepository.ts` | `getBySlug(slug), list()` |
| IProfessionalRepository | `domain/src/interfaces/IProfessionalRepository.ts` | `listByTenant(tenantId), getById(tenantId, id), create(prof), update(tenantId, id, prof)` |
| IServiceRepository | `domain/src/interfaces/IServiceRepository.ts` | `listByTenant(tenantId), getById(tenantId, id), create(svc), update(tenantId, id, svc)` |

## Auth Contract (Frontend ↔ Backend)

| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| `loginWithEmail` | `POST /auth/login` | `{email, password}` | `{access_token, refresh_token, customer}` |
| `loginWithGoogle` | `POST /auth/google` | `{google_token}` | `{access_token, refresh_token, customer}` |
| `loginWithFacebook` | `POST /auth/facebook` | `{facebook_token}` | `{access_token, refresh_token, customer}` |
| `exchangeCode` | `POST /auth/exchange` | `{code, redirect_uri, state}` | `{access_token, refresh_token, customer}` |
| `me` | `GET /auth/me` | — | `Customer` |
| `logout` | `POST /auth/logout` | — | `void` |
| `refreshToken` | `POST /auth/refresh` | `{refresh_token}` | `{access_token, refresh_token}` |

## Tenant Flow

```
/login (email or social)
    → tokens stored
    → me() called
    → redirect to /selecionar-tenant
        → TenantProvider loads tenant list
        → user picks tenant
        → currentTenant stored
        → redirect to /dashboard
            → PlatformLayout injects X-Tenant-Id header
            → all API calls scoped to tenant
```

## Whitelabel Token Merge

`useWhitelabelViewModel` merges in priority order:
1. `tenant.branding_override.*` (user custom)
2. `segment.branding.*` (segment default)
3. Fallback to platform default colors

## Http Client Interceptors

- **Request**: `Authorization: Bearer <token>`, `X-Tenant-Id: <currentTenant.id>`
- **Response 401**: silent refresh via `/auth/refresh`, retry original request
- **Refresh fail**: clear tokens, redirect to login

## Files Modified in This Session

- `packages/domain/src/entities/Segment.ts`
- `packages/domain/src/entities/Professional.ts`
- `packages/domain/src/entities/Service.ts`
- `packages/domain/src/entities/Plan.ts`
- `packages/domain/src/entities/FinancialRecord.ts`
- `packages/domain/src/entities/Tenant.ts`
- `packages/domain/src/interfaces/ISegmentRepository.ts`
- `packages/domain/src/interfaces/IProfessionalRepository.ts`
- `packages/domain/src/interfaces/IServiceRepository.ts`
- `packages/data/src/segments/SegmentRepository.ts`
- `packages/data/src/professionals/ProfessionalRepository.ts`
- `packages/data/src/services/ServiceRepository.ts`
- `packages/data/src/auth/AuthRepository.ts`
- `packages/view-models/src/auth/useAuthViewModel.ts`
- `packages/view-models/src/tenant/useTenantSwitcherViewModel.ts`
- `packages/view-models/src/tenant/useWhitelabelViewModel.ts`
- `apps/web/src/providers/AuthProvider.tsx`
- `apps/web/src/lib/tokenStorage.ts`
- `apps/web/src/lib/httpClient.ts`
- `apps/web/src/components/ProtectedRoute.tsx`
- `apps/web/src/app/(public)/login/page.tsx`
- `apps/web/src/app/(public)/selecionar-tenant/page.tsx`
- `apps/web/src/app/(platform)/layout.tsx`
- `apps/mobile/app/(auth)/login.tsx`
- `STATUS.md`
