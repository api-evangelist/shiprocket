---
name: shiprocket-track-and-handle-ndr
description: Track a Shiprocket shipment by AWB, shipment id or your order id, read the status codes, and act on a non-delivery (NDR) with a re-attempt or return-to-origin.
api: shiprocket:shiprocket-api
provider: shiprocket
generated: '2026-09-18'
method: generated
source: openapi/shiprocket-api-openapi.yml + asyncapi/shiprocket-tracking-webhooks.yml
operations:
  - generateToken
  - getTrackingThroughAWB
  - getTrackingThroughShipmentID
  - getTrackingDataThroughOrderID
  - getTrackingDataForMultipleAWBS
  - getAllNDRShipments
  - getSpecificNDRShipmentDetails
  - actionNDR
---

# Track a shipment and handle NDR

1. **Authenticate** — `generateToken` (`POST /auth/login`); 10-day JWT.
2. **Track** by whichever identifier you hold:
   - AWB: `getTrackingThroughAWB` (`GET /courier/track/awb/{awb_code}`)
   - Shiprocket shipment id: `getTrackingThroughShipmentID`
     (`GET /courier/track/shipment/{shipment_id}`)
   - Your own order id: `getTrackingDataThroughOrderID`
     (`GET /courier/track?order_id=&channel_id=`) — pass `channel_id` when the
     same order id exists on more than one channel
   - Batches: `getTrackingDataForMultipleAWBS` (`POST /courier/track/awbs`)
   Read `tracking_data.shipment_status` and `shipment_track[]`; the numeric
   status codes (6 Shipped, 7 Delivered, 17 Out For Delivery, 18 In Transit,
   21 Undelivered, 9/10 RTO ...) are listed in the Tracking tag description of
   the OpenAPI.
3. **Prefer the webhook for updates.** Shiprocket POSTs the same tracking
   payload to a seller-configured URL on every new scan (Settings > API >
   Webhooks; optional `x-api-key` token). Poll only as a fallback.
4. **List undelivered shipments** — `getAllNDRShipments` (`GET /ndr/all`) or
   one AWB with `getSpecificNDRShipmentDetails` (`GET /ndr/{AWB}`).
5. **Act on the NDR** — `actionNDR` (`POST /ndr/{awb}/action`) with
   `action` = `re-attempt` or `return` and `comments`. A 202
   `{"status":"Data Updated Sucessfully"}` acknowledges it.

## Rules

- `actionNDR` has no undo; a `return` starts RTO. Confirm with the seller first.
- Tracking calls are read-only and safe to retry.
- 404 on a tracking call means the AWB/shipment id is unknown, not that the
  shipment is lost — check the id you passed.
