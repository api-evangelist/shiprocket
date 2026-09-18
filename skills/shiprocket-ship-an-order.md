---
name: shiprocket-ship-an-order
description: Create an order in Shiprocket, pick a serviceable courier, assign the AWB, schedule the pickup and fetch the label — the core forward-shipping flow.
api: shiprocket:shiprocket-api
provider: shiprocket
generated: '2026-09-18'
method: generated
source: openapi/shiprocket-api-openapi.yml + conventions/shiprocket-conventions.yml + mcp/shiprocket-mcp.yml
operations:
  - generateToken
  - getAllPickupLocations
  - checkCourierServiceability
  - createCustomOrder
  - generateAWBForShipment
  - requestForShipmentPickup
  - generateLabel
  - generateManifest
---

# Ship an order with Shiprocket

Base URL `https://apiv2.shiprocket.in/v1/external`. Every call after step 1 sends
`Authorization: Bearer <token>`.

1. **Authenticate** — `generateToken` (`POST /auth/login`) with the API user's
   email + password (created in the panel under Settings > API). Cache the JWT;
   it is valid for 10 days. Re-login on a 401.
2. **Resolve the pickup location** — `getAllPickupLocations`
   (`GET /settings/company/pickup`). `createCustomOrder` needs the location's
   `pickup_location` nickname exactly as configured; it cannot be a new address.
3. **Rate-shop** — `checkCourierServiceability`
   (`GET /courier/serviceability?pickup_postcode=&delivery_postcode=&weight=&cod=`)
   and choose a `courier_company_id` from `data.available_courier_companies[]`
   (rate, etd, rating). Skip this to let Shiprocket's courier-priority rules pick.
4. **Create the order** — `createCustomOrder` (`POST /orders/create/adhoc`).
   Required per the provider's parameter table: `order_id` (your reference),
   `order_date`, `pickup_location`, billing name/address/city/pincode/state/
   country/email/phone, `shipping_is_billing`, `order_items[]`, `payment_method`
   (COD|Prepaid), `sub_total`, `length`, `breadth`, `height`, `weight`. The
   response carries the **Shiprocket** `order_id` and `shipment_id` — use those
   from here on, not your own reference.
5. **Assign the AWB** — `generateAWBForShipment` (`POST /courier/assign/awb`)
   with `shipment_id` and optional `courier_id`. Read `awb_code` from the response.
6. **Schedule pickup** — `requestForShipmentPickup`
   (`POST /courier/generate/pickup`) with `shipment_id[]` and optional
   `pickup_date`.
7. **Label + manifest** — `generateLabel` (`POST /courier/generate/label`,
   `shipment_id[]`) returns `label_url` (PDF); `generateManifest`
   (`POST /manifests/generate`, `shipment_id[]`) for the courier handover.

## Rules an agent must respect

- **No idempotency key exists.** Retrying step 4 after a timeout creates a
  second order; check `getAllOrders` (`search=<your order_id>`) before retrying.
- **No sandbox.** Every request with valid credentials changes live account data
  and can incur shipping charges.
- **Reversal** is `cancelAnOrder` (`POST /orders/cancel`, `ids[]`) before
  dispatch or `cancelAShipment` (`POST /orders/cancel/shipment/awbs`); the docs
  state no cut-off window, so confirm with the seller before cancelling.
- Errors are plain JSON `{message}`; a 422 carries a per-field `errors{}` map.
  A 429 means the undocumented rate limit was hit — back off, there is no
  `Retry-After`.
