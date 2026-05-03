# COPILOT AGENT PROMPT — PHASE 1
# ServeNow: User Module — Implementation Plan Update
# Zero Breaking Changes | Versioned | Additive Only

---

## CONTEXT

You are a senior .NET 8 developer working on **ServeNow** — a hyperlocal AI-powered services marketplace.

**Stack:** .NET 8, C#, ASP.NET Core, CQRS/MediatR, EF Core 8, MassTransit/RabbitMQ, Azure SQL, Angular 17 standalone AOT.

**CRITICAL RULE — READ THIS FIRST:**
> Phase 1 is PLAN AND SCAFFOLD ONLY.
> Do NOT modify any existing controller, service, DbContext, migration, or entity.
> Do NOT change any existing API route, response DTO, or database table.
> All new code goes into NEW files only.
> Existing users, existing auth flows, existing booking flows — must continue to work unchanged.

---

## YOUR TASK FOR PHASE 1

**Update the implementation plan and scaffold the project structure** for the new User Module microservice.

This is a READ + PLAN + SCAFFOLD task. No business logic yet.

---

## STEP 1 — AUDIT EXISTING CODEBASE

Scan @workspace and answer:

1. Does `ServeNow.Identity` or `ServeNow.UserService` already exist?
   - If `ServeNow.Identity` exists → we will ADD to it using API versioning (v2)
   - If neither exists → we will CREATE `ServeNow.UserService` as a new project

2. Are any of these already implemented?
   - `AuthController.cs` with `/api/v1/auth/*` routes
   - `UsersController.cs` or `UserController.cs`
   - `UserDbContext` or `IdentityDbContext`
   - EF Core migrations under `ServeNow.Identity` or `ServeNow.UserService`

3. What NuGet packages are already referenced in the solution?
   - `Asp.Versioning.Mvc` or `Microsoft.AspNetCore.Mvc.Versioning`
   - `MediatR`
   - `MassTransit.RabbitMQ`
   - `BCrypt.Net-Next`

Report findings in this format before doing anything:

```
AUDIT RESULT:
- ServeNow.Identity exists: YES / NO
- ServeNow.UserService exists: YES / NO
- AuthController (v1) exists: YES / NO (path: ...)
- UserDbContext exists: YES / NO (path: ...)
- API Versioning package: YES / NO
- Existing Auth routes: [list them]
```

---

## STEP 2 — ADD API VERSIONING (if not present)

**Only if `Asp.Versioning.Mvc` is NOT already installed:**

Add to the existing Identity/User API project:

```xml
<!-- In .csproj -->
<PackageReference Include="Asp.Versioning.Mvc" Version="8.1.0" />
<PackageReference Include="Asp.Versioning.Mvc.ApiExplorer" Version="8.1.0" />
```

Register in `Program.cs` — ADD these lines, do NOT remove or replace anything existing:

```csharp
// ADD after existing builder.Services lines — do not touch existing registrations
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version")
    );
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});
```

**IMPORTANT:** If existing controllers have routes like `/api/auth/...` (without version), do NOT change them. They stay as-is. New controllers will use `/api/v1/users/...`.

---

## STEP 3 — SCAFFOLD NEW PROJECT STRUCTURE

Create the following folder/file structure inside the existing solution.
All files should be EMPTY stubs with XML doc comments only — no implementation yet.

### If `ServeNow.Identity` exists → add inside it:

```
ServeNow.Identity/
├── Application/
│   ├── Users/
│   │   ├── Commands/
│   │   │   ├── RegisterUser/
│   │   │   │   ├── RegisterUserCommand.cs          ← STUB
│   │   │   │   └── RegisterUserCommandHandler.cs   ← STUB
│   │   │   ├── UpdateProfile/
│   │   │   │   ├── UpdateProfileCommand.cs         ← STUB
│   │   │   │   └── UpdateProfileCommandHandler.cs  ← STUB
│   │   │   └── DeactivateUser/
│   │   │       ├── DeactivateUserCommand.cs        ← STUB
│   │   │       └── DeactivateUserCommandHandler.cs ← STUB
│   │   └── Queries/
│   │       ├── GetUserById/
│   │       │   ├── GetUserByIdQuery.cs             ← STUB
│   │       │   └── GetUserByIdQueryHandler.cs      ← STUB
│   │       └── GetUserList/
│   │           ├── GetUserListQuery.cs             ← STUB
│   │           └── GetUserListQueryHandler.cs      ← STUB
│   ├── Roles/
│   │   └── Commands/
│   │       ├── AssignRole/
│   │       │   ├── AssignRoleCommand.cs            ← STUB
│   │       │   └── AssignRoleCommandHandler.cs     ← STUB
│   │       └── RevokeRole/
│   │           ├── RevokeRoleCommand.cs            ← STUB
│   │           └── RevokeRoleCommandHandler.cs     ← STUB
│   └── Delegations/
│       └── Commands/
│           ├── CreateDelegation/
│           │   ├── CreateDelegationCommand.cs      ← STUB
│           │   └── CreateDelegationCommandHandler.cs ← STUB
│           └── RevokeDelegation/
│               ├── RevokeDelegationCommand.cs      ← STUB
│               └── RevokeDelegationCommandHandler.cs ← STUB
├── Domain/
│   ├── Entities/                                   ← NEW entities only, do NOT edit existing
│   │   ├── UserProfile.cs                          ← STUB
│   │   ├── Role.cs                                 ← STUB (only if not already exists)
│   │   ├── Permission.cs                           ← STUB (only if not already exists)
│   │   ├── UserRole.cs                             ← STUB
│   │   ├── Delegation.cs                           ← STUB
│   │   └── AuditLog.cs                             ← STUB
│   └── Enums/
│       ├── UserStatus.cs                           ← STUB
│       └── DelegationStatus.cs                     ← STUB
└── Controllers/
    └── v1/
        └── UsersController.cs                      ← STUB with [ApiVersion("1.0")]
```

### If neither project exists → create new `ServeNow.UserService` with full Clean Architecture:

```
ServeNow.UserService/
├── ServeNow.UserService.Api/
│   ├── Controllers/v1/
│   │   └── UsersController.cs                      ← STUB
│   └── Program.cs                                  ← STUB
├── ServeNow.UserService.Application/
│   ├── Users/Commands/                             ← STUB folders
│   ├── Users/Queries/                              ← STUB folders
│   ├── Roles/Commands/                             ← STUB folders
│   └── Delegations/Commands/                       ← STUB folders
├── ServeNow.UserService.Domain/
│   ├── Entities/                                   ← STUB entities
│   └── Enums/                                      ← STUB enums
└── ServeNow.UserService.Infrastructure/
    ├── Persistence/
    │   └── UserDbContext.cs                        ← STUB (separate DB from existing)
    └── Repositories/                               ← STUB interfaces
```

---

## STEP 4 — STUB FILE TEMPLATES

For every stub file, generate this pattern (C# records for Commands/Queries, classes for entities):

### Command Stub Example — `RegisterUserCommand.cs`
```csharp
namespace ServeNow.Identity.Application.Users.Commands.RegisterUser;

/// <summary>
/// Command to register a new user in ServeNow.
/// Handled by <see cref="RegisterUserCommandHandler"/>.
/// </summary>
/// <remarks>
/// Phase 2 will implement full registration logic.
/// Does NOT affect existing auth flow in AuthController v1.
/// </remarks>
public sealed record RegisterUserCommand(
    string MobileNumber,
    string FullName,
    string Email,
    string Password,
    string UserType // "Customer" | "Provider"
) : IRequest<RegisterUserResult>;

/// <summary>
/// Result returned after successful registration.
/// </summary>
public sealed record RegisterUserResult(
    Guid UserId,
    string MobileNumber,
    string UserType
);
```

### Handler Stub Example — `RegisterUserCommandHandler.cs`
```csharp
namespace ServeNow.Identity.Application.Users.Commands.RegisterUser;

/// <summary>
/// Handles <see cref="RegisterUserCommand"/>.
/// Implementation deferred to Phase 2.
/// </summary>
public sealed class RegisterUserCommandHandler
    : IRequestHandler<RegisterUserCommand, RegisterUserResult>
{
    // TODO Phase 2: inject IUserRepository, IOtpService, IPublishEndpoint

    /// <summary>
    /// Phase 2 will implement: validate → hash password → save user → publish UserRegisteredEvent.
    /// </summary>
    public Task<RegisterUserResult> Handle(
        RegisterUserCommand request,
        CancellationToken cancellationToken)
        => throw new NotImplementedException("Implemented in Phase 2.");
}
```

### Controller Stub Example — `UsersController.cs`
```csharp
using Asp.Versioning;

namespace ServeNow.Identity.Controllers.v1;

/// <summary>
/// User management endpoints — API v1.
/// These are NEW endpoints and do NOT conflict with existing AuthController routes.
/// </summary>
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/users")]
public sealed class UsersController : ControllerBase
{
    // TODO Phase 2: inject ISender (MediatR)

    /// <summary>
    /// POST /api/v1/users/register
    /// Registers a new Customer or Provider.
    /// </summary>
    [HttpPost("register")]
    [ProducesResponseType(typeof(RegisterUserResult), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public Task<IActionResult> Register([FromBody] RegisterUserCommand command,
        CancellationToken ct)
        => throw new NotImplementedException("Implemented in Phase 2.");

    /// <summary>
    /// GET /api/v1/users/{id}
    /// Returns user profile by ID.
    /// </summary>
    [HttpGet("{id:guid}")]
    [Authorize]
    [ProducesResponseType(typeof(UserProfileDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public Task<IActionResult> GetById(Guid id, CancellationToken ct)
        => throw new NotImplementedException("Implemented in Phase 2.");
}
```

### Entity Stub Example — `Delegation.cs`
```csharp
namespace ServeNow.Identity.Domain.Entities;

/// <summary>
/// Represents a time-bound permission delegation from one user to another.
/// Implemented in Phase 2.
/// </summary>
public sealed class Delegation
{
    public Guid Id { get; private set; }
    public Guid GrantorUserId { get; private set; }
    public Guid GranteeUserId { get; private set; }
    public string Permission { get; private set; } = string.Empty;
    public DateTime ValidFrom { get; private set; }
    public DateTime ValidTo { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }

    // TODO Phase 2: add domain methods, EF Core config
}
```

---

## STEP 5 — UPDATE IMPLEMENTATION PLAN FILE

Locate the existing implementation plan file in the workspace (could be named `PLANNER.md`, `DEVELOPER.md`, `implementation-plan.md`, or similar in `.github/` folder).

**ADD** the following section at the end — do NOT overwrite any existing content:

```markdown
---

## USER MODULE — PHASED IMPLEMENTATION PLAN
**Added:** Phase 1 Scaffold | Zero breaking changes

### Phase 1 — Scaffold (current)
- [x] API Versioning added to project
- [x] Folder structure scaffolded under `/Application/Users`, `/Domain/Entities`
- [x] All CQRS Command/Query stubs created
- [x] UsersController v1 stub created at `/api/v1/users`
- [x] Domain entity stubs: UserProfile, Role, Permission, UserRole, Delegation, AuditLog
- [ ] Phase 2 — Auth & Registration logic
- [ ] Phase 3 — Role & Permission management
- [ ] Phase 4 — Delegation & Access Control
- [ ] Phase 5 — Profile Management & Admin endpoints

### Versioning Strategy
| Version | Scope | Status |
|---------|-------|--------|
| v1 (existing) | Original AuthController routes — untouched | ✅ Live |
| v1 (new) | New UsersController under `/api/v1/users` | 🚧 Phase 1 |
| v2 (future) | Breaking changes only if needed | 🔜 Future |

### Non-Breaking Guarantee
- All existing `/api/auth/*` routes → unchanged
- All existing JWT token contracts → unchanged
- All existing DB tables → no new migrations in Phase 1
- Existing Angular routes and guards → unchanged
```

---

## PHASE 1 DEFINITION OF DONE

Before completing Phase 1, verify:

- [ ] `dotnet build` succeeds with zero errors
- [ ] Existing unit tests (if any) still pass
- [ ] No existing controller files were modified
- [ ] No existing DbContext or migration files were modified
- [ ] No existing `Program.cs` registrations were removed
- [ ] `UsersController` stub compiles but throws `NotImplementedException` on all endpoints
- [ ] API Versioning is registered without replacing existing middleware
- [ ] Scaffold folder structure matches Step 3 exactly

---

## WHAT NOT TO DO IN PHASE 1

❌ Do NOT implement any business logic  
❌ Do NOT add EF Core migrations  
❌ Do NOT modify `AuthController.cs`  
❌ Do NOT change any existing DTOs or response models  
❌ Do NOT touch `IdentityDbContext` or any existing DbContext  
❌ Do NOT change Angular routing or guards  
❌ Do NOT add RabbitMQ consumers yet  

---

## NEXT PHASES (for reference only — do NOT implement now)

| Phase | Scope |
|-------|-------|
| Phase 2 | Registration logic, OTP verification, JWT for new users |
| Phase 3 | Role & Permission CRUD, claim-based authorization |
| Phase 4 | Delegation create/revoke/auto-expire (Hangfire) |
| Phase 5 | Profile management, Admin user list, Provider approval |
| Phase 6 | Audit log viewer, CSV export, Swagger docs finalize |

---

*ServeNow — User Module Phase 1 Prompt | Copilot Agent Mode | Zero Breaking Changes*
