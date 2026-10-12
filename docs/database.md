# Database Schema & Migrations

Supabase PostgreSQL. Every schema alteration is authored as a migration file in `/supabase/migrations/` (36 sequential migrations total). Never edit schemas via the Supabase web dashboard.

> [!NOTE]
> For the comprehensive portfolio database specifications and full entity catalog, see [`docs/portfolio/04-database.md`](file:///d:/flutter_projects/solen_web/docs/portfolio/04-database.md).

---

## 1. Placements Normalization

Garments support arbitrary printable areas (front, back, left sleeve, right sleeve, hood, pocket) via normalized placement rows:

```sql
-- One row per printable placement supported by a garment template (mockup).
create table public.mockup_placements (
  id uuid not null default gen_random_uuid (),
  mockup_id uuid not null references public.mockups (id) on delete cascade,
  placement_key text not null,        -- 'front' | 'back' | 'left_sleeve' | 'hood' | ...
  display_name text not null,         -- English display name ('Front', 'Back')
  display_name_ar text not null,      -- Arabic display name ('الجهة الأمامية', 'الجهة الخلفية')
  display_name_ku text null,          -- Kurdish display name ('ڕووی پێشەوە', 'ڕووی پشتەوە')
  sort_order int not null default 0,
  template_image_url text null,       -- Flat 2D undistorted editing surface; falls back to base_image_url
  base_image_url text not null,       -- Photographic mockup used for 2D composite preview render
  overlay_image_url text null,        -- Garment fabric texture, highlights, and shadow overlay
  displacement_map_url text null,     -- Grayscale height map for fabric folds
  displacement_strength real not null default 0.03
    constraint chk_displacement_strength check (displacement_strength between 0 and 0.2),
  base_image_width integer null,
  base_image_height integer null,
  printable_area_x real not null,
  printable_area_y real not null,
  printable_area_width real not null,
  printable_area_height real not null,
  print_width_cm numeric(6,2) not null,
  print_height_cm numeric(6,2) not null,
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint mockup_placements_unique unique (mockup_id, placement_key)
);

-- One row per placement per customer design (draft & active editing state).
create table public.user_design_placements (
  id uuid not null default gen_random_uuid (),
  user_design_id uuid not null references public.user_designs (id) on delete cascade,
  mockup_placement_id uuid not null references public.mockup_placements (id) on delete restrict,
  placement_key text null,
  design_image_url text null,
  design_position_x real null default 0.5,    -- Normalized center X [0.0 - 1.0]
  design_position_y real null default 0.5,    -- Normalized center Y [0.0 - 1.0]
  design_scale real null default 1.0,         -- Contain fit = 1.0
  design_rotation real null default 0.0,      -- Clockwise degrees [0.0 - 360.0)
  preview_image_url text null,                -- Offscreen canvas composite preview
  print_file_url text null,                   -- 300 DPI flat transparent PNG
  source_dpi real null,
  is_print_ready boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint user_design_placements_unique unique (user_design_id, mockup_placement_id)
);
```

---

## 2. PostgreSQL Master Enums

```sql
-- 5 Platform Roles
create type public.user_role as enum ('customer', 'designer', 'admin', 'production_manager', 'printer');

-- Store Moderation Status
create type public.store_status as enum ('pending', 'approved', 'rejected');

-- Design Publishing Status
create type public.design_status as enum ('draft', 'pending_review', 'approved', 'rejected');

-- Order Lifecycle State Machine
create type public.order_status as enum (
  'pending',       -- Order placed via COD, awaiting confirmation
  'processing',    -- Order confirmed by merchant / admin
  'printing',      -- Dispatched to print machinery; DTF file downloaded
  'packaging',     -- Cured, heat-pressed, and packaged
  'shipped',       -- Handed over to domestic courier
  'delivered',     -- Doorstep delivery completed & cash collected
  'cancelled',     -- Cancelled before delivery (terminal state)
  'refunded'       -- Refunded post-delivery (terminal state)
);

-- Payment Collection Status
create type public.payment_status as enum ('unpaid', 'paid', 'partially_refunded', 'fully_refunded', 'failed');

-- Creator Earnings Clearance (T+3 Post-Delivery Hold)
create type public.earning_status as enum ('pending', 'cleared', 'cancelled');

-- Payout Withdrawal Request Status
create type public.payout_status as enum ('pending', 'processing', 'paid', 'failed', 'cancelled');

-- Payout Channels
create type public.payout_method_type as enum ('zain_cash', 'fib', 'qi_card', 'fastpay', 'cash_pickup');

-- 18 Iraqi Governorates
create type public.iraq_governorate as enum (
  'baghdad', 'basra', 'erbil', 'sulaymaniyah', 'duhok',
  'najaf', 'karbala', 'kirkuk', 'nineveh', 'babil',
  'anbar', 'dhi_qar', 'diyala', 'maysan', 'al_qadisiyyah',
  'muthanna', 'wasit', 'saladin'
);
```

---

## 3. Order Status State Machine Trigger

Order transitions are strictly guarded by PostgreSQL trigger function `enforce_order_status_transition()`:

```sql
CREATE OR REPLACE FUNCTION public.enforce_order_status_transition()
RETURNS trigger AS $$
BEGIN
  IF (OLD.status = 'pending'    AND NEW.status IN ('processing', 'printing', 'cancelled')) OR
     (OLD.status = 'processing' AND NEW.status IN ('printing', 'shipped', 'cancelled')) OR
     (OLD.status = 'printing'   AND NEW.status IN ('packaging', 'shipped', 'cancelled')) OR
     (OLD.status = 'packaging'  AND NEW.status IN ('shipped', 'cancelled')) OR
     (OLD.status = 'shipped'    AND NEW.status IN ('delivered', 'cancelled', 'refunded')) OR
     (OLD.status = 'delivered'  AND NEW.status = 'refunded') THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'Invalid order status transition: % -> %', OLD.status, NEW.status;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enforce_order_status_transition
  BEFORE UPDATE ON public.orders
  FOR EACH ROW EXECUTE FUNCTION enforce_order_status_transition();
```

---

## 4. Frozen Order Snapshots & Immutability Trigger

```sql
-- Shape of order_items.design_snapshot (JSONB):
-- {
--   "title": "Bespoke Solen Apparel",
--   "variant_id": "...",
--   "design_id": "...",
--   "size": "XL",
--   "color": "Classic Black",
--   "color_hex": "#1E1E1E",
--   "unit_price_iqd": 25000,
--   "quantity": 1,
--   "placements": [
--     { "placement_key": "front", "preview_image_url": "...", "design_image_url": "...", "position_x": 0.5, "position_y": 0.5, "scale": 1.0, "rotation": 0.0 }
--   ],
--   "frozen_at": "2026-10-12T14:00:00Z"
-- }

CREATE OR REPLACE FUNCTION public.protect_design_snapshot()
RETURNS trigger AS $$
BEGIN
  IF old.design_snapshot IS DISTINCT FROM new.design_snapshot THEN
    RAISE EXCEPTION 'design_snapshot is immutable once an order_item is created';
  END IF;
  RETURN new;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_protect_design_snapshot
  BEFORE UPDATE ON public.order_items
  FOR EACH ROW EXECUTE FUNCTION protect_design_snapshot();
```

---

## 5. Monetization & Fee Model C

The platform computes retail prices using database RPC `calculate_platform_fee(markup)`:

```sql
CREATE OR REPLACE FUNCTION public.calculate_platform_fee(p_designer_markup numeric)
RETURNS numeric AS $$
DECLARE
  v_fixed numeric;
  v_pct numeric;
BEGIN
  SELECT COALESCE(fixed_platform_fee, 3000), COALESCE(platform_fee_percentage, 15.0)
  INTO v_fixed, v_pct
  FROM public.global_config
  LIMIT 1;

  RETURN ROUND(v_fixed + (COALESCE(p_designer_markup, 0) * v_pct / 100.0));
END;
$$ LANGUAGE plpgsql;

-- Trigger syncs product_variants.price whenever base_cost or designer_markup changes:
-- price = base_cost + designer_markup + calculate_platform_fee(designer_markup)
```

---

## 6. Financial Clearance & Payout Invariants

- **T+3 Delivery Hold**: Earnings logged in `designer_earnings_log` remain in `pending` hold for 72 hours after `orders.delivered_at`.
- **Minimum Withdrawal**: **25,000 IQD**.
- **Daily Withdrawal Cap**: **500,000 IQD**.
- **Atomic Deduction**: RPC `request_designer_payout` debits cleared balance and registers payout in a single atomic transaction.
- **Append-Only Auditing**: `admin_audit_logs` records all admin events; `UPDATE` and `DELETE` are blocked by trigger.
