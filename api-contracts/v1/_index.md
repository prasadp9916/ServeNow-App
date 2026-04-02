# ServeNow API V1 — Contract Index

**Version:** 1.0.0  
**Last Updated:** April 1, 2026  
**Status:** Specification (Ready for Implementation)

---

## API Overview

ServeNow API is a RESTful service providing:
- **Authentication** — Phone-based OTP login with role-based access
- **Booking Management** — Create, track, and manage service bookings
- **Provider Matching** — Intelligent provider ranking and search
- **Service Categories** — Browse available service categories
- **Real-time Updates** — WebSocket for live booking status

---

## Base URL

**Production:** `https://api.servenow.com`  
**Staging:** `https://staging-api.servenow.com`  
**Development:** `http://localhost:3000`

---

## Authentication

All endpoints (except `/auth/**`) require:

```
Authorization: Bearer <jwt-token>
```

Obtained from `/api/auth/verify-otp` or `/api/auth/set-role`.

---

## Contracts by Module

### 1. Authentication (`/api/auth/`)
- `POST /api/auth/send-otp` — Send OTP to phone
- `POST /api/auth/verify-otp` — Verify OTP and get token
- `POST /api/auth/set-role` — Set user role (new users)
- `GET /api/auth/validate` — Validate token

**File:** [auth-api.md](auth-api.md)

### 2. Booking Service (`/api/bookings/`)
- `GET /api/services/categories` — Browse service categories
- `POST /api/bookings/categorize` — AI categorization of issues
- `POST /api/bookings` — Create new booking
- `GET /api/bookings/{id}` — Get booking details
- `DELETE /api/bookings/{id}` — Cancel booking
- `GET /api/bookings` — List user bookings

**File:** [booking-api.md](booking-api.md)

### 3. Matching Service (`/api/matching/`)
- `POST /api/matching/rank` — Rank providers for booking
- `GET /api/providers/{id}` — Get provider details
- `GET /api/providers/search` — Search providers

**File:** [matching-api.md](matching-api.md)

### 4. Error Handling
Standard error format, status codes, and error codes across all endpoints.

**File:** [error-handling.md](error-handling.md)

---

## Common Patterns

### Pagination

Endpoints supporting pagination:

```
GET /api/resource?limit=20&offset=0

Response:
{
  "items": [...],
  "total": 100,
  "limit": 20,
  "offset": 0,
  "hasMore": true
}
```

### Sorting

```
GET /api/resource?sortBy=createdAt&order=desc

Parameters:
- sortBy: field name to sort by
- order: asc or desc
```

### Filtering

```
GET /api/bookings?status=in_progress&categoryId=cat-plumbing

Filters are endpoint-specific
```

### Date/Time Format

All timestamps use ISO 8601 format:

```
2026-04-01T15:30:45Z
```

---

## Content Types

**Request:**
```
Content-Type: application/json
```

**Response:**
```
Content-Type: application/json; charset=utf-8
```

---

## Rate Limiting

- **Limit:** 1000 requests per hour per user
- **Headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- **Exceeded:** Returns 429 with `Retry-After` header

---

## Versioning

API versions follow `/v1/`, `/v2/`, etc. pattern.

**Current Version:** `v1`

Breaking changes create new versions. Old versions deprecated after 6 months.

---

## Testing

Use these mock credentials for development:

```
Customer:
  Phone: 9000000001
  OTP: 111111
  Token Claims: role=customer, isNewUser=false

Provider:
  Phone: 9000000002
  OTP: 222222
  Token Claims: role=provider, isNewUser=false

Admin:
  Phone: 9000000003
  OTP: 333333
  Token Claims: role=admin, isNewUser=false

New User:
  Phone: Any valid number
  OTP: 123456
  Token Claims: role=undefined, isNewUser=true → requires /api/auth/set-role
```

---

## Related Documents

- [Architecture Guide](../../ARCHITECTURE.md)
- [Booking Flow Documentation](../../servenow-booking-flow-backend.md)
- [Development Environment Setup](../../../ENVIRONMENT_SETUP.md)

