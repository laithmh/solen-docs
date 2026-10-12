# Observability & Alerts

The highest costs in Print-on-Demand stem from silent failures: sending a corrupted or low-resolution file to a physical printer, or an unfulfilled paid order. Observability is focused on preventing physical and financial waste.

> [!NOTE]
> Master guidelines: [`AGENTS.md`](file:///d:/flutter_projects/solen_web/AGENTS.md) and [`.agents/rules/`](file:///d:/flutter_projects/solen_web/.agents/rules/).

---

## 1. Error Tracking (Sentry)

- Sentry configured on both client and Next.js server runtime (`SENTRY_DSN`).
- Every catch block in `/lib/render`, `/lib/print`, and `/lib/payment` must capture exceptions with structured metadata:
  ```typescript
  Sentry.captureException(error, {
    extra: {
      placementId,
      userDesignId,
      mode, // 'preview' | 'print'
      governorate,
      paymentMethod,
    },
  });
  ```

---

## 2. Business-Critical Alerts (`ALERT_WEBHOOK_URL`)

Immediate notifications dispatched to the operational webhook (Discord, Slack, or Telegram):

1. **Print File Missing on Paid/Confirmed Order**:
   - Fires if an order is confirmed but any line item lacks a valid `print_file_url` or has `is_print_ready: false`.
2. **Local DTF Fulfillment API Failure**:
   - Order submission to local Baghdad/Erbil printing hubs fails or times out.
3. **Iraqi Payment Gateway Webhook Failure**:
   - Invalid cryptographic signature or rejected payment payload from ZainCash or FIB.
4. **Headless Server Render Crash**:
   - Playwright headless render worker times out or fails to composite garment shader.
5. **Missing Storage Assets (404)**:
   - A displacement map, overlay, or base image referenced by an active mockup returns 404.

---

## 3. Iraqi Marketplace Metrics

- **COD Delivery Success Rate**: Ratio of successfully delivered COD parcels vs. returned/refused shipments by governorate.
- **Render Latency**: Execution duration of server-side 300 DPI PNG generation.
- **Studio Conversion Funnel**: Rate of creators uploading designs → previewing in 3D → publishing to storefront.
