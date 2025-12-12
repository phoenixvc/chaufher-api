# ChaufHER API — Technical Design

This repository describes the `chaufher-api` service: a .NET 9 backend that provides the business logic, REST APIs, and real-time messaging for the ChaufHER platform.

Document status: Final. Last updated: 2025-12-12.

## Table of contents

- Overview
- Quick start
- Architecture & design
- Core flows
- System interfaces
- Testing & observability
- Deployment
- Contributing

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

## System interfaces

REST endpoints (examples):

- `/api/rides`
- `/api/users`
- `/api/drivers`
- `/api/payments`
- `/api/notifications`

SignalR hubs (example):

- `/hubs/rideevents` — authenticated hub for real-time ride updates

Security:

- OAuth2 / JWT bearer tokens for API and hub authentication

Third-party integrations:

- Payment gateways (PCI-compliant adapters)
- Notification providers (push, SMS, email)

Compatibility & contract docs:

- OpenAPI partial (safety & ride): [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)
- Postman contract tests: [docs/postman/README.md](docs/postman/README.md)
- Compatibility checklist: [docs/compatibility/compatibility-checklist.md](docs/compatibility/compatibility-checklist.md)

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
