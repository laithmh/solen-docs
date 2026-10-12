# Code Standards

> [!NOTE]
> Master guidelines: [`AGENTS.md`](file:///d:/flutter_projects/solen_web/AGENTS.md) and [`.agents/rules/`](file:///d:/flutter_projects/solen_web/.agents/rules/).

---

## 1. Language & Architecture

- **TypeScript**: `strict: true` across the monorepo. No `any` without an explicit explanatory comment.
- **Server Boundaries**:
  - Supabase database operations, price calculations, and print generation reside in `/lib/`, never inline within UI components.
  - Route handlers (`/app/api/**`) remain thin: validate input, call `/lib` function, return standard HTTP response.
- **Component Colocation**: Feature-specific components reside in `/components/<feature>/`. Shared primitives live in `/components/ui/`.
- **Exports**: Named exports across all files. Default exports are reserved exclusively for Next.js convention files (`page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`).

---

## 2. Naming Conventions

- **Files**: `kebab-case.tsx` / `kebab-case.ts`.
- **Components & Types**: `PascalCase`.
- **Functions & Variables**: `camelCase`.
- **Database Columns**: `snake_case` in Postgres; mapped to `camelCase` at the `/lib/supabase` boundary.
- **Booleans**: Prefixed with auxiliary verbs (`isPrintReady`, `isActive`, `hasDisplacementMap`).

---

## 3. Iraqi Market, Currency & Formatting

- **Currency Calculations**:
  - Always store and calculate Iraqi Dinar amounts as **integers** (`bigint` or `number` in JS without decimals).
  - Exchange rates from USD to IQD are fetched from `global_config` dynamically at transaction time.
- **Numerals**:
  - Standard Western digits (`1234567890`) must be used exclusively. Use `formatIQD(amount)` from `/lib/utils` which guarantees English digits with thousands commas and appropriate currency label (`د.ع` or `IQD`).
- **Phone Validation**:
  - Always validate Iraqi mobile numbers (`^(?:\+964|00964|0)?7[3-9]\d{8}$`) using `/lib/validation/phone.ts`.
- **Mandatory Trilingual Localization (`ar`, `en`, `ku`)**:
  - **Zero hardcoded text**: Never hardcode user-visible strings into components or pages.
  - Every new feature, UI primitive, modal, or page must add translation keys to all three dictionary files:
    - `messages/ar.json` (Arabic, default)
    - `messages/en.json` (English)
    - `messages/ku.json` (Kurdish Sorani)
  - Components render text using `const t = useTranslations('Namespace')`.
  - Directional styling must use logical Tailwind utilities (`ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`) to support both RTL (`ar`, `ku`) and LTR (`en`).

---

## 4. Render Pipeline Standards

- All shader and compositing logic lives strictly in `/lib/render/renderPlacement.ts`.
- Use the documented normalized coordinate space:
  - `(design_position_x, design_position_y)` represents artwork center within the printable area `[0.0, 1.0]`.
  - `design_scale` is contain-fit relative.
  - `design_rotation` is clockwise degrees `[0.0, 360.0)`.

---

## 5. UI Primitives & Design System Adherence

- Build and extend primitives in `/components/ui/` (`Button`, `Card`, `RecessedInput`, `Chip`).
- Use CSS variable tokens (`--bg-surface`, `--shadow-raised-sm`, `--primary-accent`); never write arbitrary hex values in components.
- Support RTL seamlessly using logical CSS / Tailwind utilities (`ms-*`, `me-*`, `ps-*`, `pe-*`).

---

## 6. Complexity & Modularity Limits

- **Strict File Limit**: Every source or test file must be strictly **under 400 lines** of code (`npm run audit:lines`).
- **Components**: Aim for under ~150 lines. Decompose large views into:
  - Extracted subcomponents for distinct UI sections (cards, grids, summary drawers, filter bars).
  - Custom hooks for business logic, drag handling, and data orchestration (`use*`).
  - Dedicated `types.ts` for domain models and component prop contracts.
- **Functions**: Under ~40 lines with a single, clear responsibility.
- **Props**: Max 6–7 props per component. If more are needed, group related properties into a config object.

---

## 7. Dead Code & Dependency Hygiene

- **Zero Dead Code**: Always run `npm run audit:dead-code` (Knip) before committing changes.
- **Prune Unused Exports**: Do not export functions or interfaces unless consumed by other modules or part of a documented public API.
- **Zero Unused Dependencies**: Never install packages that are not directly imported and used in the production codebase.

---

## 8. Security Standards by Design

- **Open Redirect Protection**:
  - All redirection destinations (e.g. auth callbacks, signout redirects, post-checkout navigation) must pass through `getSafeRedirectUrl(url, fallback)`. Never redirect to raw unvalidated user input.
- **Database Function Search Paths**:
  - Always declare `SET search_path = public, pg_temp;` on all Postgres functions and triggers to prevent search path hijacking.
- **Least-Privilege Database Execution**:
  - Revoke `EXECUTE` privileges from `anon` on all administrative, role check, and `SECURITY DEFINER` RPC functions (`REVOKE EXECUTE ... FROM anon, PUBLIC`).
- **Secret Key Isolation**:
  - `SUPABASE_SERVICE_ROLE_KEY`, payment provider private keys (`ZAINCASH_SECRET`, `FIB_CLIENT_SECRET`), and fulfillment tokens must remain strictly server-side. Never prefix them with `NEXT_PUBLIC_` or import them in client components.
- **HTTP Security Headers**:
  - Enforce `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, and `Referrer-Policy: strict-origin-when-cross-origin` in `next.config.ts`.
- **Input Validation**:
  - Validate all client inputs, route handler payloads, and server actions using Zod schemas (`lib/auth/validation.ts`, etc.).

---

## 9. Performance & Optimization Standards

- **Postgres RLS Optimization**:
  - Always wrap `auth.uid()` in `(select auth.uid())` within Row-Level Security policies. This enables Postgres to cache the user ID across rows instead of re-executing per row.
- **Covering Foreign Key Indexes**:
  - Every foreign key constraint must have a corresponding covering index to eliminate sequential table scans during join operations.
- **Single Permissive Policy per Action**:
  - Avoid creating multiple permissive policies for the same role and action on a single table. Consolidate policies by action (`INSERT`, `UPDATE`, `DELETE`) to avoid redundant policy execution overhead.
- **Client Bundle Code Splitting**:
  - Heavy 3D or 2D rendering libraries (`three`, `fabric`) must be dynamically imported with `next/dynamic` or isolated to dedicated client stages to minimize initial bundle size for shoppers.
- **Resource Cleanup on Unmount**:
  - Canvas instances, Three.js WebGL renderers/scenes, and window resize listeners must always be disposed/removed in `useEffect` cleanup routines.
