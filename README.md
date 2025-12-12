# ChaufHER API Technical Design

## Product Overview

The **chaufher-api** service is a core backend component of the ChaufHER platform, responsible for orchestrating all interactions between the ChaufHER applications (mobile, web), real-time communications, business logic, and persistent storage. Built with .NET 9 and designed around a hexagonal architecture, the service exposes RESTful endpoints and real-time messaging (SignalR), manages rides, user accounts, driver management, payments, and notifications, and integrates closely with cloud infrastructure on Azure. Its core purpose is to provide a highly available, scalable, and secure backbone for all user and driver interactions in the ChaufHER ecosystem.

## Purpose

The primary purpose of **chaufher-api** is to centralize business logic and expose clean, well-documented APIs for the ChaufHER platform’s various clients. It solves key challenges, including:

* Encapsulating domain complexity such as ride matching, pricing algorithms, and driver assignment.

* Providing a real-time notification channel for dynamic ride states (using SignalR).

* Safely handling user authentication, authorization, and sensitive data.

* Integrating with payment and notification providers securely and reliably.

* Decoupling persistent storage and external system dependencies for maintainability and scalability.

* Offering robust APIs as the single point of orchestration for ChaufHER mobile apps, web applications, and internal/admin operations.

**Scenarios:**

* A rider books a ride through the mobile app. chaufher-api matches with an available driver and manages ride state until completion.

* A driver receives real-time ride requests, accepts them, and their status is updated for riders accordingly.

* Payment is processed at ride completion and receipts are sent.

## Target Audience

This document is intended for:

* **Backend Engineers** developing and maintaining the chaufher-api service.

* **System Architects** responsible for the overall ChaufHER system design.

* **DevOps Engineers** managing deployments, infrastructure, and observability.

* **QA/Test Engineers** defining verification and validation strategies for the API surface.

**Usage:**  

It serves as a definitive resource for understanding the system, onboarding new team members, designing integrations, planning infrastructure scaling, and implementing robust test and deployment strategies.

## Expected Outcomes

By following this technical design, we expect:

* **High Reliability:** Ensured by robust error handling, health monitoring, and rollback strategies.

* **Scalability:** Via stateless service design, auto-scaling on Azure, and support for horizontal database scaling.

* **Security:** Proper boundaries for authentication/authorization and the protection of sensitive data.

* **Developer Experience:** Comprehensive documentation (OpenAPI/Swagger), logical code organization, and clear extension points.

* **Operational Transparency:** Comprehensive metrics, logs, and traces, enabling rapid issue detection and root cause analysis.

**Key Metrics:**

## Design Details

**Layers & Structure:**

* **API Layer:** Exposes REST endpoints, SignalR hubs, and admin/internal surfaces.

* **Application Layer:** Coordinates domain operations, implements use cases, and enforces business rules.

* **Domain Layer:** Contains core logic and domain models (rides, users, drivers, payments).

* **Infrastructure Layer:** Implements persistence (EF Core/PostgreSQL), integration services (payments, notifications), and external comms (Azure services).

* **Ports/Adapters (Hexagonal):** Clear separation between primary (clients) and secondary (infrastructure) adapters; supports testability and replacement.

**Patterns:**

* Dependency Injection (via .NET DI)

* CQRS for separating read/write operations

* Event-driven for ride state transitions and notifications

**Key Modules:**

* Ride Management, User/Driver Profiles, Payment Processing, Notifications, Admin Operations

## Architectural Overview

**Component Diagram Description:**

* **Clients:** ChaufHER mobile/web apps communicate over HTTPS and SignalR.

* **API Gateway/Load Balancer:** Directs requests to chaufher-api instances.

* **chaufher-api (App Service/Container):** .NET 9, hexagonal architecture.

  * REST APIs (Rides, Users, Drivers, Payments, Notifications)

  * SignalR Hubs for real-time ride updates

  * EF Core Persistence to Azure PostgreSQL

* **Integration Services:** Outbound calls to payment gateways, notification providers, and reporting tools.

* **Other Repos:**

  * chaufher-app (mobile client), chaufher-web (web client), chaufher-infra (Terraform/Bicep/etc.), chaufher-workspace (shared tooling/scripts)

**Architecture Principles:**

* Hexagonal design enabling isolation of core business logic

* Real-time features via SignalR

* Cloud-native, leveraging Azure’s PaaS resources for scalability and reliability

## Data Structures and Algorithms

### Core Domain Models

### Key Algorithms and Flows

* **Matching:** Find the most suitable driver by proximity, availability, and rating. Uses geospatial queries and scoring.

* **Dynamic Pricing:** Fare calculation based on distance, time, and surge multipliers.

* **Status Updates:** Event-driven state machine—new, accepted, en route, in progress, completed, canceled—propagated via SignalR.

* **Settlement:** Payment authorization at ride booking, with final capture/release post-ride, using external payment gateway adapters.

**Justification:**

* EF Core entities efficiently map aggregates for ACID operations.

* Real-time features scale with SignalR and distributed state management.

* Business logic isolated for testability and future extensibility.

## System Interfaces

* **REST Endpoints:**

  * `/api/rides`

  * `/api/users`

  * `/api/drivers`

  * `/api/payments`

  * `/api/notifications`

  * Protected with OAuth2/JWT bearer authentication.

* **SignalR Hubs:**

  * `/hubs/rideevents`: authenticated connection, push ride status/activity

* **Third-party Integrations:**

  * Payment Processor SDKs/API (PCI compliant, HTTPS)

  * Notification Services (Push, SMS, Email)

  * Reporting (Telemetry/Event Hub as needed)

* **Internal Interfaces:**

  * Admin/ops endpoints (restricted access)

  * Shared messaging/events via Azure Service Bus or similar as needed

**Standards:** OpenAPI v3, OAuth2, REST, JSON, HTTPS, WebSocket (SignalR)

## User Interfaces

While **chaufher-api** exposes no end-user UI, developer-facing interfaces include:

* **Swagger/OpenAPI UI:** At `/swagger` for interactive documentation, test calls, and schema discovery.

* **Admin/Ops Endpoints:** Secure endpoints for operational health, feature flags, and incident diagnosis.

This approach supports rapid developer onboarding, robust API discoverability, and operational transparency.

## Hardware Interfaces

* **Azure App Service or Azure Container Apps:** For stateless microservice deployment.

* **Azure Database for PostgreSQL:** Provides reliable, scalable persistent storage.

* **Azure SignalR Service (optional):** For scaling real-time connections beyond a single node.

* **Load Balancer:** Azure Front Door or Application Gateway for high availability.

* **Networking:** VNETs, subnets, NSGs as per Azure best practices.

**Assumptions:**

* All components reside in the same region to minimize latency.

* Sufficient autoscaling and instance count provisioned per anticipated workload.

## Testing Plan

The plan guarantees code quality, correctness, and system stability:

* **Unit Testing:** Domain logic, validation, algorithm correctness.

* **Integration Testing:** Persistence (PostgreSQL), Service endpoints, Payment/Notification adapters.

* **Contract Testing:** API boundary verification and backward compatibility checks.

* **Performance Testing:** Load and stress testing, SignalR connection density.

* **End-to-End Testing:** Simulate critical business workflows.

## Test Strategies

* **Domain testing:** fakes/mocks for core services and domain logic.

* **Persistence:** Use in-memory (SQLite) and ephemeral PostgreSQL for CI/integration.

* **API:** Automated test suite using tools like xUnit, FluentAssertions, and REST clients.

* **SignalR:** Test harness for message delivery, client connectivity, and scale.

* **Edge Cases:** Simulate failures (network partitions, db failover, third-party API errors).

## Testing Tools

* **Unit Testing:** xUnit, NUnit

* **Mocking:** Moq, NSubstitute

* **Integration:** Testcontainers.NET (for ephemeral DB), EF Core’s in-memory provider

* **API/Contract:** Swagger/OpenAPI linter, Postman, REST-assured

* **E2E:** SpecFlow, Selenium (for admin surfaces), custom test runners

* **CI/CD:** GitHub Actions, Azure Pipelines with automated test orchestration

## Testing Environments

* Automated promotion from dev → staging → prod gated by passing tests and approvals.

## Test Cases

## Reporting and Metrics

* **Metrics:** Inbound request rate, latency, error rate, SignalR message delivery stats

* **Logs:** Structured, context-rich via Serilog or Microsoft.Extensions.Logging

* **Tracing:** Distributed tracing (OpenTelemetry) for request correlation

* **Dashboards:** Azure Monitor, Application Insights, custom Grafana dashboards

* **Alerting:** Real-time alerts for service degradation, error spikes, or infra faults

All metrics and logs are exported to central monitoring tools for team visibility and operational response.

## Deployment Plan

The chaufher-api is shipped via an automated pipeline, leveraging IaC and containerized deployments.

### Deployment Environment

* **Azure App Service/Container Apps:** Autoscaling, rolling deployment, zero downtime upgrades.

* **Azure Database for PostgreSQL:** Managed DB with high availability.

* **Networking:** Secured by private endpoints or appropriate firewall rules.

* **Secrets Management:** Azure Key Vault or environment-based secrets.

* **Disaster Recovery:** Automated backups and geo-redundancy for DB.

### Deployment Tools

* **CI/CD Pipelines:** GitHub Actions, Azure Pipelines

* **IaC:** chaufher-infra (Terraform/Bicep scripts)

* **Container Registry:** Azure Container Registry or GitHub Container Registry

* **chaufher-workspace:** Shared scripts for build/deploy/test steps

### Deployment Steps

1. **Code Commit:** Trigger CI pipeline, build, run automated tests.

2. **Build:** Containerize application if needed; push to registry.

3. **Infra Provision:** chaufher-infra provisions/updates necessary resources.

4. **Deploy:** Rolling deployment to App Service or Container App (staging slot first).

5. **Smoke Tests:** Automated checks against staging, rollback if failures detected.

6. **Production Promotion:** Slot swap/go-live after validation.

**Rollback:** Automatic revert to previous deployment if metrics/health checks fail.

## Post-Deployment Verification

* **Smoke Tests:** Automated API and SignalR connectivity runs.

* **Health Checks:** Verify `/healthz` endpoint, DB/EK connectivity, SignalR hub liveness.

* **Log/Metric Review:** No error spikes or abnormal patterns.

* **Manual Sanity:** (as needed) critical API operations via Swagger UI.

## Continuous Deployment

**Approach:**

* Implemented via branch protection, automated tests with required pass rates.

* Canary/blue-green deployments for incremental production rollout.

* Feature flags for progressive exposure of new features.

**Safeguards:**

* Rollback triggers if any defined SLOs are breached post-release.

* Alerting, monitoring, and human-in-the-loop escalation for critical paths.

**Benefits:**

* Reduced release friction and time-to-market.

* Rapid feedback on feature and fix success.

* Improved service quality and developer confidence.

**Requirements:**

* Mature test automation and high integration test reliability.

* Comprehensive observability and fast rollback mechanisms.