# ServeNow — User Module Implementation Plan

> **Based on:** `ServeNow_UserModule_CDD.md` v1.0 · April 2026  
> **Stories:** 17 (AUTH · REG · PROF · ACCT · NOTIF)  
> **Approach:** Additive — existing booking flow, demo users, and current auth endpoints remain functional throughout  
> **Last reviewed:** April 3, 2026 — code audit corrections applied

---

## Backward-Compatibility Guarantees

| Guarantee | Detail |
|---|---|
| Demo users preserved | `aaaaaaaa-…` (customer), `11111111-…`, `22222222-…` (providers) with OTP `123456` continue to work — new `AppUser` columns have `DEFAULT false/null`; dev OTP bypass retained in `SendOtpCommand` |
| Existing auth endpoints preserved | `/api/auth/register`, `/api/auth/send-otp`, `/api/auth/verify-otp`, `/api/auth/set-role` keep current request/response shapes |
| BookingService unaffected | `ProviderId` Guid references remain valid; no cross-service DB access introduced |
| My Bookings page preserved | `GET /api/bookings/dashboard` and `MyBookingsComponent` from prior session remain unchanged |

---

## Resolved Decisions

| # | Decision | Resolution | Rationale |
|---|---|---|---|
| D1 | OTP migration strategy | **Keep both in parallel.** Old `/api/auth/send-otp` and `/api/auth/verify-otp` continue to use `AppUser.OtpCode` (plain text, `123456` in dev). New `/api/auth/otp/send` and `/api/auth/otp/verify` use `OtpRecord` + BCrypt. | Demo users continue working with no migration risk |
| D2 | Database migration tooling | **EF Core migrations** (`dotnet ef migrations add`) | AuthService already uses `DbContext`. EF migrations integrate directly and are faster to set up than Liquibase here |
| D3 | `/api/users/me/bookings` routing | **Ocelot pass-through** to BookingService `GET /api/bookings` with JWT forwarded | Simpler, no inter-service coupling |
| D4 | Provider reviews (US-PROF-03) | **Stub endpoint** returning empty list now | No `Review` entity exists yet in any service |
| D5 | Profile photo upload | **Defer** — serve placeholder avatar for now | No Azure Blob Storage account configured |
| D6 *(new)* | Refresh token delivery | **JSON response body**, stored in `PreferencesService` (same as access token) | App uses Capacitor `Preferences` for native storage. HttpOnly cookies do not work reliably in PWA/Capacitor context |

---

## Existing State vs CDD Spec — Gap Summary

| Area | Exists Today | CDD Spec Requires | Gap Type |
|---|---|---|---|
| `AppUser` fields | 8 fields — mutable `class`, Name, OtpCode… | 15+ fields (FullName, IsDeleted, FcmToken, NotificationPrefs, Addresses…) | Additive EF Core migration |
| OTP storage | Plain text in `AppUser.OtpCode` | Separate `OtpRecord` table, BCrypt hashed | New table (parallel, not replacing) |
| OTP lockout | None | Redis `otp:lock:{mobile}` after 5 failures | New Redis dependency |
| JWT | Access token only | Access + 30-day refresh token | New `RefreshTokens` table + endpoints |
| JWT denylist | None | Redis `jwt:denylist:{jti}` | New Redis key |
| Admin role | Enum: Customer, Provider only | Add `Admin` value | 1-line enum change |
| Soft delete | None | `IsDeleted` + EF global query filter | Migration + context update |
| MediatR (AuthService) | None — direct EF in controller | MediatR handlers (MediatR 12.4.1 already in `ServeNow.Shared`) | Add `<ProjectReference>` + wire up |
| FluentValidation | None | All commands validated (FluentValidation 11.11 already in `ServeNow.Shared`) | Add `<ProjectReference>` + register |
| `Provider` entity | None | Separate entity with `ProviderStatus` | New EF table |
| Saved addresses | None | `SavedAddress` owned collection on AppUser | Migration + `UseNetTopologySuite()` |
| Notification history | None | `NotificationHistory` table in NotificationService | New entity + EF Core added to NotifSvc |
| Angular auth refresh | Silent fail (clears token on 401) | Retry with refresh token on 401 | Interceptor upgrade |
| Profile pages | None | Customer/Provider profile + addresses | New Angular feature area |
| Account management | None | Change mobile, delete account, suspend | New Angular feature area |
| Notification UI | None | History list + preference toggles | New Angular feature area |
| Admin UI | None | Provider approval, user management | New Angular feature area |

---

## Sprint Plan

| Sprint | Stories | Focus | Priority |
|---|---|---|---|
| **Sprint 0** | — | Prerequisites: project reference, EF migrations, Redis config | **Do first — unblocks everything** |
| Sprint 1 | US-AUTH-01 to 05 | Auth core: register, OTP, JWT refresh, logout | Critical |
| Sprint 2 | US-REG-01 to 03 | Provider registration + admin review | High |
| Sprint 3 | US-PROF-01 to 04 | Profile, availability, ratings, booking history | High |
| Sprint 4 | US-ACCT-01 to 03 | Account ops: mobile change, delete, suspend | Medium |
| Sprint 5 | US-NOTIF-01 to 02 | Notifications: FCM, SMS, history, preferences | Medium |

---

## Sprint 0 — Prerequisites

> Resolves blockers **B1** (missing `ServeNow.Shared` project reference), **B4** (`EnsureCreated` does not migrate existing DB), and decision **D2** (EF Core migrations chosen). **Must complete before Sprint 1.**

| # | Action | File(s) | Notes |
|---|---|---|---|
| 0.1 | Add `ServeNow.Shared` project reference to AuthService | `ServeNow.AuthService.csproj` | Add `<ProjectReference Include="..\ServeNow.Shared\ServeNow.Shared.csproj" />`. Brings in MediatR 12.4.1, FluentValidation 11.11, NetTopologySuite — no separate installs for those |
| 0.2 | Install `StackExchange.Redis` in AuthService | `ServeNow.AuthService.csproj` | `<PackageReference Include="StackExchange.Redis" Version="2.8.16" />` — only package not already in Shared |
| 0.3 | Install `MassTransit.RabbitMQ` in AuthService | `ServeNow.AuthService.csproj` | `<PackageReference Include="MassTransit.RabbitMQ" Version="8.3.6" />` — AuthService needs to publish events (used in Sprint 2+) |
| 0.4 | Replace `EnsureCreated` with `MigrateAsync` | `AuthService/Program.cs` | Replace `db.Database.EnsureCreatedAsync()` with `db.Database.MigrateAsync()`. Seed data stays as an idempotent block after migrate: `if (!await db.Users.AnyAsync()) { ... }` |
| 0.5 | Create initial EF Core migration | Terminal | `dotnet ef migrations add InitialCreate --project ServeNow.AuthService --startup-project ServeNow.AuthService` — snapshots current `AppUser` table so all future migrations are incremental |
| 0.6 | Add Redis connection strings | `appsettings.json`, `appsettings.Development.json` | Dev: `"Redis": "localhost:6379"`. Docker env override: `Redis=redis:6379` (matches Docker service name). Redis container is already running in the Docker stack |
| 0.7 | Seed demo admin user | `AuthService/Program.cs` | Add alongside existing 3 demo users: `Id = Guid.Parse("ffffffff-ffff-ffff-ffff-ffffffffffff")`, `Name = "Admin"`, `MobileNumber = "9000000000"`, `Role = UserRole.Admin`, `OtpCode = "123456"` |

---

## Sprint 1 — AUTH (US-AUTH-01 to US-AUTH-05)

### Stories

- **US-AUTH-01** Register with mobile number (Customer)
- **US-AUTH-02** Register as a service provider
- **US-AUTH-03** Login via OTP (BCrypt-hashed, Redis lockout after 5 failures)
- **US-AUTH-04** Refresh access token silently (30-day rotation)
- **US-AUTH-05** Logout from device (Redis denylist)

### Backend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 1.B.1 | Extend `AppUser` class — add nullable properties | `AuthService/Domain/AppUser.cs` | Keep as **`class`** (not record — EF change tracking requires mutable class). Add: `string? FullName`, `bool IsDeleted = false`, `DateTimeOffset? DeletedAt`, `DateTimeOffset? SuspendedUntil`, `string? SuspensionReason`, `string? FcmToken`, `Language PreferredLang = Language.Telugu`. Keep existing `Name` property unchanged |
| 1.B.2 | Add `Admin` to `UserRole` enum | `AuthService/Domain/AppUser.cs` | Additive — no existing records affected |
| 1.B.3 | Add `Language` enum | `AuthService/Domain/AppUser.cs` | `public enum Language { Telugu = 0, English = 1 }` |
| 1.B.4 | Create `OtpRecord` entity | `AuthService/Domain/OtpRecord.cs` | `Guid Id`, `string Mobile`, `string CodeHash` (BCrypt), `int Attempts = 0`, `bool IsUsed = false`, `DateTimeOffset ExpiresAt`, `DateTimeOffset CreatedAt`. Separate from `AppUser.OtpCode` which stays for old endpoints per D1 |
| 1.B.5 | Create `RefreshToken` entity | `AuthService/Domain/RefreshToken.cs` | `Guid Id`, `Guid UserId`, `string TokenHash` (SHA-256 of raw token), `DateTimeOffset ExpiresAt`, `DateTimeOffset CreatedAt`, `DateTimeOffset? RevokedAt` |
| 1.B.6 | Create `NotificationPrefs` class | `AuthService/Domain/NotificationPrefs.cs` | Plain class (owned entity): `bool PushEnabled = true`, `bool SmsEnabled = true` |
| 1.B.7 | Update `AuthDbContext` | `AuthService/AuthDbContext.cs` | Add `DbSet<OtpRecord> OtpRecords`, `DbSet<RefreshToken> RefreshTokens`. Configure: `OwnsOne(u => u.NotificationPrefs)`. Add `HasQueryFilter(u => !u.IsDeleted)` global soft-delete filter. Add `b.Property(u => u.IsDeleted).HasDefaultValue(false)` — this generates `DEFAULT (0)` in migration, protecting existing rows from NULL vs false mismatch |
| 1.B.8 | Create EF migration for Sprint 1 schema | Terminal | `dotnet ef migrations add UserModuleSprint1 --project ServeNow.AuthService` — adds nullable columns on `AppUser`, `NotificationPrefs_*` owned columns, `OtpRecords` table, `RefreshTokens` table |
| 1.B.9 | Create `ICacheService` interface | `AuthService/Services/ICacheService.cs` | `Task<bool> ExistsAsync(string key, CancellationToken ct)`, `Task SetAsync(string key, string value, TimeSpan ttl, CancellationToken ct)`, `Task DeleteAsync(string key, CancellationToken ct)`, `Task<string?> GetAsync(string key, CancellationToken ct)` |
| 1.B.10 | Create `RedisCacheService` | `AuthService/Services/RedisCacheService.cs` | Implements `ICacheService` using `IConnectionMultiplexer`. Registered as `AddSingleton<IConnectionMultiplexer>(ConnectionMultiplexer.Connect(config["Redis"]!))` |
| 1.B.11 | Create `ITokenService` interface | `AuthService/Services/ITokenService.cs` | `Task<string> IssueAccessTokenAsync(AppUser user)`, `Task<string> IssueRefreshTokenAsync(Guid userId, CancellationToken ct)`, `Task<(string accessToken, string refreshToken)> RotateRefreshTokenAsync(string incomingToken, CancellationToken ct)`, `Task RevokeAsync(string jti, TimeSpan remaining, CancellationToken ct)` |
| 1.B.12 | Create `JwtTokenService` | `AuthService/Services/JwtTokenService.cs` | Builds on existing `JwtHelper`. Access token: same claims as today (`sub, name, phone, role`) plus `jti = Guid.NewGuid().ToString()`. Refresh token: raw `Guid.NewGuid().ToString()`, stored as `SHA-256(rawToken)` in `RefreshTokens` table. Raw token returned to client |
| 1.B.13 | Create `IAppUserRepository` + `AppUserRepository` | `AuthService/Domain/IAppUserRepository.cs`, `AuthService/Repositories/AppUserRepository.cs` | `ExistsAsync(string mobile)`, `GetByMobileAsync(string mobile)`, `GetAsync(Guid id)`, `AddAsync(AppUser)`, `SaveAsync(AppUser)`. Note: global query filter on `DbContext` means `GetAsync` won't find soft-deleted users by default — correct behaviour |
| 1.B.14 | Create `IOtpRepository` + `OtpRepository` | `AuthService/Domain/IOtpRepository.cs`, `AuthService/Repositories/OtpRepository.cs` | `GetLatestAsync(string mobile)` — orders by `CreatedAt DESC`, returns first non-used non-expired record. `SaveAsync(OtpRecord)` |
| 1.B.15 | Create `SendOtpCommand` + handler | `AuthService/Commands/Auth/SendOtpCommand.cs` | Generates 6-digit random OTP. **Dev bypass**: `var code = config["Dev:FixedOtp"] ?? GenerateRandom6Digits()` — preserves demo user flow with `123456`. BCrypt-hashes it. Saves to `OtpRecord` with `ExpiresAt = UtcNow + 5min`. Returns `Unit` |
| 1.B.16 | Create `RegisterCustomerCommand` + handler | `AuthService/Commands/Auth/RegisterCustomerCommand.cs` | Checks duplicate mobile via `IAppUserRepository.ExistsAsync` → throw `ConflictException("MOBILE_ALREADY_REGISTERED")`. Creates `AppUser` with `Role = Customer`, `FullName = cmd.FullName`. Sends `new SendOtpCommand(cmd.MobileNumber)` via `ISender` |
| 1.B.17 | Create `RegisterProviderCommand` + handler | `AuthService/Commands/Auth/RegisterProviderCommand.cs` | Creates `AppUser` with `Role = Provider`. Saves `AppUser`. Sprint 2 will create the `Provider` record — handler can call `ISender.Send(new CreateProviderProfileCommand(...))` which is introduced in Sprint 2 |
| 1.B.18 | Create `VerifyOtpCommand` + handler | `AuthService/Commands/Auth/VerifyOtpCommand.cs` | 1) Check `otp:lock:{mobile}` → 429. 2) Load `OtpRecord` → 404 `OTP_NOT_FOUND`. 3) Check `ExpiresAt` → 400 `OTP_EXPIRED`. 4) BCrypt verify — on fail: increment Attempts, lock Redis at 5 failures (15 min), 400 `OTP_INVALID`. On success: mark `IsUsed = true`, issue `accessToken` + `refreshToken` via `ITokenService`. Return `AuthTokens` |
| 1.B.19 | Create `RefreshTokenCommand` + handler | `AuthService/Commands/Auth/RefreshTokenCommand.cs` | SHA-256 hash incoming token. Find in `RefreshTokens` where not revoked and `ExpiresAt > UtcNow` → 401. Load `AppUser`. Rotate: mark old `RevokedAt = UtcNow`, create new refresh token. Issue new access token. Return `AuthTokens` |
| 1.B.20 | Create `LogoutCommand` + handler | `AuthService/Commands/Auth/LogoutCommand.cs` | SHA-256 hash incoming refresh token, set `RevokedAt` in DB. Extract `jti` from access token string (decode JWT claims without validation). Call `ICacheService.SetAsync("jwt:denylist:{jti}", "1", remaining TTL)` |
| 1.B.21 | Create FluentValidation validators | `AuthService/Validators/` | `RegisterCustomerValidator`: phone `^[6-9]\d{9}$`, name 2–60 chars. `RegisterProviderValidator`: same + `ServiceCategory` not empty. `VerifyOtpValidator`: `Code` matches `^\d{6}$`. Register via `AddValidatorsFromAssembly(typeof(RegisterCustomerValidator).Assembly)` |
| 1.B.22 | Check if `ValidationBehavior` exists in Shared | `ServeNow.Shared/Behaviours/` | If a MediatR `IPipelineBehavior<,>` validation behaviour exists there, register it in AuthService `Program.cs`. If not, add a minimal one that throws `ValidationException` to trigger `ProblemDetails` |
| 1.B.23 | Create `JwtDenylistMiddleware` | `AuthService/Middleware/JwtDenylistMiddleware.cs` | After `UseAuthentication`. For authenticated requests: read `jti` claim from `HttpContext.User`. Check `ICacheService.ExistsAsync("jwt:denylist:{jti}")`. Return `401 { code: "TOKEN_INVALID" }` if found |
| 1.B.24 | Create `AccountSuspendedMiddleware` | `AuthService/Middleware/AccountSuspendedMiddleware.cs` | For authenticated requests: read `sub` claim. Load user via `IAppUserRepository.GetAsync(userId)`. If `SuspendedUntil > UtcNow`, return `403 { code: "ACCOUNT_SUSPENDED" }`. Cache the suspended check per user for 60 seconds to avoid a DB hit on every request |
| 1.B.25 | Add new controller endpoints | `AuthService/Controllers/AuthController.cs` | `POST /api/auth/register/customer`, `POST /api/auth/register/provider`, `POST /api/auth/otp/send`, `POST /api/auth/otp/verify`, `POST /api/auth/token/refresh`, `POST /api/auth/logout`. All thin — `return Ok(await sender.Send(new XxxCommand(...)))`. Existing endpoints unchanged |
| 1.B.26 | Wire all new services in `Program.cs` | `AuthService/Program.cs` | `AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(SendOtpCommand).Assembly))`, `AddValidatorsFromAssembly(...)`, `AddSingleton<IConnectionMultiplexer>(...)`, `AddScoped<ICacheService, RedisCacheService>()`, `AddScoped<ITokenService, JwtTokenService>()`, `AddScoped<IAppUserRepository, AppUserRepository>()`, `AddScoped<IOtpRepository, OtpRepository>()` |
| 1.B.27 | Register middleware | `AuthService/Program.cs` | `app.UseMiddleware<JwtDenylistMiddleware>()` and `app.UseMiddleware<AccountSuspendedMiddleware>()` — must appear **after** `app.UseAuthentication()` |
| 1.B.28 | Ocelot — no new routes for Sprint 1 | — | **No action.** Existing `/api/auth/{everything}` wildcard in `ocelot.Docker.json` already routes all new auth endpoints to `auth-service:8080` |

### Frontend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 1.F.1 | Update `auth.models.ts` | `auth/models/auth.models.ts` | Add `interface AuthTokens { accessToken: string; refreshToken: string; expiresIn: number }`. Add `type Language = 'te' \| 'en'`. Add `interface NotificationPrefs { pushEnabled: boolean; smsEnabled: boolean }` |
| 1.F.2 | Upgrade `AuthService` | `auth/services/auth.service.ts` | Add private `_refreshToken` signal stored in `PreferencesService` under key `'auth:refresh'`. Add `refreshAndRetry(req, next)` method: calls `POST /api/auth/token/refresh` with stored refresh token, on success updates both stored tokens and retries original request, on failure calls `logout()`. Keep all existing methods |
| 1.F.3 | Upgrade `AuthInterceptorFn` | `core/interceptors/auth.interceptor.ts` | Current behaviour: catches 401, calls `auth.logout()`. New: catches 401, calls `auth.refreshAndRetry(req, next)` — only falls through to `logout()` if refresh fails. Skip `/api/auth/token/refresh` from the retry interceptor to prevent infinite loop |
| 1.F.4 | Update `phone-login` component | `auth/pages/phone-login/` | Call `POST /api/auth/otp/send` (new endpoint) — this works for both new and existing users. Can retire `send-otp` call here if desired, or keep both for backward compat |
| 1.F.5 | Update `otp-verify` component | `auth/pages/otp-verify/` | Call `POST /api/auth/otp/verify`. On `AuthTokens` response, store both `accessToken` (existing) and `refreshToken` (new) via `auth.saveSession(...)` |
| 1.F.6 | Create `register-customer.component` | `auth/pages/register/register-customer.component.ts` | Standalone. Form: mobile (10-digit regex validation), fullName (2–60 chars). Calls `POST /api/auth/register/customer`. On 201, stores mobile in router state and navigates to `/auth/verify-otp` |
| 1.F.7 | Create `register-provider.component` | `auth/pages/register/register-provider.component.ts` | Same as above plus service category dropdown (values from `GET /api/services`). Calls `POST /api/auth/register/provider` |
| 1.F.8 | Add logout button to navigation | Wherever nav/header exists | On click: calls `POST /api/auth/logout` with current `{ refreshToken }`, then `auth.logout()` to clear local state regardless of API result |
| 1.F.9 | Update `auth.routes.ts` | `auth/auth.routes.ts` | Add `/auth/register-customer`, `/auth/register-provider` lazy-loaded routes |

### API Contract — Sprint 1

```
POST /api/auth/register/customer
  Body:  { mobileNumber: string, fullName: string }
  201:   { userId: string, message: "OTP sent" }
  400:   { code: "VALIDATION_FAILED", errors: [{ field, message }] }
  409:   { code: "MOBILE_ALREADY_REGISTERED" }

POST /api/auth/register/provider
  Body:  { mobileNumber: string, fullName: string, serviceCategory: string }
  201:   { userId: string, message: "OTP sent" }
  409:   { code: "MOBILE_ALREADY_REGISTERED" }

POST /api/auth/otp/send
  Body:  { mobileNumber: string }
  200:   { otpSent: true }
  429:   { code: "MOBILE_LOCKED", retryAfterSeconds: 900 }

POST /api/auth/otp/verify
  Body:  { mobileNumber: string, code: string }
  200:   { accessToken: string, refreshToken: string, expiresIn: 3600 }
  400:   { code: "OTP_INVALID", attemptsRemaining: 3 }
  400:   { code: "OTP_EXPIRED" }
  429:   { code: "MOBILE_LOCKED" }

POST /api/auth/token/refresh
  Body:  { refreshToken: string }
  200:   { accessToken: string, refreshToken: string, expiresIn: 3600 }
  401:   { code: "TOKEN_INVALID" }

POST /api/auth/logout
  Auth:  Bearer JWT
  Body:  { refreshToken: string }
  204:   No Content
```

---

## Sprint 2 — REG (US-REG-01 to US-REG-03)

### Stories

- **US-REG-01** Admin views pending provider registrations
- **US-REG-02** Admin approves or rejects a provider (SMS notification in preferred language)
- **US-REG-03** Rejected provider re-submits after 7-day cooldown

### Backend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 2.B.1 | Create `Provider` entity | `AuthService/Domain/Provider.cs` | `Guid Id` (FK = `AppUser.Id`, shared primary key — one-to-one by shared PK). `string ServiceCategory`, `ProviderStatus Status`, `DateTimeOffset? RejectedAt`, `string? RejectionReason`, `DateTimeOffset? LastApprovedAt` |
| 2.B.2 | Create `ProviderStatus` enum | `AuthService/Domain/Provider.cs` | `PendingVerification, Active, Rejected, Suspended` |
| 2.B.3 | Configure `Provider` in `AuthDbContext` | `AuthDbContext.cs` | `modelBuilder.Entity<Provider>(b => { b.HasKey(p => p.Id); b.HasOne<AppUser>().WithOne().HasForeignKey<Provider>(p => p.Id); });`. Add `DbSet<Provider> Providers` |
| 2.B.4 | Seed `Provider` rows for existing provider demo users | `AuthService/Program.cs` | For `11111111-…` and `22222222-…`: if no `Provider` row exists, seed with `Status = Active` — ensures existing provider demo users have corresponding `Provider` records so Sprint 2 queries don't return empty |
| 2.B.5 | Create `AuditLog` entity | `AuthService/Domain/AuditLog.cs` | `Guid Id`, `Guid AdminId`, `Guid TargetUserId`, `string Action`, `DateTimeOffset Timestamp`, `string? Notes`. Add `DbSet<AuditLog> AuditLogs` |
| 2.B.6 | Create EF migration | Terminal | `dotnet ef migrations add UserModuleSprint2 --project ServeNow.AuthService` — adds `Providers` table, `AuditLogs` table |
| 2.B.7 | Add event contracts to `ServeNow.Shared` | **`ServeNow.Shared/Contracts/Events.cs`** | Append to existing file: `record ProviderRegisteredEvent(Guid ProviderId, string Mobile)`, `record ProviderApprovedEvent(Guid ProviderId, string Mobile, string Language)`, `record ProviderRejectedEvent(Guid ProviderId, string Mobile, string Reason, string Language)`. **Must go in `ServeNow.Shared` — not in AuthService — because NotificationService's `.csproj` only references Shared, not AuthService** |
| 2.B.8 | Wire MassTransit publishing in AuthService `Program.cs` | `AuthService/Program.cs` | Add `AddMassTransit` with RabbitMQ config matching other services (`rabbitmq` host, same credentials) |
| 2.B.9 | Create `ProviderDto` | `AuthService/DTOs/ProviderDto.cs` | `Id, FullName, Mobile, ServiceCategory, Status, RegisteredAt, RejectionReason` |
| 2.B.10 | Create `IProviderRepository` + `ProviderRepository` | `AuthService/Domain/`, `AuthService/Repositories/` | `GetAsync(Guid id)`, `GetPagedAsync(ProviderStatus? filter, int page, int size)`, `SaveAsync(Provider)` |
| 2.B.11 | Create `GetPendingProvidersQuery` + handler | `AuthService/Queries/Admin/GetPendingProvidersQuery.cs` | Returns `PagedList<ProviderDto>`, filterable by `ProviderStatus`, 20 per page |
| 2.B.12 | Create `ApproveProviderCommand` + handler | `AuthService/Commands/Admin/ApproveProviderCommand.cs` | Sets `Provider.Status = Active`, writes `AuditLog`, publishes `ProviderApprovedEvent` via `IPublishEndpoint` |
| 2.B.13 | Create `RejectProviderCommand` + handler | `AuthService/Commands/Admin/RejectProviderCommand.cs` | Sets `Status = Rejected`, records `RejectedAt = UtcNow`, stores reason, writes `AuditLog`, publishes `ProviderRejectedEvent` |
| 2.B.14 | Create `ReapplyProviderCommand` + handler | `AuthService/Commands/Admin/ReapplyProviderCommand.cs` | Checks `RejectedAt + 7 days > UtcNow` → 409. Resets to `PendingVerification`, clears `RejectionReason`. Publishes `ProviderRegisteredEvent` |
| 2.B.15 | Create `RejectProviderValidator` | `AuthService/Validators/RejectProviderValidator.cs` | `Reason`: NotEmpty, length 10–500 chars |
| 2.B.16 | Create `ProvidersController` | `AuthService/Controllers/ProvidersController.cs` | `[Route("api")]`. Endpoints: `GET /api/admin/providers/pending [Authorize(Roles="Admin")]`, `PUT /api/admin/providers/{id}/approve [Authorize(Roles="Admin")]`, `PUT /api/admin/providers/{id}/reject [Authorize(Roles="Admin")]`, `PUT /api/providers/{id}/reapply [Authorize(Roles="Provider")]` |
| 2.B.17 | Create `ProviderApprovedConsumer` | `NotificationService/Consumers/ProviderApprovedConsumer.cs` | Consumes `ProviderApprovedEvent`. Sends FCM or SMS. Message in preferred language: `Language == "Telugu"` → Telugu text |
| 2.B.18 | Create `ProviderRejectedConsumer` | `NotificationService/Consumers/ProviderRejectedConsumer.cs` | Consumes `ProviderRejectedEvent`. Sends SMS with rejection reason |
| 2.B.19 | Register new consumers in NotificationService | `NotificationService/Program.cs` | `x.AddConsumer<ProviderApprovedConsumer>()`, `x.AddConsumer<ProviderRejectedConsumer>()` |
| 2.B.20 | Add Ocelot routes for admin + reapply | `ocelot.Docker.json`, `ocelot.Development.json` | Add `GET/PUT /api/admin/{everything}` → `auth-service:8080`. Add `PUT /api/providers/{providerId}/reapply` as **specific path entry** (not wildcard) placed **before** the existing `/api/providers/me/{everything}` → matching-service catch-all to prevent conflict |

### Frontend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 2.F.1 | Create `admin/` feature area | `features/admin/` | New lazy-loaded module |
| 2.F.2 | Create admin role guard | `core/guards/admin.guard.ts` | `CanActivateFn` — `auth.userRole() === 'admin' ? true : router.createUrlTree(['/'])` |
| 2.F.3 | Create `PendingProvidersComponent` | `admin/pending-providers/pending-providers.component.ts` | Table columns: Name, Mobile, Category, Registered Date, Status. Approve button → `PUT /api/admin/providers/{id}/approve`. Reject button → opens inline reason input → `PUT /api/admin/providers/{id}/reject`. Status filter tabs |
| 2.F.4 | Create `admin.routes.ts` | `admin/admin.routes.ts` | `/admin/providers/pending` guarded by `adminGuard` |
| 2.F.5 | Register admin routes in `app.routes.ts` | `app.routes.ts` | Lazy-load admin feature, apply `adminGuard` |

---

## Sprint 3 — PROF (US-PROF-01 to US-PROF-04)

### Stories

- **US-PROF-01** Customer views and edits profile (name, language, addresses; photo deferred per D5)
- **US-PROF-02** Provider sets service area (radius, working hours, IsOnline toggle)
- **US-PROF-03** Provider views their ratings and reviews (stub per D4)
- **US-PROF-04** Customer views booking history with filters

### Backend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 3.B.1 | Add `Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite` to AuthService | `ServeNow.AuthService.csproj` | `<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite" Version="8.0.11" />`. This is the EF spatial bridge — separate from the `NetTopologySuite` core package already in Shared. Update `UseSqlServer(cs, o => o.UseNetTopologySuite())` in `Program.cs` |
| 3.B.2 | Create `SavedAddress` value object | `AuthService/Domain/SavedAddress.cs` | `Guid Id`, `string Label`, `string FullAddress`, `string PinCode`, `NetTopologySuite.Geometries.Point Location` (SRID 4326). `ICollection<SavedAddress> Addresses` added to `AppUser` |
| 3.B.3 | Configure `SavedAddress` in `AuthDbContext` | `AuthDbContext.cs` | `OwnsMany(u => u.Addresses, a => { a.HasKey(x => x.Id); a.Property(x => x.Location).HasColumnType("geography"); })` |
| 3.B.4 | Add availability fields to `Provider` entity | `AuthService/Domain/Provider.cs` | Add `int ServiceRadiusKm = 5`, `string? WorkingHoursJson` (JSON-serialized per-day schedule) |
| 3.B.5 | Create EF migration | Terminal | `dotnet ef migrations add UserModuleSprint3 --project ServeNow.AuthService` — adds `SavedAddresses` table, `ServiceRadiusKm` + `WorkingHoursJson` columns on `Providers` |
| 3.B.6 | Create `AppUserDto` | `AuthService/DTOs/AppUserDto.cs` | `Id, FullName, Mobile, Role, PreferredLang, IsOnline, NotificationPrefs, Addresses` |
| 3.B.7 | Create `GetMyProfileQuery` + handler | `AuthService/Queries/Profile/GetMyProfileQuery.cs` | Reads `sub` claim from `IHttpContextAccessor`. Returns `AppUserDto` |
| 3.B.8 | Create `UpdateCustomerProfileCommand` + handler | `AuthService/Commands/Profile/UpdateCustomerProfileCommand.cs` | Updates `FullName`, `PreferredLang`. Validates name 2–60 chars |
| 3.B.9 | Create `AddSavedAddressCommand` + handler | `AuthService/Commands/Profile/AddSavedAddressCommand.cs` | Validates `Addresses.Count < 3` before adding. Converts `(double Latitude, double Longitude)` → `new Point(Longitude, Latitude) { SRID = 4326 }` (note: NTS takes X=longitude, Y=latitude) |
| 3.B.10 | Create `RemoveSavedAddressCommand` + handler | `AuthService/Commands/Profile/RemoveSavedAddressCommand.cs` | Removes by `AddressId`, verifies it belongs to requesting user |
| 3.B.11 | Create `UpdateProviderAvailabilityCommand` + handler | `AuthService/Commands/Profile/UpdateProviderAvailabilityCommand.cs` | Updates `Provider.ServiceRadiusKm` (validate 2–25), `WorkingHoursJson` (serialize input), `AppUser.IsOnline` |
| 3.B.12 | Create `GetProviderReviewsQuery` + handler — STUB | `AuthService/Queries/Profile/GetProviderReviewsQuery.cs` | Returns `PagedList<ReviewDto>` with empty `Items`, `TotalCount = 0`. Comment: "Stub — Reviews feature pending Review entity in BookingService" |
| 3.B.13 | Create `UsersController` | `AuthService/Controllers/UsersController.cs` | `[Authorize]` on all. `GET /api/users/me`, `PUT /api/users/me`, `PUT /api/users/me/addresses`, `DELETE /api/users/me/addresses/{addressId}` |
| 3.B.14 | Create `ProviderProfileController` | `AuthService/Controllers/ProviderProfileController.cs` | `[Route("api/auth/providers")]`. Endpoints: `PUT /api/auth/providers/availability [Authorize(Roles="Provider")]`, `GET /api/auth/providers/reviews [Authorize(Roles="Provider")]`. **Path prefix `/api/auth/providers/` deliberately avoids the existing Ocelot conflict:** `/api/providers/me/{everything}` → MatchingService |
| 3.B.15 | Add Ocelot routes for profile + provider profile | `ocelot.Docker.json`, `ocelot.Development.json` | Add: `GET /api/users/me` → `auth-service:8080`, `PUT /api/users/me` → `auth-service:8080`, `PUT /api/users/me/addresses` → `auth-service:8080`, `DELETE /api/users/me/addresses/{addressId}` → `auth-service:8080`, `/api/auth/providers/{everything}` → `auth-service:8080`. Place all new entries **before** the existing `/api/auth/{everything}` catch-all |
| 3.B.16 | Add Ocelot route for booking history | `ocelot.Docker.json`, `ocelot.Development.json` | `GET /api/users/me/bookings` → `booking-service:8080` with `DownstreamPathTemplate: "/api/bookings"`. Ocelot will forward the Bearer JWT unchanged |

### Frontend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 3.F.1 | Create `profile/` feature area | `features/profile/` | New lazy-loaded feature |
| 3.F.2 | Create `CustomerProfileComponent` | `profile/customer-profile/customer-profile.component.ts` | Editable name field, language toggle (Telugu/English — calls `PUT /api/users/me` on change), saved addresses list with add/remove. Placeholder avatar displayed (photo upload deferred) |
| 3.F.3 | Create `ProviderAvailabilityComponent` | `profile/provider-availability/provider-availability.component.ts` | Radius slider (2–25 km), hours-per-day grid (open/close time per weekday + "closed" checkbox), IsOnline toggle |
| 3.F.4 | Create `ProviderReviewsComponent` | `profile/provider-reviews/provider-reviews.component.ts` | Shows empty state: "No reviews yet — average 0.0". Ready to display data when review feature ships |
| 3.F.5 | Enhance existing `MyBookingsComponent` | `customer/my-bookings/my-bookings.component.ts` | Add status filter tabs (Upcoming / Completed / Cancelled), pagination. **Component already exists from prior session** |
| 3.F.6 | Create `profile.service.ts` | `profile/services/profile.service.ts` | `getProfile()` → `GET /api/users/me`, `updateProfile(data)` → `PUT /api/users/me`, `addAddress(addr)` → `PUT /api/users/me/addresses`, `removeAddress(id)` → `DELETE /api/users/me/addresses/{id}` |
| 3.F.7 | Add profile routes | `customer.routes.ts`, `provider.routes.ts` | Customer: `/customer/profile`. Provider: `/provider/profile`, `/provider/availability`, `/provider/reviews` |

---

## Sprint 4 — ACCT (US-ACCT-01 to US-ACCT-03)

### Stories

- **US-ACCT-01** Customer changes registered mobile number (dual OTP verification)
- **US-ACCT-02** User deletes own account (soft delete, OTP confirmation, 30-day background hard delete)
- **US-ACCT-03** Admin suspends a user account (Redis denylist, SMS notification)

### Backend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 4.B.1 | Add event contracts to `ServeNow.Shared` | `ServeNow.Shared/Contracts/Events.cs` | Append: `record UserSuspendedEvent(Guid UserId, string Mobile, string Reason, string Language)`, `record AccountDeletedEvent(Guid UserId, DateTimeOffset DeletedAt)` |
| 4.B.2 | Create `ChangeMobileCommand` + handler | `AuthService/Commands/Account/ChangeMobileCommand.cs` | Split into two steps: `InitiateChangeMobileCommand` (sends OTP to new number, checks not already registered) and `ConfirmChangeMobileCommand` (verifies OTP on old + new number, updates `MobileNumber`, issues new JWT, revokes all existing `RefreshTokens` for user) |
| 4.B.3 | Create `SoftDeleteAccountCommand` + handler | `AuthService/Commands/Account/SoftDeleteAccountCommand.cs` | Calls BookingService via `IHttpClientFactory("BookingService")` to check active bookings → 409 `ACTIVE_BOOKING_EXISTS`. Sets `IsDeleted = true`, `DeletedAt = UtcNow`. Bulk-revokes all user's `RefreshTokens`. Publishes `AccountDeletedEvent` |
| 4.B.4 | Create `SuspendUserCommand` + handler | `AuthService/Commands/Account/SuspendUserCommand.cs` | Admin only. Sets `SuspendedUntil`, optional expiry. Fetches all non-revoked `RefreshTokens` for target user → bulk revoke. Adds current session's `jti` to Redis denylist. Writes `AuditLog`. Publishes `UserSuspendedEvent` |
| 4.B.5 | Create `SuspendUserValidator` | `AuthService/Validators/SuspendUserValidator.cs` | `Reason`: NotEmpty, 10–500 chars. `SuspendedUntil`: must be `> DateTimeOffset.UtcNow` when provided |
| 4.B.6 | Register `IHttpClientFactory` for BookingService | `AuthService/Program.cs` | `builder.Services.AddHttpClient("BookingService", c => c.BaseAddress = new Uri(config["Services:BookingService"]!))`. Add `"Services:BookingService": "http://localhost:5001"` to `appsettings.Development.json` and `"http://booking-service:8080"` to Docker config |
| 4.B.7 | Create `AccountController` | `AuthService/Controllers/AccountController.cs` | `PUT /api/users/me/mobile/initiate`, `PUT /api/users/me/mobile/confirm`, `DELETE /api/users/me [Authorize]`, `PUT /api/admin/users/{id}/suspend [Authorize(Roles="Admin")]` |
| 4.B.8 | Create `UserSuspendedConsumer` | `NotificationService/Consumers/UserSuspendedConsumer.cs` | Sends SMS notification of suspension in preferred language |
| 4.B.9 | Create `HardDeleteSchedulerConsumer` | `NotificationService/Consumers/HardDeleteSchedulerConsumer.cs` | Listens to `AccountDeletedEvent`. Logs details. Schedules hard delete: if MassTransit delayed messages are available use `IMessageScheduler`, otherwise log a TODO for Hangfire integration |
| 4.B.10 | Register new consumers | `NotificationService/Program.cs` | `x.AddConsumer<UserSuspendedConsumer>()`, `x.AddConsumer<HardDeleteSchedulerConsumer>()` |
| 4.B.11 | Add Ocelot routes for account endpoints | `ocelot.Docker.json`, `ocelot.Development.json` | `PUT /api/users/me/mobile/{everything}` → `auth-service:8080`, `DELETE /api/users/me` → `auth-service:8080`, `PUT /api/admin/users/{everything}` → `auth-service:8080` |

### Frontend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 4.F.1 | Create `account/` feature area | `features/account/` | New lazy-loaded feature |
| 4.F.2 | Create `ChangeMobileComponent` | `account/change-mobile/change-mobile.component.ts` | Step 1: enter new number, submit → initiate. Step 2: two OTP inputs (old number OTP + new number OTP) → confirm. On success, update stored tokens and notify nav |
| 4.F.3 | Create `DeleteAccountComponent` | `account/delete-account/delete-account.component.ts` | Danger zone confirmation text. OTP input. Shows "Cannot delete — active booking exists" guidance on 409. On 204 success, calls `auth.logout()` |
| 4.F.4 | Add account routes | `customer.routes.ts`, `provider.routes.ts` | `/account/change-mobile`, `/account/delete` |

---

## Sprint 5 — NOTIF (US-NOTIF-01 to US-NOTIF-02)

### Stories

- **US-NOTIF-01** Customer receives push + SMS notifications on booking status changes (in preferred language)
- **US-NOTIF-02** User manages notification preferences (Push/SMS toggles; OTP/payment always on)

### Backend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 5.B.1 | Add EF Core to NotificationService | `ServeNow.NotificationService.csproj` | `<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.11" />`, `<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.11" />`. NotificationService currently has **no EF Core** — this is a prerequisite for `NotificationHistory` |
| 5.B.2 | Create `NotificationDbContext` | `NotificationService/NotificationDbContext.cs` | Connection string: `ServeNow_Notification` (new database). Register `AddDbContext<NotificationDbContext>` in `Program.cs`. Use `EnsureCreated` initially then migrate to `MigrateAsync` |
| 5.B.3 | Create `NotificationHistory` entity | `NotificationService/Domain/NotificationHistory.cs` | `Guid Id`, `Guid UserId`, `string Title`, `string Body`, `string Type`, `DateTimeOffset SentAt`, `DateTimeOffset? ReadAt`. Add index on `(UserId, SentAt)` |
| 5.B.4 | Create EF migration | Terminal | `dotnet ef migrations add InitialNotificationHistory --project ServeNow.NotificationService` |
| 5.B.5 | Add MediatR to NotificationService | `NotificationService/Program.cs` | `AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(GetNotificationHistoryQuery).Assembly))` |
| 5.B.6 | Create `GetNotificationHistoryQuery` + handler | `NotificationService/Queries/GetNotificationHistoryQuery.cs` | Paginated 20/page. Filter: `SentAt > UtcNow - 30 days`. Ordered newest first. User ID from JWT `sub` claim via `IHttpContextAccessor` |
| 5.B.7 | Create `UpdateNotificationPrefsCommand` + handler | `AuthService/Commands/Notif/UpdateNotificationPrefsCommand.cs` | Updates `AppUser.NotificationPrefs` via `IAppUserRepository`. Invalidates `notif:prefs:{userId}` from Redis via `ICacheService` |
| 5.B.8 | Update all existing consumers to check prefs | `NotificationService/Consumers/*.cs` | All consumers: before sending, load `NotificationPrefs` from Redis key `notif:prefs:{userId}` (fallback: HTTP GET `/api/users/me`). Skip if `PushEnabled = false` / `SmsEnabled = false`. **Never skip** for OTP or payment-related event types |
| 5.B.9 | Update `ProviderAcceptedConsumer` | `NotificationService/Consumers/ProviderAcceptedConsumer.cs` | Add i18n message selection based on `PreferredLang`. Persist notification to `NotificationHistory` after sending |
| 5.B.10 | Create `NotificationsController` | `NotificationService/Controllers/NotificationsController.cs` | `GET /api/users/me/notifications [Authorize]` — paginated history. `PUT /api/users/me/notifications/preferences [Authorize]` — forwards to AuthService via `IHttpClientFactory` (or publishes command via MassTransit) |
| 5.B.11 | Add Ocelot routes | `ocelot.Docker.json`, `ocelot.Development.json` | `GET /api/users/me/notifications` → `notification-service:8080`. `PUT /api/users/me/notifications/preferences` → `auth-service:8080` |

### Frontend Tasks

| # | Action | File(s) | Notes |
|---|---|---|---|
| 5.F.1 | Create `notifications/` feature area | `features/notifications/` | New lazy-loaded feature |
| 5.F.2 | Create `NotificationHistoryComponent` | `notifications/notification-history/notification-history.component.ts` | Scrollable list, unread dot indicator. Tap to mark as read |
| 5.F.3 | Create `NotificationPrefsComponent` | `notifications/notification-prefs/notification-prefs.component.ts` | Push/SMS toggle per event type. OTP and payment rows always-on and disabled in UI |
| 5.F.4 | Create `notifications.service.ts` | `notifications/services/notifications.service.ts` | `getHistory()` → `GET /api/users/me/notifications`, `updatePrefs(prefs)` → `PUT /api/users/me/notifications/preferences` |
| 5.F.5 | Add notification routes | `customer.routes.ts`, `provider.routes.ts` | `/notifications`, `/notifications/preferences` |

---

## API Gateway Routes — All Sprints

> All downstream ports are **8080** (Kestrel default inside Docker). External dev ports (5001, 5004…) are host port mappings only — never used in Ocelot config.

```
Sprint 0 — no Ocelot changes
Sprint 1 — no Ocelot changes
  Reason: existing "/api/auth/{everything}" → auth-service:8080 already covers all new auth endpoints

Sprint 2 — add to ocelot.Docker.json + ocelot.Development.json
  GET|PUT|DELETE  /api/admin/{everything}              → auth-service:8080
  PUT             /api/providers/{providerId}/reapply  → auth-service:8080   [specific, before wildcard]

Sprint 3 — add to ocelot config
  GET             /api/users/me                        → auth-service:8080
  PUT             /api/users/me                        → auth-service:8080
  PUT             /api/users/me/addresses              → auth-service:8080
  DELETE          /api/users/me/addresses/{addressId}  → auth-service:8080
  GET             /api/users/me/bookings               → booking-service:8080  [downstream: /api/bookings]
  *               /api/auth/providers/{everything}     → auth-service:8080     [new prefix, no conflict]

Sprint 4 — add to ocelot config
  PUT             /api/users/me/mobile/{everything}    → auth-service:8080
  DELETE          /api/users/me                        → auth-service:8080
  PUT             /api/admin/users/{everything}         → auth-service:8080

Sprint 5 — add to ocelot config
  GET             /api/users/me/notifications          → notification-service:8080
  PUT             /api/users/me/notifications/preferences → auth-service:8080
```

**Ocelot conflict note:** `/api/providers/me/{everything}` already routes to `matching-service:8080`. Provider profile endpoints in Sprint 3 use the `/api/auth/providers/` prefix deliberately to avoid this.

---

## New File Count Summary

### Backend (AuthService + NotificationService)

| Category | Count |
|---|---|
| Domain entities | 5 — `OtpRecord`, `RefreshToken`, `Provider`, `AuditLog`, `NotificationHistory` |
| Value objects / enums | 4 — `SavedAddress`, `NotificationPrefs`, `Language`, `ProviderStatus` |
| MediatR commands | 14 |
| MediatR queries | 5 |
| FluentValidation validators | 5 |
| Services + interfaces | 6 — `ICacheService`, `RedisCacheService`, `ITokenService`, `JwtTokenService`, `IAppUserRepository`, `IOtpRepository` + implementations |
| Repositories | 3 — `AppUserRepository`, `OtpRepository`, `ProviderRepository` |
| Controllers (new) | 5 — `UsersController`, `ProvidersController`, `AccountController`, `ProviderProfileController`, `NotificationsController` |
| Event contracts in Shared | 5 new appended to `ServeNow.Shared/Contracts/Events.cs` |
| MassTransit consumers (new) | 4 new + 1 upgraded — `ProviderApprovedConsumer`, `ProviderRejectedConsumer`, `UserSuspendedConsumer`, `HardDeleteSchedulerConsumer`, updated `ProviderAcceptedConsumer` |
| Middleware | 2 — `JwtDenylistMiddleware`, `AccountSuspendedMiddleware` |
| **Total new .cs files** | ~55 |

### Frontend (Angular)

| Category | Count |
|---|---|
| New standalone components | 10 |
| New services | 3 — `profile.service.ts`, `notifications.service.ts`, upgraded `auth.service.ts` |
| New feature areas | 4 — `admin/`, `profile/`, `account/`, `notifications/` |
| Auth page updates + new | 4 — `phone-login`, `otp-verify` updated; `register-customer`, `register-provider` new |
| Interceptor upgrade | 1 — `auth.interceptor.ts` refresh + retry |
| **Total new/modified Angular files** | ~25 |

---

## Error Response Codes (ProblemDetails — RFC 7807)

```
400  OTP_INVALID                Wrong OTP code
400  OTP_EXPIRED                OTP 5-min window passed
400  VALIDATION_FAILED          FluentValidation failure — includes errors[] array
401  TOKEN_EXPIRED              Access token expired (standard JWT validation)
401  TOKEN_INVALID              Revoked or malformed token (denylist hit)
403  INSUFFICIENT_ROLE          Non-admin accessing admin endpoint
403  ACCOUNT_SUSPENDED          User suspended — includes suspendedUntil in response
404  USER_NOT_FOUND
404  OTP_NOT_FOUND
404  PROVIDER_NOT_FOUND
409  MOBILE_ALREADY_REGISTERED
409  ACTIVE_BOOKING_EXISTS
429  MOBILE_LOCKED              OTP lockout active — includes retryAfterSeconds
```

---

## Redis Key Reference

| Key | TTL | Purpose |
|---|---|---|
| `otp:lock:{mobile}` | 15 min | Mobile lockout after 5 OTP failures |
| `jwt:denylist:{jti}` | Access token remaining TTL | Revoked access tokens (logout, suspend) |
| `notif:prefs:{userId}` | 1 hour | Cached notification preferences |

> Refresh tokens are persisted in the **SQL `RefreshTokens` table** (not Redis) to support rotation, auditing, and bulk revocation.

---

## Known Constraints

| # | Constraint | Detail |
|---|---|---|
| K1 | `AppUser` stays as `class` | CDD spec defines it as `record` but EF Core change tracking requires a mutable class. All new properties added as standard class properties |
| K2 | `ServeNow Orchestrator.md` | Out of scope for this plan — that document describes a separate cross-cutting orchestration pattern with its own implementation lifecycle |
| K3 | Profile photo upload | Deferred per D5. `PUT /api/users/me/photo` can be added when Azure Blob Storage account is available |
| K4 | Provider reviews | Stub endpoint only. Full implementation requires a `Review` entity in BookingService after the booking completion flow is finalized |
| K5 | NTS coordinate order | `NetTopologySuite.Geometries.Point` takes `(X=longitude, Y=latitude)` — the opposite of the common (lat, lng) convention. All address conversion code must use `new Point(longitude, latitude)` |
