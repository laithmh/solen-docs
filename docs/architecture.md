# Architecture & Technical Specifications

> [!NOTE]
> Primary rules entry point: [`AGENTS.md`](file:///d:/flutter_projects/solen_web/AGENTS.md). Additional rules: [`.agents/rules/`](file:///d:/flutter_projects/solen_web/.agents/rules/).
> For comprehensive portfolio documentation, see [`docs/portfolio/`](file:///d:/flutter_projects/solen_web/docs/portfolio).

---

## 1. Render Pipeline Invariants & Coordinate Standard

### Coordinate Origin & Transformation Convention
All transform values stored in `user_design_placements` (`design_position_x`, `design_position_y`, `design_scale`, `design_rotation`) follow a strict normalized coordinate standard relative to `mockup_placements.printable_area_*`:

- **Printable Area Bounds**: `printable_area_x`, `printable_area_y`, `printable_area_width`, `printable_area_height` in pixels on the mockup base image.
- **Position (`design_position_x`, `design_position_y`)**:
  - Normalized UV coordinates representing the **center** of the artwork inside the printable area.
  - Range: `[0.0, 1.0]` where `(0.5, 0.5)` is perfectly centered, `(0.0, 0.0)` is top-left, and `(1.0, 1.0)` is bottom-right.
  - Pixel position calculation:
    $$\text{pixelX} = \text{printable\_area\_x} + \text{design\_position\_x} \times \text{printable\_area\_width}$$
    $$\text{pixelY} = \text{printable\_area\_y} + \text{design\_position\_y} \times \text{printable\_area\_height}$$
- **Scale (`design_scale`)**:
  - `1.0` represents standard **`contain` fit** (the artwork's largest dimension fills 100% of the corresponding printable area dimension while preserving original aspect ratio).
  - Minimum scale clamped to `0.05` (`MIN_DESIGN_SCALE`).
- **Rotation (`design_rotation`)**:
  - Clockwise angle in degrees `[0.0, 360.0)`.
  - Rotates strictly around the design's own center `(design_position_x, design_position_y)`.

---

## 2. Repo Layout (Next.js 16 Monorepo)

```
/app
  /[locale]
    /                      # Landing page (hero, profit simulator, shipping calculator)
    /store/[storeSlug]     # Creator's branded storefront & catalog
    /p/[productSlug]       # Product details & placement switcher
    /catalog               # Marketplace discovery catalog
    /checkout              # Iraqi checkout (18 governorates, COD, vouchers)
    /checkout/success      # Order confirmation with WhatsApp bridge
    /track                 # Self-serve order tracking (order # + phone guard)
    /studio/[mockupId]     # 2D gesture drag studio with live compositing & DPI gauge
    /dashboard             # Creator dashboard (overview, products, orders, payouts, settings)
    /dashboard/production  # Fulfillment Production Hub (job queue, packing slips, dispatch)
    /dashboard/admin       # Platform governance (designs, mockups, users, settings, audit)
    /(auth)                # Login, register, password reset, email verification
  /api
    /orders                # Order submission, Zod validation, pricing resolver
    /admin/print-file/[id] # Sharp-powered 300 DPI transparent DTF export
    /auth                  # Supabase auth session, signout, callback
/components
  /ui                      # Design system tokens (Button, Card, Input, Badge, Dialog)
  /studio                  # Studio wizard, 2D canvas, gesture drag, DPI gauge
  /storefront              # Cart drawer, governorate select, payment selector
  /merchant                # Creator dashboard tabs, orders drawer, WhatsApp actions
  /production              # Job cards, placement inspector, printable packing slips
  /admin                   # Curation tables, settings forms, audit log viewer
/lib
  /supabase                # Browser, server, and admin client factories
  /render                  # renderPlacement.ts (2D canvas compositing), math.ts (UV & DPI)
  /print                   # dtf-generator.ts (Node.js Sharp 300 DPI physical engine)
  /orders                  # pricing.ts, promo-service.ts, track-service.ts, schema.ts
  /security                # shared-rate-limit.ts, rate-limit.ts, file-validation.ts
  /notifications           # In-app notifications, Telegram alerts, WhatsApp link builders
/messages                  # Trilingual i18n JSON files (ar.json, ku.json, en.json)
/supabase
  /migrations              # 36 sequential PostgreSQL migrations
/tests
  /unit                    # 56 Vitest test suites (445 tests: math, security, state machine)
  /e2e                     # Playwright end-to-end specs (COD checkout, tracking, studio)
```

---

## 3. The Dual-Pipeline Render Engine

Solen decouples user interface rendering from manufacturing print file generation:

```
Shared Geometry: UV Center (u, v) + Contain Fit Scale + Clockwise Rotation
  ├── Client Preview: lib/render/renderPlacement.ts
  │     └── HTML5 Canvas 2D compositing: source-in tinting + multiply fabric shading (<15 KB JS)
  └── Server DTF: lib/print/dtf-generator.ts
        └── Node.js Sharp: Physical cm @ 300 DPI, Lanczos3 scale, 100% transparent PNG-24
```

- **`mode: 'preview'` (Client UI Preview)**:
  - Composites base photographic garment blank with dynamic color tinting (`source-in`).
  - Renders artwork at normalized coordinates with aspect-preserving contain fit.
  - Multiplies ambient fabric wrinkle and shadow overlays (`multiply` blend mode).
- **`mode: 'print'` (Physical DTF Manufacturing)**:
  - Target canvas calculated directly from physical centimeters at 300 DPI:
    $$\text{widthPx} = \text{round}\left(\frac{\text{print\_width\_cm}}{2.54} \times 300\right)$$
    $$\text{heightPx} = \text{round}\left(\frac{\text{print\_height\_cm}}{2.54} \times 300\right)$$
  - Flat artwork only, transparent background, zero garment shading, zero fabric texture, clipped to printable bounds.
  - Output is uncompressed PNG-24 at 300 DPI.

---

## 4. End-to-End Data Flow

1. **Mockup Calibration**: Admin configures base garment photography, printable boundaries (`printable_area_x/y/w/h`), physical dimensions (`print_width_cm/height_cm`), and fabric texture overlay.
2. **Creator Studio**: Creator opens `/studio/[mockupId]`. The 2D canvas loads placement zones. Creator drags, scales, and rotates graphics with live DPI feedback.
3. **Atomic Publish**: Creator sets profit margin and publishes. The client renders 1024px composite previews, uploads original and preview assets to Supabase Storage, creates records in `user_designs` and `user_design_placements`, and invokes RPC `create_design_and_auto_variants` to generate SKU variants. If any step fails, an automated rollback cleans up storage and draft records.
4. **Shopper Checkout**: Shopper orders via COD.
   - Client sends cart items and Iraqi delivery address (18 governorates, district, landmark).
   - Server validates phone format (`+964 7X`) and enforces distributed rate limits.
   - Server recalculates prices and creator markups from database rows (`product_variants`).
   - Server freezes immutable `design_snapshot` JSONB into `order_items` (protected by DB trigger).
   - Non-blocking alerts dispatched to Telegram operations bot and creator in-app bell.
5. **Print Fulfillment**: Order advances to `processing` / `printing`. Print operator opens Production Hub (`/dashboard/production`), downloads 300 DPI transparent DTF print files generated on-the-fly by Sharp, prints packing slips with piece codes, and hands parcels to domestic couriers.
6. **Delivery & Clearance**: Courier collects COD cash. Order is marked `delivered`. Creator earnings remain in a 72-hour safety hold (T+3) before becoming eligible for payout withdrawal via ZainCash, FIB, or Qi Card.

---

## 5. Iraqi Localization & Market Adaptation

- **Trilingual Routing (`next-intl`)**:
  - Trilingual routes: `/[locale]/...` (`ar`, `ku`, `en`).
  - Arabic and Kurdish Sorani render in native Right-to-Left (`dir="rtl"`).
  - 100% key parity across all 3 locales (2,542 keys per file) verified via CI.
- **Typography**:
  - Arabic & Kurdish: **IBM Plex Sans Arabic**.
  - English: **Playfair Display** (headings) + **IBM Plex Sans** (body).
- **Numerals Standard**: Western Arabic numerals (`0123456789`) are rendered across all languages to eliminate payment and tracking ambiguity.
- **Logistics**:
  - 18 Iraqi Governorates dropdown with regional shipping tariffs stored in `governorate_shipping_rates`.
  - Required district and landmark fields to address informal Iraqi street layouts.
  - Phone validation supporting Iraqi telecom networks (Zain, Asiacell, Korek).

---

## 6. Service Boundaries & Quality Standards

- **Payments**: Server-authoritative. Client prices are purely informational; server validates all amounts against database variants and active promo codes. Cash on Delivery is active; mobile wallets operate via structured deferred verification.
- **Role-Based Routing**: Middleware (`proxy.ts`) and server guards inspect `profiles.role` (`customer`, `designer`, `printer`, `production_manager`, `admin`).
- **File Length & Clean Code**: Strict 400-line per file ceiling (`npm run audit:lines`) and zero dead code (`npm run audit:dead-code`).
- **Test Invariant**: 56 Vitest test suites (445 tests) must pass with zero errors in CI before deployment.
