# Engineering Case Study: Solen — Iraq's Premier Print-on-Demand Marketplace

![Project Category](https://img.shields.io/badge/Category-Full--Stack%20E--Commerce%20%2F%20POD-blue)
![Tech Stack](https://img.shields.io/badge/Stack-Next.js%2016%20%7C%20TypeScript%20%7C%20Supabase%20%7C%20Sharp-black)
![Test Coverage](https://img.shields.io/badge/Tests-445%20Passing%20(56%20Suites)-brightgreen)
![Localization](https://img.shields.io/badge/i18n-Trilingual%20(Arabic%20%7C%20Kurdish%20%7C%20English)-orange)

---

## 1. Executive Summary

**Solen** (سولين) is a high-performance B2B2C Print-on-Demand (POD) and creator commerce marketplace engineered specifically for the operational realities of Iraq's emerging digital economy. 

Prior to Solen, Iraqi content creators and digital artists operated almost exclusively through social media direct messages (Instagram/TikTok DMs), bearing heavy upfront capital risks to buy bulk inventory and suffering high delivery failure rates under informal Cash on Delivery (COD) logistics. 

As the Lead / Founding Full-Stack Engineer, I architected and shipped Solen’s complete web platform from scratch: an interactive browser-based 2D garment design studio, a dual-pipeline rendering engine (client-side 2D compositing + server-side 300 DPI transparent DTF export), a guest-optimized checkout engine spanning all 18 Iraqi governorates, an industrial fulfillment production hub, and a complete financial clearance ledger with zero-float integer arithmetic.

The MVP was engineered under strict architectural constraints, backed by 36 PostgreSQL migrations, and hardened by a 4-stage CI pipeline verifying **445 automated unit/integration tests** and Playwright end-to-end specs.

---

## 2. The Challenge: Building E-Commerce for Iraq's Market Realities

Building an apparel e-commerce platform for emerging markets presents complex infrastructure challenges that off-the-shelf Western SaaS platforms (Shopify, Printful, Stripe) cannot solve:

1. **Cash on Delivery (COD) Dominates (>85%)**: Traditional POD platforms rely on immediate credit card capture via Stripe. Under COD, the platform bears the financial risk of printing and shipping physical garments before customer cash is collected at the doorstep. A high return or cancellation rate could bankrupt operations.
2. **Informal Geography & Addressing**: Iraq lacks a universal postal code system. Addresses rely on informal landmarks, neighborhood names, and governorate-specific shipping tariffs across 18 distinct provinces.
3. **Bandwidth & Device Constraints**: Many users access the web via mobile devices on mobile data networks. Heavy 3D WebGL or 3D engine bundles (Three.js, 100+ MB assets) cause mobile browser crashes, battery drain, and sluggish load times.
4. **Linguistic Plurality**: Iraq requires native support for Arabic (official), Kurdish Sorani (official in Kurdistan), and English, with seamless Right-to-Left (RTL) typography and layout switching.

---

## 3. Engineering Innovations & Key Solutions

### A. Dual-Pipeline Rendering Engine (Lightweight UI + Industrial 300 DPI DTF)
**The Problem**: Garment customizers typically choose between low-quality flat 2D previews or heavy 3D WebGL meshes that are too resource-intensive for mobile shoppers. Furthermore, sending a distorted 3D texture to a physical printing press produces blurry prints.

**The Solution**: I designed a decoupled rendering engine governed by a single mathematical coordinate standard:
- **Client-Side Preview (`renderPlacement.ts`)**: Utilizes lightweight HTML5 2D canvas compositing. It loads the garment photograph, applies dynamic color tinting using `source-in` composite blending, draws the artwork using contain-fit UV coordinates, and composites garment wrinkle and shadow layers using `multiply` blend modes. The result is photorealistic fabric depth in under 15 KB of JavaScript.
- **Server-Side DTF Generation (`dtf-generator.ts`)**: Built with Node.js and `sharp`. It converts physical garment dimensions (`print_width_cm`, `print_height_cm`) into exact 300 DPI pixel dimensions, scales the original artwork using high-quality Lanczos3 resampling, applies rotation strictly around the artwork center, and exports a 100% transparent PNG-24 with physical density metadata. **Zero garment texture or background color is ever sent to printing machinery.**

```
Physical Print Dimension Formula (300 DPI):
Canvas Width (px)  = Round((Print Width cm / 2.54) * 300)
Canvas Height (px) = Round((Print Height cm / 2.54) * 300)
```

### B. Anti-Fraud COD State Machine & Immutable Snapshots
**The Problem**: In COD commerce, malicious actors or competitors can place fraudulent orders to exhaust inventory and inflate printing expenses. Furthermore, if a creator edits or deletes a design after an order is placed, fulfillment would print incorrect graphics.

**The Solution**:
1. **Immutable Snapshot Guarantee**: At checkout, the server compiles a complete `design_snapshot` JSONB embedding variant ID, placement coordinates, color hex code, and high-resolution artwork URLs. A PostgreSQL trigger (`protect_design_snapshot`) rejects any `UPDATE` statements targeting this snapshot post-creation.
2. **Rigid Order State Machine**: Enforced at the database trigger level (`enforce_order_status_transition`), guaranteeing orders can only advance through authorized stages:
   $$\text{pending} \rightarrow \text{processing} \rightarrow \text{printing} \rightarrow \text{packaging} \rightarrow \text{shipped} \rightarrow \text{delivered}$$
   Cancellation is restricted before delivery, and refunds are only permitted after physical delivery.
3. **Multi-Tier Rate Limiting**: Distributed rate limiters backed by a shared PostgreSQL `rate_limits` table restrict orders to 5 requests / 60s per IP and 3 orders / 60s per validated Iraqi phone number.

### C. Zero-Float Financial Ledger & T+3 Delivery Clearance
**The Problem**: Floating-point mathematics (`0.1 + 0.2 = 0.30000000000000004`) in currency calculation creates cumulative accounting leaks. Additionally, paying out creators immediately upon order placement exposes the platform to complete loss if a COD parcel is refused at the door.

**The Solution**:
1. **Integer Arithmetic**: All transactions, commissions, and shipping fees in Iraqi Dinars (`IQD`) are strictly typed as `bigint` / `integer` across the entire stack.
2. **T+3 Delivery Clearance Engine**: Creator profits are logged in `designer_earnings_log` as `pending`. When an order is marked `delivered` and cash collected, a 72-hour safety hold begins. Only after 72 hours do funds transition to `cleared`.
3. **Atomic Payouts RPC (`request_designer_payout`)**: Validates a 25,000 IQD minimum threshold and a 500,000 IQD daily maximum cap, deducting funds and generating a payout batch atomically in a single Postgres transaction.

### D. Multi-Asset Publishing with Resilient Storage Rollbacks
**The Problem**: Publishing a custom garment requires multiple sequential network operations: rendering 1024px preview blobs, uploading original and composite images to Supabase Storage, inserting database records, and generating SKU variants across in-stock sizes and colors. If the user disconnects halfway, orphan files clog storage and corrupted draft rows pollute the catalog.

**The Solution**: Implemented a resilient transactional publishing hook ([`use-studio-publisher.ts`](file:///d:/flutter_projects/solen_web/components/studio/use-studio-publisher.ts)) that tracks every created storage path and database ID. In the event of any network or validation failure, it initiates an automated cleanup routine that removes all uploaded storage files and purges draft database records.

---

## 4. Architectural Highlights & Tech Stack

```
Frontend:           Next.js 16 (App Router) • React 19 • Tailwind CSS v4 • Lucide
State & Routing:    next-intl (Trilingual ar/ku/en) • Custom Gesture Drag Engine
Graphics & Print:   HTML5 Canvas 2D Compositing • Node.js Sharp (300 DPI DTF)
Backend & Auth:     Supabase (PostgreSQL 15) • Row-Level Security (RLS) • Supabase Auth
Validation:         Zod v4 • Magic-Byte File Inspection • Iraqi Phone Regex
Quality & CI:       Vitest (445 tests) • Playwright E2E • GitHub Actions (4 Stages)
```

---

## 5. Verified Engineering Metrics

All metrics derived directly from the active production codebase:

| Metric | Measured Value | Significance |
|---|---|---|
| **TypeScript / TSX Source Files** | **408 files** | Complete end-to-end platform architecture. |
| **Total Lines of Code** | **~71,000 LOC** | Comprehensive application logic and automated test suites. |
| **Max File Length Invariant** | **≤ 400 lines / file** | Strictly enforced modular architecture (`npm run audit:lines`). |
| **Database Migrations** | **36 sequential migrations** | Fully reproducible declarative Postgres infrastructure. |
| **Relational Database Entities** | **25 tables, 9 enums, 50+ RPCs** | Comprehensive data model covering commerce, logistics, and audit. |
| **Automated Test Coverage** | **445 passing tests (56 suites)** | 100% pass rate across unit, math, security, and integration tests. |
| **Localization Parity** | **2,542 keys (100% parity)** | Identical coverage across Arabic, Kurdish Sorani, and English. |
| **Dead Code / Unused Exports** | **0 errors (Knip verified)** | Clean codebase free of orphan components and unused dependencies. |

---

## 6. Key Takeaways & Lessons Learned

1. **Local Context Drives Architecture**: Designing for an emerging market requires engineering for trust first. Features like phone-verified order tracking, landmark delivery fields, and WhatsApp merchant chat links proved far more impactful than generic e-commerce conventions.
2. **Zero-Dependency Graphics Win on Mobile**: Replacing heavy 3D rendering libraries with pure HTML5 Canvas composite operations provided instantaneous previews, preserved battery life on budget Android devices, and eliminated megabytes of client bundle overhead.
3. **Database Constraints Outlive Application Code**: Enforcing business invariants (immutable snapshots, state machine transitions, append-only logs) directly in PostgreSQL triggers prevents data corruption regardless of which API route or background script initiates the change.
