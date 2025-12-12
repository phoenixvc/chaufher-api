# ChaufHER API — Technical Design

This repository describes the `chaufher-api` service: a .NET 9 backend that provides the business logic, REST APIs, and real-time messaging for the ChaufHER platform.

Document status: Final. Last updated: 2025-12-12.

## Table of contents

- [Overview](#overview)
- [Quick start](#quick-start)
- [Architecture & design](#architecture--design)
- [Core flows](#core-flows)
- [Multi-repo architecture](#multi-repo-architecture)
- [Client integration](#client-integration)
- [System interfaces](#system-interfaces)
- [Testing & observability](#testing--observability)
- [Test fixtures & seed data](#test-fixtures--seed-data)
- [Deployment](#deployment)
- [Related repositories](#related-repositories)
- [Backward compatibility & versioning](#backward-compatibility--versioning)
- [Development guidelines](#development-guidelines)
- [Security guidelines](#security-guidelines)
- [Troubleshooting](#troubleshooting)
- [License](#license)
- [Contributing](#contributing)

## Overview

`chaufher-api` is a stateless service (recommended) implementing a hexagonal architecture. It coordinates ride booking and lifecycle, driver state, payments, and notifications; it exposes REST endpoints and SignalR hubs for real-time updates.

Goals:

- High availability and horizontal scalability
- Clear separation of domain logic, application orchestration, and infrastructure adapters
- Testability via ports/adapters and automated tests

## Quick start

Prerequisites:

- .NET 9 SDK
- PostgreSQL (or Testcontainers/ephemeral DB for local runs)

Common quick steps (example):

1. Build: `dotnet build`
2. Run: `dotnet run --project src/Chaufher.Api`
3. Open Swagger: `http://localhost:{port}/swagger`

Adjust connection strings and secrets via environment variables or Azure Key Vault in production.

**Environment Variables Reference:**

Required:

- `ASPNETCORE_ENVIRONMENT` — `Development`, `Staging`, `Production`
- `ConnectionStrings__DefaultConnection` — PostgreSQL connection string
- `AzureKeyVault__VaultUrl` — Azure Key Vault endpoint
- `Auth__JwtSigningKey` — Secret key for JWT signing (fetch from Key Vault in prod)
- `SignalR__ConnectionString` — Azure SignalR Service endpoint (if using managed service)

Optional:

- `ASPNETCORE_URLS` — Server URL (default: `http://localhost:5000`)
- `Logging__LogLevel` — Min log level (`Debug`, `Information`, `Warning`, `Error`)
- `ApplicationInsights__InstrumentationKey` — Azure App Insights key for telemetry

Full list: see `.env.example` in the repo.

**IDE Configuration:**

VS Code:

1. Install extensions: C#, REST Client, Azure Tools
2. Open workspace: `code .`
3. Create `.vscode/launch.json` for debugging (template in repo)
4. Use `F5` to debug

Visual Studio:

1. Open `chaufher-api.sln`
2. Right-click solution → Manage Secrets (stores env vars locally)
3. F5 to debug

**API Documentation:**

Swagger/OpenAPI specs available at:

- Dev: `http://localhost:5000/swagger`
- Staging: `https://api-staging.chaufher.azure.com/swagger`
- Production: `https://api.chaufher.azure.com/swagger`

## Architecture & design

Layering:

- API Layer — controllers, SignalR hubs, health checks
- Application Layer — use-cases, command/query orchestration
- Domain Layer — entities, aggregates, domain services
- Infrastructure Layer — EF Core/Postgres, external adapters (payments, notifications)

Patterns:

- Dependency Injection (built-in .NET DI)
- CQRS where read/write separation improves scalability
- Event-driven state transitions for ride lifecycle and notifications

Principles:

- Keep domain pure and side-effect free
- Isolate external integrations behind adapter interfaces
- Prefer small, testable units and composition over monolithic services

## Core flows

- Matching: geospatial queries + scoring (proximity, availability, rating)
- Pricing: distance/time + surge multipliers
- Ride lifecycle: new → accepted → en route → in progress → completed/canceled (events propagated to clients)
- Payments: authorization at booking, final capture on completion (adapter pattern)

## Multi-repo architecture

The ChaufHER platform is a coordinated multi-repository ecosystem:

```
                    chaufher-workspace (entry point, shared docs)
                            |
        ┌───────────────────┼───────────────────┐
        |                   |                   |
   chaufher-app       chaufher-web         chaufher-api
 (Flutter mobile)     (React admin)       (.NET backend)
        |                   |                   |
        └───────────────────┼───────────────────┘
                            |
                     chaufher-infra
                  (Azure IaC, CI/CD)
```

- **chaufher-workspace:** Monorepo coordination, shared documentation, local development setup.
- **chaufher-api:** Core backend (this repo); provides REST APIs, SignalR hubs, business logic.
- **chaufher-app:** Flutter mobile client (riders, drivers); consumes API.
- **chaufher-web:** React admin portal; real-time ride oversight, incident escalation, compliance reporting.
- **chaufher-infra:** Azure infrastructure as code (Bicep/Terraform), CI/CD pipelines, deployment automation.

All repositories are version-controlled independently but designed for seamless integration via well-defined APIs and contracts.

## Client integration

Both `chaufher-app` and `chaufher-web` depend on stable, well-documented API contracts. This section outlines what clients expect from the backend.

### Authentication & Authorization

- **REST APIs:** JWT bearer token (OAuth2 / Azure AD B2C); clients pass `Authorization: Bearer {token}` header.
- **SignalR Hubs:** Same JWT auth; clients negotiate using bearer token in query param or Authorization header.
- **Key Vault Integration:** Signing keys, secrets, and API keys are stored in Azure Key Vault per environment; clients fetch via secure managed identity.

### Expected Endpoints (clients depend on these)

- `POST /api/safety-events` — trigger panic; see [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)
- `GET /api/safety-events/{eventId}` — query safety event status
- `GET /api/rides/{rideId}/status` — lightweight ride status query
- `GET /api/rides/{rideId}` — full ride details
- `POST /api/rides` — create a new ride request
- `GET /api/users/{userId}` — user profile
- `GET /api/drivers/{driverId}` — driver profile

Full spec: [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)

### Real-Time Events (SignalR)

Clients subscribe to the `/hubs/rideevents` hub and listen for:

- **`ride.updated`** — fired when ride state changes (accepted, en-route, completed, canceled)
  - Payload: `{ rideId, status, driverLocation?, eta?, fare? }`
- **`safety.event.update`** — fired when a safety event escalates or resolves
  - Payload: `{ eventId, status, message, escalationLevel?, assignedOps? }`

Full schema definitions: [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)

### Idempotency & Retry Semantics

- Clients SHOULD send `Idempotency-Key` header (UUID) on safety-event POST to deduplicate retries.
- Server returns `409` if duplicate key matches an existing event (safe to retry).
- Server may return `202 Accepted` for async processing; clients should poll event status.
- For offline scenarios, clients cache events locally and retry when network resumes.

### Client Checklist

Before integrating with the API, ensure:

- [ ] Auth tokens are obtained securely (Azure AD / OAuth2 flow).
- [ ] JWT tokens are refreshed before expiry; rejected 401s trigger re-login.
- [ ] All API calls include proper error handling and exponential backoff for retries.
- [ ] SignalR connection is established and listeners for `ride.updated` and `safety.event.update` are active.
- [ ] Panic/safety events are sent with an `Idempotency-Key`; local caching supports offline dispatch.
- [ ] Secrets (API keys, signing keys) are fetched from Key Vault, never hardcoded.

See [docs/compatibility/compatibility-checklist.md](docs/compatibility/compatibility-checklist.md) for a detailed checklist.

## System interfaces

REST endpoints (examples):

- `/api/rides`
- `/api/users`
- `/api/drivers`
- `/api/payments`
- `/api/notifications`

SignalR hubs (example):

- `/hubs/rideevents` — authenticated hub for real-time ride updates

Health & readiness probes (Kubernetes/Azure):

- `GET /healthz` — liveness probe (is the API alive?)
- `GET /ready` — readiness probe (can it handle traffic?)

Security:

- OAuth2 / JWT bearer tokens for API and hub authentication

Third-party integrations:

- Payment gateways (PCI-compliant adapters)
- Notification providers (push, SMS, email)

### Error Handling & Response Conventions

All error responses follow this standard format:

```json
{
  "code": "INVALID_REQUEST",
  "message": "Human-readable error message",
  "details": { "field": "error details" },
  "traceId": "correlation-id-for-logs"
}
```

HTTP Status Codes:

- `200 OK` — Success
- `201 Created` — Resource created
- `202 Accepted` — Request accepted, async processing (check `retryAfter`)
- `400 Bad Request` — Validation error
- `401 Unauthorized` — Missing or invalid auth token
- `403 Forbidden` — Authenticated but not authorized
- `404 Not Found` — Resource doesn't exist
- `409 Conflict` — Duplicate (e.g., idempotency key match)
- `429 Too Many Requests` — Rate limit exceeded
- `500 Internal Server Error` — Server fault

### Logging Conventions

Structured logging via Serilog or Microsoft.Extensions.Logging:

```csharp
logger.LogInformation(
  "Ride created: {RideId}, Rider: {RiderId}, Correlation: {CorrelationId}",
  rideId, riderId, correlationId
);
```

All logs include:

- Timestamp (ISO 8601 UTC)
- Log Level (Information, Warning, Error, Critical)
- Correlation ID (trace across app/infra layers)
- Structured properties (JSON serialized)

Logs are exported to:

- Azure Log Analytics (central aggregation)
- Application Insights (performance + errors)
- Local console (dev)

Compatibility & contract docs:

- OpenAPI partial (safety & ride): [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)
- Postman contract tests: [docs/postman/README.md](docs/postman/README.md)
- Compatibility checklist: [docs/compatibility/compatibility-checklist.md](docs/compatibility/compatibility-checklist.md)

## Test fixtures & seed data

For integration testing and local development, the API provides mock data and seed scripts:

**Test Data Setup:**

- `scripts/seed-dev.sql` — populates dev environment with synthetic users, drivers, rides, payments.
- `tests/fixtures/` — reusable test data payloads (JSON files for requests/responses).
- Testcontainers.NET — ephemeral PostgreSQL for CI integration tests.
- EF Core in-memory provider — fast unit test database.

**Mock Data Patterns:**

```csharp
// Example: fixture for safety-event POST
var safetyEventFixture = new {
  tripId = "trip_test_001",
  userId = "user_test_alice",
  timestamp = DateTime.UtcNow,
  location = new { lat = 47.6205, lon = -122.3493, accuracy = 5 },
  deviceInfo = new { platform = "android", appVersion = "1.2.3" }
};
```

**Integration Test Example:**

Tests in `tests/Integration/` use Testcontainers to spin up a real PostgreSQL instance, seed test data, and validate API behavior end-to-end. See [tests/README.md](tests/README.md) for full setup.

## Testing & observability

Testing strategy:

- Unit: xUnit + mocking (Moq / NSubstitute)
- Integration: Testcontainers or ephemeral PostgreSQL for EF Core tests
- Contract/API: OpenAPI/Swagger-driven checks
- E2E: scenarios for core user journeys

Observability:

- Structured logging (Serilog or Microsoft.Extensions.Logging)
- Metrics and traces (OpenTelemetry → Application Insights / Prometheus)
- Dashboards + alerts in Azure Monitor / Grafana

## Deployment

Recommended pipeline steps:

1. CI build + tests (GitHub Actions)
2. Build container image (if containerized) and push to registry
3. Deploy to staging (App Service / Container Apps / AKS)
4. Smoke tests + health checks
5. Promote to production (canary / slot swap)

Secrets: Azure Key Vault. DB: Azure Database for PostgreSQL (managed).

## Related repositories

ChaufHER API is part of a multi-repository workspace. Refer to the appropriate repo for integration and development guidance:

- **[chaufher-workspace](https://github.com/phoenixvc/chaufher-workspace)** — Monorepo entry point, shared documentation, and local development setup. Start here.
- **[chaufher-app](https://github.com/phoenixvc/chaufher-app)** — Flutter mobile client (iOS/Android). Consumes the API via REST and SignalR.
- **[chaufher-web](https://github.com/phoenixvc/chaufher-web)** — React admin portal. Real-time ride oversight, incident escalation, compliance reporting.
- **[chaufher-infra](https://github.com/phoenixvc/chaufher-infra)** — Azure infrastructure as code (Bicep/Terraform), CI/CD pipelines, and deployment automation.

For cross-repo questions or coordination, start with chaufher-workspace.

## Backward Compatibility & Versioning

**API Versioning:**

- Current: v1 (stable, no breaking changes without notice)
- Versioning: Semantic versioning (MAJOR.MINOR.PATCH)
- Breaking changes: Require new major version (v2, v3, etc.)

**Deprecation Policy:**

- Deprecated endpoints/fields are marked with `Deprecated: true` in OpenAPI.
- Minimum 3 months notice before removal (published in release notes).
- Example: field removed in v2.0.0 → deprecated in v1.2.0 (released 3+ months prior).

**Migration Guide:**

When upgrading API versions:

1. Check [CHANGELOG.md](CHANGELOG.md) for breaking changes.
2. Update client endpoints (e.g., `/api/v2/rides` if applicable).
3. Test in staging before production.
4. See related repos (chaufher-app, chaufher-web) for client-side migrations.

## Development Guidelines

### Naming Conventions

- **Endpoints:** kebab-case with resource nouns: `/api/safety-events`, `/api/rides/{rideId}/status`
- **Controllers:** PascalCase, plural resource name: `RidesController`, `SafetyEventsController`
- **Methods:** PascalCase: `CreateRide()`, `GetSafetyEventStatus()`
- **Variables:** camelCase: `rideId`, `driverLocation`
- **Constants:** UPPER_SNAKE_CASE: `MAX_RETRY_ATTEMPTS`, `DEFAULT_TIMEOUT_MS`
- **Database columns:** snake_case: `ride_id`, `created_at`

### Commit Conventions

Use conventional commits:

```
<type>(<scope>): <subject>

<body>

<footer>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `style`

Examples:

```
feat(safety): add panic event escalation to ops
fix(auth): resolve JWT token refresh race condition
docs(api): update OpenAPI partial with new endpoints
```

### Git Workflow

**Branching Strategy:**

- `main` — production-ready
- `feature/your-feature` — features
- `fix/issue-description` — bugfixes
- `chore/task` — non-code changes

**Pull Request Flow:**

1. Create feature branch from `main`
2. Push and open PR with description
3. All checks must pass (tests, linters, code review)
4. Squash and merge on approval
5. Delete branch after merge

### Code Quality Standards

- **Linting:** `dotnet format` (enforced in CI)
- **Analyzers:** StyleCop, SonarAnalyzer (strict mode)
- **Test Coverage:** Minimum 80% for critical business logic
- **Complexity:** McCabe complexity < 10 per method
- **Async:** Prefer async/await for I/O-bound operations

## Security Guidelines

**Secrets Management:**

- Never commit secrets to git (keys, tokens, passwords)
- Store all secrets in Azure Key Vault per environment
- Use managed identities for service-to-service auth
- Rotate credentials every 90 days (automated)

**What NOT to commit:**

- `.env` files with sensitive values
- API keys, JWT signing keys
- Database passwords
- Private certificates

**Code Patterns:**

- Use parameterized queries to prevent SQL injection
- Validate and sanitize all user inputs
- Use HTTPS/TLS for all external comms
- Apply OWASP Top 10 mitigations
- Log security events (auth failures, rate limits, admin actions)
- Never log passwords, tokens, or PII

## Troubleshooting

**Common Setup Issues**

*Port 5000 already in use*

```bash
# Find process
lsof -i :5000

# Kill or use different port
dotnet run --urls "http://localhost:5001"
```

*PostgreSQL connection fails*

```bash
# Verify connection string format
# Should be: Server=localhost;Database=chaufher_dev;User Id=postgres;Password=...;

# Test connection
psql -h localhost -U postgres -d chaufher_dev
```

*EF Core migration fails*

```bash
# Ensure DB exists and is accessible
dotnet ef database update

# If stuck, reset (dev only!)
dotnet ef database drop --force
dotnet ef database update
```

*SignalR connection timeout*

```bash
# Check Azure SignalR Service is running (staging/prod)
# Dev: ensure /hubs/rideevents endpoint is registered
# Client: verify correct hub URL and auth token
```

*Tests fail locally but pass in CI*

```bash
# Clear build artifacts
rm -rf bin obj
dotnet clean
dotnet build
dotnet test

# Check env vars match CI
echo $ASPNETCORE_ENVIRONMENT
```

**Getting Help**

- **Documentation:** See [chaufher-workspace](https://github.com/phoenixvc/chaufher-workspace) for cross-repo docs
- **Issues:** Open a GitHub issue with:
  - Error message and stack trace
  - Steps to reproduce
  - Environment (OS, .NET version)
  - Attempted fixes
- **Slack:** #engineering-backend (for quick questions)
- **Escalation:** Contact the platform lead for critical blockers

## License

[Add License Here — e.g., MIT, Apache 2.0, or proprietary]

Copyright (c) 2025 ChaufHER. All rights reserved.

## Contributing

- Keep domain logic in the `Domain` layer and avoid direct infra calls from it
- Add unit tests for new business rules and integration tests for persistence or external adapters
- Follow the existing CI checks and branch protection rules

This README was reorganized for clarity and quicker onboarding. See the repo for implementation details and the code-level docs.

## Docs & Contract Tests

- OpenAPI partial (safety & ride): [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)
- Postman contract tests and instructions: [docs/postman/README.md](docs/postman/README.md)
- Compatibility checklist: [docs/compatibility/compatibility-checklist.md](docs/compatibility/compatibility-checklist.md)

Key integration notes:

- Idempotency: clients SHOULD send an `Idempotency-Key` header or `idempotencyId` in the payload for safety events; server returns `409` when a duplicate key maps to an existing event.
- Async semantics: server may respond `202 Accepted` for queued safety events; clients should poll `/api/safety-events/{eventId}` for final state.
- SignalR events: the app listens to `/hubs/rideevents` and consumes `ride.updated` and `safety.event.update` messages (see compatibility checklist).

CI (Newman) example

We include a sample GitHub Actions workflow to run the Postman collection against a test/staging API. Secrets expected: `TEST_BASE_URL`, `TEST_BEARER_TOKEN`.

Workflow path: `.github/workflows/newman.yml` (example included in repo).

Local run (Newman):

```bash
# install newman
npm install -g newman

# run (replace values)
newman run docs/postman/safety-event-collection.json --env-var "baseUrl=http://localhost:5000" --env-var "token=YOUR_JWT" --env-var "idempotencyKey=$(uuidgen)"
```

Recommended next steps:

- Expand the OpenAPI partial to include SignalR event schemas and more examples.
- Add negative/idempotency tests to the Postman collection and run them in CI.
- Convert the Postman contract to a Pact or contract-testing framework if you want stronger consumer-driven testing.
