# Testing

Stack: **Vitest** (unit & integration) + **Playwright** (e2e, RTL layout, and headless render verification).

> [!NOTE]
> Master guidelines: [`AGENTS.md`](file:///d:/flutter_projects/solen_web/AGENTS.md) and [`.agents/rules/`](file:///d:/flutter_projects/solen_web/.agents/rules/).

---

## 1. Unit Tests (Vitest) — Pure Logic

Located under `/tests/unit/`:

- **Coordinate & Transform Math**:
  - Verification of UV mapping: `(0.5, 0.5)` center maps accurately to printable area center.
  - `contain` fit calculations: aspect ratio preservation across varying aspect ratios (portrait, landscape, square).
  - Rotation matrix around custom center `(design_position_x, design_position_y)`.
- **DPI & Print Resolution Math**:
  - Pixel calculation: `print_width_cm / 2.54 * 300`.
  - Resolution warning thresholds (detecting low source DPI < 150 at target size).
- **Iraqi Market Logic**:
  - Currency calculations: Integer IQD math, rounding rules, promo code deductions.
  - Iraqi phone number regex validation: testing Zain (`078x`), Asiacell (`077x`), Korek (`075x`), international prefix formats (`+9647...`, `009647...`, `07...`), and rejecting invalid numbers.

---

## 2. Integration Tests (Vitest + Supabase)

Located under `/tests/integration/`:

- **Row-Level Security (RLS)**:
  - Verify that authenticated user A cannot update user B's designs or placements.
  - Verify storefront public can read active mockups and placements.
- **Order Snapshot Immutability**:
  - Attempting to update `order_items.design_snapshot` after insert must trigger `protect_design_snapshot()` and reject the mutation with a Postgres error.
- **Server Price Recalculation**:
  - Submitting tampered client-side prices during checkout must be overridden by server recomputation.

---

## 3. End-to-End Tests (Playwright)

Located under `/tests/e2e/`:

- **Creator Studio Workflow**:
  - Open `/studio/[mockupId]`, upload image, position/scale on flat template.
  - Switch tabs between multiple placements (Front, Back, Sleeves) — verify state persistence across tabs.
  - Toggle 3D Preview — verify Three.js canvas mounts and renders without shader compilation errors.
  - Publish design — assert `preview_image_url` is returned and accessible.
- **Shopper & Checkout Workflow (Iraqi Market)**:
  - Browse storefront in Arabic (`/ar`), select product, choose size and color.
  - Add to cart, proceed to `/ar/checkout`.
  - Fill Iraqi address (Governorate: Baghdad, District: Karrada, Landmark: Near Kahramana).
  - Select Cash on Delivery (COD), enter Iraqi phone number, submit order.
  - Assert order created with valid status and frozen snapshot.
- **Bidirectional / RTL Testing**:
  - Run checkout and studio in Arabic (`dir="rtl"`) with **IBM Plex Sans Arabic** to verify layout mirroring and that English digits (`123`) remain intact.

---

## 4. Render Pipeline Regression Suite

- **Print Output Fidelity**: Run `renderPlacement(mode: 'print')` with fixed test artwork. Assert:
  - Exact dimensions match target physical dimensions at 300 DPI.
  - Transparent alpha channel is 100% intact.
  - Zero displacement warping, zero garment shadows.
- **Preview Headless Parity**: Assert headless server render produces pixel similarity (visual regression threshold < 2%) with client-side interactive preview.
