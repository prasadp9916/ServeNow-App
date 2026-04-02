# ServeNow API V1 — Authentication Contracts

## Overview

All authentication endpoints return a JWT token that must be included in subsequent requests via the `Authorization: Bearer <token>` header.

---

## 1. Send OTP

**Endpoint:** `POST /api/auth/send-otp`

### Request
```json
{
  "phone": "+919000000001"
}
```

### Response (200 OK)
```json
{
  "otpSent": true,
  "sessionId": "session-abc123",
  "expiresIn": 300
}
```

### Errors
- **400 Bad Request** — Invalid phone format
- **429 Too Many Requests** — Rate limit exceeded

---

## 2. Verify OTP

**Endpoint:** `POST /api/auth/verify-otp`

### Request
```json
{
  "sessionId": "session-abc123",
  "otp": "123456"
}
```

### Response (200 OK) — Returning User
```json
{
  "token": "eyJhbGc...",
  "isNewUser": false,
  "role": "customer",
  "expiresIn": 3600
}
```

### Response (200 OK) — New User (requires role selection)
```json
{
  "token": "eyJhbGc...",
  "isNewUser": true,
  "role": null,
  "expiresIn": 3600
}
```

### Errors
- **401 Unauthorized** — Invalid or expired OTP
- **410 Gone** — Session expired

---

## 3. Set Role (New Users Only)

**Endpoint:** `POST /api/auth/set-role`

**Headers:** `Authorization: Bearer <token>`

### Request
```json
{
  "role": "customer"
}
```

### Response (200 OK)
```json
{
  "token": "eyJhbGc...",
  "role": "customer",
  "expiresIn": 3600
}
```

### Notes
- Only valid for new users (isNewUser = true)
- Allowed roles: `customer`, `provider`, `admin`
- Returns new token with role claims embedded

---

## 4. Validate Token

**Endpoint:** `GET /api/auth/validate`

**Headers:** `Authorization: Bearer <token>`

### Response (200 OK)
```json
{
  "valid": true,
  "userId": "user-123",
  "role": "customer",
  "expiresIn": 3400
}
```

### Errors
- **401 Unauthorized** — Invalid or expired token

---

## JWT Token Structure

All tokens follow this claims structure:

```json
{
  "sub": "user-123",
  "phone": "+919000000001",
  "role": "customer",
  "isNewUser": false,
  "exp": 1672531200,
  "iat": 1672527600
}
```

**Token Expiry:** 1 hour (3600 seconds)

---

## Test Credentials

### Development Mock Data
```
CUSTOMER  Phone: 9000000001   OTP: 111111
PROVIDER  Phone: 9000000002   OTP: 222222
ADMIN     Phone: 9000000003   OTP: 333333
NEW USER  Any valid phone     OTP: 123456  → Role Select required
```

---

## Error Responses

All endpoints return errors in this format:

```json
{
  "error": {
    "code": "INVALID_OTP",
    "message": "The OTP provided is invalid or expired.",
    "details": {
      "attemptsRemaining": 2
    }
  }
}
```

**Standard Codes:**
- `INVALID_OTP` — Wrong OTP
- `SESSION_EXPIRED` — Session no longer valid
- `RATE_LIMITED` — Too many requests
- `INVALID_PHONE` — Phone format invalid
- `SERVER_ERROR` — Unexpected server error

