---
name: Monitor Harri platform events and webhook delivery
description: Poll the Harri events log for platform activity, filter by event code, location and published window, check delivery status, and list the webhook subscriptions configured for your client.
api: openapi/harri-employee-openapi.yml
operations: [v2_events_list, GetEventById, ListClientSubscriptions, ListFranchiseeEvents, franchisees_v2_events_list, FranchiseesGetEventById, ListFranchiseeSubscriptions]
---

# Monitor Harri platform events and webhook delivery

Harri emits webhook event notifications and also keeps a queryable event log on the REST API. Use the
log to reconcile anything the webhook receiver may have missed, and to audit delivery.

Auth, base URL and rate limits: see `harri-onboard-new-hire.md`.

## Poll the event log

`v2_events_list` (`GET /api/v2/events`) accepts:

- `event_code` — the event type. The catalog of valid codes is not published publicly; read the codes off
  your own event history rather than assuming them.
- `location_id` — narrow to one location.
- `status` — `SUCCESS`, `FAILED` or `IN_PROGRESS`.
- `published_gte` / `published_lte` — ISO 8601 date-times bounding the publish window.
- `limit` (default 10) and `page` (default 1).

Use `GetEventById` (`GET /api/v1/events/{eventId}`) to pull one event in full.

Do **not** call `GetEvents` at `/api/v1/events` — it is flagged deprecated. Use the v2 list.

## Reconcile failed deliveries

Poll with `status=FAILED` over the window since your last successful reconciliation, then re-fetch the
underlying record with the relevant employee or location operation. Harri publishes no replay or
re-delivery operation, so reconciliation is a pull, not a retry.

## Check your subscriptions

`ListClientSubscriptions` (`GET /api/v1/subscriptions`) returns the subscriptions for the authenticated
client. Franchise operators use `ListFranchiseeSubscriptions`
(`GET /api/v1/franchisees/{franchiseeId}/subscriptions`).

Subscriptions are **read-only over the API** — there is no create, update or delete operation in the
published spec. Webhook endpoints are provisioned out of band with Harri, so treat this operation as an
audit check, not a management surface.

## Franchisee variants

`franchisees_v2_events_list` (`GET /api/v2/franchisees/{franchiseeId}/events`),
`ListFranchiseeEvents` (`GET /api/v1/franchisees/{franchiseeId}/events`) and
`FranchiseesGetEventById` mirror the corporate operations.

## Operating notes

- A polling loop is the easiest way to burn the 400 requests/minute budget. Widen the window and raise
  `limit` rather than shortening the interval.
- Harri publishes no AsyncAPI document, so there is no machine-readable message schema for the webhook
  payloads. See `asyncapi/harri-webhooks.yml` for what is and is not published.
