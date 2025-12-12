ChaufHER API — Postman contract tests (safety-event)

This folder contains a minimal Postman collection to validate the safety-event contract between mobile clients and the API.

Files:

- `safety-event-collection.json` — Postman collection with two requests: create safety event, get event status.

Quick run with Newman:

1. Install Newman (if not installed):

```bash
npm install -g newman
```

2. Run the collection (provide runtime variables):

```bash
newman run safety-event-collection.json --env-var "baseUrl=http://localhost:5000" --env-var "token=YOUR_JWT" --env-var "idempotencyKey=$(uuidgen)"
```

Notes:

- Replace `baseUrl` and `token` with your local API URL and a valid bearer token.
- The collection is intentionally minimal; expand it with negative tests (429, 401, 422) and idempotency checks.
- Consider automating this in CI using `newman` and a staging environment.
