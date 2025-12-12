# ChaufHER API — Technical Design

The backend service powering the ChaufHER ride-sharing platform. Built with .NET 9, providing REST APIs, real-time messaging, and business logic for riders, drivers, and operations.

**Document status:** Final | **Last updated:** 2025-12-12

---

## Table of Contents

### Business & Product
- [Overview](#overview)
- [Purpose & Value](#purpose--value)
- [Core Capabilities](#core-capabilities)
- [Multi-Repo Architecture](#multi-repo-architecture)

### Architecture & Design
- [Architecture Overview](#architecture-overview)
- [Design Philosophy](#design-philosophy)
- [Core Flows](#core-flows)
- [System Interfaces](#system-interfaces)

### Integration
- [Client Integration](#client-integration)
- [Related Repositories](#related-repositories)
- [Backward Compatibility](#backward-compatibility)

### Developer Guide
- [Quick Start](#quick-start)
- [Development Guidelines](#development-guidelines)
- [Testing & Observability](#testing--observability)
- [Deployment](#deployment)
- [Security Guidelines](#security-guidelines)
- [Troubleshooting](#troubleshooting)

### Reference
- [API Documentation](#api-documentation)
- [Contributing](#contributing)
- [License](#license)

---

# Business & Product

## Overview

`chaufher-api` is the core backend service for the ChaufHER platform, implementing business logic for:

- **Ride Management**: Booking, matching, lifecycle tracking, completion
- **Safety Features**: Panic events, real-time location tracking, incident escalation
- **Driver Operations**: Availability, earnings, performance metrics
- **Payment Processing**: Authorization, capture, refunds, dispute handling
- **Real-Time Communication**: WebSocket-based updates for riders and drivers

### Technical Foundation

- **Platform**: .NET 9 (C#)
- **Architecture**: Hexagonal (Ports & Adapters)
- **Database**: PostgreSQL (Azure managed)
- **Real-Time**: SignalR for WebSocket communication
- **Deployment**: Azure App Service / Container Apps

---

## Purpose & Value

### Business Problems Solved

1. **Real-Time Coordination**: Instant updates between riders, drivers, and operations
2. **Safety Assurance**: Immediate panic response with automated escalation
3. **Scalable Matching**: Efficient driver-rider pairing based on proximity and availability
4. **Payment Reliability**: PCI-compliant payment processing with fraud detection
5. **Operational Visibility**: Live dashboards for ride monitoring and incident management

### Technical Challenges Addressed

1. **High Availability**: Stateless design enables horizontal scaling
2. **Data Consistency**: Event-driven architecture ensures reliable state transitions
3. **Integration Flexibility**: Adapter pattern isolates external dependencies
4. **Testing Confidence**: Hexagonal architecture enables comprehensive test coverage
5. **Maintainability**: Clear separation of concerns reduces technical debt

---

## Core Capabilities

### Ride Management

- **Booking**: Create ride requests with pickup/dropoff locations
- **Matching**: Geospatial queries with scoring (proximity, availability, rating)
- **Pricing**: Distance/time calculation with surge multipliers
- **Lifecycle**: State machine (requested → accepted → en-route → in-progress → completed/canceled)
- **Tracking**: Real-time driver location updates via SignalR

### Safety & Security

- **Panic Events**: Immediate alert creation with automatic escalation
- **Location Tracking**: Continuous GPS monitoring during rides
- **Incident Management**: Ops dashboard integration for emergency response
- **Authentication**: OAuth2/JWT with Azure AD B2C integration
- **Authorization**: Role-based access control (rider, driver, ops, admin)

### Payment Processing

- **Authorization**: Hold funds at booking time
- **Capture**: Finalize payment on ride completion
- **Refunds**: Automated and manual refund processing
- **Disputes**: Integration with payment gateway dispute systems
- **Compliance**: PCI DSS compliant through adapter pattern

### Real-Time Communication

- **SignalR Hubs**: WebSocket connections for instant updates
- **Event Broadcasting**: Ride status changes, safety alerts, driver location
- **Fallback**: Long-polling for clients without WebSocket support
- **Scalability**: Azure SignalR Service for distributed connections

---

## Multi-Repo Architecture

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

### Repository Responsibilities

| Repository | Purpose | Technology | Depends On |
|------------|---------|------------|------------|
| **chaufher-workspace** | Monorepo coordination, shared docs | Markdown, scripts | All repos |
| **chaufher-api** | Backend services (this repo) | .NET 9, PostgreSQL | chaufher-infra |
| **chaufher-app** | Mobile client (iOS/Android) | Flutter/Dart | chaufher-api |
| **chaufher-web** | Admin portal | React/TypeScript | chaufher-api |
| **chaufher-infra** | Infrastructure as code | Bicep/Terraform | - |

### Integration Points

- **API Contracts**: OpenAPI specs define stable interfaces
- **Real-Time Events**: SignalR hub contracts for WebSocket communication
- **Authentication**: Shared JWT tokens from Azure AD B2C
- **Deployment**: Infrastructure managed by chaufher-infra
- **Monitoring**: Shared Application Insights workspace

---

# Architecture & Design

## Architecture Overview

### Hexagonal Architecture (Ports & Adapters)

```
┌─────────────────────────────────────────────────────────────┐
│                      API Layer                               │
│  Controllers │ SignalR Hubs │ Health Checks │ Middleware    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  Application Layer                           │
│  Use Cases │ Command/Query Handlers │ Orchestration         │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Domain Layer                              │
│  Entities │ Aggregates │ Domain Services │ Business Rules   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                Infrastructure Layer                          │
│  EF Core/PostgreSQL │ Payment Adapters │ Notification       │
│  External APIs │ Azure Services │ Message Queue             │
└─────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

#### API Layer (Presentation)
- HTTP request/response handling
- SignalR hub management
- Authentication/authorization enforcement
- Request validation and error formatting
- Health check endpoints

#### Application Layer (Use Cases)
- Business workflow orchestration
- Command/query handling (CQRS pattern)
- Transaction management
- Event publishing
- Cross-cutting concerns (logging, caching)

#### Domain Layer (Business Logic)
- Pure business rules (no side effects)
- Entity lifecycle management
- Aggregate consistency enforcement
- Domain events
- Value objects and specifications

#### Infrastructure Layer (Adapters)
- Database access (EF Core)
- External service integration
- Message queue implementation
- File storage
- Third-party API clients

---

## Design Philosophy

### Core Principles

#### 1. Domain-Driven Design (DDD)
- **Ubiquitous Language**: Code reflects business terminology
- **Bounded Contexts**: Clear boundaries between subdomains
- **Aggregates**: Consistency boundaries for entities
- **Domain Events**: Decouple business logic from side effects

#### 2. Separation of Concerns
- **Domain Purity**: No infrastructure dependencies in domain layer
- **Adapter Pattern**: External integrations behind interfaces
- **Dependency Inversion**: High-level modules don't depend on low-level details

#### 3. Testability
- **Unit Tests**: Pure domain logic without mocks
- **Integration Tests**: Adapter implementations with real dependencies
- **Contract Tests**: API compliance with OpenAPI specs
- **E2E Tests**: Critical user journeys

#### 4. Scalability
- **Stateless Design**: No server-side session state
- **Horizontal Scaling**: Multiple instances behind load balancer
- **Async Processing**: Long-running operations via message queue
- **Caching**: Redis for frequently accessed data

#### 5. Observability
- **Structured Logging**: Correlation IDs across requests
- **Distributed Tracing**: OpenTelemetry for request flows
- **Metrics**: Performance counters and business KPIs
- **Alerting**: Proactive monitoring for anomalies

### Design Patterns

| Pattern | Usage | Benefit |
|---------|-------|---------|
| **CQRS** | Separate read/write models | Optimized queries, scalability |
| **Repository** | Data access abstraction | Testability, flexibility |
| **Unit of Work** | Transaction management | Consistency, rollback support |
| **Strategy** | Algorithm selection (pricing, matching) | Extensibility, A/B testing |
| **Observer** | Event-driven notifications | Decoupling, async processing |
| **Adapter** | External service integration | Isolation, replaceability |

---

## Core Flows

### Ride Booking Flow

```
1. Rider creates ride request
   ↓
2. System validates pickup/dropoff locations
   ↓
3. Matching algorithm finds available drivers
   ↓
4. Driver scoring (proximity, rating, availability)
   ↓
5. Ride offer sent to top driver(s)
   ↓
6. Driver accepts → Ride state: ACCEPTED
   ↓
7. Payment authorization (hold funds)
   ↓
8. Real-time updates via SignalR
   ↓
9. Driver arrives → Ride state: EN_ROUTE
   ↓
10. Ride starts → Ride state: IN_PROGRESS
    ↓
11. Ride completes → Ride state: COMPLETED
    ↓
12. Payment capture (finalize transaction)
    ↓
13. Rating/feedback collection
```

### Safety Event Flow

```
1. User triggers panic button
   ↓
2. Safety event created with location
   ↓
3. Immediate notification to ops dashboard
   ↓
4. Automatic escalation based on severity
   ↓
5. Real-time location tracking activated
   ↓
6. Emergency contacts notified (SMS/push)
   ↓
7. Ops team assignment
   ↓
8. Resolution tracking and follow-up
```

### Payment Processing Flow

```
1. Ride booking → Payment authorization
   ↓
2. Funds held (not captured)
   ↓
3. Ride in progress → No payment action
   ↓
4. Ride completed → Calculate final fare
   ↓
5. Payment capture (finalize amount)
   ↓
6. Receipt generation and delivery
   ↓
7. Dispute handling (if applicable)
```

---

## System Interfaces

### REST API Endpoints

#### Rides
- `POST /api/rides` — Create new ride request
- `GET /api/rides/{rideId}` — Get ride details
- `GET /api/rides/{rideId}/status` — Lightweight status check
- `PATCH /api/rides/{rideId}/cancel` — Cancel ride
- `GET /api/rides/history` — User ride history

#### Safety
- `POST /api/safety-events` — Trigger panic/safety event
- `GET /api/safety-events/{eventId}` — Query event status
- `PATCH /api/safety-events/{eventId}/resolve` — Mark resolved

#### Users
- `GET /api/users/{userId}` — User profile
- `PATCH /api/users/{userId}` — Update profile
- `GET /api/users/{userId}/payment-methods` — Saved payment methods

#### Drivers
- `GET /api/drivers/{driverId}` — Driver profile
- `PATCH /api/drivers/{driverId}/availability` — Update availability
- `GET /api/drivers/{driverId}/earnings` — Earnings summary

#### Payments
- `POST /api/payments/authorize` — Authorize payment
- `POST /api/payments/capture` — Capture payment
- `POST /api/payments/refund` — Process refund

#### Notifications
- `POST /api/notifications/send` — Send notification
- `GET /api/notifications/preferences` — User preferences

### SignalR Hubs

#### `/hubs/rideevents` (Authenticated)

**Server → Client Events:**

- `ride.updated` — Ride state change
  - Payload: `{ rideId, status, driverLocation?, eta?, fare? }`
- `safety.event.update` — Safety event status change
  - Payload: `{ eventId, status, message, escalationLevel?, assignedOps? }`
- `driver.location.updated` — Driver position update
  - Payload: `{ driverId, location: { lat, lon }, heading, speed }`

**Client → Server Methods:**

- `SubscribeToRide(rideId)` — Subscribe to ride updates
- `UnsubscribeFromRide(rideId)` — Unsubscribe from ride
- `UpdateDriverLocation(location)` — Driver sends location

### Health & Monitoring

- `GET /healthz` — Liveness probe (is the API alive?)
- `GET /ready` — Readiness probe (can it handle traffic?)
- `GET /metrics` — Prometheus-compatible metrics

### Error Response Format

All errors follow this standard structure:

```json
{
  "code": "INVALID_REQUEST",
  "message": "Human-readable error message",
  "details": { "field": "error details" },
  "traceId": "correlation-id-for-logs"
}
```

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful request |
| 201 | Created | Resource created |
| 202 | Accepted | Async processing (check `retryAfter`) |
| 400 | Bad Request | Validation error |
| 401 | Unauthorized | Missing/invalid auth token |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate (idempotency key match) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server fault |

---

# Integration

## Client Integration

Both `chaufher-app` (mobile) and `chaufher-web` (admin) depend on stable API contracts.

### Authentication & Authorization

**REST APIs:**
- JWT bearer token (OAuth2 / Azure AD B2C)
- Header: `Authorization: Bearer {token}`
- Token refresh before expiry (401 triggers re-login)

**SignalR Hubs:**
- Same JWT auth
- Token in query param or Authorization header during negotiation

**Key Vault Integration:**
- Signing keys stored in Azure Key Vault per environment
- Clients fetch via managed identity (no hardcoded secrets)

### Idempotency & Retry Semantics

**Idempotency:**
- Clients SHOULD send `Idempotency-Key` header (UUID) for safety events
- Server returns `409 Conflict` if duplicate key matches existing event
- Safe to retry on network failure

**Async Processing:**
- Server may return `202 Accepted` for queued operations
- Clients poll status endpoint for completion
- `Retry-After` header indicates wait time

**Offline Support:**
- Clients cache events locally
- Retry with same idempotency key when network resumes

### Client Integration Checklist

Before integrating with the API:

- [ ] Auth tokens obtained securely (Azure AD / OAuth2 flow)
- [ ] JWT tokens refreshed before expiry
- [ ] Error handling with exponential backoff for retries
- [ ] SignalR connection established with event listeners
- [ ] Panic/safety events sent with `Idempotency-Key`
- [ ] Local caching supports offline dispatch
- [ ] Secrets fetched from Key Vault (never hardcoded)
- [ ] Correlation IDs included in all requests
- [ ] Rate limiting respected (429 responses)
- [ ] Proper error handling for all status codes

Full checklist: [docs/compatibility/compatibility-checklist.md](docs/compatibility/compatibility-checklist.md)

### Expected Client Behavior

**Mobile App (chaufher-app):**
- Maintains persistent SignalR connection during rides
- Sends location updates every 5 seconds (drivers)
- Implements offline queue for safety events
- Shows real-time driver location on map
- Handles push notifications for ride updates

**Admin Portal (chaufher-web):**
- Monitors all active rides in real-time
- Receives immediate safety event alerts
- Provides ops dashboard for incident management
- Generates compliance reports
- Manages driver approvals and suspensions

---

## Related Repositories

ChaufHER API is part of a multi-repository workspace:

| Repository | Purpose | Link |
|------------|---------|------|
| **chaufher-workspace** | Monorepo entry point, shared docs | [GitHub](https://github.com/phoenixvc/chaufher-workspace) |
| **chaufher-app** | Flutter mobile client (iOS/Android) | [GitHub](https://github.com/phoenixvc/chaufher-app) |
| **chaufher-web** | React admin portal | [GitHub](https://github.com/phoenixvc/chaufher-web) |
| **chaufher-infra** | Azure IaC (Bicep/Terraform), CI/CD | [GitHub](https://github.com/phoenixvc/chaufher-infra) |

### Cross-Repo Integration

**Infrastructure Dependencies:**
- Deployment managed by `chaufher-infra`
- Environment variables configured via Key Vault
- CI/CD pipelines defined in `chaufher-infra`
- See [chaufher-infra/MOBILE_CHECKLIST.md](https://github.com/phoenixvc/chaufher-infra/blob/main/MOBILE_CHECKLIST.md)

**Client Dependencies:**
- Mobile app consumes REST + SignalR APIs
- Admin portal uses same authentication
- Shared OpenAPI specs for contract testing
- See [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)

---

## Backward Compatibility

### API Versioning

- **Current Version**: v1 (stable)
- **Versioning Scheme**: Semantic versioning (MAJOR.MINOR.PATCH)
- **Breaking Changes**: Require new major version (v2, v3, etc.)
- **URL Format**: `/api/v1/rides` (version in path)

### Deprecation Policy

- Deprecated endpoints/fields marked with `Deprecated: true` in OpenAPI
- Minimum 3 months notice before removal
- Published in release notes and CHANGELOG
- Example: Field removed in v2.0.0 → deprecated in v1.2.0 (3+ months prior)

### Migration Guide

When upgrading API versions:

1. Check [CHANGELOG.md](CHANGELOG.md) for breaking changes
2. Update client endpoints (e.g., `/api/v2/rides`)
3. Test in staging before production
4. Update related repos (chaufher-app, chaufher-web)
5. Monitor error rates after deployment

### Compatibility Testing

- **Contract Tests**: Postman collections validate API contracts
- **Integration Tests**: Testcontainers verify database interactions
- **E2E Tests**: Critical user journeys across versions
- **CI Validation**: All tests run on every PR

---

# Developer Guide

## Quick Start

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| **.NET SDK** | 9.0+ | Build and run the API |
| **PostgreSQL** | 14+ | Database (or Testcontainers) |
| **Docker** | Latest | Optional: containerized dependencies |
| **Azure CLI** | Latest | Optional: Key Vault access |

### Initial Setup

1. **Clone Repository**
2. **Install Dependencies**
3. **Configure Environment**
4. **Run Database Migrations**
5. **Start API**
6. **Verify Setup**

### Environment Variables

**Required:**

- `ASPNETCORE_ENVIRONMENT` — Development, Staging, Production
- `ConnectionStrings__DefaultConnection` — PostgreSQL connection string
- `AzureKeyVault__VaultUrl` — Azure Key Vault endpoint
- `Auth__JwtSigningKey` — JWT signing key (from Key Vault in prod)
- `SignalR__ConnectionString` — Azure SignalR Service (if using managed)

**Optional:**

- `ASPNETCORE_URLS` — Server URL (default: http://localhost:5000)
- `Logging__LogLevel` — Min log level (Debug, Information, Warning, Error)
- `ApplicationInsights__InstrumentationKey` — Azure App Insights

Full reference: `.env.example` in repo

### IDE Configuration

**VS Code:**
1. Install extensions: C#, REST Client, Azure Tools
2. Open workspace
3. Create `.vscode/launch.json` (template in repo)
4. Press F5 to debug

**Visual Studio:**
1. Open `chaufher-api.sln`
2. Right-click solution → Manage Secrets
3. Press F5 to debug

---

## Development Guidelines

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| **Endpoints** | kebab-case | `/api/safety-events` |
| **Controllers** | PascalCase, plural | `RidesController` |
| **Methods** | PascalCase | `CreateRide()` |
| **Variables** | camelCase | `rideId`, `driverLocation` |
| **Constants** | UPPER_SNAKE_CASE | `MAX_RETRY_ATTEMPTS` |
| **DB Columns** | snake_case | `ride_id`, `created_at` |

### Commit Conventions

Use conventional commits:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `style`

**Examples:**
- `feat(safety): add panic event escalation to ops`
- `fix(auth): resolve JWT token refresh race condition`
- `docs(api): update OpenAPI partial with new endpoints`

### Git Workflow

**Branching:**
- `main` — production-ready
- `feature/your-feature` — new features
- `fix/issue-description` — bugfixes
- `chore/task` — non-code changes

**Pull Request Flow:**
1. Create feature branch from `main`
2. Push and open PR with description
3. All checks must pass (tests, linters, review)
4. Squash and merge on approval
5. Delete branch after merge

### Code Quality Standards

- **Linting**: `dotnet format` (enforced in CI)
- **Analyzers**: StyleCop, SonarAnalyzer (strict mode)
- **Test Coverage**: Minimum 80% for critical business logic
- **Complexity**: McCabe complexity < 10 per method
- **Async**: Prefer async/await for I/O-bound operations

---

## Testing & Observability

### Testing Strategy

| Test Type | Framework | Purpose | Coverage Target |
|-----------|-----------|---------|-----------------|
| **Unit** | xUnit + Moq | Domain logic, pure functions | 90%+ |
| **Integration** | Testcontainers | Database, adapters | 80%+ |
| **Contract** | Postman/Newman | API compliance | 100% endpoints |
| **E2E** | Custom scenarios | Critical user journeys | Key flows |

### Test Fixtures & Seed Data

- `scripts/seed-dev.sql` — Synthetic data for dev environment
- `tests/fixtures/` — Reusable test payloads (JSON)
- Testcontainers.NET — Ephemeral PostgreSQL for CI
- EF Core in-memory — Fast unit test database

### Observability

**Structured Logging:**
- Serilog or Microsoft.Extensions.Logging
- All logs include correlation ID, timestamp, structured properties
- Export to Azure Log Analytics + Application Insights

**Distributed Tracing:**
- OpenTelemetry for request flows
- Trace IDs propagated across services
- Visualized in Application Insights

**Metrics:**
- Performance counters (request duration, error rate)
- Business KPIs (rides/hour, safety events, revenue)
- Prometheus-compatible `/metrics` endpoint

**Dashboards:**
- Azure Monitor for infrastructure metrics
- Application Insights for application performance
- Grafana for custom visualizations

---

## Deployment

### Pipeline Overview

1. **CI Build**: Compile, lint, run tests
2. **Container Build**: Create Docker image, push to registry
3. **Deploy to Staging**: Azure App Service / Container Apps
4. **Smoke Tests**: Health checks, contract validation
5. **Promote to Production**: Canary / blue-green deployment

### Infrastructure

Managed by [chaufher-infra](https://github.com/phoenixvc/chaufher-infra):
- Azure App Service (recommended)
- Azure Container Apps (alternative)
- Azure Kubernetes Service (future)

### Secrets Management

- All secrets stored in Azure Key Vault per environment
- Managed identities for service-to-service auth
- No secrets in source code or environment files
- Automatic rotation every 90 days

### Database

- Azure Database for PostgreSQL (managed)
- Automated backups (continuous, 15-min RPO)
- Geo-replication for disaster recovery
- Connection pooling via PgBouncer

---

## Security Guidelines

### Secrets Management

**Never commit:**
- `.env` files with sensitive values
- API keys, JWT signing keys
- Database passwords
- Private certificates
- Connection strings with credentials

**Always:**
- Store secrets in Azure Key Vault
- Use managed identities for service auth
- Rotate credentials every 90 days
- Use environment variables for configuration

### Code Security

- Use parameterized queries (prevent SQL injection)
- Validate and sanitize all user inputs
- Use HTTPS/TLS for all external communication
- Apply OWASP Top 10 mitigations
- Never log passwords, tokens, or PII

### Security Logging

Log security events:
- Authentication failures
- Authorization denials
- Rate limit violations
- Admin actions
- Safety event creation

---

## Troubleshooting

### Common Issues

**Port 5000 already in use:**
- Find process and kill, or use different port

**PostgreSQL connection fails:**
- Verify connection string format
- Test connection with psql

**EF Core migration fails:**
- Ensure database exists and is accessible
- Reset database (dev only)

**SignalR connection timeout:**
- Check Azure SignalR Service status
- Verify hub URL and auth token

**Tests fail locally but pass in CI:**
- Clear build artifacts
- Check environment variables match CI

### Getting Help

- **Documentation**: [chaufher-workspace](https://github.com/phoenixvc/chaufher-workspace)
- **Issues**: Open GitHub issue with error details
- **Contact**: hans@chaufher.app

---

## API Documentation

### OpenAPI Specifications

- **Swagger UI**: Available at `/swagger` endpoint
- **OpenAPI Partial**: [docs/openapi/safety-and-ride-partial.yaml](docs/openapi/safety-and-ride-partial.yaml)
- **Postman Collection**: [docs/postman/README.md](docs/postman/README.md)
- **Compatibility Checklist**: [docs/compatibility/compatibility-checklist.md](docs/compatibility/compatibility-checklist.md)

### Environment URLs

- **Dev**: http://localhost:5000/swagger
- **Staging**: https://api-staging.chaufher.azure.com/swagger
- **Production**: https://api.chaufher.azure.com/swagger

### Contract Testing

Run Postman collection with Newman for automated contract validation in CI.

---

## Contributing

### Guidelines

- Keep domain logic in Domain layer (no infrastructure dependencies)
- Add unit tests for new business rules
- Add integration tests for persistence/adapters
- Follow existing CI checks and branch protection
- Update OpenAPI specs for API changes
- Document breaking changes in CHANGELOG

### Code Review Checklist

- [ ] Code follows naming conventions
- [ ] Tests added/updated
- [ ] No secrets in code
- [ ] OpenAPI specs updated
- [ ] CHANGELOG updated (if applicable)
- [ ] Breaking changes documented

---

## License

Copyright (c) 2025 ChaufHER. All rights reserved.

[Add License Here — e.g., MIT, Apache 2.0, or proprietary]
