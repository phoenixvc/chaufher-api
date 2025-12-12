# Compatibility Checklist — chaufher-api ↔ chaufher-app

This checklist summarizes the key integration points the mobile app depends on. Keep this file updated when changing API endpoints, auth flows, or hub events.

Endpoints

- POST `/api/safety-events` — accepts safety event payload (see OpenAPI partial)
- GET `/api/safety-events/{eventId}` — query status of safety event
- GET `/api/rides/{rideId}/status` — lightweight ride status

Auth

- OAuth2 / JWT bearer tokens for REST APIs
- SignalR hubs require the same bearer token during negotiation (Authorization header or access_token query param)

SignalR / Real-time events

- Hub: `/hubs/rideevents`
- Events the app consumes:
  - `ride.updated` — payload: { rideId, status, driverLocation?, eta? }
  - `safety.event.update` — payload: { eventId, status, message }

Idempotency & Retry

- Clients SHOULD send `Idempotency-Key` header or `idempotencyId` in payload for safety events.
- Server returns `409` for duplicate idempotency keys with existing event info.
- Server may return `202` for queued processing; clients should poll event status.

Docs & Tests

- OpenAPI partial: `docs/openapi/safety-and-ride-partial.yaml`
- Postman/Newman collection: `docs/postman/safety-event-collection.json`
- Postman run instructions: `docs/postman/README.md`

Notes

- Update this checklist whenever the OpenAPI partial changes.
- Consider adding machine-checked contract tests (Pact/Postman) to CI.
