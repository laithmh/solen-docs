# Security

> [!NOTE]
> Master guidelines: [`AGENTS.md`](file:///d:/flutter_projects/solen_web/AGENTS.md) and [`.agents/rules/`](file:///d:/flutter_projects/solen_web/.agents/rules/).

---

## 1. Row-Level Security (Supabase)

RLS is mandatory on every table with user-owned or role-restricted rows. No table ships without explicit policies:

- **`user_designs` & `user_design_placements`**: Readable and writable only by `creator_auth_id = auth.uid()` or users with the `admin` role.
- **`orders` & `order_items`**:
  - Customers can read their own orders (`orders.customer_id = auth.uid()`).
  - Designers can read line items corresponding to their products (`designer_id = auth.uid()`).
  - Only backend service-role or admin can update order fulfillment states.
- **`mockups` & `mockup_placements`**: Public read for storefront display; mutations restricted to `admin` role.
- **`designer_payout_methods` & `payouts`**: Designers can read and modify their own payment methods; payout status updates are strictly admin/service-role.
- **`profiles`**: Users can read and update their own display name; role elevation (`customer` → `designer` → `admin`) is restricted to server-side admin functions.

---

## 2. Iraqi Payment & Checkout Security

- **Server-Side Pricing**: Never trust prices submitted by client requests. The server recalculates:
  $$\text{Total IQD} = \sum (\text{Variant Base} + \text{Designer Markup}) \times \text{Qty} + \text{Governorate Shipping Fee} - \text{Validated Discount}$$
- **Cash on Delivery (COD) Fraud Mitigation**:
  - COD is vulnerable to fake orders. Enforce server-side regex validation on Iraqi phone numbers (`^(?:\+964|00964|0)?7[3-9]\d{8}$`).
  - Rate-limit checkout submissions per IP and phone number to prevent bot abuse.
  - Implement SMS/WhatsApp OTP verification on high-value or first-time COD orders.
- **Payment Webhook Verification**:
  - Webhooks from ZainCash and First Iraqi Bank (FIB) must verify cryptographic HMAC signatures before updating order payment status.

---

## 3. File Uploads & Asset Protection

- **MIME & Dimension Validation**: Server-side verification of image format (PNG, JPEG, SVG) and maximum file size (50MB) before accepting artwork.
- **EXIF Stripping**: Automatically remove GPS and camera metadata from user-uploaded images to protect creator privacy.
- **Storage Scoping**: User uploads are stored in private Supabase Storage buckets under `designs/{userId}/{fileId}`. Raw high-resolution assets are accessible only by the creator and the server-side render worker.
- **Rate Limiting**: Rate-limit upload endpoints to prevent Denial-of-Service via resource-intensive image processing.

---

## 4. Print File Integrity

- Print files (`print_file_url`) are generated server-side using `/lib/render/renderPlacement.ts` from stored source assets.
- Never allow a client to upload or submit a "final print file" directly to prevent tampering or injecting malicious payloads into DTF print queues.

---

## 5. Secrets Management

- `SUPABASE_SERVICE_ROLE_KEY`, `ZAINCASH_SECRET`, `FIB_CLIENT_SECRET`, and print provider API credentials must remain strictly server-side.
- Ensure no secret environment variable is ever prefixed with `NEXT_PUBLIC_`.
