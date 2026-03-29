# Modular Monolith with Clean Architecture — Implementation Guide

A practical, high-level reference for developers and architects building .NET APIs using the hybrid pattern: **Modular Monolith boundaries between business domains** with **Clean Architecture layering inside complex modules**. No code — only definitions, rules, folder structures, and file names. All examples use EF Core and are based on an ecommerce API.

---

## Table of Contents

### Part 1: Understanding the Architecture
- [Architecture Overview](#architecture-overview)
- [Core Principles](#core-principles)

### Part 2: Solution Structure
- [Solution Structure](#solution-structure)
- [The Host Project](#the-host-project)
- [The Shared Project](#the-shared-project)
- [Module Anatomy: Complex vs Simple](#module-anatomy-complex-vs-simple)
  - [Complex Module (Clean Architecture Inside)](#complex-module-clean-architecture-inside)
  - [Simple Module (Flat Core)](#simple-module-flat-core)
- [Project Reference Rules](#project-reference-rules)

### Part 3: Inside a Module
- [Layer-by-Layer Guide](#layer-by-layer-guide)
  - [Contracts — The Module's Public API](#contracts--the-modules-public-api)
  - [Domain — Business Logic, No Dependencies](#domain--business-logic-no-dependencies)
  - [Core — Services, Data Access, and Endpoints](#core--services-data-access-and-endpoints)
- [Key Concepts](#key-concepts)
  - [Value Objects](#value-objects)
  - [Domain Entities vs DTOs](#domain-entities-vs-dtos)
  - [Two Levels of Interfaces](#two-levels-of-interfaces)
  - [The Module Registration File](#the-module-registration-file)
  - [Standard API Result](#standard-api-result)

### Part 4: How Things Work Together
- [Cross-Module Communication](#cross-module-communication)
- [Authentication](#authentication)
- [Error Handling](#error-handling)
- [Middleware Pipeline Order](#middleware-pipeline-order)
- [Validation Strategy](#validation-strategy)
- [Multi-Tenancy](#multi-tenancy)
- [Configuration and Settings per Module](#configuration-and-settings-per-module)
- [Dependency Injection Registration Pattern](#dependency-injection-registration-pattern)
- [Endpoint Organization](#endpoint-organization)
- [Mapping Strategy](#mapping-strategy)
- [EF Core Organization](#ef-core-organization)
- [Logging and Observability](#logging-and-observability)
- [Testing Strategy](#testing-strategy)

### Part 5: Putting It All Together
- [Request → Response Workflows](#request--response-workflows)
  - [Workflow 1: Create Product (single module)](#workflow-1-create-product-single-module)
  - [Workflow 2: Place Order (cross-module)](#workflow-2-place-order-cross-module-interface-approach)
- [How to Add a New Module](#how-to-add-a-new-module)
- [When a Module Should Graduate from Simple to Complex](#when-a-module-should-graduate-from-simple-to-complex)

### Part 6: Reference
- [Common Mistakes](#common-mistakes)
- [Setting Up the Solution (dotnet CLI)](#setting-up-the-solution-dotnet-cli)
- [Adopting the Events Approach](#adopting-the-events-approach)
- [Glossary](#glossary)

## Architecture Overview

This architecture combines two patterns that solve different problems:

**Modular Monolith** solves the inter-module problem: how do I keep business domains (Catalog, Orders, Payments) from coupling to each other's internals? Each module owns its data, its logic, and its API surface. Modules communicate only through public contracts.

**Clean Architecture** solves the intra-module problem: how do I keep business logic inside a module free from infrastructure concerns? Dependencies point inward — Domain knows nothing about databases, Application defines interfaces that Infrastructure implements.

The hybrid gives you **two layers of protection**: compiler-enforced boundaries between modules (no module can access another's internals) and compiler-enforced dependency direction within complex modules (inner layers never reference outer layers).

The key insight: **not every module needs Clean Architecture internally**. A complex Catalog module with rich pricing rules benefits from the layering. A simple Notifications module that just sends emails can stay flat. Each module chooses its own internal complexity.

---

## Core Principles

**1. Modules own their data.** Each module has its own EF Core `DbContext`. The Catalog module never queries the Orders module's tables directly. If Catalog needs order data, it asks through Orders' public contract.

**2. Dependencies point inward within a module.** In a complex module, Domain references nothing. Application references Domain. Infrastructure references Application. This is classic Clean Architecture scoped to one module.

**3. Modules communicate through Contracts only.** Every module exposes a Contracts project containing DTOs and service interfaces. Other modules reference *only* Contracts — never the internal projects (Domain or Core).

**4. No Core-to-Core, no Domain-to-Domain.** The hard boundary rule. A module's internal projects are invisible to every other module. The compiler enforces this through `.csproj` references.

**5. Shared stays small.** The Shared project contains truly cross-cutting abstractions: module registration interface, tenant provider, current user service, base entity. If Shared keeps growing, you're leaking module responsibilities into it.

**6. Each module can choose its own complexity.** A module with 3 entities and simple CRUD uses Contracts + Core (2 projects). A module with 15 entities, value objects, and complex business rules uses Contracts + Domain + Application + Infrastructure (4 projects). This decision is per-module, not system-wide.

---

## Solution Structure

```
EcommerceApp.sln
│
├── src/
│   ├── EcommerceApp.Host/                          ← the startup project (ASP.NET API)
│   ├── EcommerceApp.Shared/                        ← cross-cutting abstractions
│   │
│   └── Modules/
│       ├── Catalog/                                ← COMPLEX: 3 internal projects
│       │   ├── EcommerceApp.Modules.Catalog.Contracts/
│       │   ├── EcommerceApp.Modules.Catalog.Domain/
│       │   └── EcommerceApp.Modules.Catalog.Core/
│       │
│       ├── Orders/                                 ← COMPLEX: 3 internal projects
│       │   ├── EcommerceApp.Modules.Orders.Contracts/
│       │   ├── EcommerceApp.Modules.Orders.Domain/
│       │   └── EcommerceApp.Modules.Orders.Core/
│       │
│       ├── Payments/                               ← SIMPLE: 2 projects
│       │   ├── EcommerceApp.Modules.Payments.Contracts/
│       │   └── EcommerceApp.Modules.Payments.Core/
│       │
│       ├── Customers/                              ← SIMPLE: 2 projects
│       │   ├── EcommerceApp.Modules.Customers.Contracts/
│       │   └── EcommerceApp.Modules.Customers.Core/
│       │
│       ├── Auth/                                   ← SIMPLE: 2 projects
│       │   ├── EcommerceApp.Modules.Auth.Contracts/
│       │   └── EcommerceApp.Modules.Auth.Core/
│       │
│       └── Notifications/                          ← SIMPLE: 2 projects
│           ├── EcommerceApp.Modules.Notifications.Contracts/
│           └── EcommerceApp.Modules.Notifications.Core/
│
└── tests/
    ├── Modules/
    │   ├── Catalog/
    │   │   ├── EcommerceApp.Modules.Catalog.Domain.Tests/
    │   │   └── EcommerceApp.Modules.Catalog.Core.Tests/
    │   ├── Orders/
    │   │   ├── EcommerceApp.Modules.Orders.Domain.Tests/
    │   │   └── EcommerceApp.Modules.Orders.Core.Tests/
    │   ├── EcommerceApp.Modules.Payments.Tests/
    │   ├── EcommerceApp.Modules.Customers.Tests/
    │   └── EcommerceApp.Modules.Notifications.Tests/
    └── EcommerceApp.Integration.Tests/             ← cross-module integration tests
```

---

## The Host Project

The Host is the ASP.NET startup project. It wires everything together but contains no business logic. Its responsibilities are: configuring the HTTP pipeline, registering modules, applying middleware, and loading configuration.

```
EcommerceApp.Host/
├── Program.cs                          ← configures services, registers all modules, builds pipeline
├── appsettings.json                    ← global configuration (connection strings, JWT settings, tenant config)
├── appsettings.Development.json
├── Middleware/
│   ├── ExceptionHandlingMiddleware.cs  ← global exception → HTTP response mapping
│   ├── TenantResolutionMiddleware.cs   ← resolves tenant from header, subdomain, or route
│   └── RequestLoggingMiddleware.cs     ← structured logging for all requests
└── Extensions/
    └── ModuleRegistrationExtensions.cs ← helper to discover and register all IModule implementations
```

`Program.cs` iterates all modules and calls each one's registration method. This keeps the startup file clean — it doesn't know the internals of any module, just that each module implements `IModule` and knows how to register itself.

---

## The Shared Project

Contains abstractions and utilities that are genuinely cross-cutting — used by multiple modules and not owned by any single one. This project should stay **small**. If it grows beyond a handful of files, something that belongs in a module is leaking into Shared.

```
EcommerceApp.Shared/
├── Abstractions/
│   ├── IModule.cs                      ← interface every module implements for registration
│   └── ICurrentUserService.cs          ← resolves authenticated user's ID, roles, claims
├── Results/
│   ├── ApiResult.cs                    ← standard response wrapper: Success, Code, Message, Data, Errors
│   └── PagedResult.cs                  ← extends ApiResult with Page, PageSize, TotalCount, TotalPages
├── Multitenancy/
│   ├── ITenantProvider.cs              ← resolves current tenant context
│   ├── TenantProvider.cs               ← implementation (reads from middleware-set context)
│   └── TenantContext.cs                ← tenant ID, connection string, tenant-specific config
├── Auth/
│   ├── JwtTokenGenerator.cs            ← generates JWT tokens
│   └── JwtSettings.cs                  ← token configuration (secret, expiry, issuer)
├── Persistence/
│   ├── BaseEntity.cs                   ← shared base with Id, CreatedAt, UpdatedAt
│   └── IAuditableEntity.cs             ← interface for entities that track who modified them
└── Exceptions/
    ├── NotFoundException.cs            ← base exception for "entity not found" across modules
    └── ForbiddenException.cs           ← base exception for authorization failures
```

### What does NOT belong in Shared

- Module-specific DTOs, entities, or services
- Repository interfaces (these belong in each module's Core)
- Business rules or domain logic of any kind
- EF Core configurations or DbContext classes
- Module-specific middleware or filters

---

## Module Anatomy: Complex vs Simple

### Complex Module (Clean Architecture Inside)

Use this structure when a module has rich business logic, multiple entities with relationships, value objects, domain rules that need protection from infrastructure, or multiple infrastructure dependencies.

**3 projects: Contracts + Domain + Core** (recommended default)

```
Modules/Catalog/
│
├── EcommerceApp.Modules.Catalog.Contracts/        ← PUBLIC: what other modules see
│   ├── DTOs/
│   │   ├── ProductDto.cs                          ← product data exposed to other modules
│   │   ├── ProductSummaryDto.cs                   ← lightweight version for lists
│   │   ├── CategoryDto.cs
│   │   └── PriceQuoteDto.cs                       ← price with currency, used by Orders
│   └── Interfaces/
│       └── ICatalogService.cs                     ← public operations other modules can call
│
├── EcommerceApp.Modules.Catalog.Domain/           ← PROTECTED: pure business logic, zero dependencies
│   ├── Entities/
│   │   ├── Product.cs                             ← rich entity with behavior methods
│   │   ├── Category.cs
│   │   ├── ProductVariant.cs
│   │   └── ProductImage.cs
│   ├── ValueObjects/
│   │   ├── Money.cs                               ← amount + currency, immutable, self-validating
│   │   ├── Sku.cs                                 ← stock keeping unit with format validation
│   │   └── ProductDimensions.cs                   ← weight, height, width
│   ├── Enums/
│   │   ├── ProductStatus.cs                       ← Draft, Active, Discontinued
│   │   └── PricingTier.cs                         ← Retail, Wholesale, Premium
│   └── Exceptions/
│       ├── InvalidPriceException.cs
│       ├── DuplicateSkuException.cs
│       └── ProductDiscontinuedException.cs
│
└── EcommerceApp.Modules.Catalog.Core/             ← INTERNAL: services, data access, endpoints
    ├── Interfaces/
    │   ├── IProductRepository.cs                  ← data access contract (for testability)
    │   └── ICategoryRepository.cs
    ├── Services/
    │   └── CatalogService.cs                      ← implements ICatalogService from Contracts
    ├── DTOs/
    │   ├── CreateProductDto.cs                    ← input for creating a product (internal, not in Contracts)
    │   ├── UpdateProductDto.cs
    │   ├── CreateCategoryDto.cs
    │   └── ProductFilterDto.cs                    ← filtering/pagination parameters
    ├── Validators/
    │   ├── CreateProductValidator.cs              ← input validation rules for CreateProductDto
    │   ├── UpdateProductValidator.cs
    │   └── CreateCategoryValidator.cs
    ├── Mappings/
    │   └── CatalogMappingProfile.cs               ← entity ↔ DTO mapping logic
    ├── Persistence/
    │   ├── CatalogDbContext.cs                     ← EF Core DbContext scoped to this module
    │   ├── Configurations/
    │   │   ├── ProductConfiguration.cs             ← EF Core entity → table mapping
    │   │   ├── CategoryConfiguration.cs
    │   │   ├── ProductVariantConfiguration.cs
    │   │   └── ProductImageConfiguration.cs
    │   ├── Repositories/
    │   │   ├── ProductRepository.cs                ← implements IProductRepository from Core/Interfaces
    │   │   └── CategoryRepository.cs
    │   ├── Migrations/
    │   │   ├── 20250101_InitialCatalog.cs
    │   │   └── 20250115_AddProductVariants.cs
    │   └── Seeders/
    │       └── CategorySeeder.cs                   ← seed data for default categories
    ├── Endpoints/
    │   ├── GetProducts/
    │   │   ├── GetProductsEndpoint.cs              ← minimal API endpoint definition
    │   │   └── GetProductsResponse.cs              ← response shape specific to this endpoint
    │   ├── GetProductById/
    │   │   ├── GetProductByIdEndpoint.cs
    │   │   └── ProductDetailResponse.cs
    │   ├── CreateProduct/
    │   │   └── CreateProductEndpoint.cs
    │   ├── UpdateProduct/
    │   │   └── UpdateProductEndpoint.cs
    │   ├── DeleteProduct/
    │   │   └── DeleteProductEndpoint.cs
    │   └── Categories/
    │       ├── GetCategoriesEndpoint.cs
    │       └── CreateCategoryEndpoint.cs
    └── CatalogModule.cs                            ← implements IModule, registers all DI for this module
```

**Project references for 3-project modules:**

- **Contracts** → Shared only
- **Domain** → nothing (zero dependencies — this is the whole point)
- **Core** → Domain + Contracts + Shared + other modules' Contracts

**What this gives you:** The most valuable boundary — Domain isolation — is compiler-enforced. `Product.cs` cannot import EF Core or any infrastructure package. Value objects, entity behavior, and domain exceptions remain pure and testable with zero mocking. Services, repositories, validators, endpoints, and data access coexist in Core, separated by folders and conventions.

**What you trade:** Services can technically call `CatalogDbContext` directly, bypassing `IProductRepository`. In a 4-project setup the compiler blocks this; in a 3-project setup, code reviews catch it. For most teams, this is an acceptable trade-off.

### Adding an Application layer (optional, 4 projects)

If a module requires stricter internal separation — for example, a large team working on the module, multiple infrastructure dependencies you want to swap independently, or a requirement to test use case orchestration without any infrastructure — you can split Core into Application + Infrastructure:

- **Contracts + Domain + Application + Core** becomes **Contracts + Domain + Application + Infrastructure**
- Application holds: services, repository interfaces, internal DTOs, validators, mappings
- Infrastructure holds: DbContext, configurations, repository implementations, endpoints, module registration
- Application references Domain + Contracts. Infrastructure references Application. The compiler now prevents services from calling DbContext directly.

This is standard Clean Architecture applied inside the module. Use it when the complexity justifies the overhead — not as a default.

| Module complexity | Projects | Structure |
|---|---|---|
| Simple (mostly CRUD) | 2 | Contracts + Core |
| Complex (rich domain, business rules) | 3 | Contracts + Domain + Core |
| Extra complex (large team, multiple infra deps) | 4 | Contracts + Domain + Application + Infrastructure |

### Simple Module (Flat Core)

Use this structure when a module is mostly CRUD, has few entities, minimal business logic, and doesn't justify the overhead of Domain + Application + Infrastructure separation.

**2 projects: Contracts + Core**

```
Modules/Notifications/
│
├── EcommerceApp.Modules.Notifications.Contracts/
│   ├── DTOs/
│   │   └── NotificationDto.cs
│   └── Interfaces/
│       └── INotificationService.cs
│
└── EcommerceApp.Modules.Notifications.Core/
    ├── Domain/
    │   ├── Notification.cs
    │   ├── NotificationTemplate.cs
    │   └── NotificationType.cs
    ├── Services/
    │   └── NotificationService.cs                  ← implements INotificationService from Contracts
    │                                                  calls SendGrid, Twilio SDKs directly
    ├── Endpoints/
    │   ├── GetNotifications/
    │   │   └── GetNotificationsEndpoint.cs
    │   └── MarkAsRead/
    │       └── MarkAsReadEndpoint.cs
    ├── Persistence/
    │   ├── NotificationsDbContext.cs
    │   ├── Configurations/
    │   │   └── NotificationConfiguration.cs
    │   └── Migrations/
    └── NotificationsModule.cs
```

In a simple module, services can call `DbContext` directly — no repository abstraction needed. Validation can use data annotations on DTOs or inline checks in the service. The tradeoff is less structure for less ceremony.

---

## Project Reference Rules

These rules are non-negotiable. They are what makes the architecture work, and they are enforced by the compiler through `.csproj` references.

### Between modules (the hard boundary)

```
Host ──────────→ all *.Core projects (for module registration)
Host ──────────→ Shared

Any module ────→ other modules' Contracts ONLY
Any module ────→ Shared

Module A Core ──✕→ Module B Core          ← NEVER: this is the rule that makes it work
Module A Domain ✕→ Module B Domain        ← NEVER
```

### Within a complex module (3 projects)

```
Contracts ─────→ Shared (and nothing else)
Domain ────────→ nothing (zero dependencies)
Core ──────────→ Domain + Contracts + Shared + other modules' Contracts
```

### Within a complex module with Application layer (4 projects, optional)

```
Contracts ─────→ Shared (and nothing else)
Domain ────────→ nothing (zero dependencies)
Application ───→ Domain + Contracts
Infrastructure → Application (and therefore transitively Domain + Contracts)
```

### Within a simple module (2 projects)

```
Contracts ─────→ Shared (and nothing else)
Core ──────────→ Contracts + Shared + other modules' Contracts
```

---

## Layer-by-Layer Guide

### Contracts — The Module's Public API

**What it is:** The only project other modules are allowed to reference. It defines the module's external face — what it offers to the outside world.

**What it contains:**

- **DTOs** — Data shapes that cross module boundaries. `ProductDto`, `PriceQuoteDto`. These are the return types and parameter types used in the module's public interface.
- **Interfaces** — Service contracts that other modules can call. `ICatalogService` with methods like `GetProductsByIds()`, `GetPriceQuote()`, `ReduceStock()`. Implementations live in Core — never in Contracts.

**What it must NOT contain:**

- Implementations of any kind
- Domain entities or value objects
- Repository interfaces (those are internal to Core)
- Any reference to EF Core, ASP.NET, or infrastructure packages

**Design rule:** Keep Contracts as thin as possible. Every type you add here is a public commitment — other modules will depend on it, and changing it requires coordinating across modules.

### Domain — Business Logic, No Dependencies

**What it is:** The innermost layer of a complex module. Contains the business rules that would be true even if the software didn't exist. "A product must have a positive price" is a business rule. "A product is stored in a SQL table" is not.

**What it contains:**

- **Entities** — Rich objects with behavior. A `Product` entity doesn't just hold data — it has methods like `ApplyDiscount()`, `Discontinue()`, `AddVariant()` that enforce business rules. Entities have an identity (an ID that persists across state changes).
- **Value Objects** — Objects defined by their attributes, not identity. `Money(100, "USD")` is equal to any other `Money(100, "USD")`. They are immutable and self-validating. See [Value Objects](#value-objects) section.
- **Enums** — Domain-level enumerations. `ProductStatus`, `PricingTier`.
- **Exceptions** — Domain-specific exceptions. `InvalidPriceException`, `DuplicateSkuException`. These represent violations of business rules, not infrastructure errors.

**What it must NOT contain:**

- Any reference to EF Core, ASP.NET, or any NuGet package that isn't pure .NET
- Repository interfaces (those belong in Core)
- DTOs (those belong in Core or Contracts)
- Anything that depends on infrastructure

**The test:** If you can copy the Domain project into a completely different application (console app, desktop app, different API) and it compiles with zero changes, the Domain layer is clean.

### Core — Services, Data Access, and Endpoints

**What it is:** The project that holds everything except the public API (Contracts) and pure business logic (Domain). It contains use case orchestration, data access, HTTP endpoints, validation, and module registration. In the 3-project structure, Core combines what would be Application and Infrastructure in a full Clean Architecture setup.

**What it contains:**

- **Service implementations** — `CatalogService` implements `ICatalogService` from Contracts. This is where use cases live: "create a product" means validate the input, map DTO to entity, call the repository, return a DTO.
- **Repository interfaces** — `IProductRepository`, `ICategoryRepository`. Defined in `Core/Interfaces/` for testability. Implementation lives in `Core/Persistence/Repositories/`.
- **Internal DTOs** — `CreateProductDto`, `UpdateProductDto`. These are the input shapes for use cases. They may differ from Contracts DTOs (Contracts DTOs are for cross-module communication, Core DTOs are for endpoint-to-service communication within the module).
- **Validators** — Validation classes for input DTOs. `CreateProductValidator` checks that name is not empty, price is positive, SKU format is valid. You can use any validation library or write validation manually.
- **Mappings** — Entity ↔ DTO transformation logic. Can use a mapping library or manual mapping methods. See [Mapping Strategy](#mapping-strategy).
- **EF Core DbContext** — `CatalogDbContext` scoped to this module's entities only. See [EF Core Organization](#ef-core-organization).
- **Entity Configurations** — EF Core `IEntityTypeConfiguration<T>` classes that map domain entities to database tables.
- **Repository implementations** — `ProductRepository` implements `IProductRepository`. Uses `CatalogDbContext` internally.
- **Migrations** — EF Core migrations scoped to this module's DbContext.
- **External service calls** — Services call external SDKs directly (Stripe, SendGrid, S3) without an intermediate interface. Only introduce an abstraction when you have a real need to swap providers — such as multi-tenant systems where different tenants use different providers.
- **Endpoints** — Minimal API endpoint definitions. Each endpoint is a thin adapter: it receives an HTTP request, calls the service, and returns an HTTP response. Organized by feature in subfolders.
- **Module registration** — `CatalogModule.cs` implements `IModule` and registers all DI for this module: DbContext, repositories, services, validators.

**What it must NOT contain:**

- Business rules or domain logic (those belong in Domain)
- Types meant for cross-module consumption (those belong in Contracts)

---

## Key Concepts

### Value Objects

A value object is defined by its attributes, not by an identity. Two `Money(100, "USD")` instances are equal regardless of being different objects in memory. Value objects follow three rules:

**Equality by value** — Override `Equals()` and `GetHashCode()` to compare all properties.

**Immutability** — Once created, never changes. Need a different amount? Create a new `Money`.

**Self-validation** — A `Money` with a negative amount or an `EmailAddress` without `@` can never exist. The constructor rejects invalid state.

The benefit: if your code holds a value object reference, it's guaranteed to be valid. No scattered validation across services and controllers.

Common ecommerce value objects: `Money` (amount + currency), `Address` (street + city + zip + country), `Sku` (formatted stock keeping unit), `EmailAddress` (validated email format), `PhoneNumber`, `DateRange`, `Percentage`.

### Domain Entities vs DTOs

| Aspect | Domain Entity | DTO |
|---|---|---|
| **Has identity** | Yes (ID) | No |
| **Has behavior** | Yes (methods enforce rules) | No (pure data carrier) |
| **Mutable** | Yes (through controlled methods) | Typically yes (for deserialization) |
| **Where it lives** | Domain layer | Core or Contracts |
| **Crosses boundaries** | Never leaves the module | Designed to cross boundaries |
| **EF Core maps it** | Yes (via Core configurations) | Never |

Entities never leave the module. When another module or an endpoint needs product data, the service maps the `Product` entity to a `ProductDto` and returns the DTO. The entity stays internal.

### Two Levels of Interfaces

This architecture produces two distinct kinds of interfaces. Understanding the difference is essential:

| Interface | Defined in | Purpose | Who consumes it |
|---|---|---|---|
| `ICatalogService` | Contracts | Module's public API | Any other module |
| `IProductRepository` | Core/Interfaces | Internal data access contract | Core/Persistence only |

`ICatalogService` is the **external face** — what the Orders module calls when it needs product prices. `IProductRepository` is **internal plumbing** — how services get data without coupling to EF Core directly. Other modules never see `IProductRepository`.

### The Module Registration File

Every module (complex or simple) has a registration file that implements `IModule`. This file is the single entry point the Host uses to wire up the module.

`CatalogModule.cs` lives in Core (where the DbContext, repositories, and services live). It registers: the `CatalogDbContext`, all repository implementations, the `CatalogService`, all validators, and the module's minimal API endpoints.

The Host's `Program.cs` discovers all `IModule` implementations and calls their registration methods. This keeps the startup file clean and each module self-contained.

### Standard API Result

Every endpoint should return a consistent response shape. Instead of returning raw DTOs or inconsistent error formats, wrap all responses in a standard result type that lives in Shared so every module uses the same structure.

**Where it lives:** `Shared/Results/ApiResult.cs`

```
EcommerceApp.Shared/
├── Abstractions/
│   ├── IModule.cs
│   └── ICurrentUserService.cs
├── Results/
│   ├── ApiResult.cs                    ← generic wrapper: Success, Code, Message, Data, Errors
│   └── PagedResult.cs                  ← extends ApiResult with pagination: Page, PageSize, TotalCount, TotalPages
├── Multitenancy/
│   └── ...
```

**What `ApiResult<T>` represents:**

- `Success` (bool) — whether the operation succeeded
- `Code` (string) — a machine-readable code identifying the result: `"PRODUCT_CREATED"`, `"VALIDATION_ERROR"`, `"PAYMENT_DECLINED"`. Clients can switch on this code. Useful for translations — the client is responsible for mapping codes to localized messages
- `Message` (string, nullable) — a human-readable summary of what happened. Most useful when `Success = false` to give the caller context. For successful responses, this is typically null or a simple confirmation
- `Data` (T, nullable) — the response payload. Null on failure
- `Errors` (object, nullable) — a dictionary-like object where each property is a field name and its value is an array of error objects with `code` and `message`. For errors not tied to a specific field, use a general key like `"general"`. This structure supports field-level validation errors and general errors in the same shape

The `Code` is a string (not an HTTP status code) because it represents application-level results. Your API can define custom codes more specific than HTTP statuses. The endpoint maps `Code` to the appropriate HTTP status code when returning the response, keeping services unaware of HTTP concerns.

**How it looks in practice:**

A successful `POST /api/catalog/products` returns:

```json
{
  "success": true,
  "code": "PRODUCT_CREATED",
  "message": null,
  "data": {
    "id": "abc-123",
    "name": "Wireless Headphones",
    "price": { "amount": 79.99, "currency": "USD" },
    "status": "Active"
  },
  "errors": null
}
```

A failed `POST /api/catalog/products` with validation errors returns:

```json
{
  "success": false,
  "code": "VALIDATION_ERROR",
  "message": "One or more validation errors occurred.",
  "data": null,
  "errors": {
    "name": [
      { "code": "REQUIRED", "message": "Product name is required." }
    ],
    "price": [
      { "code": "GREATER_THAN", "message": "Price must be greater than zero." },
      { "code": "INVALID_CURRENCY", "message": "Currency code 'XYZ' is not supported." }
    ]
  }
}
```

A `GET /api/catalog/products` with pagination returns:

```json
{
  "success": true,
  "code": "OK",
  "message": null,
  "data": [
    { "id": "abc-123", "name": "Wireless Headphones", "price": { "amount": 79.99, "currency": "USD" } },
    { "id": "def-456", "name": "USB-C Cable", "price": { "amount": 12.99, "currency": "USD" } }
  ],
  "errors": null,
  "page": 1,
  "pageSize": 20,
  "totalCount": 87,
  "totalPages": 5
}
```

A `GET /api/catalog/products/{id}` when the product doesn't exist returns:

```json
{
  "success": false,
  "code": "NOT_FOUND",
  "message": "Product with id 'xyz-789' was not found.",
  "data": null,
  "errors": null
}
```

A domain-specific failure like a payment decline:

```json
{
  "success": false,
  "code": "PAYMENT_DECLINED",
  "message": "The payment could not be processed.",
  "data": null,
  "errors": {
    "general": [
      { "code": "CARD_DECLINED", "message": "The card was declined by the issuing bank." }
    ]
  }
}
```

A combined failure with both field-level and general errors:

```json
{
  "success": false,
  "code": "ORDER_CREATION_FAILED",
  "message": "The order could not be created.",
  "data": null,
  "errors": {
    "shippingAddress": [
      { "code": "REQUIRED", "message": "Shipping address is required." }
    ],
    "general": [
      { "code": "INSUFFICIENT_STOCK", "message": "Product 'abc-123' has only 2 units available." }
    ]
  }
}
```

**Why errors are structured this way:** The object-based errors structure (field → array of `{code, message}`) serves two purposes. First, it maps naturally to form validation — a frontend can iterate over fields and display errors next to the corresponding input. Second, the `code` on each error supports client-side translations — the client can use the code (`"REQUIRED"`, `"GREATER_THAN"`, `"CARD_DECLINED"`) as a translation key and ignore the `message`, or use the `message` as a fallback for untranslated codes.

**Why it lives in Shared:** Every module returns `ApiResult<T>` from its services. If it lived in a module's Contracts, other modules would need to reference it — which violates the boundary rules. Shared is the correct home because it's a cross-cutting concern used by all modules equally.

**How services use it:** Services in Core return `ApiResult<ProductDto>` instead of throwing exceptions for expected business failures. The endpoint receives the result and maps the `Code` to the appropriate HTTP status code. The `ExceptionHandlingMiddleware` in Host catches any unhandled exceptions and wraps them in an `ApiResult` with `Code = "INTERNAL_ERROR"` and a generic `Message`, so the client always gets the same shape regardless of what happened internally.

---

## Cross-Module Communication

Modules communicate by injecting interfaces from each other's Contracts and calling them directly. The call is a regular method invocation resolved through DI — no bus, no handlers, no infrastructure beyond standard dependency injection.

**Example:** Orders needs to verify product prices before placing an order. `OrderService` (in Orders.Core) injects `ICatalogService` (from Catalog.Contracts) and calls `GetProductsByIds()`. The call is immediate, returns data, and OrderService continues.

**Project reference:** Orders.Core references Catalog.Contracts. That's the only connection between the two modules.

**How it works in practice:** When Orders needs to verify product prices, it calls `ICatalogService.GetProductsByIds()`. When Orders needs to reduce stock after placing an order, it calls `ICatalogService.ReduceStock()`. When Orders needs to trigger payment, it calls `IPaymentService.ProcessPayment()`. All interactions are direct, synchronous, and debuggable — you can step through the entire cross-module flow with a debugger.

**When you outgrow interfaces:** If you find that one module has too many dependencies on other modules' interfaces, or that a single action needs to notify many modules simultaneously, consider introducing integration events as an alternative communication mechanism. See [Adopting the Events Approach](#adopting-the-events-approach) for the complete guide on when and how to add events to this architecture.

---

## Authentication

Auth has two sides: the **actions** users take (login, register, refresh token) and the **cross-cutting behavior** that protects other endpoints (JWT validation, current user resolution). These live in different places.

**Auth as a module** — Login, register, forgot password, reset password, and refresh token are user-facing features. They belong in a simple Auth module with Contracts + Core, just like any other module.

```
Modules/Auth/
├── EcommerceApp.Modules.Auth.Contracts/
│   ├── DTOs/
│   │   ├── LoginDto.cs
│   │   ├── RegisterDto.cs
│   │   ├── TokenResponseDto.cs
│   │   └── RefreshTokenDto.cs
│   └── Interfaces/
│       └── IAuthService.cs
│
└── EcommerceApp.Modules.Auth.Core/
    ├── Domain/
    │   ├── User.cs
    │   ├── RefreshToken.cs
    │   └── Role.cs
    ├── Services/
    │   └── AuthService.cs                          ← implements IAuthService from Contracts
    ├── Validators/
    │   ├── LoginValidator.cs
    │   ├── RegisterValidator.cs
    │   └── ResetPasswordValidator.cs
    ├── Endpoints/
    │   ├── Login/
    │   │   └── LoginEndpoint.cs
    │   ├── Register/
    │   │   └── RegisterEndpoint.cs
    │   ├── RefreshToken/
    │   │   └── RefreshTokenEndpoint.cs
    │   ├── ForgotPassword/
    │   │   └── ForgotPasswordEndpoint.cs
    │   └── ResetPassword/
    │       └── ResetPasswordEndpoint.cs
    ├── Persistence/
    │   ├── AuthDbContext.cs
    │   ├── Configurations/
    │   │   ├── UserConfiguration.cs
    │   │   └── RefreshTokenConfiguration.cs
    │   └── Migrations/
    └── AuthModule.cs
```

**Auth as cross-cutting** — JWT validation, token generation, and current user resolution are used by every module. These live in Shared.

- `Shared/Auth/JwtTokenGenerator.cs` — creates and validates JWT tokens
- `Shared/Auth/JwtSettings.cs` — token configuration (secret, expiry, issuer)
- `Shared/Abstractions/ICurrentUserService.cs` — resolves the authenticated user's ID, roles, and claims from the HTTP context

The Auth module's `AuthService` uses `JwtTokenGenerator` from Shared to create tokens. Every other module uses `ICurrentUserService` from Shared to know who's making the request. The auth middleware in Host validates the JWT on every request and populates the user context that `ICurrentUserService` reads.

**Endpoint protection** stays on each endpoint — each endpoint declares its own access requirements (`[Authorize]`, `[Authorize(Roles = "Admin")]`, or policy-based authorization). Authorization decisions are visible where the route is defined, not centralized elsewhere.

---

## Error Handling

Errors flow through the system in two ways: **expected failures** returned as `ApiResult` with error codes, and **unexpected exceptions** caught by global middleware.

### Expected failures (service returns error result)

When a service encounters a business failure — product not found, validation failed, insufficient stock — it returns an `ApiResult` with `Success = false` and a descriptive `Code`. The endpoint maps the code to the appropriate HTTP status and returns it. No exception is thrown.

This is the preferred path for any failure the system anticipates. The flow is explicit: the service decides the outcome, the endpoint translates it to HTTP.

### Unexpected exceptions (middleware catches)

When something genuinely unexpected happens — a database connection drops, a null reference, an external API timeout — an exception propagates up. The `ExceptionHandlingMiddleware` in Host catches it and wraps it in a standard `ApiResult`.

The middleware maps known exception types to appropriate responses:

| Exception | Code | HTTP Status |
|---|---|---|
| `NotFoundException` (from Shared) | `NOT_FOUND` | 404 |
| `ForbiddenException` (from Shared) | `FORBIDDEN` | 403 |
| `UnauthorizedAccessException` | `UNAUTHORIZED` | 401 |
| `ValidationException` | `VALIDATION_ERROR` | 400 |
| Any unhandled exception | `INTERNAL_ERROR` | 500 |

For `INTERNAL_ERROR`, the middleware logs the full exception with stack trace but returns only a generic message to the client — never expose internal details in production.

### The convention

Services should return `ApiResult` for expected outcomes (both success and business failures). Exceptions are reserved for truly unexpected situations. If you find yourself writing `throw new ProductNotFoundException()` in a service, consider returning `ApiResult.Fail("NOT_FOUND", "Product not found")` instead — it's more explicit, carries a human-readable message, and doesn't rely on exception flow for control.

The exception middleware exists as a safety net, not as the primary error handling mechanism.

---

## Middleware Pipeline Order

The order middleware is registered in `Program.cs` matters. Each middleware wraps the next, so the first registered runs first on the way in and last on the way out.

```
1. ExceptionHandlingMiddleware        ← outermost: catches ALL unhandled exceptions
2. RequestLoggingMiddleware           ← logs every request/response with timing
3. Authentication middleware          ← validates JWT, populates User identity
4. TenantResolutionMiddleware         ← resolves tenant from header/subdomain/claim
5. Authorization middleware           ← checks [Authorize] policies
6. Endpoint routing                   ← routes to the correct module endpoint
```

**Why this order:**

- Exception handling wraps everything — any middleware or endpoint that throws is caught
- Logging runs before auth so you can log even unauthenticated requests (useful for diagnosing auth failures)
- Authentication runs before tenant resolution because some systems resolve the tenant from a JWT claim
- Tenant resolution runs before authorization because some authorization policies are tenant-specific
- Authorization runs before endpoint routing so unauthorized requests are rejected before touching module code

---

## Validation Strategy

Validation happens at four levels. Each level has a specific place in the architecture.

### Input validation (Core/Validators/)

Validates the shape of incoming data: required fields, string lengths, format constraints, numeric ranges. Each input DTO has a corresponding validator file in `Core/Validators/`.

| DTO | Validator file |
|---|---|
| `CreateProductDto` | `Core/Validators/CreateProductValidator.cs` |
| `UpdateProductDto` | `Core/Validators/UpdateProductValidator.cs` |
| `PlaceOrderDto` | `Core/Validators/PlaceOrderValidator.cs` |

Validators run before business logic — if a `CreateProductDto` has no name, reject it immediately without touching the domain. You can use any validation library or write validation logic manually. The validator returns an `ApiResult` with `Code = "VALIDATION_ERROR"` and field-level errors if validation fails.

### Business rule validation (Domain/Entities/)

Validates domain invariants: "a product cannot have a negative price," "a discontinued product cannot be reactivated," "an order must have at least one item." Lives inside entity constructors and methods.

| Rule | File |
|---|---|
| Product price must be positive | `Domain/Entities/Product.cs` (constructor or `SetPrice()` method) |
| Money cannot have negative amount | `Domain/ValueObjects/Money.cs` (constructor) |
| Order must have at least one item | `Domain/Entities/Order.cs` (constructor or `AddItem()` method) |

These rules are enforced regardless of how the entity is created — through an endpoint, a background job, or a test. When a domain rule is violated, the entity throws a domain exception (e.g., `InvalidPriceException` from `Domain/Exceptions/`).

### Cross-module validation (through Contracts)

When a module needs to validate data that belongs to another module: "does this customer exist before placing an order?" The validation happens in the service file that orchestrates the use case.

| Rule | File | How |
|---|---|---|
| Customer must exist before placing order | `Orders.Core/Services/OrderService.cs` | Calls `ICustomerService.GetCustomerById()` from Customers.Contracts |
| Product must be in stock before adding to order | `Orders.Core/Services/OrderService.cs` | Calls `ICatalogService.GetProductsByIds()` from Catalog.Contracts |

The calling module never queries another module's database directly. If the cross-module call returns a failure, the service returns an `ApiResult` with the appropriate error.

### Cross-cutting validation (Host/Middleware/)

Tenant existence, authentication token validity, rate limiting. Lives in middleware that runs before any module code.

| Rule | File |
|---|---|
| JWT token is valid | `Host/Middleware/` (ASP.NET authentication middleware) |
| Tenant exists and is active | `Host/Middleware/TenantResolutionMiddleware.cs` |
| Rate limit not exceeded | `Host/Middleware/RateLimitingMiddleware.cs` (if applicable) |

---

## Multi-Tenancy

Tenant resolution flows from the outside in:

**Host layer** — `TenantResolutionMiddleware` reads the tenant identifier from the HTTP request (header, subdomain, route segment, or JWT claim) and sets `TenantContext` in the DI scope.

**Shared layer** — `ITenantProvider` and `TenantContext` make the current tenant available to any module. Services resolve `ITenantProvider` to get the tenant ID.

**Module layer** — Each module's `DbContext` applies a global query filter for tenant isolation. Every query automatically filters by `TenantId` without developers remembering to add `.Where(x => x.TenantId == tenantId)` on every query.

This is the cleanest multi-tenant approach because each module controls its own data isolation. The Catalog module could use row-level filtering (all tenants in one database, filtered by `TenantId`), while the Payments module could use database-per-tenant for stricter financial isolation.

---

## Configuration and Settings per Module

Each module can have its own settings class for module-specific configuration. All configuration lives in the Host's `appsettings.json` (the single configuration source), but each module reads only the section it owns.

**In `appsettings.json`** — module settings are grouped by module name:

```json
{
  "ConnectionStrings": {
    "Catalog": "Server=...;Database=EcommerceApp;",
    "Orders": "Server=...;Database=EcommerceApp;",
    "Payments": "Server=...;Database=EcommerceApp;"
  },
  "Jwt": {
    "Secret": "...",
    "Issuer": "EcommerceApp",
    "ExpiryMinutes": 60
  },
  "Modules": {
    "Catalog": {
      "MaxProductsPerCategory": 500,
      "ImageMaxSizeMb": 10
    },
    "Payments": {
      "StripeApiKey": "sk_live_...",
      "WebhookSecret": "whsec_..."
    },
    "Notifications": {
      "SendGridApiKey": "SG...",
      "FromEmail": "noreply@ecommerce.com"
    }
  }
}
```

**In the module** — each module defines a settings class in Core and binds it during registration:

```
Catalog.Core/
├── Settings/
│   └── CatalogSettings.cs             ← MaxProductsPerCategory, ImageMaxSizeMb
```

The module's `CatalogModule.cs` binds the settings from `configuration.GetSection("Modules:Catalog")` and registers it in DI. Services inject `IOptions<CatalogSettings>` or `CatalogSettings` directly.

**Connection strings** — each module's DbContext gets its connection string from `ConnectionStrings:{ModuleName}`. All modules can share the same connection string (same database, different schemas) or use different databases.

---

## Dependency Injection Registration Pattern

Every module implements `IModule` from Shared. This interface has two methods: one for registering services, one for registering endpoints.

**`IModule` interface** (in `Shared/Abstractions/`):

```
IModule
├── RegisterServices(IServiceCollection services, IConfiguration configuration)
├── RegisterEndpoints(IEndpointRouteBuilder endpoints)
```

**What `CatalogModule.cs` registers in `RegisterServices`:**

- `CatalogDbContext` with the connection string from configuration
- Repository implementations: `IProductRepository` → `ProductRepository`
- Service implementations: `ICatalogService` → `CatalogService`
- Validators: `CreateProductValidator`, `UpdateProductValidator`
- Module-specific settings: `CatalogSettings` from configuration section

**What `CatalogModule.cs` registers in `RegisterEndpoints`:**

- All endpoint route mappings grouped under a common prefix: `/api/catalog`
- Each endpoint file defines its route, HTTP method, authorization policy, and handler

**How `Program.cs` discovers modules:**

The Host scans assemblies for all types implementing `IModule`, instantiates them, and calls both registration methods. This can be done with assembly scanning or by explicitly listing modules — explicit listing is simpler and avoids reflection surprises.

```
Program.cs calls:
├── CatalogModule.RegisterServices(services, configuration)
├── OrdersModule.RegisterServices(services, configuration)
├── PaymentsModule.RegisterServices(services, configuration)
├── CustomersModule.RegisterServices(services, configuration)
├── NotificationsModule.RegisterServices(services, configuration)
├── AuthModule.RegisterServices(services, configuration)
│
then after app.Build():
├── CatalogModule.RegisterEndpoints(app)
├── OrdersModule.RegisterEndpoints(app)
├── ... (same for all modules)
```

---

## Endpoint Organization

Endpoints use ASP.NET minimal APIs. Each feature gets its own endpoint file. Endpoints are thin — they receive the HTTP request, call the service, and return the response. No business logic lives in endpoints.

**Route convention:** Each module prefixes its routes with `/api/{module-name}`. Catalog endpoints start with `/api/catalog`, Orders with `/api/orders`. This is set up in `RegisterEndpoints` within the module registration file.

**One endpoint file, one route:**

```
Catalog.Core/Endpoints/
├── GetProducts/
│   ├── GetProductsEndpoint.cs          ← GET /api/catalog/products
│   └── GetProductsResponse.cs          ← response shape specific to this endpoint
├── GetProductById/
│   ├── GetProductByIdEndpoint.cs       ← GET /api/catalog/products/{id}
│   └── ProductDetailResponse.cs
├── CreateProduct/
│   └── CreateProductEndpoint.cs        ← POST /api/catalog/products
├── UpdateProduct/
│   └── UpdateProductEndpoint.cs        ← PUT /api/catalog/products/{id}
└── DeleteProduct/
    └── DeleteProductEndpoint.cs        ← DELETE /api/catalog/products/{id}
```

**What an endpoint does:**

1. Defines the route, HTTP method, and authorization requirements
2. Receives the request (body, route params, query params)
3. Calls the service method (e.g., `ICatalogService.CreateProduct(dto)`)
4. Receives an `ApiResult<T>` from the service
5. Maps the `ApiResult.Code` to an HTTP status code
6. Returns the response

**Endpoint-specific response DTOs vs Contracts DTOs:** Endpoints can define their own response shapes in the endpoint folder (e.g., `GetProductsResponse.cs` with only the fields the list view needs) or reuse DTOs from Contracts/Core. Use endpoint-specific DTOs when the endpoint needs a different shape than what the service returns — for example, a list endpoint that returns fewer fields than a detail endpoint.

---

## Mapping Strategy

Mapping transforms domain entities into DTOs and vice versa. You have two approaches, and you can mix them within the same project.

**Convention-based mapping libraries** — Libraries like AutoMapper or Mapster reduce boilerplate by automatically mapping properties with matching names. They work well for simple, flat mappings where the entity and DTO shapes are similar. Use whichever library your team prefers — the architecture doesn't depend on a specific one.

**Manual mapping** — Write the mapping yourself as a static method, extension method, or a dedicated mapper class. No library dependency, full control, easy to debug. Each mapping is explicit — you see exactly what goes where.

**When to use which:**

- **Library-based mapping** works well for straightforward conversions where most properties have the same name and type: `Product` → `ProductDto`
- **Manual mapping** is better when the mapping involves logic (conditional fields, computed values, combining data from multiple sources), when the entity and DTO shapes are significantly different, or when you want zero external dependencies

**The pragmatic approach:** Start with manual mapping. It's explicit, has zero dependencies, and is easy for any developer to understand. Introduce a mapping library only when you notice repetitive boilerplate across many mappings in a module. You can use a library in one module and manual mapping in another — the choice is per-module, not system-wide.

**Where mappings live:** In `Core/Mappings/` for modules that use a mapping library. For manual mapping, the logic can live in the service methods directly, in static methods on the DTO class, or in a dedicated mapper class in `Core/Mappings/`.

---

## EF Core Organization

### One DbContext per module

Each module has its own `DbContext`. Catalog has `CatalogDbContext`, Orders has `OrdersDbContext`, Payments has `PaymentsDbContext`. They can point to the same physical database but use different schemas, or they can point to different databases entirely.

### Entity configurations in Core

EF Core entity-to-table mappings live in `Core/Persistence/Configurations/`. Each entity gets its own configuration file implementing `IEntityTypeConfiguration<T>`. This keeps the DbContext class clean — it just registers configurations, it doesn't contain mapping logic.

### Migrations per module

Each module has its own migration history in `Core/Persistence/Migrations/`. When you run `dotnet ef migrations add`, you specify which DbContext to target. This means modules can evolve their database schemas independently.

### Schema separation

For modules sharing a physical database, use EF Core schema configuration to separate tables:

- Catalog entities → `catalog` schema (`catalog.Products`, `catalog.Categories`)
- Orders entities → `orders` schema (`orders.Orders`, `orders.OrderItems`)
- Payments entities → `payments` schema (`payments.Payments`)

This prevents table name collisions and makes ownership immediately visible in the database.

### Global query filters for tenancy

Each module's `DbContext` applies a global query filter for tenant isolation in `OnModelCreating`. This ensures every query automatically filters by tenant without manual `.Where()` clauses.

### No cross-module joins

Because each module has its own DbContext, you cannot write a LINQ query that joins Products with Orders. If the Orders module needs product data, it calls `ICatalogService` — not the database. This is intentional: it enforces the module boundary at the data level.

---

## Logging and Observability

### Structured logging

Use structured logging (Serilog is the most common choice in .NET) so log entries are queryable, not just readable. Every log entry should include: timestamp, log level, message, and contextual properties.

### Correlation IDs

Every incoming HTTP request should get a unique correlation ID (generated in middleware or from a request header if the client sends one). This ID is attached to every log entry, every cross-module service call, and every database operation within that request. When something fails, you search logs by correlation ID and see the entire request flow across modules.

**Where it lives:** `Host/Middleware/RequestLoggingMiddleware.cs` generates or reads the correlation ID and stores it in the DI scope. Services access it through a shared `ICorrelationIdProvider` in Shared (or through `IHttpContextAccessor`).

### What to log

| What | Where | Level |
|---|---|---|
| Every HTTP request and response (method, path, status, duration) | `RequestLoggingMiddleware` | Information |
| Cross-module service calls (which module called which, duration) | Service layer | Debug |
| Validation failures | Validators / middleware | Warning |
| Business rule violations (insufficient stock, payment declined) | Services | Warning |
| Unhandled exceptions | `ExceptionHandlingMiddleware` | Error |
| External API calls (Stripe, SendGrid — request, response, duration) | Service layer | Information |
| Database query performance (slow queries) | EF Core logging | Warning |

### What NOT to log

- Passwords, tokens, API keys, or any credentials
- Full credit card numbers or sensitive PII
- Request/response bodies in production (unless redacted) — log them in development only
- Health check endpoints (they generate noise)

---

## Testing Strategy

### Domain tests (complex modules only)

Test business rules in isolation. No mocking needed — Domain has no dependencies. Test entity behavior: "when I call `Product.ApplyDiscount(20)`, the price decreases by 20%." Test value object validation: "creating `Money(-5, "USD")` throws `ArgumentException`."

Location: `tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain.Tests/`

### Core tests (complex modules)

Test use case orchestration. Mock repository interfaces (defined in `Core/Interfaces/`). Test that `CatalogService.CreateProduct()` validates input, creates the entity, calls the repository, and returns the correct DTO.

Location: `tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Core.Tests/`

### Module integration tests

Test the module end-to-end with a real (in-memory or containerized) database. Test that endpoints return correct responses, that EF Core mappings work, that repository queries return expected results.

Location: `tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Integration.Tests/` or `tests/Modules/EcommerceApp.Modules.Payments.Tests/`

### Cross-module integration tests

Test interactions between modules. Test that placing an order correctly calls `ICatalogService` and `IPaymentService`, and that the full flow produces the expected results.

Location: `tests/EcommerceApp.Integration.Tests/`

---

## Request → Response Workflows

### Workflow 1: Create Product (single module)

**Scenario:** `POST /api/catalog/products` — create a new product. Stays within the Catalog module.

```
1. REQUEST ARRIVES
   └── Host/Middleware/TenantResolutionMiddleware.cs           → resolves tenant
   └── Host/Middleware/ExceptionHandlingMiddleware.cs           → wraps pipeline

2. ROUTING
   └── Catalog.Core/Endpoints/CreateProduct/
       └── CreateProductEndpoint.cs                            → receives HTTP request, extracts body

3. VALIDATION
   └── Catalog.Core/Validators/CreateProductValidator.cs       → validates CreateProductDto

4. USE CASE ORCHESTRATION
   └── Catalog.Core/Services/CatalogService.cs                 → maps DTO → entity, applies business rules

5. DOMAIN LOGIC
   └── Catalog.Domain/Entities/Product.cs                      → constructor enforces invariants
   └── Catalog.Domain/ValueObjects/Money.cs                    → validates price amount + currency

6. PERSISTENCE
   └── Catalog.Core/Persistence/Repositories/ProductRepository.cs → calls CatalogDbContext.Products.Add()
   └── Catalog.Core/Persistence/CatalogDbContext.cs            → SaveChanges(), tenant filter applied

7. MAPPING
   └── Catalog.Core/Mappings/CatalogMappingProfile.cs          → Product entity → ProductDto

8. RESPONSE
   └── CreateProductEndpoint returns ProductDto as HTTP 201
```

### Workflow 2: Place Order (cross-module)

**Scenario:** `POST /api/orders` — place an order. Orders calls Catalog and Payments synchronously.

```
1. REQUEST ARRIVES
   └── Host/Middleware/TenantResolutionMiddleware.cs              → resolves tenant

2. ROUTING
   └── Orders.Core/Endpoints/PlaceOrder/
       └── PlaceOrderEndpoint.cs                                 → receives request

3. VALIDATION
   └── Orders.Core/Validators/PlaceOrderValidator.cs             → validates input

4. CROSS-MODULE: get product prices
   └── Catalog.Contracts/Interfaces/ICatalogService.cs           → interface
   └── Orders.Core calls ICatalogService.GetProductsByIds() via DI
   └── Catalog.Core/Services/CatalogService.cs                   → executes internally

5. BUSINESS LOGIC
   └── Orders.Core/Services/OrderService.cs                      → creates order, calculates totals
   └── Orders.Domain/Entities/Order.cs                           → domain invariants

6. CROSS-MODULE: process payment
   └── Payments.Contracts/Interfaces/IPaymentService.cs          → interface
   └── Orders.Core calls IPaymentService.ProcessPayment() via DI

7. PERSISTENCE
   └── Orders.Core/Persistence/OrdersDbContext.cs                → SaveChanges()

8. CROSS-MODULE: reduce stock
   └── Orders.Core calls ICatalogService.ReduceStock() via DI

9. RESPONSE
   └── PlaceOrderEndpoint returns OrderDto as HTTP 201
```

---

## How to Add a New Module

Step-by-step checklist for adding a new module. Example: adding a **Shipping** module as a simple module (Contracts + Core).

**1. Create projects and references:**

```bash
dotnet new classlib -n EcommerceApp.Modules.Shipping.Contracts -o src/Modules/Shipping/EcommerceApp.Modules.Shipping.Contracts
dotnet sln add src/Modules/Shipping/EcommerceApp.Modules.Shipping.Contracts

dotnet new classlib -n EcommerceApp.Modules.Shipping.Core -o src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core
dotnet sln add src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core

dotnet add src/Modules/Shipping/EcommerceApp.Modules.Shipping.Contracts reference src/EcommerceApp.Shared
dotnet add src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core reference src/Modules/Shipping/EcommerceApp.Modules.Shipping.Contracts
dotnet add src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core reference src/EcommerceApp.Shared
dotnet add src/EcommerceApp.Host reference src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core
```

**2. Add cross-module references** (only if Shipping needs to call other modules):

```bash
dotnet add src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core reference src/Modules/Orders/EcommerceApp.Modules.Orders.Contracts
```

**3. Create the folder structure in Contracts:**

- `DTOs/ShippingRateDto.cs`, `ShipmentDto.cs`
- `Interfaces/IShippingService.cs`

**4. Create the folder structure in Core:**

- `Domain/Shipment.cs`, `ShippingCarrier.cs`
- `Services/ShippingService.cs` (implements `IShippingService`)
- `Endpoints/GetShippingRates/GetShippingRatesEndpoint.cs`
- `Endpoints/CreateShipment/CreateShipmentEndpoint.cs`
- `Persistence/ShippingDbContext.cs`
- `Persistence/Configurations/ShipmentConfiguration.cs`
- `ShippingModule.cs` (implements `IModule`)

**5. Register the module** in Host's `Program.cs` — add `ShippingModule` to the module list.

**6. Add configuration** in `appsettings.json` under `Modules:Shipping` and `ConnectionStrings:Shipping`.

**7. Create and run migrations:**

```bash
dotnet ef migrations add InitialShipping --project src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core --startup-project src/EcommerceApp.Host --context ShippingDbContext
dotnet ef database update --project src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core --startup-project src/EcommerceApp.Host --context ShippingDbContext
```

**8. Add tests:**

```bash
dotnet new xunit -n EcommerceApp.Modules.Shipping.Tests -o tests/Modules/EcommerceApp.Modules.Shipping.Tests
dotnet sln add tests/Modules/EcommerceApp.Modules.Shipping.Tests
dotnet add tests/Modules/EcommerceApp.Modules.Shipping.Tests reference src/Modules/Shipping/EcommerceApp.Modules.Shipping.Core
```

If the module grows complex enough to warrant a Domain project, follow the graduation process described in [When a Module Should Graduate from Simple to Complex](#when-a-module-should-graduate-from-simple-to-complex).

---

## When a Module Should Graduate from Simple to Complex

A module starts as Contracts + Core (simple). Consider adding a Domain project (Contracts + Domain + Core) when:

- **The service class exceeds ~300 lines** — it's doing too much and needs separation between orchestration (Core) and business rules (Domain)
- **You have value objects or domain invariants** — these deserve a pure Domain layer with no dependencies
- **You want to unit test business rules without mocking infrastructure** — a Domain layer with zero dependencies makes this trivial
- **A different team takes ownership** — the internal layering gives them clear boundaries to work within

The transition is safe: create a Domain project, move entities, value objects, enums, and exceptions into it, update `.csproj` references, and nothing outside the module changes. Contracts stays exactly the same, so no other module is affected.

---

## Common Mistakes

**Putting module-specific types in Shared.** If only one module uses it, it belongs in that module. Shared is for cross-cutting concerns used by three or more modules.

**Exposing domain entities in Contracts.** Contracts should only contain DTOs and service interfaces. If `Product.cs` (the entity) appears in Contracts, other modules can depend on your internal domain model — and any change to the entity breaks them.

**Referencing another module's Core or Infrastructure.** This is the cardinal rule violation. If you need something from another module, it must be exposed through Contracts. If it's not in Contracts and you need it, either add it to Contracts or reconsider your module boundaries.

**Fat Shared project.** If Shared has 30+ files, audit it. Most of those files probably belong in a specific module. Shared should have fewer than 15 files in a typical system.

**Skipping the interface in Core for repositories.** If `CatalogService` directly instantiates `CatalogDbContext` instead of depending on `IProductRepository`, you lose testability — you can't mock the data access layer without an interface. Repository interfaces in `Core/Interfaces/` are the one internal abstraction that consistently earns its place.

**Over-abstracting external services.** Creating `IPaymentGateway` + `StripePaymentGateway` + `PayPalPaymentGateway` when you only use Stripe adds complexity without value. Services can call external SDKs directly. Only introduce an abstraction when you have a real need to swap providers at runtime (e.g., multi-tenant systems where different tenants use different payment providers). Don't abstract what you aren't actually swapping.

**Identical DTOs in Contracts and Application.** The DTOs serve different purposes. `ProductDto` in Contracts is the cross-module data shape — stable, versioned carefully. `CreateProductDto` in Application is the internal input shape — can change freely without affecting other modules. Don't conflate them.

**Cross-module database joins.** If you find yourself wanting to join Catalog's `Products` table with Orders' `OrderItems` table, you're violating module data ownership. Use an interface call to get the data you need.

**One giant migration history.** Each module should have its own migrations for its own DbContext. Don't put all migrations in one shared folder.

---

## Setting Up the Solution (dotnet CLI)

These commands create the full solution structure from scratch. Run them from the root directory where you want the solution to live.

### Create the solution and folder structure

```bash
# Create the solution
dotnet new sln -n EcommerceApp

# Create the folder structure
mkdir -p src/Modules/Catalog
mkdir -p src/Modules/Orders
mkdir -p src/Modules/Payments
mkdir -p src/Modules/Customers
mkdir -p src/Modules/Notifications
mkdir -p tests/Modules/Catalog
mkdir -p tests/Modules/Orders
```

### Create the Host and Shared projects

```bash
# Host (ASP.NET Web API)
dotnet new webapi -n EcommerceApp.Host -o src/EcommerceApp.Host
dotnet sln add src/EcommerceApp.Host

# Shared (class library)
dotnet new classlib -n EcommerceApp.Shared -o src/EcommerceApp.Shared
dotnet sln add src/EcommerceApp.Shared
```

### Create a complex module (Catalog — 3 projects)

```bash
# Contracts
dotnet new classlib -n EcommerceApp.Modules.Catalog.Contracts -o src/Modules/Catalog/EcommerceApp.Modules.Catalog.Contracts
dotnet sln add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Contracts

# Domain
dotnet new classlib -n EcommerceApp.Modules.Catalog.Domain -o src/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain
dotnet sln add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain

# Core
dotnet new classlib -n EcommerceApp.Modules.Catalog.Core -o src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core
dotnet sln add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core
```

### Create another complex module (Orders — 3 projects)

```bash
dotnet new classlib -n EcommerceApp.Modules.Orders.Contracts -o src/Modules/Orders/EcommerceApp.Modules.Orders.Contracts
dotnet sln add src/Modules/Orders/EcommerceApp.Modules.Orders.Contracts

dotnet new classlib -n EcommerceApp.Modules.Orders.Domain -o src/Modules/Orders/EcommerceApp.Modules.Orders.Domain
dotnet sln add src/Modules/Orders/EcommerceApp.Modules.Orders.Domain

dotnet new classlib -n EcommerceApp.Modules.Orders.Core -o src/Modules/Orders/EcommerceApp.Modules.Orders.Core
dotnet sln add src/Modules/Orders/EcommerceApp.Modules.Orders.Core
```

### Create simple modules (2 projects each)

```bash
# Payments
dotnet new classlib -n EcommerceApp.Modules.Payments.Contracts -o src/Modules/Payments/EcommerceApp.Modules.Payments.Contracts
dotnet sln add src/Modules/Payments/EcommerceApp.Modules.Payments.Contracts

dotnet new classlib -n EcommerceApp.Modules.Payments.Core -o src/Modules/Payments/EcommerceApp.Modules.Payments.Core
dotnet sln add src/Modules/Payments/EcommerceApp.Modules.Payments.Core

# Customers
dotnet new classlib -n EcommerceApp.Modules.Customers.Contracts -o src/Modules/Customers/EcommerceApp.Modules.Customers.Contracts
dotnet sln add src/Modules/Customers/EcommerceApp.Modules.Customers.Contracts

dotnet new classlib -n EcommerceApp.Modules.Customers.Core -o src/Modules/Customers/EcommerceApp.Modules.Customers.Core
dotnet sln add src/Modules/Customers/EcommerceApp.Modules.Customers.Core

# Notifications
dotnet new classlib -n EcommerceApp.Modules.Notifications.Contracts -o src/Modules/Notifications/EcommerceApp.Modules.Notifications.Contracts
dotnet sln add src/Modules/Notifications/EcommerceApp.Modules.Notifications.Contracts

dotnet new classlib -n EcommerceApp.Modules.Notifications.Core -o src/Modules/Notifications/EcommerceApp.Modules.Notifications.Core
dotnet sln add src/Modules/Notifications/EcommerceApp.Modules.Notifications.Core
```

### Add project references

```bash
# Host references
dotnet add src/EcommerceApp.Host reference src/EcommerceApp.Shared
dotnet add src/EcommerceApp.Host reference src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core
dotnet add src/EcommerceApp.Host reference src/Modules/Orders/EcommerceApp.Modules.Orders.Core
dotnet add src/EcommerceApp.Host reference src/Modules/Payments/EcommerceApp.Modules.Payments.Core
dotnet add src/EcommerceApp.Host reference src/Modules/Customers/EcommerceApp.Modules.Customers.Core
dotnet add src/EcommerceApp.Host reference src/Modules/Notifications/EcommerceApp.Modules.Notifications.Core

# Catalog (complex module)
dotnet add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Contracts reference src/EcommerceApp.Shared
dotnet add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core reference src/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain
dotnet add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core reference src/Modules/Catalog/EcommerceApp.Modules.Catalog.Contracts
dotnet add src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core reference src/EcommerceApp.Shared
# Domain references NOTHING — this is enforced by not adding any reference

# Orders (complex module)
dotnet add src/Modules/Orders/EcommerceApp.Modules.Orders.Contracts reference src/EcommerceApp.Shared
dotnet add src/Modules/Orders/EcommerceApp.Modules.Orders.Core reference src/Modules/Orders/EcommerceApp.Modules.Orders.Domain
dotnet add src/Modules/Orders/EcommerceApp.Modules.Orders.Core reference src/Modules/Orders/EcommerceApp.Modules.Orders.Contracts
dotnet add src/Modules/Orders/EcommerceApp.Modules.Orders.Core reference src/EcommerceApp.Shared
# Orders.Core needs Catalog and Payments Contracts for cross-module calls
dotnet add src/Modules/Orders/EcommerceApp.Modules.Orders.Core reference src/Modules/Catalog/EcommerceApp.Modules.Catalog.Contracts
dotnet add src/Modules/Orders/EcommerceApp.Modules.Orders.Core reference src/Modules/Payments/EcommerceApp.Modules.Payments.Contracts

# Payments (simple module)
dotnet add src/Modules/Payments/EcommerceApp.Modules.Payments.Contracts reference src/EcommerceApp.Shared
dotnet add src/Modules/Payments/EcommerceApp.Modules.Payments.Core reference src/Modules/Payments/EcommerceApp.Modules.Payments.Contracts
dotnet add src/Modules/Payments/EcommerceApp.Modules.Payments.Core reference src/EcommerceApp.Shared

# Customers (simple module)
dotnet add src/Modules/Customers/EcommerceApp.Modules.Customers.Contracts reference src/EcommerceApp.Shared
dotnet add src/Modules/Customers/EcommerceApp.Modules.Customers.Core reference src/Modules/Customers/EcommerceApp.Modules.Customers.Contracts
dotnet add src/Modules/Customers/EcommerceApp.Modules.Customers.Core reference src/EcommerceApp.Shared

# Notifications (simple module)
dotnet add src/Modules/Notifications/EcommerceApp.Modules.Notifications.Contracts reference src/EcommerceApp.Shared
dotnet add src/Modules/Notifications/EcommerceApp.Modules.Notifications.Core reference src/Modules/Notifications/EcommerceApp.Modules.Notifications.Contracts
dotnet add src/Modules/Notifications/EcommerceApp.Modules.Notifications.Core reference src/EcommerceApp.Shared
```

### Create test projects

```bash
# Complex module tests
dotnet new xunit -n EcommerceApp.Modules.Catalog.Domain.Tests -o tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain.Tests
dotnet sln add tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain.Tests
dotnet add tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain.Tests reference src/Modules/Catalog/EcommerceApp.Modules.Catalog.Domain

dotnet new xunit -n EcommerceApp.Modules.Catalog.Core.Tests -o tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Core.Tests
dotnet sln add tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Core.Tests
dotnet add tests/Modules/Catalog/EcommerceApp.Modules.Catalog.Core.Tests reference src/Modules/Catalog/EcommerceApp.Modules.Catalog.Core

# Simple module tests
dotnet new xunit -n EcommerceApp.Modules.Payments.Tests -o tests/Modules/EcommerceApp.Modules.Payments.Tests
dotnet sln add tests/Modules/EcommerceApp.Modules.Payments.Tests
dotnet add tests/Modules/EcommerceApp.Modules.Payments.Tests reference src/Modules/Payments/EcommerceApp.Modules.Payments.Core

# Cross-module integration tests
dotnet new xunit -n EcommerceApp.Integration.Tests -o tests/EcommerceApp.Integration.Tests
dotnet sln add tests/EcommerceApp.Integration.Tests
```

### Verify the solution builds

```bash
dotnet build
```

After running these commands, you have the full skeleton ready. Delete the auto-generated `Class1.cs` files from each class library and start adding your actual classes following the folder structures described in this document.

---

## Adopting the Events Approach

This document uses the **interface approach** (synchronous, direct calls) as the default throughout. This section explains how to adopt the **events approach** (asynchronous, decoupled) — either fully or selectively — and provides the complete folder structure, infrastructure needed, migration steps, and failure handling strategy.

The events approach doesn't replace interfaces entirely. Most real-world systems use both: interfaces for reads and queries (where you need a response), events for side effects and reactions (where you're announcing something happened). The goal of this section is to give you everything you need to introduce events into the architecture described in this document.

### Why Move to Events

The interface approach works well, but it creates **direct coupling** between modules. When `OrderService` calls `ICatalogService.ReduceStock()`, Orders depends on Catalog's contract for that specific operation. If three modules need to react when an order is placed, OrderService ends up with three injected dependencies and three sequential calls.

Events invert this relationship. Orders publishes `OrderPlacedEvent` and doesn't know or care who reacts. Catalog, Payments, and Notifications each decide independently how to handle it. This gives you:

- **Decoupled writes** — the publisher doesn't depend on the consumers
- **One-to-many reactions** — one event, unlimited handlers, no publisher changes
- **Independent evolution** — adding a new reaction (e.g., analytics tracking) means adding a handler in one module, touching zero code in the publishing module
- **Cleaner module Contracts** — fewer methods on service interfaces, because write operations move to event handlers
- **Easier path to microservices** — events map naturally to message queues when you eventually extract modules

### What Changes Structurally

Compared to the interface-only architecture described in this document, the events approach adds three things:

**1. Event bus infrastructure in Shared** — `IEventBus` interface and `InProcessEventBus` implementation. This is the one-time setup cost.

**2. Events folder in each module's Contracts** — Integration event classes (`OrderPlacedEvent`, `PaymentCompletedEvent`) that define what happened and carry the data other modules need to react.

**3. EventHandlers folder in each module's Infrastructure or Core** — Handler classes that subscribe to other modules' events and execute the reaction logic.

Everything else stays the same: project references, module boundaries, Clean Architecture layers inside complex modules, the Host, the Shared project.

### Folder Structure with Events

This is the complete solution structure with the events approach applied. Compare it to the interface-only structure earlier in this document — the additions are annotated.

```
EcommerceApp.sln
│
├── src/
│   ├── EcommerceApp.Host/
│   │   ├── Program.cs                              ← also registers IEventBus and discovers event handlers
│   │   ├── appsettings.json
│   │   └── Middleware/
│   │       ├── ExceptionHandlingMiddleware.cs
│   │       ├── TenantResolutionMiddleware.cs
│   │       └── RequestLoggingMiddleware.cs
│   │
│   ├── EcommerceApp.Shared/
│   │   ├── Abstractions/
│   │   │   ├── IModule.cs
│   │   │   ├── IEventBus.cs                        ← NEW: publish/subscribe contract
│   │   │   ├── IEventHandler.cs                    ← NEW: marker interface for event handlers
│   │   │   └── ICurrentUserService.cs
│   │   ├── EventBus/
│   │   │   └── InProcessEventBus.cs                ← NEW: in-memory implementation
│   │   ├── Multitenancy/
│   │   │   ├── ITenantProvider.cs
│   │   │   ├── TenantProvider.cs
│   │   │   └── TenantContext.cs
│   │   ├── Auth/
│   │   │   ├── JwtTokenGenerator.cs
│   │   │   └── JwtSettings.cs
│   │   ├── Persistence/
│   │   │   ├── BaseEntity.cs
│   │   │   └── IAuditableEntity.cs
│   │   └── Exceptions/
│   │       ├── NotFoundException.cs
│   │       └── ForbiddenException.cs
│   │
│   └── Modules/
│       │
│       ├── Catalog/  ← COMPLEX MODULE with events
│       │   │
│       │   ├── EcommerceApp.Modules.Catalog.Contracts/
│       │   │   ├── DTOs/
│       │   │   │   ├── ProductDto.cs
│       │   │   │   ├── ProductSummaryDto.cs
│       │   │   │   ├── CategoryDto.cs
│       │   │   │   └── PriceQuoteDto.cs
│       │   │   ├── Interfaces/
│       │   │   │   └── ICatalogService.cs           ← SLIMMER: write methods removed (now handled by events)
│       │   │   └── Events/                          ← NEW: integration events this module publishes
│       │   │       ├── ProductCreatedEvent.cs
│       │   │       ├── ProductPriceChangedEvent.cs
│       │   │       ├── ProductOutOfStockEvent.cs
│       │   │       └── StockReducedEvent.cs
│       │   │
│       │   ├── EcommerceApp.Modules.Catalog.Domain/
│       │   │   ├── Entities/
│       │   │   │   ├── Product.cs
│       │   │   │   ├── Category.cs
│       │   │   │   ├── ProductVariant.cs
│       │   │   │   └── ProductImage.cs
│       │   │   ├── ValueObjects/
│       │   │   │   ├── Money.cs
│       │   │   │   ├── Sku.cs
│       │   │   │   └── ProductDimensions.cs
│       │   │   ├── Enums/
│       │   │   │   ├── ProductStatus.cs
│       │   │   │   └── PricingTier.cs
│       │   │   └── Exceptions/
│       │   │       ├── InvalidPriceException.cs
│       │   │       ├── DuplicateSkuException.cs
│       │   │       └── ProductDiscontinuedException.cs
│       │   │
│       │   ├── EcommerceApp.Modules.Catalog.Application/
│       │   │   ├── Interfaces/
│       │   │   │   ├── IProductRepository.cs
│       │   │   │   └── ICategoryRepository.cs
│       │   │   ├── Services/
│       │   │   │   └── CatalogService.cs
│       │   │   ├── DTOs/
│       │   │   │   ├── CreateProductDto.cs
│       │   │   │   ├── UpdateProductDto.cs
│       │   │   │   ├── CreateCategoryDto.cs
│       │   │   │   └── ProductFilterDto.cs
│       │   │   ├── Validators/
│       │   │   │   ├── CreateProductValidator.cs
│       │   │   │   ├── UpdateProductValidator.cs
│       │   │   │   └── CreateCategoryValidator.cs
│       │   │   └── Mappings/
│       │   │       └── CatalogMappingProfile.cs
│       │   │
│       │   └── EcommerceApp.Modules.Catalog.Infrastructure/
│       │       ├── Persistence/
│       │       │   ├── CatalogDbContext.cs
│       │       │   ├── Configurations/
│       │       │   │   ├── ProductConfiguration.cs
│       │       │   │   ├── CategoryConfiguration.cs
│       │       │   │   ├── ProductVariantConfiguration.cs
│       │       │   │   └── ProductImageConfiguration.cs
│       │       │   ├── Repositories/
│       │       │   │   ├── ProductRepository.cs
│       │       │   │   └── CategoryRepository.cs
│       │       │   ├── Migrations/
│       │       │   └── Seeders/
│       │       │       └── CategorySeeder.cs
│       │       ├── EventHandlers/                   ← NEW: reacts to OTHER modules' events
│       │       │   ├── OrderPlacedReduceStockHandler.cs       ← listens to Orders.Contracts.Events
│       │       │   └── OrderCancelledRestoreStockHandler.cs   ← listens to Orders.Contracts.Events
│       │       ├── Endpoints/
│       │       │   ├── GetProducts/
│       │       │   │   ├── GetProductsEndpoint.cs
│       │       │   │   └── GetProductsResponse.cs
│       │       │   ├── GetProductById/
│       │       │   │   ├── GetProductByIdEndpoint.cs
│       │       │   │   └── ProductDetailResponse.cs
│       │       │   ├── CreateProduct/
│       │       │   │   └── CreateProductEndpoint.cs
│       │       │   ├── UpdateProduct/
│       │       │   │   └── UpdateProductEndpoint.cs
│       │       │   ├── DeleteProduct/
│       │       │   │   └── DeleteProductEndpoint.cs
│       │       │   └── Categories/
│       │       │       ├── GetCategoriesEndpoint.cs
│       │       │       └── CreateCategoryEndpoint.cs
│       │       └── CatalogModule.cs                 ← also registers event handlers
│       │
│       ├── Orders/  ← COMPLEX MODULE with events
│       │   │
│       │   ├── EcommerceApp.Modules.Orders.Contracts/
│       │   │   ├── DTOs/
│       │   │   │   ├── OrderDto.cs
│       │   │   │   └── OrderItemDto.cs
│       │   │   ├── Interfaces/
│       │   │   │   └── IOrderService.cs
│       │   │   └── Events/                          ← NEW: events this module publishes
│       │   │       ├── OrderPlacedEvent.cs
│       │   │       ├── OrderCancelledEvent.cs
│       │   │       ├── OrderCompletedEvent.cs
│       │   │       └── OrderShippedEvent.cs
│       │   │
│       │   ├── EcommerceApp.Modules.Orders.Domain/
│       │   │   ├── Entities/
│       │   │   │   ├── Order.cs
│       │   │   │   └── OrderItem.cs
│       │   │   ├── ValueObjects/
│       │   │   │   └── ShippingAddress.cs
│       │   │   └── Enums/
│       │   │       └── OrderStatus.cs
│       │   │
│       │   ├── EcommerceApp.Modules.Orders.Application/
│       │   │   ├── Interfaces/
│       │   │   │   └── IOrderRepository.cs
│       │   │   ├── Services/
│       │   │   │   └── OrderService.cs              ← publishes events instead of calling other modules' write methods
│       │   │   ├── DTOs/
│       │   │   │   ├── PlaceOrderDto.cs
│       │   │   │   └── CancelOrderDto.cs
│       │   │   ├── Validators/
│       │   │   │   └── PlaceOrderValidator.cs
│       │   │   └── Mappings/
│       │   │       └── OrderMappingProfile.cs
│       │   │
│       │   └── EcommerceApp.Modules.Orders.Infrastructure/
│       │       ├── Persistence/
│       │       │   ├── OrdersDbContext.cs
│       │       │   ├── Configurations/
│       │       │   │   └── OrderConfiguration.cs
│       │       │   ├── Repositories/
│       │       │   │   └── OrderRepository.cs
│       │       │   └── Migrations/
│       │       ├── EventHandlers/                   ← NEW: reacts to other modules' events
│       │       │   ├── PaymentCompletedHandler.cs            ← listens to Payments.Contracts.Events
│       │       │   └── PaymentFailedHandler.cs               ← listens to Payments.Contracts.Events
│       │       ├── Endpoints/
│       │       │   ├── PlaceOrder/
│       │       │   │   └── PlaceOrderEndpoint.cs
│       │       │   ├── CancelOrder/
│       │       │   │   └── CancelOrderEndpoint.cs
│       │       │   └── GetOrderById/
│       │       │       └── GetOrderByIdEndpoint.cs
│       │       └── OrdersModule.cs
│       │
│       ├── Payments/  ← SIMPLE MODULE with events
│       │   │
│       │   ├── EcommerceApp.Modules.Payments.Contracts/
│       │   │   ├── DTOs/
│       │   │   │   └── PaymentResultDto.cs
│       │   │   ├── Interfaces/
│       │   │   │   └── IPaymentService.cs
│       │   │   └── Events/                          ← NEW
│       │   │       ├── PaymentCompletedEvent.cs
│       │   │       ├── PaymentFailedEvent.cs
│       │   │       └── RefundProcessedEvent.cs
│       │   │
│       │   └── EcommerceApp.Modules.Payments.Core/
│       │       ├── Domain/
│       │       │   └── Payment.cs
│       │       ├── Services/
│       │       │   └── PaymentService.cs              ← calls Stripe SDK directly, no gateway abstraction
│       │       ├── Endpoints/
│       │       │   └── ProcessPayment/
│       │       │       └── ProcessPaymentEndpoint.cs
│       │       ├── EventHandlers/                   ← NEW
│       │       │   └── OrderPlacedProcessPaymentHandler.cs   ← listens to Orders.Contracts.Events
│       │       ├── Persistence/
│       │       │   ├── PaymentsDbContext.cs
│       │       │   └── Migrations/
│       │       └── PaymentsModule.cs
│       │
│       ├── Customers/  ← SIMPLE MODULE with events
│       │   │
│       │   ├── EcommerceApp.Modules.Customers.Contracts/
│       │   │   ├── DTOs/
│       │   │   │   └── CustomerDto.cs
│       │   │   ├── Interfaces/
│       │   │   │   └── ICustomerService.cs
│       │   │   └── Events/                          ← NEW
│       │   │       └── CustomerRegisteredEvent.cs
│       │   │
│       │   └── EcommerceApp.Modules.Customers.Core/
│       │       ├── Domain/
│       │       │   ├── Customer.cs
│       │       │   └── Address.cs
│       │       ├── Services/
│       │       │   └── CustomerService.cs
│       │       ├── Endpoints/
│       │       │   ├── RegisterCustomer/
│       │       │   │   └── RegisterCustomerEndpoint.cs
│       │       │   └── GetCustomerProfile/
│       │       │       └── GetCustomerProfileEndpoint.cs
│       │       ├── Persistence/
│       │       │   ├── CustomersDbContext.cs
│       │       │   └── Migrations/
│       │       └── CustomersModule.cs
│       │
│       └── Notifications/  ← SIMPLE MODULE, pure event consumer
│           │
│           ├── EcommerceApp.Modules.Notifications.Contracts/
│           │   ├── DTOs/
│           │   │   └── NotificationDto.cs
│           │   └── Interfaces/
│           │       └── INotificationService.cs
│           │
│           └── EcommerceApp.Modules.Notifications.Core/
│               ├── Domain/
│               │   ├── Notification.cs
│               │   └── NotificationTemplate.cs
│               ├── Services/
│               │   └── NotificationService.cs         ← calls SendGrid, Twilio SDKs directly
│               ├── Endpoints/
│               │   └── GetNotifications/
│               │       └── GetNotificationsEndpoint.cs
│               ├── EventHandlers/                   ← this module is MOSTLY event handlers
│               │   ├── OrderPlacedSendConfirmationHandler.cs
│               │   ├── OrderShippedSendTrackingHandler.cs
│               │   ├── PaymentFailedSendAlertHandler.cs
│               │   ├── CustomerRegisteredSendWelcomeHandler.cs
│               │   └── RefundProcessedSendNotificationHandler.cs
│               ├── Persistence/
│               │   ├── NotificationsDbContext.cs
│               │   └── Migrations/
│               └── NotificationsModule.cs
│
└── tests/
    ├── Modules/
    │   ├── Catalog/
    │   │   ├── EcommerceApp.Modules.Catalog.Domain.Tests/
    │   │   ├── EcommerceApp.Modules.Catalog.Application.Tests/
    │   │   └── EcommerceApp.Modules.Catalog.Integration.Tests/
    │   ├── Orders/
    │   │   ├── EcommerceApp.Modules.Orders.Domain.Tests/
    │   │   ├── EcommerceApp.Modules.Orders.Application.Tests/
    │   │   └── EcommerceApp.Modules.Orders.Integration.Tests/
    │   ├── EcommerceApp.Modules.Payments.Tests/
    │   ├── EcommerceApp.Modules.Customers.Tests/
    │   └── EcommerceApp.Modules.Notifications.Tests/
    └── EcommerceApp.Integration.Tests/
```

### Event Infrastructure in Shared

The event bus requires three components in Shared:

**`IEventBus`** (in `Shared/Abstractions/`) — The publish interface. Has a single method: `Publish<TEvent>(TEvent event)`. Modules inject this to publish events. They never reference the implementation.

**`IEventHandler<TEvent>`** (in `Shared/Abstractions/`) — The handler marker interface. Every event handler implements this with a `Handle(TEvent event)` method. The event bus discovers handlers through DI registration.

**`InProcessEventBus`** (in `Shared/EventBus/`) — The in-memory implementation. When `Publish()` is called, it resolves all `IEventHandler<TEvent>` implementations from the DI container and invokes them. For a monolith, this is sufficient. When you extract to microservices, you swap this implementation for `RabbitMqEventBus` or `AzureServiceBusEventBus` — no module code changes.

### Events in Contracts

Each module that publishes events adds an `Events/` folder to its Contracts project. An integration event is a simple DTO — it carries the data other modules need to react but has no behavior.

**Naming convention:** `{What happened}Event`. Past tense, because events describe something that already occurred: `OrderPlacedEvent`, `PaymentCompletedEvent`, `ProductPriceChangedEvent`.

**What an event carries:** The minimum data needed for handlers to react without calling back to the publisher. `OrderPlacedEvent` should include the order ID, customer ID, list of product IDs with quantities, and the total amount. If a handler needs more detail, it calls the publisher's service interface (a synchronous read) — but the event should cover 80% of cases.

**Ownership rule:** An event is defined in the Contracts of the module where the action happened. `OrderPlacedEvent` lives in Orders.Contracts because Orders is where the order was placed. Catalog doesn't define this event — it only consumes it.

### Event Handlers in Infrastructure and Core

Event handlers live in the consuming module — the module that reacts to someone else's event.

**In a complex module:** Handlers go in `Infrastructure/EventHandlers/` because they typically need access to the module's DbContext or repositories (which are Infrastructure concerns).

**In a simple module:** Handlers go in `Core/EventHandlers/` since Core has direct access to everything.

**Naming convention:** `{EventName}{WhatThisModuleDoes}Handler`. Describes the event being handled and the action being taken: `OrderPlacedReduceStockHandler` (Catalog reacts to order by reducing stock), `PaymentCompletedHandler` (Orders reacts to payment by updating order status), `OrderPlacedSendConfirmationHandler` (Notifications reacts to order by sending email).

**One handler per reaction.** If Catalog needs to do two things when an order is placed (reduce stock and update availability), use two handlers: `OrderPlacedReduceStockHandler` and `OrderPlacedUpdateAvailabilityHandler`. Each handler has a single responsibility and can fail independently.

**Project reference:** The handler's module references the publishing module's Contracts (to access the event class). The publishing module has no reference to the consuming module — it doesn't know handlers exist.

### Step-by-Step Migration from Interfaces to Events

This is how you convert a specific interaction from synchronous interface call to asynchronous event. Do this per interaction, not all at once.

**Before (interface approach):** `OrderService` calls `ICatalogService.ReduceStock(productId, quantity)` after placing an order.

**Migration steps:**

**Step 1 — Add event bus (one-time, if not already done).** Add `IEventBus` and `IEventHandler<T>` to `Shared/Abstractions/`. Add `InProcessEventBus` to `Shared/EventBus/`. Register in Host's `Program.cs`.

**Step 2 — Define the event.** Create `Orders.Contracts/Events/OrderPlacedEvent.cs`. Include all data Catalog needs: order ID, list of product IDs with quantities. This event may already exist if other interactions use it.

**Step 3 — Create the handler.** Create `Catalog.Infrastructure/EventHandlers/OrderPlacedReduceStockHandler.cs`. This handler implements `IEventHandler<OrderPlacedEvent>`. Move the stock reduction logic from wherever it was called synchronously into this handler. The handler injects `IProductRepository` (from Catalog.Application) and reduces stock.

**Step 4 — Register the handler.** In `CatalogModule.cs`, register `OrderPlacedReduceStockHandler` as an `IEventHandler<OrderPlacedEvent>` in the DI container.

**Step 5 — Swap the call.** In `OrderService`, replace `ICatalogService.ReduceStock(...)` with `IEventBus.Publish(new OrderPlacedEvent(...))`. If `ICatalogService` was injected only for `ReduceStock()`, remove the dependency entirely.

**Step 6 — Clean up Contracts.** If `ReduceStock()` on `ICatalogService` is no longer called by anyone, remove it from the interface. The interface gets slimmer — it now only exposes read operations that other modules call synchronously.

**What doesn't change:** The Contracts/Core/Domain/Application/Infrastructure boundaries. No `.csproj` references change (Catalog.Infrastructure already references Orders.Contracts, or if it doesn't, you add it — which is allowed). No other module is affected.

### What Stays Synchronous (Even with Events)

Not everything should become an event. These interactions remain as interface calls:

**Reads and queries.** When Orders needs to know the current price of a product, it calls `ICatalogService.GetProductsByIds()`. You can't "fire and forget" a question — you need the answer before continuing. All `Get*` methods on service interfaces stay synchronous.

**Validation that blocks the operation.** When Orders needs to verify a customer exists before placing an order, it calls `ICustomerService.GetCustomerById()`. If the customer doesn't exist, the order must be rejected. This cannot be asynchronous.

**Operations where immediate consistency is required.** If your business rules say "an order cannot be placed if stock is zero," and this check must be atomic, the stock check is a synchronous call. The stock *reduction* after the order is persisted can be an event.

**The resulting pattern:** Service interfaces (`ICatalogService`, `ICustomerService`) shrink to read-only operations. Write operations (reduce stock, process payment, send notification) move to event handlers. Interfaces become query surfaces, events become command channels.

### Handling Failures in Event Handlers

When an event handler fails, the primary operation (e.g., the order) has already succeeded and returned a response to the client. This is the fundamental trade-off of events: you gain decoupling but you lose immediate consistency.

**Retry strategy.** The `InProcessEventBus` should support retry logic — if `OrderPlacedReduceStockHandler` throws an exception, retry 2-3 times before marking it as failed. This handles transient errors (database timeout, brief network issue).

**Dead letter handling.** After retries are exhausted, the failed event should be logged to a dead letter store (a database table like `FailedEvents` with the event data, error message, and timestamp). An admin or background job can review and replay failed events.

**Compensating actions.** For critical failures (payment processing fails after order is persisted), the system needs a compensating action: publish a `PaymentFailedEvent` that Orders handles by updating the order status to "Payment Failed" or "Cancelled." This is eventual consistency — the system reaches the correct state, just not instantaneously.

**Idempotency.** Event handlers must be idempotent — processing the same event twice should produce the same result. If `OrderPlacedReduceStockHandler` runs twice for the same order, stock should only be reduced once. Use the event's order ID to check "have I already processed this?"

**Monitoring.** With events, you lose the linear call stack that makes debugging easy with interfaces. Invest in structured logging: log when events are published (with correlation ID), when handlers start and complete, and when handlers fail. A correlation ID that follows from the HTTP request through event publishing to handler execution is essential.

### Events Approach Common Mistakes

**Making everything an event.** If a module needs data to continue its operation, use an interface call. Events are for side effects and reactions, not for fetching data. "Get product prices" is a query, not an event.

**Fat events.** Including too much data in an event (the entire product entity, all customer details) couples consumers to the publisher's internal structure. Include IDs and the minimum data needed. Handlers that need more detail can make a synchronous read call.

**Forgetting idempotency.** If a handler processes `OrderPlacedEvent` twice because of a retry, and stock is reduced twice, you have a data integrity bug. Every handler must handle duplicate delivery.

**No failure visibility.** Events fail silently if you don't implement logging and dead letter handling. In production, a failed event handler is invisible unless you actively monitor for it. With interface calls, failures are immediately visible as HTTP 500 responses.

**Circular event chains.** Module A publishes event → Module B handles it and publishes another event → Module A handles that event and publishes another → infinite loop. Design events as reactions to business actions, not as triggers for other events that trigger more events. Keep chains short and linear.

**Publishing events before persisting.** Always persist the primary operation first, then publish the event. If `OrderService` publishes `OrderPlacedEvent` before calling `SaveChanges()`, and the save fails, handlers are already reacting to an order that doesn't exist.

---

## Glossary

**Bounded Context** — A DDD concept where a specific domain model applies. In this architecture, each module roughly maps to one bounded context.

**Catalog** — The product information authority. Owns product creation, descriptions, SKUs, variants, categories, and pricing. Answers "what can you buy and how much does it cost?"

**Orders** — The purchase lifecycle manager. Owns placing, tracking, and canceling orders. Receives product data from Catalog and payment confirmations from Payments.

**Payments** — The financial transaction handler. Owns payment gateway integration, transaction processing, and refund management.

**Customers** — The user identity and profile manager. Owns registration, profiles, addresses, and preferences.

**Notifications** — The communication dispatcher. Sends emails, SMS, and push notifications in reaction to events from other modules.

**Cart** — The pre-purchase staging area. Owns the temporary collection of items a customer intends to buy, before checkout hands off to Orders.

**Contracts** — A module's public API project. Contains DTOs and service interfaces that other modules may reference.

**Core** — A module's implementation project. In a simple module (2 projects), Core contains everything: domain, services, persistence, and endpoints. In a complex module (3 projects), Core contains services, repository interfaces, validators, data access, endpoints, and module registration — everything except domain logic and the public API.

**Domain Layer** — The innermost layer of a complex module. Contains entities, value objects, enums, and exceptions. Has zero dependencies. Only exists as a separate project in complex modules (3+ projects).

**Application Layer** (optional) — An additional layer that can be introduced between Domain and Core for extra-complex modules (4 projects). Contains service implementations, repository interfaces, validators, and mappings. References Domain and Contracts. Infrastructure then references Application. This follows standard Clean Architecture and is only needed when compiler-enforced separation between orchestration and data access is required.

**Value Object** — An object defined by its attributes (not identity), that is immutable and self-validating. Examples: `Money`, `Address`, `EmailAddress`.

**Integration Event** — A message published by one module to notify other modules that something happened. Defined in the publisher's Contracts, handled in consumers' Core. See [Adopting the Events Approach](#adopting-the-events-approach).

**Module Registration (IModule)** — An interface every module implements. The Host calls it at startup to register the module's services, DbContext, and endpoints.

**Global Query Filter** — An EF Core feature that automatically applies a `.Where()` clause to every query. Used for tenant isolation so developers don't manually filter by `TenantId`.
