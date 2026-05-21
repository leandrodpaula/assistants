# FastAPI ↔ TypeScript Domain Field Mapping

> Session: 2026-05-21. Fase 2 (Client Monorepo) — Grupos 2.4–2.6.
> Context: Python FastAPI backend returns `snake_case`; TypeScript domain entities use `camelCase`. This reference documents the collision points and resolution for the TeAjuda platform.

## Why This Happens

FastAPI (Pydantic) serializes Python models with `snake_case` field names by default:
```python
class ProfessionalOut(BaseModel):
    id: str
    name: str
    tenant_id: str        # snake_case
    is_active: bool       # snake_case
    schedule_settings: dict  # snake_case
```

TypeScript domain uses `camelCase`:
```ts
export interface Professional {
  id: string;
  name: string;
  tenantId: string;     // camelCase
  isActive: boolean;    // camelCase
  scheduleSettings: ScheduleSettings;  // camelCase
}
```

## Collision Points Discovered

### Schedule Entity
| API Field (snake_case) | Domain Field (camelCase) | Resolution |
|------------------------|--------------------------|------------|
| `tenant_id` | `tenant_id` (kept as-is) | Added alias |
| `customer_name` | `customer_name` (kept as-is) | Added alias |
| `start_time` | `date` (canonical) | Added `start_time?: string` alias |
| `duration_minutes` | — | Added `duration_minutes?: number` alias |
| `notes` | — | Added `notes?: string` alias |
| `service_id` | — | Added `service_id?: string` alias |
| `professional_id` | — | Added `professional_id?: string` alias |
| `created_by` | — | Added `created_by?: string` alias |
| `agent_id` | — | Added `agent_id?: string` alias |

### Professional Entity
| API Field (snake_case) | Domain Field (camelCase) | Resolution |
|------------------------|--------------------------|------------|
| `tenant_id` | `tenantId` | Added alias |
| `is_active` | `isActive` | Consumer fixed to `isActive` |
| `color` | `color` | Added `color?: string` to domain |

### Service Entity
| API Field (snake_case) | Domain Field (camelCase) | Resolution |
|------------------------|--------------------------|------------|
| `tenant_id` | `tenantId` | Repository handles mapping |
| `duration_minutes` | `durationMinutes` | Consumer fixed to `durationMinutes` |
| `is_active` | `isActive` | Consumer fixed to `isActive` |
| `is_service` | — | Used only in `createService` DTO |

## Resolution Strategy Used

**Dual-field entities (Strategy A)** — see `SKILL.md` Pitfall #1.

Domain entities carry optional snake_case aliases so pages can consume raw API responses without a transform layer:

```ts
// packages/domain/src/entities/Schedule.ts
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

Consumers must handle `undefined`:
```tsx
// Safe consumption
{s.start_time ? new Date(s.start_time as string).toLocaleTimeString() : "--:--"}
```

## When to Migrate to Strategy B (Repository Transform)

Strategy A is appropriate while:
- The API contract is still stabilizing
- The team is small and velocity matters more than purity
- There's no shared OpenAPI spec generating both sides

Migrate to Strategy B (explicit mappers in `data/`) when:
- The API is stable and versioned
- Multiple frontends (web, mobile, portal) consume the same backend
- You want strict domain purity for complex business logic

## Type-Check Gotchas

1. **`as string` casts on optional aliases**
   `new Date(s.start_time)` fails because `start_time` is `string | undefined`.
   Fix: `new Date(s.start_time as string)` or guard with `if (s.start_time)`.

2. **Property does not exist on type**
   If the domain entity is missing an alias, TypeScript reports:
   ```
   TS2339: Property 'duration_minutes' does not exist on type 'Service'.
   ```
   Fix: add the optional alias to the domain entity, then re-export from `index.ts`.

3. **Renaming consumer code**
   When fixing `is_active` → `isActive`, search for ALL occurrences in the consuming page/component, including JSX conditionals:
   ```tsx
   // Before (broken)
   <Badge variant={p.is_active ? "success" : "default"}>
   
   // After (fixed)
   <Badge variant={p.isActive ? "success" : "default"}>
   ```
