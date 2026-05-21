# PMFlow — Production Tracking System

**Type:** Professional  
**Status:** Production — deployed in an active manufacturing facility  
**Built:** Solo — every layer, from database schema to frontend UI  
**Git:** Private

*Full-stack production management platform for industrial manufacturing operations*

---

## System Overview

PMFlow is a full-stack web application designed to manage, monitor, and track production operations on a manufacturing floor. It covers the full lifecycle of a production job — from planning and scheduling, through real-time execution tracking, to completion and documentation.

The system replaces manual processes with a centralized digital platform that integrates directly with production lines, tracks pallets and materials, generates production paperwork, and provides real-time visibility into line performance.

---

## Architecture

### Pattern: Clean Architecture (4 Layers)

```
1 - Domain           → Entities, Interfaces, Business Rules
2 - Application      → Services, Use Cases, Business Logic
3 - Presentation     → Minimal API Endpoints, Request Objects (DTOs)
4 - Infrastructure   → EF Core Repositories, DbContext, External Integrations
```

Each layer depends only on layers above it. Infrastructure is the outermost layer and knows nothing about presentation. Domain is the innermost and has no external dependencies.

### Backend Design Patterns

**Repository Pattern**
- Generic base: `RepositoryBase<T>` implementing `IRepositoryBase<T>`
- Specialized repositories per domain: `JobRepository`, `PalletRepository`, `RealTimeInfoRepository`, `EventRepository`
- All repositories registered via DI as scoped services

**Unit of Work**
- `IUnitOfWork` with `BeginTransaction()`, `Commit()`, `Rollback()`, `SaveChanges()`
- Manages multi-entity database transactions at the service layer
- Prevents partial writes on complex operations

**Service Layer with Generic Base**
- `ServiceBase<T>` — generic CRUD service extended by domain-specific services
- Specialized: `JobServices`, `RealTimeInfoService`, `DocumentsServices`, `PaperWorkService`
- Domain logic encapsulated inside services, not leaked to endpoints

**Reflection-Based Auto-Registration**
- `EntityTypes.Types` — discovers all entity classes at startup via reflection
- `BaseCRUDEndPoint.CRUDEndpoints<TEntity>()` — auto-generates standard CRUD endpoints for all entities
- Entities marked with `[DoNotUseCRUDE]` get custom endpoint implementations instead
- Significantly reduces boilerplate — adding a new entity automatically registers its repository, service, and endpoints

**Minimal APIs**
- No MVC controllers — endpoints registered as lightweight extension methods
- Custom endpoints for complex domain operations (Job lifecycle, real-time, document printing)

### Frontend Architecture

**Framework:** Next.js (file-based routing) with React

**Component Layering:**
```
Pages → Sections (feature groups) → Components (reusable UI)
```

**State Management Strategy (layered):**
- `useState` — local UI state
- Custom hooks (`useSearch`, `useStore`) — feature-scoped data and filtering
- Redux Toolkit slices — global app features
- Context API — authentication and theme settings
- `localStorage` — client-side caching with TTL

**Authentication (Pluggable):**
- Default: JWT stored in `sessionStorage`
- Swappable providers: Auth0, Firebase, AWS Amplify, Azure MSAL
- `withAuthGuard()` HOC protects routes

---

## Technology Stack

### Backend

| Technology | Version |
|---|---|
| .NET | 7.0 |
| ASP.NET Core Minimal APIs | 7.0 |
| Entity Framework Core | 7.0.9 |
| SQL Server | — |
| Swagger / OpenAPI | 6.4.0 |
| MSTest + Moq + FluentAssertions | — |

### Frontend

| Technology | Version |
|---|---|
| Next.js | 16.1.6 |
| React | 19.2.4 |
| Material-UI (MUI) | 7.3.9 |
| Redux Toolkit | 2.11.2 |
| ApexCharts | 5.10.4 |
| Formik + Yup | — |
| @react-pdf/renderer | 4.3.2 |
| PowerBI Client | 2.23.10 |
| TypeScript | 5.9.3 |

---

## Domain Model

The system models a manufacturing operation with 30+ entities across these groups:

**Production Core**
- `Job` — central entity; a production run with status, brand, batch, line assignment, and pallet count
- `Batch` — raw material batch linked to jobs
- `Pallet` — physical pallet unit tracked per job
- `Brand`, `Presentation`, `ModelProduct` — product classification hierarchy
- `Line` — physical production line

**Job Lifecycle & Status**
- `JobStatus`: Planned → Scheduled → PreRunning → Executing → Stopped → Finished / Canceled
- `JobExecution` — execution records per job

**Job Paperwork (nested documentation per job)**
- `JobPaperWork` — parent document per job
- `JobPreRunInfo`, `JobStartUpInfo`, `JobRunInfo`, `JobEndRunInfo`, `JobCIPInfo`, `JobFinalCheck`

**Real-Time Monitoring**
- `RealTimeInfo` — live metrics from machines and lines
- `RealTimeInfoJob` — real-time data linked to active jobs
- `Event` — system events from the production floor

---

## Key Features

- **Job lifecycle management** — create, schedule, start, stop, finish production jobs
- **Real-time line monitoring** — live metrics per production line and machine
- **Pallet tracking** — add, remove, print labels per job
- **Production paperwork** — structured digital records for each execution phase
- **Document generation** — Word document generation via Office Interop
- **Label printing** — job and pallet label printing integration
- **Analytics dashboard** — ApexCharts-based production analytics
- **PowerBI integration** — embedded reporting
- **Multi-environment support** — dev/staging/production database configurations

---

## Key Achievements

**Reflection-based auto-registration engine** — Designed a system that automatically discovers all domain entities at startup and registers their repositories, services, and CRUD endpoints without manual wiring per entity. Adding a new entity requires only defining the class; everything else registers itself. Reduced boilerplate significantly as the domain grew to 30+ entities.

**Full job lifecycle state machine** — Designed and implemented complete state management for production jobs: Planned → Scheduled → PreRunning → Executing → Stopped → Finished / Canceled. Each transition has its own business logic, validations, and paperwork records.

**Structured production paperwork** — Each job generates a layered digital record — pre-run checks, start-up info, runtime entries, CIP records, end-of-run data, and final quality checks. Replaced paper-based records and made production history searchable and auditable.

**Real-time production monitoring** — Implemented endpoints and frontend components for live monitoring of production lines — speed, status, progress per line, and active job tracking visible to operators and managers on the floor.

---

## Technical Strengths (Confirmed by Code Audit)

**Architecture & Structure**
- Clean Architecture layers are genuinely enforced — no cross-layer shortcuts or leaky dependencies
- Domain stays pure: no infrastructure references, no framework coupling inside entities
- `ServiceBase<T>` + specialized services is a clean pattern — generic logic is shared, domain-specific logic is isolated

**Dependency Injection & Auto-Registration**
- Reflection-based entity discovery at startup automatically registers repositories and services
- All registrations are scoped correctly; no singleton anti-patterns

**Transaction Management**
- `IUnitOfWork` with `BeginTransaction()`, `Commit()`, `Rollback()` properly wraps multi-entity operations
- All job lifecycle operations use `IUnitOfWork` exclusively

**Business Logic Isolation**
- Job state machine transitions validated in `JobHelper.cs` — invalid transitions throw before reaching the database
- `StartJobAtLine()`, `FinishJob()`, `StopJob()` contain real domain logic, not just CRUD pass-throughs

---

## Known Gaps & Technical Debt

Honest assessment of what is missing or needs improvement. These are identified gaps, not failures — recognizing them is part of engineering maturity.

### Authentication & Authorization — Not Implemented
The API currently has no authentication. CORS is fully open. No JWT scheme configured.

**What's needed:** JWT authentication middleware, role-based authorization on endpoints, and a restricted CORS policy per environment.

### Credentials — Resolved
All credentials have been removed from committed files. `appsettings.json` is now secret-free. SQL Server credentials and the Kestrel certificate block have been moved to `appsettings.Development.json`, which is gitignored.

### Configuration — No IOptions\<T\> or Secrets Management
Configuration is read directly from `appsettings.json`. No strongly-typed configuration using `IOptions<T>`, no environment variable support, no separation between dev and production secrets.

### Logging — No Structured Logging
No Serilog or structured logging. Default ASP.NET Core logging only. Critical operations (job start, finish, stop) have no audit trail.

### Error Handling — No Global Middleware
No centralized exception handling middleware. Error handling is scattered across individual endpoint try-catch blocks. Internal exception details are no longer returned to the client, but no global `ProblemDetails` shape is in place.

### Unit Tests — Minimal Coverage (~2–3%)
Only one test file (`JobTest.cs`) covering 5 methods. No tests for `PaperWorkService`, `DocumentsService`, `RealTimeInfoService`, CRUD endpoints, or validation logic.

### EF Core Performance — Lazy Loading Enabled, N+1 Risk
Lazy loading is enabled globally. Domain entities are returned directly from endpoints without DTOs. Navigation properties on computed entity fields trigger database queries at JSON serialization time.

### DTOs — Domain Entities Exposed Directly
All generic CRUD endpoints return raw domain entities. No output DTOs, no mapping layer.

### Validation — Manual and Inconsistent
No validation framework. Validation exists only as inline if-checks in one endpoint file. Entity properties have no constraints.

---

| Area | Status |
|---|---|
| Authentication & Authorization | Not implemented |
| Credentials | Resolved |
| Configuration (IOptions\<T\>) | Not implemented |
| Structured Logging (Serilog) | Not implemented |
| Global Error Handling | Not implemented |
| Unit Test Coverage | ~2–3% |
| EF Core Performance (N+1) | Active risk |
| DTOs / Data Exposure | Not implemented |
| Input Validation | Inconsistent |
