# ServeNow API V1 — Error Handling Standards

## Overview

All API errors follow a consistent format with structured error codes, human-readable messages, and optional detail objects.

---

## Error Response Format

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "timestamp": "2026-04-01T15:00:00Z",
    "traceId": "00-trace-id-00",
    "details": {
      "field": "value",
      "additionalContext": "info"
    }
  }
}
```

---

## HTTP Status Codes

| Status | Meaning | Example |
|--------|---------|---------|
| **200** | Success | Request completed |
| **201** | Created | Resource created |
| **204** | No Content | Success, no body |
| **400** | Bad Request | Invalid input |
| **401** | Unauthorized | Missing/invalid token |
| **403** | Forbidden | No permission |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Duplicate/conflict |
| **422** | Unprocessable Entity | Validation error |
| **429** | Too Many Requests | Rate limited |
| **500** | Server Error | Unexpected error |
| **503** | Service Unavailable | Temporary outage |

---

## Standard Error Codes

### Authentication Errors (401, 403)

| Code | HTTP | Message | Action |
|------|------|---------|--------|
| `TOKEN_EXPIRED` | 401 | Session expired. Please login again. | Re-authenticate |
| `TOKEN_INVALID` | 401 | Invalid or malformed token. | Re-authenticate |
| `NOT_AUTHENTICATED` | 401 | Authentication required. | Login required |
| `INSUFFICIENT_PERMISSIONS` | 403 | You do not have permission. | Request elevated access |
| `ROLE_RESTRICTED` | 403 | This resource requires a different role. | Role mismatch |

### Validation Errors (400, 422)

| Code | HTTP | Message | Action |
|------|------|---------|--------|
| `INVALID_REQUEST` | 400 | Request body is invalid JSON. | Check JSON syntax |
| `MISSING_FIELD` | 422 | Required field is missing. | Provide all fields |
| `INVALID_FORMAT` | 422 | Field format is invalid. | Check format spec |
| `VALUE_OUT_OF_RANGE` | 422 | Value exceeds limits. | Provide valid value |
| `DUPLICATE_RECORD` | 409 | Record already exists. | Use existing record |

### Not Found Errors (404)

| Code | HTTP | Message | Action |
|------|------|---------|--------|
| `RESOURCE_NOT_FOUND` | 404 | Resource doesn't exist. | Verify ID |
| `BOOKING_NOT_FOUND` | 404 | Booking not found. | Check booking ID |
| `PROVIDER_NOT_FOUND` | 404 | Provider not found. | Check provider ID |
| `CATEGORY_NOT_FOUND` | 404 | Category not found. | Check category ID |

### Business Logic Errors (400, 409)

| Code | HTTP | Message | Action |
|------|------|---------|--------|
| `INVALID_STATE` | 400 | Operation not allowed in current state. | Check status |
| `BOOKING_LIMIT_REACHED` | 409 | User has max active bookings. | Complete existing |
| `PROVIDER_UNAVAILABLE` | 400 | Provider currently unavailable. | Choose another |
| `LOCATION_OUT_OF_SERVICE` | 400 | Location outside service area. | Choose valid area |
| `INSUFFICIENT_BALANCE` | 400 | Insufficient payment balance. | Add funds |

### Rate Limiting (429)

| Code | HTTP | Message | Action |
|------|------|---------|--------|
| `RATE_LIMITED` | 429 | Too many requests. Retry after X seconds. | Wait & retry |
| `OTP_ATTEMPTS_EXCEEDED` | 429 | Too many OTP attempts. Try again later. | Request new OTP |

### Server Errors (500, 503)

| Code | HTTP | Message | Action |
|------|------|---------|--------|
| `INTERNAL_ERROR` | 500 | Unexpected server error. | Retry later |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable. | Retry later |
| `DATABASE_ERROR` | 500 | Database operation failed. | Retry later |
| `EXTERNAL_SERVICE_ERROR` | 500 | External service error. | Retry later |

---

## Validation Error Example

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "timestamp": "2026-04-01T15:00:00Z",
    "traceId": "00-12345-abcde",
    "details": {
      "validationErrors": [
        {
          "field": "phone",
          "rule": "format",
          "message": "Phone must be in format +91XXXXXXXXXX"
        },
        {
          "field": "budget.maxAmount",
          "rule": "min",
          "message": "Must be at least 100"
        }
      ]
    }
  }
}
```

---

## Rate Limiting Headers

All successful responses include rate limit headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1672531200
```

When rate limited (429):

```
Retry-After: 60
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531260
```

---

## Handling Errors in Client

### Recommended Client Logic

```typescript
async function handleApiError(error: any) {
  const code = error.error?.code;
  const status = error.status;

  // Re-authenticate on token errors
  if (code === 'TOKEN_EXPIRED' || code === 'TOKEN_INVALID') {
    window.location.href = '/auth/login';
    return;
  }

  // Alert user for validation errors
  if (status === 422 || status === 400) {
    showValidationDialog(error.error.details.validationErrors);
    return;
  }

  // Retry on server errors with exponential backoff
  if (status >= 500) {
    await retryWithBackoff(() => apiCall(), 3);
    return;
  }

  // Show generic error message
  showErrorToast(error.error.message);
}
```

