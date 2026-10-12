# Setup & Environment

> [!NOTE]
> Master guidelines: [`AGENTS.md`](file:///d:/flutter_projects/solen_web/AGENTS.md) and [`.agents/rules/`](file:///d:/flutter_projects/solen_web/.agents/rules/).

---

## 1. Prerequisites

- **Node.js**: v20+ LTS, npm
- **Supabase CLI**: `npm install -g supabase` (optional for local database when ready, or connect to a remote Supabase project)
- **Playwright Browsers**: `npx playwright install chromium` (for headless 3D render pipeline and e2e testing)

---

## 2. Environment Variables

Create a `.env.local` file in the project root based on `.env.example`:

```bash
# App Configuration
NEXT_PUBLIC_SITE_URL=http://localhost:3000
NEXT_PUBLIC_DEFAULT_LOCALE=ar

# Supabase (Local or Cloud Project)
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Iraqi Payment Gateways (Test / Sandbox)
ZAINCASH_MERCHANT_ID=test_merchant_id
ZAINCASH_SECRET=test_secret
ZAINCASH_MSISDN=9647800000000
FIB_CLIENT_ID=test_fib_client
FIB_CLIENT_SECRET=test_fib_secret

# Local Print Provider / Fulfillment
PRINT_PROVIDER_API_KEY=test_provider_key
PRINT_PROVIDER_BASE_URL=https://api.printprovider.com

# Currency & Rates
EXCHANGE_RATE_FALLBACK=1310

# Observability
SENTRY_DSN=
ALERT_WEBHOOK_URL=
```

---

## 3. Local Development Workflow

```bash
# 1. Install dependencies
npm install

# 2. Database Setup
# When ready to start local Supabase in Docker:
#   supabase start
#   supabase db reset
# Or link to a hosted Supabase project:
#   supabase link --project-ref <your-ref>
#   supabase db push

# 3. Start Next.js Development Server
npm run dev
```

---

## 4. Verification & Testing Commands

```bash
npm run lint              # ESLint checks
npm run typecheck         # TypeScript strict compilation check
npm run test              # Vitest unit & coordinate transformation tests
npm run test:e2e          # Playwright headless preview & print render tests
```

---

## 5. Common Pitfalls

- **Headless Render URLs**: Playwright headless render tests require absolute HTTP URLs to fetch base garment and displacement maps.
- **RTL & Fonts**: Arabic typography uses **IBM Plex Sans Arabic** loaded via `next/font/google`. Ensure internet connectivity during the first build so Google Fonts can be fetched and cached locally.
- **RLS Verification**: Never run client-facing tests using `SUPABASE_SERVICE_ROLE_KEY`; always use the authenticated user token and anon key to verify actual policy enforcement.
