# Custom .NET Clean Architecture + CQRS Template

**Type:** Personal  
**Status:** Completed — available as a Visual Studio 2022 project template  
**Built:** Solo  
**Git:** Coming soon

---

## Overview

A Visual Studio 2022 project template that generates a complete, production-ready .NET 9 backend solution with all enterprise architectural patterns pre-wired. Built after delivering PMFlow to encode the solution to every structural gap found in that system — no validation framework, no DTO mapping, inconsistent error handling, minimal test coverage, manual DI wiring per entity. Every new project using this template starts with all those problems already solved.

The repository contains two folders: `Template/` with `MyProject.zip` (the actual VS 2022 template file) and `Result/` with a working generated solution (`MyProject1.sln`) showing exactly what gets produced.

A developer adds only new entities and their DTOs. Everything else — repository registration, service wiring, MediatR handler discovery, validator discovery, CRUD endpoint generation — registers itself automatically at startup via reflection.

---

## Feature Flows

### CQRS via MediatR

All operations are expressed as Commands (state-changing) or Queries (read-only). Each has its own Handler. No business logic lives in endpoints — endpoints dispatch to MediatR and return the result.

`ApplicationServiceRegistration.cs` scans the Application assembly at startup and registers all handlers automatically — no manual registration per feature. Adding a Command means creating the Request and Handler classes; the template picks them up.

Cross-cutting concerns (validation) run as a pipeline Behavior before every handler without touching handler code.

### Validation via FluentValidation (Pipeline Behavior)

`ValidationBehavior.cs` intercepts every Command before it reaches the handler. It discovers all `AbstractValidator<TRequest>` classes in the assembly at startup and invokes the matching one per request. Invalid commands are rejected with a structured `string[]` error array. Validators auto-discovered — no manual registration.

### DTO Mapping via AutoMapper

Domain entities are never returned directly from endpoints. `MyTemplateProfile` is a single AutoMapper profile registered at startup that discovers all entity-to-DTO mappings. Adding a new entity requires only defining its DTO and the mapping entry — the profile registration is already wired.

### Result\<T\> Pattern (Railway-Oriented Programming)

Every operation returns `Result<T>` (in Domain, zero NuGet dependencies). Either a success carrying a typed value, or a failure carrying a `string[]` error array. No raw exceptions propagate to the API layer.

```
Result<T>.Success(value)      → HTTP 200
Result<T>.Failure(errors)     → HTTP 400 / 404 / 500
```

### Generic CRUD Endpoints + Auto-Registration

`EndPointRegistration.cs` scans for all `IEntity` types at startup. For each entity not marked with `[NoAutoEndpoint]`, `BaseCRUDEndPoint` generates five standard routes: GET all, GET by ID, POST, PUT, DELETE. Entities needing custom behavior use `[NoAutoEndpoint]` to opt out and define their own endpoints.

### Repository Pattern + Unit of Work + Service Layer

`RepositoryBase<T>` provides all standard data access (`AddAsync`, `UpdateAsync`, `DeleteAsync`, `GetAsync`, `GetAllAsync`, `GetFilteredAsync`, `ExistsAsync`). `InfrastructureServiceRegistration.cs` registers `IRepositoryBase<T>` → `RepositoryBase<T>` for every `IEntity` type automatically.

`UnitOfWork` wraps EF Core transactions. All multi-step operations run in a transaction, preventing partial writes.

### Testing + Data Seeding

An xUnit test project is included and configured from project generation. The in-memory EF Core database means integration tests run without a real database. `DataSeeder.cs` seeds sample entities at startup in Development.

---

## Component Summary

| Component | Technology | Version | What It Solves |
|---|---|---|---|
| CQRS | MediatR | 14.1.0 | Auto-discovers handlers; enables pipeline behaviors |
| Validation | FluentValidation | 12.1.1 | Auto-discovered validators; rejects invalid commands before handlers |
| Mapping | AutoMapper | 16.1.1 | Decouples API contract from domain entities |
| Error Handling | Result\<T\> (Domain) | — | Typed success/failure; no exception propagation to API |
| Data Access | EF Core + RepositoryBase\<T\> | 9.0.14 | Generic CRUD; auto-registered per IEntity |
| Transactions | UnitOfWork | — | Atomic multi-entity operations |
| Service Layer | ServiceBase\<T, TDto\> | — | Generic CQRS services; extensible per entity |
| Endpoint Gen | BaseCRUDEndPoint + reflection | — | 5 CRUD routes per IEntity; \[NoAutoEndpoint\] opt-out |
| Testing | xUnit | 2.9.2 | Test project present from day one |
| Sample Data | DataSeeder | — | Development data at startup |

---

## Patterns Used

- **Clean Architecture (6-project)** — unidirectional dependencies enforced by project references
- **`IEntity` interface** — single marker driving all auto-registration: repositories, services, CRUD endpoints
- **`[NoAutoEndpoint]` attribute** — opt-out for entities needing custom endpoint logic
- **CQRS via MediatR** — commands and queries separated; pipeline behaviors for cross-cutting concerns
- **FluentValidation as pipeline behavior** — auto-discovered; runs before every command handler
- **AutoMapper + single reflection profile** — zero per-entity registration
- **Result\<T\>** — typed success/failure in Domain; no framework coupling
- **RepositoryBase\<T\> + UnitOfWork** — generic data access; atomic transactions
- **DataSeeder** — sample data at startup in Development

---

## Technologies

- .NET 9.0 / ASP.NET Core Minimal APIs
- MediatR 14.1.0
- AutoMapper 16.1.1
- FluentValidation 12.1.1
- Entity Framework Core 9.0.14
- xUnit 2.9.2
- Swagger / OpenAPI (Swashbuckle)
