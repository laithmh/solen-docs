# Solen — End-to-End User & System Flows

This document details the primary lifecycle flows in Solen using visual Mermaid diagrams, mapping customer touchpoints, server actions, background workers, and database state changes.

---

## 1. Creator Journey: Onboarding, Studio Design & Storefront Publishing

```mermaid
sequenceDiagram
    autonumber
    actor Creator as Creator / Artist
    participant Studio as Studio Wizard (/studio/[id])
    participant Canvas as 2D Flat Canvas & Drag Engine
    participant Math as Coordinate & DPI Math Engine
    participant Publisher as useStudioPublisher Hook
    participant Storage as Supabase Storage
    participant DB as Postgres DB & RPCs

    Creator->>Studio: Select Garment Blank (e.g. Boxy Tee)
    Studio->>Canvas: Initialize Placement Tabs (Front, Back, Sleeves)
    Creator->>Canvas: Upload PNG/WEBP Artwork
    Math->>Math: Compute Contain Fit & Initial Center (0.5, 0.5)
    Creator->>Canvas: Drag, Scale, Rotate, Pick Garment Colors
    Canvas->>Math: Recalculate Effective Print DPI
    Math-->>Studio: DPI Gauge Update (Green >= 300, Yellow >= 150, Red < 150)
    Creator->>Studio: Set Profit Margin (e.g. 8,000 IQD) & Title
    Creator->>Studio: Click "Publish Design"
    Studio->>Publisher: Trigger Atomic Publish Pipeline
    Publisher->>Canvas: Render Offscreen 1024px Composite Previews
    Publisher->>Storage: Upload Original Artwork (user-design-originals)
    Publisher->>Storage: Upload Composite Previews (user-design-previews)
    Publisher->>DB: INSERT into user_designs (status: pending_review)
    Publisher->>DB: INSERT into user_design_placements (UV coords, scale, rot)
    Publisher->>DB: RPC create_design_and_auto_variants (Matrix for in-stock sizes/colors)
    alt Failure in any step
        Publisher->>Storage: Rollback: Delete uploaded storage files
        Publisher->>DB: Rollback: Delete draft design records
        Publisher-->>Creator: Show Error Alert with retry
    else Success
        DB-->>Publisher: Design & Variants Created Successfully
        Publisher-->>Creator: Show Publish Success Card with Shareable URL & QR Code
    end
```

---

## 2. Shopper Experience: Discovery, Cart & Iraqi COD Checkout

```mermaid
sequenceDiagram
    autonumber
    actor Shopper as Shopper / Buyer
    participant Store as Creator Storefront (/store/[slug])
    participant Product as Product View (/p/[slug])
    participant Cart as Cart Drawer (CartProvider)
    participant Checkout as Checkout Page (/checkout)
    participant OrderAPI as Order API Handler (/api/orders)
    participant RateLimiter as Shared Postgres Rate Limiter
    participant DB as Postgres DB (orders, order_items)
    participant Telegram as Telegram Bot Operations Alert

    Shopper->>Store: Discover Creator Store via Instagram/TikTok link
    Shopper->>Product: Select Garment, Color Swatch & Size Chip
    Shopper->>Cart: Add to Cart (Stored in LocalStorage & Context)
    Shopper->>Checkout: Proceed to Checkout (Guest Checkout)
    Shopper->>Checkout: Enter Name, Iraqi Phone (07XXXXXXXXX), Governorate, District, Landmark
    Shopper->>Checkout: (Optional) Apply Promo Voucher Code
    Shopper->>Checkout: Select Payment Method (Cash on Delivery - COD)
    Shopper->>OrderAPI: POST /api/orders (Payload with cart items & address)
    OrderAPI->>RateLimiter: Check IP rate limit (max 5/60s) & Phone rate limit (max 3/60s)
    OrderAPI->>DB: Re-verify item prices & in-stock status from product_variants
    OrderAPI->>DB: RPC validate_promo_code (Compute discount if applied)
    OrderAPI->>DB: Lookup governorate shipping fee (governorate_shipping_rates)
    OrderAPI->>OrderAPI: Build immutable design_snapshot JSONB
    OrderAPI->>DB: INSERT into orders (status: pending, payment: unpaid)
    OrderAPI->>DB: INSERT into order_items (with frozen design_snapshot)
    OrderAPI->>DB: INSERT into designer_earnings_log (status: pending)
    OrderAPI->>Telegram: Dispatch Real-time Telegram Order Alert (Non-blocking)
    OrderAPI-->>Checkout: Return 201 Created (orderId, orderNumber)
    Checkout->>Shopper: Redirect to /checkout/success (Displays Order #, Delivery Notice)
```

---

## 3. Customer Order Tracking (Self-Serve Portal)

```mermaid
sequenceDiagram
    autonumber
    actor Shopper as Customer
    participant Portal as Tracking Portal (/track)
    participant Service as track-service.ts
    participant DB as Supabase Postgres

    Shopper->>Portal: Enters Order Reference (e.g. SOL-4829) + Phone Number (0770XXXXXXX)
    Portal->>Service: lookupOrderTracking(orderRef, rawPhone)
    Service->>Service: Normalize Iraqi Phone to digits
    Service->>DB: Query orders where order_number = ref AND customer_phone = normalizedPhone
    alt Order not found or phone mismatch
        DB-->>Service: Null / Empty
        Service-->>Portal: Error: "Order not found or phone number does not match"
    else Match found
        DB-->>Service: Return Order, Status, Governorate, Items Snapshot
        Service-->>Portal: Render Order Tracker View:
        Note over Portal: 1. Status Stepper: Confirmed -> Processing -> Printing -> Dispatched -> Delivered<br/>2. Delivery Address & Estimated Window<br/>3. Ordered Items List with sizes & colors<br/>4. Direct WhatsApp link for instant support
    end
```

---

## 4. Fulfillment & Print Hub Production Workflow

```mermaid
sequenceDiagram
    autonumber
    actor Printer as Print Shop Operator
    participant Hub as Production Hub (/dashboard/production)
    participant PrintAPI as Print API (/api/admin/print-file/[id])
    participant Generator as Sharp DTF Generator (dtf-generator.ts)
    participant Action as Production Server Actions
    participant DB as Postgres DB

    Printer->>Hub: Open Production Queue (Filtered by 'processing' or 'printing')
    Hub->>DB: Fetch jobs with assigned piece codes (e.g. BAG-089-L-BLK)
    Printer->>Hub: Click "Inspect Placement & Download Print File"
    Hub->>PrintAPI: GET /api/admin/print-file/[placementId]
    PrintAPI->>Generator: generateDtfPrintFile(artworkBuffer, print_w_cm, print_h_cm, UV, scale, rot)
    Generator->>Generator: Calculate physical canvas (300 DPI), contain fit, lanczos3 scale
    Generator-->>PrintAPI: 300 DPI Transparent PNG-24 Buffer
    PrintAPI-->>Printer: Browser download of DTF Print File
    Printer->>Hub: Click "Print Packing Slip"
    Hub-->>Printer: Opens Print-ready Packing Slip (Barcode, QR, Piece Codes, Address)
    Printer->>Action: advanceProductionJobAction(orderId, 'packaging')
    Action->>DB: Update order status (Trigger verifies allowed transition)
    Printer->>Action: handoverToCourierAction(orderIds, courierName: "Al-Zajel", manifest: "MNF-992")
    Action->>DB: Batch update order status to 'shipped'
```

---

## 5. Creator Earnings Clearance & Payout Lifecycle (T+3)

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Operations / Admin
    actor Courier as Iraqi Courier
    actor Creator as Creator / Merchant
    participant PayoutTab as Merchant Payouts Tab (/dashboard?tab=payouts)
    participant RPC as Postgres RPC request_designer_payout
    participant DB as Postgres DB

    Courier->>Admin: Courier collects cash from customer and marks package delivered
    Admin->>DB: confirmDeliveryAndCodCashAction(orderId)
    DB->>DB: orders.status = 'delivered', payment_status = 'paid', delivered_at = NOW()
    Note over DB: Earnings remain in 'pending' hold for 72 hours (T+3 Delivery Hold)<br/>to safeguard against doorstep returns or disputes.
    Note over DB: After 72 hours, earnings transition to 'cleared'
    Creator->>PayoutTab: Views Available Withdrawable Balance
    Creator->>PayoutTab: Submits Payout Request (Channel: ZainCash / FIB / Qi Card, Account ID)
    PayoutTab->>RPC: request_designer_payout(p_amount, p_channel, p_identifier, p_holder)
    RPC->>RPC: Verify minimum threshold (25,000 IQD) & maximum cap (500,000 IQD)
    RPC->>RPC: Atomically deduct amount from cleared balance and create row in payouts
    RPC-->>PayoutTab: Payout Request Registered (Status: 'pending')
    Admin->>DB: Admin reviews batch and marks payout 'paid' with transaction reference
    DB-->>Creator: Creator receives mobile wallet notification in Iraq
```

---

## 6. Admin Governance & Platform Curation Flow

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Platform Admin
    participant AdminUI as Admin Dashboard (/dashboard/admin)
    participant Actions as Admin Server Actions
    participant DB as Postgres DB
    participant Audit as admin_audit_logs Table

    Admin->>AdminUI: Navigate to Designs Review (/dashboard/admin/designs)
    AdminUI->>DB: Fetch designs with status = 'pending_review'
    Admin->>Actions: reviewDesignAction(designId, action: 'approved')
    Actions->>DB: UPDATE user_designs SET design_status = 'approved', is_published = true
    Actions->>Audit: Append record to admin_audit_logs (actor, action, diff, timestamp)
    Note over Audit: Table is append-only. UPDATE and DELETE are blocked by database trigger.
    Admin->>AdminUI: Navigate to Stores Curation (/dashboard/admin/stores)
    Admin->>Actions: toggleStoreSpotlightAction(storeId, isSpotlight: true)
    Actions->>DB: UPDATE user_stores SET is_spotlight = true
    Actions->>Audit: Append record to admin_audit_logs (domain: 'stores')
    AdminUI-->>Admin: Show Instant Toast Confirmation
```
