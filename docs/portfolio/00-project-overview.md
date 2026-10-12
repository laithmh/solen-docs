# Solen — Project Overview & Core Concept

> **Solen** (سولين) is Iraq’s pioneering B2B2C Print-on-Demand (POD) and creator commerce platform, engineered to transform Iraq's rapidly expanding social media commerce into a scalable, automated merchandise ecosystem.

---

## 1. Executive Summary & The Core Idea

In emerging markets like Iraq, thousands of content creators, digital artists, calligraphers, gaming streamers, and boutique brands sell merchandise primarily through Instagram and TikTok DMs. This manual model suffers from severe operational bottlenecks:
- Creators must front capital to buy bulk inventory upfront in uncertain sizes and colors.
- Order management is handled manually via WhatsApp/Instagram chats.
- Order fulfillment relies on informal, uncalibrated local print shops with high defect rates.
- Cash on Delivery (COD) return rates are rampant due to nonexistent order verification and customer friction.
- Payment rails are fragmented between cash and emerging mobile wallets (ZainCash, FIB, Qi Card).

**Solen eliminates all friction by introducing a turnkey Print-on-Demand platform built specifically for Iraq's market realities:**
1. **For Creators (B2B)**: Zero upfront capital, zero minimum order quantities (MOQ), and zero inventory risk. A creator uploads high-resolution artwork, designs apparel in an in-browser 2D studio with realistic garment compositing, sets their own profit margin in Iraqi Dinars (`IQD`), and instantly launches a trilingual branded storefront with a shareable URL and QR code.
2. **For Shoppers (B2C)**: A frictionless localized shopping experience supporting all 18 Iraqi governorates, real-time shipping calculation, automated phone normalization (+964), Cash on Delivery (COD), order tracking via order number and phone verification, and trilingual localization (Arabic, Kurdish Sorani, and English).
3. **For Print Hubs & Platform Operations**: Automated server-side generation of 300 DPI transparent DTF (Direct-to-Film) print-ready files, automated piece-code generation for garment routing, an industrial Production Hub for print stations, and a T+3 post-delivery automated earnings clearance system.

---

## 2. Market Context: Why Iraq?

| Market Dynamic | The Challenge in Iraq | How Solen Solves It |
|---|---|---|
| **Dominant Payment Method** | Cash on Delivery (COD) accounts for >85% of Iraqi e-commerce transactions. Traditional SaaS platforms assume Stripe/Credit Cards. | Solen treats COD as a first-class citizen with mandatory Iraqi phone validation (`+964 7X`), rate-limiting to prevent spam orders, and merchant/admin order confirmation workflows before physical printing. |
| **Logistics & Geography** | Iraq comprises 18 governorates with varying delivery fees, informal street addresses, and reliance on landmarks rather than postal codes. | Solen incorporates an 18-governorate shipping engine with automated rate calculation (e.g., 5,000 IQD for Baghdad, 7,000+ IQD for outer provinces) and required landmark/district fields. |
| **Linguistic Diversity** | Iraq is officially bilingual (Arabic & Kurdish) with significant English digital usage among tech-savvy youth and diaspora. | Built with 100% trilingual key parity (2,542 keys across Arabic, Kurdish Sorani, and English), automatic RTL layout switching, and western Arabic digits standard. |
| **Apparel Manufacturing** | High-end automated POD infrastructure (like Printful or Printify) does not service Iraq locally due to customs barriers, shipping delays (3–4 weeks), and astronomical shipping fees. | Solen connects creators to domestic Baghdad and Erbil DTF printing hubs with 24–72 hour local fulfillment times. |

---

## 3. Target Personas & Value Proposition

### 1. The Digital Creator / Visual Artist (B2B)
- **Profile**: Artists, illustrators, cultural icons, influencers, and gaming clans with active followings on Instagram, TikTok, or YouTube.
- **Pain Point**: Wants to monetize their brand through custom apparel but cannot afford 500,000–2,000,000 IQD to print inventory in advance, nor do they have time to pack boxes and call couriers.
- **Solen Value**:
  - Drag-and-drop studio with real-time DPI resolution feedback (flags low-resolution graphics before printing).
  - Multi-placement support (Front, Back, Sleeves, Hood) with custom garment colorways.
  - Branded storefront (`solen.iq/store/[slug]`) customizable with bio, banner, Instagram/TikTok links.
  - Real-time earnings ledger tracking pending sales and withdrawable balance in IQD.
  - Transparent payout requests via ZainCash mobile wallet, FIB IBAN, or Qi Card.

### 2. The Iraqi Shopper (B2C)
- **Profile**: Young, mobile-first consumer discovering creator merchandise through social media links.
- **Pain Point**: Hesitant to pay upfront online; fears receiving poor quality fabric or designs that fade after one wash; frustrated by complicated checkout forms.
- **Solen Value**:
  - Lightning-fast guest checkout (no mandatory account creation).
  - Realistic garment preview displaying how the graphic looks on physical fabric.
  - Familiar COD payment option with order total calculated in clean integer IQD.
  - Self-serve order tracking portal (`/track`) by order number + mobile number without requiring login.
  - Automated WhatsApp support links to chat directly with merchants.

### 3. The Print Shop Operator (Fulfillment Partner)
- **Profile**: High-volume DTF printing facilities in Baghdad and Erbil operating industrial roll-to-roll printers and heat presses.
- **Pain Point**: Receives low-resolution JPEGs, wrong aspect ratios, and ambiguous placement instructions via Telegram, resulting in misprints and wasted blanks.
- **Solen Value**:
  - Dedicated **Production Hub** (`/dashboard/production`) filtered by status (`processing`, `printing`, `packaging`).
  - Single-click download of pristine 300 DPI transparent PNG-24 print files calculated at exact physical centimeters (`print_width_cm` × `print_height_cm`).
  - Printable **Packing Slips** with barcode/QR code and garment piece tracking codes.
  - Batch courier handover workflows with manifest tracking.

### 4. Platform Administrators
- **Profile**: Solen operations team overseeing catalog quality, marketplace safety, and financial settlements.
- **Solen Value**:
  - Design review & curation dashboard with bulk approval/rejection.
  - Immutable audit logs (`admin_audit_logs`) tracking every administrative action.
  - Governorates shipping rate manager and global exchange rate configuration.
  - Print shop remittance & financial settlement ledger (Zero-Float arithmetic).

---

## 4. Business & Monetization Model (Fee Model C)

Solen operates on a transparent, risk-free marketplace commission model with zero upfront subscription fees:

```
[Retail Price IQD] = [Base Garment Cost] + [Creator Markup] + [Platform Service Fee]
```

1. **Base Garment Cost**: Covers high-grade blank apparel (e.g., 280 GSM heavy cotton t-shirt or 360 GSM fleece hoodie) + DTF printing consumable costs. Retained by the fulfillment partner or platform.
2. **Creator Markup**: Defined entirely by the creator in the Studio (e.g., 8,000 IQD). 100% of this markup belongs to the creator upon verified order delivery.
3. **Platform Service Fee**: Dynamically calculated via database RPC `calculate_platform_fee(markup)` (combining a fixed platform fee of 3,000 IQD and a 15% platform commission on the markup).
4. **T+3 Delivery Clearance**: To eliminate print and return risk under COD, creator earnings remain in `pending` hold until the package is marked `delivered` and COD cash is collected, transitioning to `cleared` after a 72-hour safety buffer.

---

## 5. Current MVP Scope & Shipped Status

Solen is currently in active MVP deployment with all core loops operational and verified by 445 automated tests:

### Shipped & Operational in MVP:
- [x] Full Trilingual Localization (Arabic, Kurdish Sorani, English) with automatic RTL/LTR handling.
- [x] Interactive 2D Creator Studio with live drag-and-drop transform, contain-fit math, and DPI quality gauge.
- [x] Client-side 2D garment compositing preview engine (canvas multiply blend, dynamic tinting, fabric overlays).
- [x] Server-side Sharp-powered 300 DPI transparent PNG-24 DTF print generator with centimeter physical calibration.
- [x] Branded creator storefronts with custom banners, logos, social links, and public product catalogs.
- [x] Guest checkout flow optimized for Iraq (18 governorates, COD, phone normalization, promotional vouchers).
- [x] Immutable design snapshot guarantee (`order_items.design_snapshot` protected by database trigger).
- [x] Rigid order state machine (`pending` → `processing` → `printing` → `packaging` → `shipped` → `delivered`).
- [x] Self-serve order tracking portal (`/track`) with hybrid phone verification.
- [x] Creator dashboard with overview KPIs, product catalog manager, order ledger, and payout requests.
- [x] Production Hub for print shop operators with job inspection, printable packing slips, and courier handover.
- [x] Admin console with design curation, store management, mockup calibration, user role controls, and immutable audit logs.
- [x] Money flow engine with T+3 delivery hold, zero-float integer arithmetic, and print shop settlements.
- [x] Security suite with shared Postgres rate limiters, magic-byte image validation, open-redirect sanitizers, and RLS.

### Roadmap (Post-MVP):
- [ ] Direct automated API gateway integration for Iraqi mobile wallets (ZainCash, FIB, Qi Card) replacing deferred transfer.
- [ ] Direct Courier API webhook synchronization (Al-Zajel, National Express, Aramex Iraq) for real-time tracking pings.
- [ ] Real-time browser-based WebGL/Three.js 3D fabric displacement shader engine.
- [ ] Native mobile apps for creators (iOS & Android) via React Native or Flutter.
- [ ] Automated SMS / WhatsApp notification webhooks for order confirmations.
