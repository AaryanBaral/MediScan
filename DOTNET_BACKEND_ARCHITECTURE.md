# MediScan .NET 10 Enterprise Backend Architecture & AI Implementation Specification

> **Target Version**: .NET 10 (C# 14)  
> **Architecture Pattern**: Clean Architecture / Domain-Driven Design (DDD) + CQRS  
> **Database**: PostgreSQL 17+ via Entity Framework Core 10 (`Npgsql`)  
> **Status**: Living Architecture Blueprint & AI Operational Mandate  

---

## 1. Executive Summary & Senior Architect Directive

This document serves as the **master engineering contract** and **strict operational blueprint** for migrating the legacy Django/DRF backend (`legacy/Backend`) to a production-grade, high-performance, strictly typed **.NET 10 backend** for **MediScan**.

MediScan is a clinical-grade medical report translator, triage, and teleconsultation platform handling **Protected Health Information (PHI)**. The system processes laboratory reports, orchestrates OCR/AI model inference pipelines, manages doctor verification, provides time-slot scheduling, processes payments with revenue splitting (Khalti), and streams clinical explanations to patients and clinicians.

Every line of code generated for this backend must reflect the standards of a **Principal .NET Architect & Senior DevOps Engineer**.

---

## 2. AI Persona & Strict Rules of Engagement

When acting as the AI developer implementing this architecture, you **MUST** adhere to the following non-negotiables. Any violation of these principles is considered an architectural failure.

### 2.1 The "Strictly Forbidden" List
1. ❌ **NO MediatR or Third-Party Mediator Libraries**: You are strictly forbidden from adding `MediatR` or similar packages. All CQRS commands, queries, and handlers must run through the project's own custom in-process dispatcher (`IDispatcher`).
2. ❌ **NO AutoMapper, Mapster, or Reflection-Based Mappers**: Zero magic mappers. All mapping between Domain Entities, Value Objects, Application DTOs, and Presentation View Models must be performed via explicit, compile-time safe, static extension methods (`ToDto()`, `ToEntity()`, `ToResponse()`).
3. ❌ **NO Minimal APIs for Business Endpoints**: All HTTP endpoints must be implemented using ASP.NET Core `ControllerBase` controllers. Controllers must be clean, slim, and adhere to strict RESTful conventions using attribute routing.
4. ❌ **NO Business Logic in Controllers**: Controllers may only unpack HTTP requests, validate basic model state, dispatch commands or queries via `IDispatcher`, and return standardized `ActionResult<ApiResponse<T>>`.
5. ❌ **NO Leaking of EF Core `DbContext` Outside Infrastructure**: The `DbContext` must never be injected into Application handlers or Controllers. Repositories and `IUnitOfWork` encapsulate persistence.
6. ❌ **NO Leaking of Domain Entities to the Web Layer**: Controllers and HTTP responses must never return Domain Entities directly. Only Application DTOs or Presentation View Models are exposed.
7. ❌ **NO Plain Text Secrets or PHI in Logs**: Logging passwords, OTPs, auth tokens, or unmasked patient identifiers is strictly prohibited.
8. ❌ **NO Synchronous Blocking Calls**: Never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`. Every I/O operation must be `async/await` with `CancellationToken` propagated through the entire call stack.

### 2.2 Modern C# 14 & .NET 10 Coding Standards
- **File-scoped namespaces** exclusively: `namespace MediScan.Application.Reports.Commands;`
- **Nullable Reference Types** enabled across all projects (`<Nullable>enable</Nullable>`). Zero compiler nullability warnings allowed.
- **Primary Constructors** used for dependency injection across classes: `public class GetReportByIdQueryHandler(IReportRepository reportRepository) : IQueryHandler<...>`
- **Records & Readonly Structs**: Use `record` for CQRS requests/DTOs and `readonly record struct` for Domain Value Objects.
- **Pattern Matching & Switch Expressions**: Prefer modern pattern matching expressions over verbose `if/else` ladders.
- **Collection Expressions**: Use modern `[...]` collection syntax instead of `new List<T>()` or `new T[] { ... }`.
- **Compile-Time Logging**: Use `[LoggerMessage]` source generators for high-throughput, allocation-free structured logging.

---

## 3. Legacy (v1 Django) to .NET 10 Domain Port Map

| Legacy Django App | Legacy Python Component | .NET 10 Clean Architecture Destination | Core Responsibilities |
| :--- | :--- | :--- | :--- |
| `authentication` | `CustomUser`, `CustomUserManager` | `MediScan.Domain.Entities.User`, `Role` | Identity, RBAC, lockout, professional profile info. |
| `authentication` | `otp.py`, `email_backend.py` | `MediScan.Domain.Entities.EmailOtp`, `IEmailService` | Cryptographic OTP generation, expiry (5 min), rate-limiting. |
| `authentication` | `views.py` (JWT & Cookies) | `MediScan.Application.Authentication.*`, `ITokenService` | JWT access tokens, cryptographically secure refresh token rotation stored in Postgres. |
| `doctor` | `DoctorLicense`, `SupportingDocument` | `MediScan.Domain.Entities.DoctorLicense`, `DoctorDocument` | License verification lifecycle (`Pending`, `Approved`, `Rejected`), file uploads. |
| `doctor` | `DoctorAvailability` | `MediScan.Domain.Entities.DoctorAvailabilitySlot` | Weekly recurring schedules (0-6) and specific date overrides, time ranges. |
| `doctor` | `DoctorPatientLink` | `MediScan.Domain.Entities.DoctorPatientLink` | Clinical link between patient & doctor with clinical observations. |
| `doctor` | `Appointment` | `MediScan.Domain.Entities.Appointment` | Status: `PendingPayment` → `Paid` → `Completed` / `Cancelled`. |
| `doctor` | `khalti_views.py` | `MediScan.Infrastructure.ExternalServices.Payments.KhaltiClient` | Khalti payment initiation, transaction verification, refund tracking. |
| `doctor` | Revenue split logic | `MediScan.Domain.Entities.PaymentTransaction` | Revenue ledger: Doctor (75%), Platform/Admin fee (25%). |
| `reports` | `Report`, `ExtractedReportData` | `MediScan.Domain.Entities.Report`, `AnalyteObservation` | PDF upload, two-step verification (`Pending` → `Extracting` → `AwaitingReview` → `Assessing` → `Completed`). |
| `reports` | `ReportResult` | `MediScan.Domain.Entities.ReportResult` | Summary, doctor summary, key findings, detected conditions, LOINC ranges. |
| `reports` | `trends` action | `MediScan.Application.Reports.Queries.GetReportTrendsQuery` | Time-series projection of numerical biomarker trends per patient. |
| `chatbot` | `ChatStreamView` | `MediScan.Presentation.Controllers.ChatbotController` | Server-Sent Events (SSE) streaming proxy injecting patient report context to LLM Gateway. |
| `admin_panel` | Admin views & verification | `MediScan.Application.Administration.*` | Doctor approval/rejection workflows with audit trails and reasons. |

---

## 4. High-Level System & Network Architecture

```
                    ┌──────────────────────────────────────────────┐
                    │          Frontend (React 19 + Vite)          │
                    └──────────────────────┬───────────────────────┘
                                           │ HTTPS (REST API & SSE)
                                           ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 MediScan API Gateway                                   │
│                        (Reverse Proxy / Cloudflare / Nginx)                            │
└──────────────────────────────────────────┬─────────────────────────────────────────────┘
                                           │
                                           ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        MediScan .NET 10 Web API Host                                   │
│                                                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ Middlewares Pipeline:                                                            │  │
│  │  1. Correlation ID  2. Request Timing  3. Global Exception (RFC 9457)            │  │
│  │  4. Security Headers 5. Rate Limiter   6. JWT Authentication & Authorization      │  │
│  └───────────────────────────────────────┬──────────────────────────────────────────┘  │
│                                          │                                             │
│  ┌───────────────────────────────────────▼──────────────────────────────────────────┐  │
│  │ Presentation Layer: API Controllers (BaseApiController, Reports, Doctors, etc.)   │  │
│  └───────────────────────────────────────┬──────────────────────────────────────────┘  │
│                                          │ Dispatches Commands / Queries               │
│  ┌───────────────────────────────────────▼──────────────────────────────────────────┐  │
│  │ Application Layer: Custom In-Process CQRS Dispatcher                             │  │
│  │  Pipeline: LoggingBehavior ──▶ ValidationBehavior ──▶ TransactionBehavior         │  │
│  │  Handlers: Handlers execute domain logic, repositories, unit of work             │  │
│  └───────────────────────┬──────────────────────────────────┬───────────────────────┘  │
│                          │                                  │                          │
│  ┌───────────────────────▼──────────────────┐   ┌───────────▼───────────────────────┐  │
│  │ Domain Layer (Pure Enterprise Logic)     │   │ SharedKernel (Primitives & Core)  │  │
│  │  Entities, Aggregates, Domain Events     │   │  Result<T>, Error, IDispatcher    │  │
│  │  Value Objects, Specifications, Enums    │   │  Entity, AggregateRoot, IClock    │  │
│  └───────────────────────┬──────────────────┘   └───────────────────────────────────┘  │
│                          │                                                             │
│  ┌───────────────────────▼──────────────────────────────────────────────────────────┐  │
│  │ Infrastructure Layer:                                                            │  │
│  │  - EF Core 10 (PostgreSQL Npgsql with JSONB, Snake_Case, Interceptors)           │  │
│  │  - Repositories & Unit of Work                                                   │  │
│  │  - Redis Distributed Cache & Rate Limiting Storage                               │  │
│  │  - S3 / MinIO Object Storage (Presigned URLs, Blob Management)                   │  │
│  │  - Outbox Background Publisher & Channel Worker                                  │  │
│  │  - External Service Clients (Khalti, AI Inference, LLM Gateway, SMTP)            │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────┬──────────────────┬──────────────────┬──────────────────────────┘
                        │                  │                  │
        ┌───────────────▼───────┐  ┌───────▼──────┐   ┌───────▼──────────────────────────┐
        │ PostgreSQL 17 (DB)    │  │ Redis 7      │   │ MinIO / S3                       │
        │ - Relational Tables   │  │ - Cache      │   │ - Report PDFs                    │
        │ - JSONB Observations  │  │ - RateLimit  │   │ - Doctor License Docs            │
        │ - Outbox Table        │  │ - OTP Locks  │   │ - Presigned Upload / Download    │
        │ - AccessEvent Audit   │  └──────────────┘   └──────────────────────────────────┘
        └───────────────────────┘
```

---

## 5. Solution & Project Directory Structure

The solution must be strictly partitioned into independent projects following Clean Architecture rules:

```
MediScan.Backend/
├── MediScan.sln
├── Directory.Build.props                  # Common C# 14 / .NET 10 compiler & analyzer settings
├── Directory.Packages.props               # Central Package Management (CPM)
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml
│   └── certs/
├── src/
│   ├── MediScan.SharedKernel/             # Zero dependencies (pure abstractions & primitives)
│   │   ├── Common/
│   │   │   ├── Result.cs                  # Functional Result pattern (Result, Result<T>)
│   │   │   ├── Error.cs                   # Standardized Error and ErrorType records
│   │   │   ├── PagedList.cs               # Pagination wrapper
│   │   │   └── Ensure.cs                  # Guard clauses
│   │   ├── Primitives/
│   │   │   ├── Entity.cs                  # Generic Entity<TId>
│   │   │   ├── AggregateRoot.cs           # AggregateRoot<TId> with DomainEvents support
│   │   │   ├── ValueObject.cs             # ValueObject base class
│   │   │   └── IDomainEvent.cs            # In-process domain event contract
│   │   ├── CQRS/
│   │   │   ├── ICommand.cs                # ICommand<TResponse> contract
│   │   │   ├── ICommandHandler.cs        # Handler contract
│   │   │   ├── IQuery.cs                  # IQuery<TResponse> contract
│   │   │   ├── IQueryHandler.cs           # Handler contract
│   │   │   ├── IDispatcher.cs             # Central custom dispatcher interface
│   │   │   └── IPipelineBehavior.cs       # Middleware pipeline for CQRS handlers
│   │   └── Interfaces/
│   │       ├── IClock.cs                  # System clock abstraction (testable UTC time)
│   │       ├── IAuditableEntity.cs        # CreatedAt, CreatedBy, UpdatedAt, UpdatedBy
│   │       └── ISoftDeletable.cs          # IsDeleted, DeletedAtUtc, DeletedBy
│   │
│   ├── MediScan.Domain/                   # Depends ONLY on SharedKernel
│   │   ├── Aggregates/
│   │   │   ├── Users/                     # User, Role, DoctorProfile, PatientProfile
│   │   │   ├── Reports/                   # Report, AnalyteObservation, ExtractedData, ReportResult
│   │   │   ├── Consultations/             # DoctorPatientLink, Appointment, DoctorComment
│   │   │   └── Billing/                   # PaymentTransaction, RevenueSplit
│   │   ├── ValueObjects/
│   │   │   ├── Email.cs
│   │   │   ├── PhoneNumber.cs
│   │   │   ├── Money.cs
│   │   │   ├── TimeRange.cs
│   │   │   └── LoincCode.cs
│   │   ├── Enums/
│   │   │   ├── UserRole.cs                # Patient, Doctor, Admin
│   │   │   ├── DoctorStatus.cs            # Unverified, Pending, Verified, Rejected
│   │   │   ├── Specialization.cs          # Cardiologist, Nephrologist, Hepatologist, etc.
│   │   │   ├── ReportStatus.cs            # Pending, Extracting, AwaitingReview, Assessing, Completed, Failed
│   │   │   ├── RiskLevel.cs               # Low, Medium, High
│   │   │   └── AppointmentStatus.cs       # PendingPayment, Paid, Cancelled, Completed
│   │   ├── Events/
│   │   │   ├── UserRegisteredDomainEvent.cs
│   │   │   ├── ReportUploadedDomainEvent.cs
│   │   │   ├── ReportVerifiedDomainEvent.cs
│   │   │   ├── AppointmentBookedDomainEvent.cs
│   │   │   └── PaymentCompletedDomainEvent.cs
│   │   └── Exceptions/
│   │       ├── DomainException.cs
│   │       └── MedicalValidationException.cs
│   │
│   ├── MediScan.Application/              # Depends on Domain & SharedKernel
│   │   ├── Abstractions/
│   │   │   ├── Repositories/              # IRepository<T>, IUserRepository, IReportRepository, etc.
│   │   │   ├── Persistence/               # IUnitOfWork, IDbConnectionFactory
│   │   │   ├── Services/                  # ITokenService, IPasswordHasher, IEmailService, IBlobStorage
│   │   │   ├── External/                  # IKhaltiService, IOcrServiceClient, IInferenceClient, ILlmGateway
│   │   │   └── Security/                  # ICurrentUserContext, IAuditLogger
│   │   ├── Behaviors/                     # Custom CQRS Behaviors (Decorator/Chain)
│   │   │   ├── LoggingBehavior.cs
│   │   │   ├── ValidationBehavior.cs
│   │   │   └── TransactionBehavior.cs
│   │   ├── Dispatcher/
│   │   │   └── Dispatcher.cs              # Concrete custom CQRS dispatcher implementation
│   │   ├── Features/                      # Sliced by feature/subdomain
│   │   │   ├── Authentication/
│   │   │   │   ├── Commands/Register/
│   │   │   │   ├── Commands/VerifyOtp/
│   │   │   │   ├── Commands/Login/
│   │   │   │   ├── Commands/RefreshToken/
│   │   │   │   └── Queries/GetProfile/
│   │   │   ├── Reports/
│   │   │   │   ├── Commands/UploadReport/
│   │   │   │   ├── Commands/CorrectReportData/
│   │   │   │   ├── Commands/TriggerAnalysis/
│   │   │   │   ├── Queries/GetReportById/
│   │   │   │   └── Queries/GetReportTrends/
│   │   │   ├── Doctors/
│   │   │   ├── Appointments/
│   │   │   ├── Billing/
│   │   │   └── Chatbot/
│   │   └── Mappings/                      # Static compile-time manual mapping extensions
│   │       ├── UserMappings.cs
│   │       ├── ReportMappings.cs
│   │       └── AppointmentMappings.cs
│   │
│   ├── MediScan.Infrastructure/           # Implements Application interfaces
│   │   ├── Persistence/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   ├── Configurations/            # Fluent API Entity Configurations (IEntityTypeConfiguration)
│   │   │   ├── Interceptors/              # AuditableEntityInterceptor, DomainEventOutboxInterceptor
│   │   │   ├── Repositories/              # GenericRepository<T>, Specialized Repositories
│   │   │   ├── UnitOfWork.cs
│   │   │   └── Migrations/
│   │   ├── Outbox/                        # Transactional Outbox Pattern
│   │   │   ├── OutboxMessage.cs
│   │   │   └── OutboxBackgroundProcessor.cs
│   │   ├── Storage/                       # S3 / MinIO blob storage implementation
│   │   ├── Authentication/                # JWT TokenService, Argon2id PasswordHasher
│   │   ├── Caching/                       # Redis cache service wrapper
│   │   ├── ExternalServices/
│   │   │   ├── Khalti/                    # Khalti payment client with Polly v8 resilience
│   │   │   ├── PythonAI/                  # Typed HTTP client to FastAPI OCR & Inference
│   │   │   └── Email/                     # FluentEmail / SmtpClient implementation
│   │   └── Audit/
│   │       └── AccessEventLogger.cs       # HIPAA compliant PHI access logging
│   │
│   └── MediScan.Presentation/             # Web API Host (ASP.NET Core Controllers)
│       ├── Controllers/                   # Inherits ControllerBase
│       │   ├── BaseApiController.cs       # Standard result handling & dispatcher injection
│       │   ├── AuthController.cs
│       │   ├── UsersController.cs
│       │   ├── DoctorsController.cs
│       │   ├── ReportsController.cs
│       │   ├── AppointmentsController.cs
│       │   ├── PaymentsController.cs
│       │   ├── ChatbotController.cs       # SSE endpoint for streaming responses
│       │   └── AdminController.cs
│       ├── Middlewares/
│       │   ├── CorrelationIdMiddleware.cs
│       │   ├── RequestTimingMiddleware.cs # Request process time logging
│       │   ├── GlobalExceptionMiddleware.cs # RFC 9457 Problem Details
│       │   └── SecurityHeadersMiddleware.cs
│       ├── Filters/
│       │   └── IdempotencyFilter.cs       # Idempotency-Key support for payments
│       ├── Extensions/
│       │   ├── DependencyInjection.cs     # Modular service collection wireups
│       │   └── SerilogExtensions.cs       # Serilog enrichers & configuration
│       ├── Program.cs                     # WebApplication builder & middleware configuration
│       └── appsettings.json
│
└── tests/
    ├── MediScan.Domain.UnitTests/         # Pure unit tests for Domain logic & Value Objects
    ├── MediScan.Application.UnitTests/    # CQRS Handlers & Validation tests using Mock/Substitute
    ├── MediScan.Infrastructure.IntegrationTests/ # EF Core & Redis tests
    ├── MediScan.Api.IntegrationTests/     # WebApplicationFactory + Testcontainers (PostgreSQL)
    └── MediScan.ArchitectureTests/        # NetArchTest enforcing Clean Architecture boundaries
```

---

## 6. Core Design Patterns & Concrete Implementations

### 6.1 Custom In-Process CQRS Dispatcher (Zero MediatR)

To eliminate third-party lock-in and runtime reflection overhead, we implement a lightweight, high-performance in-process CQRS dispatcher with typed pipeline behavior chaining.

#### Contracts (`MediScan.SharedKernel.CQRS`)

```csharp
namespace MediScan.SharedKernel.CQRS;

public interface ICommand<TResponse>
{
}

public interface IQuery<TResponse>
{
}

public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    Task<TResponse> HandleAsync(TCommand command, CancellationToken cancellationToken);
}

public interface IQueryHandler<in TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    Task<TResponse> HandleAsync(TQuery query, CancellationToken cancellationToken);
}

public interface IDispatcher
{
    Task<TResponse> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default);
    Task<TResponse> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default);
}

public delegate Task<TResponse> RequestHandlerDelegate<TResponse>();

public interface IPipelineBehavior<in TRequest, TResponse>
{
    Task<TResponse> HandleAsync(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken);
}
```

#### Concrete Dispatcher Implementation (`MediScan.Application.Dispatcher`)

```csharp
namespace MediScan.Application.Dispatcher;

using System.Collections.Concurrent;
using Microsoft.Extensions.DependencyInjection;
using MediScan.SharedKernel.CQRS;

public sealed class Dispatcher(IServiceProvider serviceProvider) : IDispatcher
{
    private static readonly ConcurrentDictionary<Type, Type> HandlerInterfaceCache = new();

    public async Task<TResponse> SendAsync<TResponse>(
        ICommand<TResponse> command, 
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(command);
        var commandType = command.GetType();
        
        var handlerType = HandlerInterfaceCache.GetOrAdd(
            commandType, 
            t => typeof(ICommandHandler<,>).MakeGenericType(t, typeof(TResponse)));

        var handler = serviceProvider.GetRequiredService(handlerType);
        var behaviors = serviceProvider.GetServices<IPipelineBehavior<ICommand<TResponse>, TResponse>>().Reverse();

        RequestHandlerDelegate<TResponse> handlerDelegate = () =>
        {
            var method = handlerType.GetMethod(nameof(ICommandHandler<ICommand<TResponse>, TResponse>.HandleAsync))!;
            return (Task<TResponse>)method.Invoke(handler, [command, cancellationToken])!;
        };

        foreach (var behavior in behaviors)
        {
            var next = handlerDelegate;
            handlerDelegate = () => behavior.HandleAsync(command, next, cancellationToken);
        }

        return await handlerDelegate();
    }

    public async Task<TResponse> QueryAsync<TResponse>(
        IQuery<TResponse> query, 
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(query);
        var queryType = query.GetType();

        var handlerType = HandlerInterfaceCache.GetOrAdd(
            queryType, 
            t => typeof(IQueryHandler<,>).MakeGenericType(t, typeof(TResponse)));

        var handler = serviceProvider.GetRequiredService(handlerType);
        var behaviors = serviceProvider.GetServices<IPipelineBehavior<IQuery<TResponse>, TResponse>>().Reverse();

        RequestHandlerDelegate<TResponse> handlerDelegate = () =>
        {
            var method = handlerType.GetMethod(nameof(IQueryHandler<IQuery<TResponse>, TResponse>.HandleAsync))!;
            return (Task<TResponse>)method.Invoke(handler, [query, cancellationToken])!;
        };

        foreach (var behavior in behaviors)
        {
            var next = handlerDelegate;
            handlerDelegate = () => behavior.HandleAsync(query, next, cancellationToken);
        }

        return await handlerDelegate();
    }
}
```

---

### 6.2 Standardized Result Monad & Error Hierarchy

We forbid throwing exceptions for expected business failures (e.g., invalid OTP, email taken, slot unavailable). All domain operations return `Result` or `Result<T>`.

#### `Error` and `ErrorType` (`MediScan.SharedKernel.Common`)

```csharp
namespace MediScan.SharedKernel.Common;

public enum ErrorType
{
    Failure = 0,
    Validation = 1,
    NotFound = 2,
    Conflict = 3,
    Unauthorized = 4,
    Forbidden = 5
}

public readonly record struct Error(string Code, string Description, ErrorType Type)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.Failure);
    public static readonly Error NullValue = new("Error.NullValue", "The specified result value is null.", ErrorType.Failure);

    public static Error Failure(string code, string description) => new(code, description, ErrorType.Failure);
    public static Error Validation(string code, string description) => new(code, description, ErrorType.Validation);
    public static Error NotFound(string code, string description) => new(code, description, ErrorType.NotFound);
    public static Error Conflict(string code, string description) => new(code, description, ErrorType.Conflict);
    public static Error Unauthorized(string code, string description) => new(code, description, ErrorType.Unauthorized);
    public static Error Forbidden(string code, string description) => new(code, description, ErrorType.Forbidden);
}
```

#### `Result<TValue>` Monad

```csharp
namespace MediScan.SharedKernel.Common;

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None || !isSuccess && error == Error.None)
            throw new InvalidOperationException("Invalid error state for result.");

        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
}

public sealed class Result<TValue> : Result
{
    private readonly TValue? _value;

    private Result(TValue? value, bool isSuccess, Error error) : base(isSuccess, error)
    {
        _value = value;
    }

    public TValue Value => IsSuccess 
        ? _value! 
        : throw new InvalidOperationException("Cannot access value of a failed result.");

    public static Result<TValue> Success(TValue value) => new(value, true, Error.None);
    public static new Result<TValue> Failure(Error error) => new(default, false, error);

    public static implicit operator Result<TValue>(TValue value) => Success(value);
    public static implicit operator Result<TValue>(Error error) => Failure(error);
}
```

---

### 6.3 FluentValidation Pipeline Behavior

All incoming commands and queries are automatically validated before touching the handler. If any validation rule fails, the pipeline short-circuits and returns a validation error result.

```csharp
namespace MediScan.Application.Behaviors;

using FluentValidation;
using MediScan.SharedKernel.Common;
using MediScan.SharedKernel.CQRS;

public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators) 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
    where TResponse : Result
{
    public async Task<TResponse> HandleAsync(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
        {
            return await next();
        }

        var context = new ValidationContext<TRequest>(request);
        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
        {
            var firstFailure = failures[0];
            var error = Error.Validation(firstFailure.PropertyName, firstFailure.ErrorMessage);

            // Dynamically construct failed Result or Result<T>
            if (typeof(TResponse) == typeof(Result))
            {
                return (TResponse)(object)Result.Failure(error);
            }

            var genericType = typeof(TResponse).GetGenericArguments()[0];
            var failureMethod = typeof(Result<>)
                .MakeGenericType(genericType)
                .GetMethod(nameof(Result<object>.Failure), [typeof(Error)])!;

            return (TResponse)failureMethod.Invoke(null, [error])!;
        }

        return await next();
    }
}
```

---

### 6.4 Compile-Time Manual Mapping (Zero AutoMapper)

To guarantee type safety, high throughput, and zero runtime reflection exceptions, all transformations are written as explicit extension methods.

```csharp
namespace MediScan.Application.Features.Reports.Mappings;

using MediScan.Application.Features.Reports.DTOs;
using MediScan.Domain.Entities;

public static class ReportMappings
{
    public static ReportResponse ToResponse(this Report report)
    {
        ArgumentNullException.ThrowIfNull(report);

        return new ReportResponse(
            Id: report.Id,
            UserId: report.UserId,
            FileUrl: report.FileUrl,
            UploadedAtUtc: report.UploadedAtUtc,
            Status: report.Status.ToString(),
            RiskLevel: report.Result?.RiskLevel.ToString(),
            ConfidenceScore: report.Result?.ConfidenceScore ?? 0.0,
            Summary: report.Result?.Summary,
            DoctorSummary: report.Result?.DoctorSummary,
            SuggestedSpecialization: report.Result?.SuggestedSpecialization?.ToString(),
            Observations: report.Observations.Select(o => o.ToResponse()).ToList()
        );
    }

    public static ObservationResponse ToResponse(this AnalyteObservation observation)
    {
        ArgumentNullException.ThrowIfNull(observation);

        return new ObservationResponse(
            LoincCode: observation.LoincCode.Value,
            Name: observation.AnalyteName,
            Value: observation.Value,
            Unit: observation.Unit,
            ReferenceRangeLow: observation.ReferenceRangeLow,
            ReferenceRangeHigh: observation.ReferenceRangeHigh,
            IsCritical: observation.IsCritical,
            InterpretationFlag: observation.InterpretationFlag
        );
    }
}
```

---

### 6.5 Production-Grade Base Controller

Controllers stay completely clean by delegating execution to the `IDispatcher` and mapping `Result<T>` to standardized HTTP responses.

```csharp
namespace MediScan.Presentation.Controllers;

using Microsoft.AspNetCore.Mvc;
using MediScan.SharedKernel.Common;
using MediScan.SharedKernel.CQRS;

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[Produces("application/json")]
public abstract class BaseApiController(IDispatcher dispatcher) : ControllerBase
{
    protected IDispatcher Dispatcher { get; } = dispatcher;

    protected ActionResult<ApiResponse<T>> HandleResult<T>(Result<T> result)
    {
        if (result.IsSuccess)
        {
            return Ok(ApiResponse<T>.SuccessResponse(result.Value));
        }

        return MapErrorToActionResult(result.Error);
    }

    protected ActionResult<ApiResponse> HandleResult(Result result)
    {
        if (result.IsSuccess)
        {
            return Ok(ApiResponse.SuccessResponse());
        }

        return MapErrorToActionResult(result.Error);
    }

    private ActionResult MapErrorToActionResult(Error error)
    {
        var statusCode = error.Type switch
        {
            ErrorType.NotFound => StatusCodes.Status404NotFound,
            ErrorType.Validation => StatusCodes.Status400BadRequest,
            ErrorType.Conflict => StatusCodes.Status409Conflict,
            ErrorType.Unauthorized => StatusCodes.Status401Unauthorized,
            ErrorType.Forbidden => StatusCodes.Status403Forbidden,
            _ => StatusCodes.Status500InternalServerError
        };

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = error.Code,
            Detail = error.Description,
            Instance = HttpContext.Request.Path
        };

        return new ObjectResult(problemDetails) { StatusCode = statusCode };
    }
}

public sealed record ApiResponse<T>(bool Success, T? Data, string? Message, DateTime TimestampUtc)
{
    public static ApiResponse<T> SuccessResponse(T data, string? message = null) =>
        new(true, data, message, DateTime.UtcNow);
}

public sealed record ApiResponse(bool Success, string? Message, DateTime TimestampUtc)
{
    public static ApiResponse SuccessResponse(string? message = null) =>
        new(true, message, DateTime.UtcNow);
}
```

---

## 7. Production-Grade Enterprise Features (Gaps Solved)

### 7.1 Request Processing Time & Correlation ID Logging Middleware
Every HTTP request must be traced with a unique `X-Correlation-Id`, its duration measured down to sub-millisecond precision, and reported both in response headers (`X-Response-Time-Ms`) and Serilog context.

```csharp
namespace MediScan.Presentation.Middlewares;

using System.Diagnostics;
using Serilog.Context;

public sealed class RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault() 
            ?? Guid.NewGuid().ToString("N");
        
        context.Response.Headers["X-Correlation-Id"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (LogContext.PushProperty("ClientIp", context.Connection.RemoteIpAddress?.ToString()))
        using (LogContext.PushProperty("UserId", context.User.Identity?.Name ?? "Anonymous"))
        {
            var stopwatch = Stopwatch.StartNew();

            try
            {
                await next(context);
            }
            finally
            {
                stopwatch.Stop();
                var elapsedMs = stopwatch.Elapsed.TotalMilliseconds;
                
                context.Response.Headers["X-Response-Time-Ms"] = elapsedMs.ToString("F2");

                if (elapsedMs > 500)
                {
                    logger.LogWarning("SLOW REQUEST: {Method} {Path} responded with {StatusCode} in {ElapsedMs:0.00}ms",
                        context.Request.Method, context.Request.Path, context.Response.StatusCode, elapsedMs);
                }
                else
                {
                    logger.LogInformation("HTTP {Method} {Path} responded with {StatusCode} in {ElapsedMs:0.00}ms",
                        context.Request.Method, context.Request.Path, context.Response.StatusCode, elapsedMs);
                }
            }
        }
    }
}
```

---

### 7.2 Global Exception Handling Middleware (RFC 9457 Problem Details)
Zero unhandled stack traces exposed to clients. All unhandled exceptions are caught, logged with correlation context, and returned as standard RFC 9457 `ProblemDetails`.

```csharp
namespace MediScan.Presentation.Middlewares;

using System.Text.Json;
using Microsoft.AspNetCore.Mvc;

public sealed class GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "An unhandled exception occurred: {Message}", ex.Message);
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/problem+json";
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Type = "https://datatracker.ietf.org/doc/html/rfc9457",
            Title = "An unexpected internal server error occurred.",
            Detail = "A critical error occurred while processing your request. Please contact support with the Correlation ID.",
            Instance = context.Request.Path
        };

        if (context.Response.Headers.TryGetValue("X-Correlation-Id", out var correlationId))
        {
            problemDetails.Extensions["correlationId"] = correlationId.ToString();
        }

        var json = JsonSerializer.Serialize(problemDetails);
        await context.Response.WriteAsync(json);
    }
}
```

---

### 7.3 HIPAA-Compliant PHI Access Audit Logging (`AccessEvent`)
In medical applications, every single access to Protected Health Information (patient identity, medical reports, lab observations, clinical comments) **MUST** produce an immutable audit log.

```csharp
namespace MediScan.Infrastructure.Audit;

using MediScan.Application.Abstractions.Security;
using MediScan.Domain.Entities;
using MediScan.Infrastructure.Persistence;

public sealed class AccessEventLogger(ApplicationDbContext dbContext, IClock clock) : IAuditLogger
{
    public async Task LogPhiAccessAsync(
        Guid userId, 
        Guid patientId, 
        string resourceType, 
        string resourceId, 
        string action, 
        string? reason,
        string clientIp,
        CancellationToken cancellationToken = default)
    {
        var accessEvent = new AccessEvent
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            PatientId = patientId,
            ResourceType = resourceType,
            ResourceId = resourceId,
            Action = action,
            Reason = reason ?? "Direct Consultation / View",
            ClientIp = clientIp,
            TimestampUtc = clock.UtcNow
        };

        dbContext.AccessEvents.Add(accessEvent);
        await dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

---

### 7.4 Transactional Outbox Pattern for Async Reliability
To guarantee that domain events (e.g. `ReportUploaded`, `PaymentVerified`, `DoctorApproved`) are dispatched reliably without two-phase commits, events are saved into an `OutboxMessages` table in the exact same database transaction as the business entity.

1. **EF Core Interceptor**: Catches domain events on aggregates before `SaveChangesAsync()`.
2. **Serializes to JSONB**: Writes into `OutboxMessages` with `ProcessedAtUtc = null`.
3. **Background HostedService**: Reads unprocessed messages using `FOR UPDATE SKIP LOCKED`, dispatches them to external queues or worker channels, and marks them processed.

---

### 7.5 Payment Idempotency & Revenue Splitting (Khalti)
- **Idempotency Filter**: For payment initiation and confirmation, clients send `Idempotency-Key` in the header. The response is cached in Redis for 24 hours. Duplicate requests return the cached result instantly, preventing double-billing.
- **Revenue Split Calculation**:
  $$\text{Admin Commission} = \text{Amount Paid} \times 0.25$$
  $$\text{Doctor Net Revenue} = \text{Amount Paid} \times 0.75$$
  Stored as distinct financial fields in `PaymentTransaction` with exact decimal arithmetic (`decimal(18,2)`).

---

### 7.6 File Upload Hardening (Security & Anti-Malware)
Medical document uploads (PDF, JPG, PNG) must pass strict security checks:
1. **Magic Byte Inspection**: Reject files where the binary header doesn't match the MIME type (prevent `.exe` renamed as `.pdf`).
2. **File Size Enforcement**: Maximum 15 MB per report, maximum 5 MB per doctor certificate.
3. **Randomized S3 Keys**: Upload directly to MinIO/S3 using UUID paths: `reports/{userId}/{guid}.pdf`. Files are never stored with original user-supplied file names on disk.
4. **Presigned Download URLs**: Clients only download files via short-lived signed URLs (15-minute expiry). Direct public bucket access is prohibited.

---

### 7.7 Resilience & Fault Tolerance (Polly v8)
Calls to external services (Khalti payment API, Python FastAPI OCR service, LLM Gateway, SMTP) must use `Microsoft.Extensions.Http.Resilience` (Polly v8):
- **Retry Policy**: 3 retries with exponential backoff and jitter for transient errors (HTTP 502, 503, 504, 408).
- **Circuit Breaker**: Break circuit after 50% failure rate over a 30-second window; stay open for 15 seconds.
- **Timeout**: Enforce strict HTTP timeouts (e.g. 10s for Khalti, 60s for OCR extraction).

---

## 8. EF Core 10 PostgreSQL Configuration & Migrations

### 8.1 Configuration Principles
- **Naming Convention**: `UseSnakeCaseNamingConvention()` so tables and columns match PostgreSQL standards (`created_at_utc`, `user_id`).
- **PostgreSQL JSONB**: Complex objects like `raw_ocr_data`, `key_findings`, and `conditions` are mapped to `jsonb` columns for querying speed and schema flexibility.
- **Concurrency Tokens**: Use `xmin` shadow property or `[Timestamp]` row versions for optimistic concurrency control on appointment slots and payment updates.
- **Global Query Filters**: Automatically applied for soft deletion (`!e.IsDeleted`).

```csharp
public class ReportConfiguration : IEntityTypeConfiguration<Report>
{
    public void Configure(EntityTypeBuilder<Report> builder)
    {
        builder.ToTable("reports");
        builder.HasKey(r => r.Id);

        builder.Property(r => r.Status)
            .HasConversion<string>()
            .HasMaxLength(32)
            .IsRequired();

        builder.HasOne(r => r.User)
            .WithMany()
            .HasForeignKey(r => r.UserId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasOne(r => r.Result)
            .WithOne(res => res.Report)
            .HasForeignKey<ReportResult>(res => res.ReportId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasQueryFilter(r => !r.IsDeleted);
    }
}
```

---

## 9. DevOps, Containerization & CI/CD Specification

### 9.1 Multi-Stage Production Dockerfile

```dockerfile
# Stage 1: Build & Publish
FROM mcr.microsoft.com/dotnet/sdk:10.0-preview-alpine AS build
WORKDIR /app

# Copy csproj files and restore dependencies via Central Package Management
COPY Directory.Build.props Directory.Packages.props ./
COPY src/*/*.csproj ./
RUN for file in $(ls *.csproj); do \
      mkdir -p src/${file%.*}/ && mv $file src/${file%.*}/; \
    done
RUN dotnet restore src/MediScan.Presentation/MediScan.Presentation.csproj

# Copy remaining source code and build
COPY src/ ./src/
WORKDIR /app/src/MediScan.Presentation
RUN dotnet publish MediScan.Presentation.csproj \
    -c Release \
    -o /out \
    --no-restore \
    /p:UseAppHost=false

# Stage 2: Runtime Image (Hardened, Chiseled / Non-Root)
FROM mcr.microsoft.com/dotnet/aspnet:10.0-preview-alpine AS runtime
WORKDIR /app

# Create non-root user and group
RUN addgroup -S mediscan && adduser -S mediscan -G mediscan
USER mediscan

# Copy published artifacts
COPY --from=build --chown=mediscan:mediscan /out ./

# Health check & ports
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_EnableDiagnostics=0

HEALTHCHECK --interval=30s --timeout=3s --retries=3 --start-period=10s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/ready || exit 1

ENTRYPOINT ["dotnet", "MediScan.Presentation.dll"]
```

---

### 9.2 Local Development `docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: mediscan-postgres
    environment:
      POSTGRES_DB: mediscan_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgrespassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: mediscan-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    container_name: mediscan-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: miniopassword
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data

  seq:
    image: datalust/seq:latest
    container_name: mediscan-seq
    environment:
      ACCEPT_EULA: Y
    ports:
      - "5341:80"

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

---

### 9.3 GitHub Actions CI/CD Pipeline (`.github/workflows/backend-ci.yml`)

```yaml
name: Backend CI/CD Pipeline

on:
  push:
    branches: [ main, dev ]
    paths:
      - 'src/**'
      - 'tests/**'
      - 'Directory.*'
  pull_request:
    branches: [ main, dev ]

jobs:
  build-and-test:
    name: Build, Lint & Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_DB: mediscan_test
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET 10 SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
          include-prerelease: true

      - name: Restore dependencies
        run: dotnet restore MediScan.sln

      - name: Check code formatting & analyzers
        run: dotnet format --verify-no-changes --severity warn

      - name: Build solution
        run: dotnet build MediScan.sln -c Release --no-restore

      - name: Run Architecture Rules Tests
        run: dotnet test tests/MediScan.ArchitectureTests/MediScan.ArchitectureTests.csproj -c Release --no-build

      - name: Run Unit Tests
        run: dotnet test tests/MediScan.Application.UnitTests/MediScan.Application.UnitTests.csproj -c Release --no-build

      - name: Run Integration Tests
        run: dotnet test tests/MediScan.Api.IntegrationTests/MediScan.Api.IntegrationTests.csproj -c Release --no-build
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Port=5432;Database=mediscan_test;Username=testuser;Password=testpassword;"
```

---

## 10. Architecture Rules Enforcement (NetArchTest)

To prevent code degradation over time, architecture tests must run in CI to strictly enforce layer boundary rules:

```csharp
namespace MediScan.ArchitectureTests;

using NetArchTest.Rules;
using Xunit;

public class ArchitectureBoundaryTests
{
    private const string DomainNamespace = "MediScan.Domain";
    private const string ApplicationNamespace = "MediScan.Application";
    private const string InfrastructureNamespace = "MediScan.Infrastructure";
    private const string PresentationNamespace = "MediScan.Presentation";

    [Fact]
    public void Domain_ShouldNotHaveDependencyOn_OtherProjects()
    {
        var result = Types.InAssembly(typeof(Domain.Entities.User).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(ApplicationNamespace, InfrastructureNamespace, PresentationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, "Domain must not reference Application, Infrastructure, or Presentation.");
    }

    [Fact]
    public void Application_ShouldNotHaveDependencyOn_InfrastructureOrPresentation()
    {
        var result = Types.InAssembly(typeof(Application.Dispatcher.Dispatcher).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(InfrastructureNamespace, PresentationNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, "Application must not reference Infrastructure or Presentation.");
    }

    [Fact]
    public void Controllers_ShouldNotDirectlyDependOn_DbContext()
    {
        var result = Types.InAssembly(typeof(Presentation.Controllers.BaseApiController).Assembly)
            .That()
            .Inherit(typeof(Microsoft.AspNetCore.Mvc.ControllerBase))
            .ShouldNot()
            .HaveDependencyOn("MediScan.Infrastructure.Persistence.ApplicationDbContext")
            .GetResult();

        Assert.True(result.IsSuccessful, "Controllers must never directly reference ApplicationDbContext.");
    }
}
```

---

## 11. AI Implementation Playbook (Step-by-Step Execution)

When the user commands you to begin implementing parts of this backend, follow this **exact sequence**:

### Phase 1: Foundation & Shared Kernel
1. Create `Directory.Build.props` and `Directory.Packages.props`.
2. Scaffold `MediScan.SharedKernel` (Result monad, Error hierarchy, Entity/AggregateRoot, CQRS contracts, IClock, IDispatcher).
3. Scaffold `MediScan.Domain` (Value Objects: Email, PhoneNumber, Money; Enums; Core Entities: User, Role, DoctorLicense, Report, AnalyteObservation, Appointment).
4. Implement `MediScan.ArchitectureTests` to lock down boundary rules before writing infrastructure.

### Phase 2: Application Core & CQRS
1. Implement `MediScan.Application.Dispatcher.Dispatcher`.
2. Implement CQRS Pipeline Behaviors: `LoggingBehavior`, `ValidationBehavior`, `TransactionBehavior`.
3. Scaffold Repository and Unit of Work interfaces (`IUserRepository`, `IReportRepository`, `IUnitOfWork`).
4. Implement static manual mappers (`UserMappings`, `ReportMappings`).
5. Write CQRS Commands, Queries, and FluentValidation rules for Authentication (Register, Login, OTP verification, Refresh Token).

### Phase 3: Infrastructure & Persistence
1. Setup EF Core 10 `ApplicationDbContext` with PostgreSQL Npgsql.
2. Implement Fluent API entity configurations with snake_case naming and JSONB columns.
3. Add EF Core Interceptors for automatic auditing (`IAuditableEntity`) and Outbox event dispatching.
4. Implement repositories and `UnitOfWork`.
5. Implement external clients: Redis Caching, S3/MinIO Blob Storage, Khalti Payment Gateway, and Argon2id Password Hasher.

### Phase 4: Presentation API & Middlewares
1. Setup `MediScan.Presentation` Web API Host.
2. Implement Middlewares: `CorrelationIdMiddleware`, `RequestTimingMiddleware`, `GlobalExceptionMiddleware`, `SecurityHeadersMiddleware`.
3. Configure Serilog structured logging with enrichers and colored console output.
4. Implement `BaseApiController` and feature controllers (`AuthController`, `ReportsController`, `DoctorsController`, `AppointmentsController`, `ChatbotController`).
5. Implement Chatbot Server-Sent Events (SSE) streaming endpoint connecting to the LLM Gateway.
6. Configure Health Checks (`/health/ready`, `/health/live`) and OpenAPI / Scalar documentation.

### Phase 5: Verification & DevOps
1. Create multi-stage `Dockerfile` and `docker-compose.yml`.
2. Configure `.github/workflows/backend-ci.yml`.
3. Write end-to-end integration tests using `WebApplicationFactory` and `Testcontainers.PostgreSql`.
4. Ensure all unit and architecture tests pass with zero warnings.

---

## 12. Pre-Completion Quality Verification Checklist

Before reporting any implementation task as complete, verify:

- [ ] Does every new class use file-scoped namespaces?
- [ ] Are all dependencies injected via Primary Constructors?
- [ ] Is `CancellationToken` passed through every single async method call?
- [ ] Are there **ZERO** usages of `MediatR` or `AutoMapper`?
- [ ] Does every endpoint inherit from `BaseApiController` and return `ActionResult<ApiResponse<T>>`?
- [ ] Are there **ZERO** raw domain entities exposed in controller responses?
- [ ] Is every input command/query covered by a `FluentValidation` validator?
- [ ] Does every PHI access write an `AccessEvent` audit record?
- [ ] Does every HTTP request log duration in `ms` and emit `X-Correlation-Id` / `X-Response-Time-Ms` headers?
- [ ] Does the project compile cleanly with zero warnings (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`)?
