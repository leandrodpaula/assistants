# TeAjuda Admin Session Patterns

Session: 2026-05-21 — Built `teajuda.app.br.admin` (Next.js 16 App Router)

## Route Structure Used

```
src/app/
  login/page.tsx                  # Public login (email + grant_type=admin)
  (admin)/
    layout.tsx                    # Sidebar + Header shell
    dashboard/page.tsx            # KPI cards (customers, tenants, schedules, MRR)
    segments/page.tsx             # Table list
    segments/novo/page.tsx        # Create form
    segments/[id]/page.tsx        # Edit form
    tenants/page.tsx              # Table list
    tenants/[id]/page.tsx         # Detail view
    customers/page.tsx            # Table list
    customers/[id]/page.tsx       # Detail view (with subscriptions)
    planos/page.tsx               # Table list
    financeiro/page.tsx           # Subscriptions table
    financeiro/mrr/page.tsx       # Recharts LineChart
```

## Sidebar Nav Items (with icons)

| Route | Label | Icon |
|-------|-------|------|
| /dashboard | Dashboard | LayoutDashboard |
| /segments | Segmentos | Layers |
| /tenants | Tenants | Building2 |
| /customers | Customers | Users |
| /planos | Planos | ClipboardList |
| /financeiro | Financeiro | CreditCard |
| /financeiro/mrr | MRR | TrendingUp |

## API Contract (Admin Endpoints)

All under `https://api.teajuda.app.br`:

```
GET  /admin/dashboard          -> { total_customers, total_tenants, total_schedules_today, mrr }
GET  /admin/segments           -> { items: Segment[] }
POST /admin/segments           -> Segment
GET  /admin/segments/{id}     -> Segment
PUT  /admin/segments/{id}     -> Segment
GET  /admin/tenants            -> { items: Tenant[] }
GET  /admin/tenants/{id}      -> TenantDetail
GET  /admin/customers          -> { items: Customer[] }
GET  /admin/customers/{id}    -> CustomerDetail
GET  /admin/plans              -> { items: Plan[] }
GET  /admin/subscriptions      -> { items: Subscription[] }
GET  /admin/mrr                -> { items: { month, mrr_cents }[] }
```

## Currency Display Pattern

Values from API are in **cents**:
```tsx
R$ {(value_cents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 2 })}
```

## Tailwind Theme Tokens Used

```css
--color-primary-500: #3b82f6;
--color-primary-600: #2563eb;
--color-sidebar-bg: #0f172a;
--color-sidebar-text: #94a3b8;
--color-sidebar-active: #ffffff;
--color-sidebar-active-bg: #1e293b;
```

## Build Result

```
✓ Generating static pages using 7 workers (11/11)
```

7 static + 4 dynamic routes generated successfully.
