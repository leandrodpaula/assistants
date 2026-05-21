---
name: admin-dashboard
description: Build internal admin dashboards and CRUD panels with Next.js App Router
triggers:
  - admin panel
  - backoffice
  - internal dashboard
  - CRUD interface
  - admin console
  - management panel
  - platform admin
---

# Admin Dashboard Skill

## When to Use

Use this skill when:
- The user asks for an admin panel, backoffice, or internal dashboard
- Building CRUD interfaces for platform management (users, tenants, segments, orders, etc.)
- Creating data-heavy tables with list / view / edit / detail flows
- Need auth-protected routes with role-based or token-based access
- Need metrics dashboards with charts (MRR, KPIs, growth)

## Architecture

### Route Groups
Use `(admin)` route group for all protected pages. Place public pages (`/login`) outside the group so middleware can distinguish them cleanly.

```
src/app/
  login/page.tsx          # Public
  (admin)/
    layout.tsx            # Sidebar + Header shell
    dashboard/page.tsx    # KPI cards
    [entity]/page.tsx     # List/table
    [entity]/novo/page.tsx    # Create form
    [entity]/[id]/page.tsx    # Edit/detail
```

### Middleware (`src/middleware.ts`)
- Protect all non-public routes (use `matcher` to exempt `_next`, `favicon.ico`, login)
- Check `admin_token` cookie
- Redirect unauthenticated to `/login`
- Do NOT check localStorage here (middleware runs server-side)

### Auth Token Storage (`src/lib/auth.ts`)
Use **dual storage** for compatibility:
- `localStorage`: primary, client-side access
- `document.cookie`: for Next.js middleware visibility
- `setToken()` writes both; `removeToken()` clears both

### API Client (`src/lib/adminApi.ts`)
Create a thin wrapper around `fetch` for **Client Components**:
- Reads token from `localStorage`
- Injects `Authorization: Bearer <token>`
- On 401: calls `removeToken()` and redirects to `/login`
- Exposes `get`, `post`, `put`, `patch`, `del` helpers

### Server API Helper (`src/lib/serverApi.ts`)
Create a server-safe fetch wrapper for **Server Components**:
- Reads the auth token from `next/headers` `cookies()`
- Injects `Authorization: Bearer <token>` into the `headers` object
- Uses `cache: "no-store"` so data is always fresh
- Exposes the same `get`, `post`, `put`, `patch`, `del` helpers

```typescript
import { cookies } from "next/headers";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "https://api.example.com";

export async function serverRequest<T>(method: string, path: string, body?: unknown): Promise<T> {
  const cookieStore = await cookies();
  const token = cookieStore.get("admin_token")?.value;
  const headers: Record<string, string> = { "Content-Type": "application/json" };
  if (token) headers["Authorization"] = `Bearer ${token}`;

  const res = await fetch(`${API_BASE}${path}`, {
    method, headers, body: body ? JSON.stringify(body) : undefined, cache: "no-store",
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

### Layout Shell (`src/app/(admin)/layout.tsx`)
- Sidebar: fixed width (~64), dark background (`bg-slate-900`), icon + label nav items
- Header: page title left, user info + logout right
- Main: `flex-1 overflow-y-auto p-6`
- Use `lucide-react` icons for nav items

## Server Components for Read-Only Pages

For **list, detail, and dashboard pages** that only display data, use **async Server Components** instead of `"use client"` + `useEffect`.

**Before (Client Component — avoid for read-only data):**
```tsx
"use client";
import { useEffect, useState } from "react";
export default function Page() {
  const [items, setItems] = useState([]);
  useEffect(() => { adminApi.get("/items").then(setItems); }, []);
  // ... loading + error states
}
```

**After (Server Component — preferred):**
```tsx
import { serverApi } from "@/lib/serverApi";
export default async function Page() {
  const { items } = await serverApi.get<{ items: Item[] }>("/items");
  return <DataTable data={items} />;
}
```

Benefits:
- No loading spinners or error states to manage in JSX
- Data is fetched server-side; token is read securely from `cookies()`
- Smaller client bundle
- Better SEO and initial paint

**Dynamic routes** use `params` as a Promise in Next.js 16+:
```tsx
interface Props { params: Promise<{ id: string }>; }
export default async function DetailPage({ params }: Props) {
  const { id } = await params;
  const item = await serverApi.get<Item>(`/items/${id}`);
  if (!item) notFound();
  return <DetailView item={item} />;
}
```

## Server Actions for Forms

For **create / edit forms**, keep the form as a Client Component but drive mutations via **Server Actions**.

**Architecture:**
```
app/(admin)/[entity]/
  novo/page.tsx          # Server Component (thin wrapper)
  [id]/page.tsx          # Server Component (fetches initial data)
  _components/
    EntityForm.tsx       # Client Component (uses "use client")
  actions.ts             # Server Actions ("use server")
```

**`actions.ts`:**
```typescript
"use server";
import { serverApi } from "@/lib/serverApi";
import { revalidatePath } from "next/cache";

export async function createEntity(formData: FormData): Promise<{ error?: string }> {
  const payload = { name: String(formData.get("name")) };
  try {
    await serverApi.post("/items", payload);
    revalidatePath("/items");
    return {};
  } catch (err: any) {
    return { error: err.message };
  }
}
```

**`EntityForm.tsx` (Client Component):**
```tsx
"use client";
import { useRouter } from "next/navigation";
export function EntityForm({ action }: { action: (fd: FormData) => Promise<{ error?: string }> }) {
  const router = useRouter();
  const [error, setError] = useState("");
  const [saving, setSaving] = useState(false);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setSaving(true);
    const result = await action(new FormData(e.currentTarget));
    setSaving(false);
    if (result.error) setError(result.error);
    else { router.push("/items"); router.refresh(); }
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <ErrorMessage message={error} />}
      <input name="name" required />
      <button type="submit" disabled={saving}>{saving ? "Saving..." : "Save"}</button>
    </form>
  );
}
```

**Edit page wrapper (Server Component):**
```tsx
export default async function EditPage({ params }: Props) {
  const { id } = await params;
  const item = await serverApi.get<Item>(`/items/${id}`);
  const boundAction = async (fd: FormData) => { "use server"; return updateItem(id, fd); };
  return <EntityForm initial={item} action={boundAction} />;
}
```

## UI Patterns

### Data Tables
```
bg-white rounded-xl border border-slate-200 overflow-hidden
table.w-full.text-sm
thead.bg-slate-50 (th: text-left px-4 py-3 font-medium text-slate-600)
tbody.divide-y.divide-slate-100 (tr.hover:bg-slate-50)
```
- Always include an empty state row (`colSpan`, centered, text-slate-400)
- Action column aligned right (Edit / View links with icons)

### Status Badges
```
inline-flex px-2 py-0.5 rounded-full text-xs font-medium
Active   -> bg-emerald-50 text-emerald-700
Inactive -> bg-slate-100 text-slate-600
Canceled -> bg-red-50 text-red-700
```

### Detail Views
- Grid layout: `grid grid-cols-1 md:grid-cols-2 gap-4`
- Each field: `<label>` (uppercase, xs, slate-500) + `<p>` (slate-800)
- JSON blocks (settings, branding): `<pre class="bg-slate-50 rounded-lg text-xs overflow-auto">`
- Related lists (tenants, subscriptions) with sub-headers

### Forms (Create / Edit)
- White card with border: `bg-white rounded-xl border border-slate-200 p-6`
- Input style: `w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-primary-500`
- Checkbox row: `flex items-center gap-2`
- Buttons: Primary (bg-primary-600) + Secondary (border border-slate-300)
- Error banner at top: `bg-red-50 text-red-700 text-sm rounded-lg`
- Loading state on submit button

### Charts (Metrics)
Use **Recharts** for time-series metrics. Because Recharts requires browser APIs, it must run in a Client Component.

**Pattern: Isolate the chart into a dedicated Client Component wrapper so the parent page remains a Server Component.**

```tsx
// components/MrrChart.tsx  
"use client";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from "recharts";
import { MrrDataPoint } from "@/types";

export function MrrChart({ data }: { data: MrrDataPoint[] }) {
  const chartData = data.map(d => ({ month: d.month, mrr: d.mrr_cents / 100 }));
  return (
    <ResponsiveContainer width="100%" height={320}>
      <LineChart data={chartData}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="month" />
        <YAxis />
        <Tooltip />
        <Line type="monotone" dataKey="mrr" stroke="#2563eb" />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```tsx
// page.tsx (Server Component)
import { serverApi } from "@/lib/serverApi";
import { MrrChart } from "@/components/MrrChart";

export default async function MrrPage() {
  const { items } = await serverApi.get<{ items: MrrDataPoint[] }>("/admin/mrr");
  return (
    <div>
      <h2>MRR</h2>
      <MrrChart data={items} />
    </div>
  );
}
```

- Keep chart card below KPI summary cards
- Format currency with `toLocaleString("pt-BR")`

## Pitfalls

1. **Do NOT put protected routes outside the `(admin)` group.** If you do, middleware and layout shell become harder to coordinate.
2. **Middleware cannot read localStorage.** Always set a cookie when storing the token, or middleware auth checks will fail.
3. **Dynamic routes (`[id]`) need null-checks.** After fetch, handle `null`, loading, and error states explicitly before rendering.
4. **Avoid `next/font` in client components.** `next/font/google` imports should stay in server layouts.
5. **Tables without empty states look broken.** Always render a "Nenhum X encontrado" row when `items.length === 0`.
6. **Currency values from APIs are usually in cents.** Divide by 100 before displaying. Keep cents internally to avoid float errors.

## Dependencies

Standard Next.js admin stack:
```json
{
  "next": "^16.0.0",
  "react": "^19.1.0",
  "tailwindcss": "^4.0.0",
  "@tailwindcss/postcss": "^4.0.0",
  "lucide-react": "^0.468.0",
  "recharts": "^2.15.0"
}
```

## Verification

After scaffolding, run:
```bash
npm run build
```
Expect all routes to generate (static + dynamic). Fix type errors before claiming completion.

## References

- See `references/patterns.md` for session-specific pattern notes (e.g., TeAjuda admin conventions).
