# ServeNow API V1 — Booking Service Contracts

## Overview

Booking endpoints allow customers to create service booking requests and track their status.

---

## 1. Get Service Categories

**Endpoint:** `GET /api/services/categories`

### Query Parameters
```
locale=en_US  (optional, defaults to en)
```

### Response (200 OK)
```json
{
  "categories": [
    {
      "categoryId": "cat-plumbing",
      "name": "Plumbing",
      "description": "Pipe repairs, leaks, installations",
      "icon": "https://cdn.servenow.local/icons/plumbing.svg",
      "order": 1
    },
    {
      "categoryId": "cat-electrical",
      "name": "Electrical",
      "description": "Wiring, switches, fixtures",
      "icon": "https://cdn.servenow.local/icons/electrical.svg",
      "order": 2
    }
  ]
}
```

---

## 2. Categorize Problem

**Endpoint:** `POST /api/bookings/categorize`

**Headers:** `Authorization: Bearer <token>`

### Request
```json
{
  "problemDescription": "Water is leaking from the kitchen sink",
  "location": {
    "latitude": 17.3850,
    "longitude": 78.4867
  }
}
```

### Response (200 OK)
```json
{
  "suggestedCategories": [
    {
      "categoryId": "cat-plumbing",
      "name": "Plumbing",
      "confidence": 0.95
    },
    {
      "categoryId": "cat-general-maintenance",
      "name": "General Maintenance",
      "confidence": 0.45
    }
  ],
  "categorizedText": "Kitchen sink leak",
  "aiModel": "gpt-4-turbo"
}
```

### Notes
- Uses Azure OpenAI for intelligent categorization
- Returns confidence scores for ranking suggestions
- Takes ~1-2 seconds due to AI processing

---

## 3. Create Booking

**Endpoint:** `POST /api/bookings`

**Headers:** `Authorization: Bearer <token>`

### Request
```json
{
  "categoryId": "cat-plumbing",
  "problemDescription": "Water is leaking from the kitchen sink",
  "preferredDate": "2026-04-05T10:00:00Z",
  "preferredTimeWindow": "morning",
  "location": {
    "latitude": 17.3850,
    "longitude": 78.4867,
    "address": "123 Street, Hyderabad, TG"
  },
  "budget": {
    "type": "flexible",
    "minAmount": 500,
    "maxAmount": 2000,
    "currency": "INR"
  }
}
```

### Response (201 Created)
```json
{
  "bookingId": "bk-ab12cd34ef56",
  "status": "Created",
  "customerId": "cust-xyz789",
  "categoryId": "cat-plumbing",
  "location": {
    "latitude": 17.3850,
    "longitude": 78.4867
  },
  "createdAt": "2026-04-01T14:30:00Z",
  "estimatedArrival": null,
  "stage": "Awaiting Provider Acceptance"
}
```

**Status Values:**
- `Created` — Booking created, awaiting matching
- `Matching` — System finding providers
- `Offered` — Providers notified, awaiting acceptance
- `Accepted` — Provider accepted, en route
- `In Progress` — Service in progress
- `Completed` — Service completed
- `Cancelled` — Booking cancelled

---

## 4. Get Booking Details

**Endpoint:** `GET /api/bookings/{bookingId}`

**Headers:** `Authorization: Bearer <token>`

### Response (200 OK)
```json
{
  "bookingId": "bk-ab12cd34ef56",
  "status": "In Progress",
  "customerId": "cust-xyz789",
  "providerId": "prov-john123",
  "provider": {
    "providerId": "prov-john123",
    "name": "John Plumber",
    "rating": 4.8,
    "phone": "+919876543210",
    "eta": "10 minutes"
  },
  "categoryId": "cat-plumbing",
  "location": {
    "latitude": 17.3850,
    "longitude": 78.4867,
    "address": "123 Street, Hyderabad, TG"
  },
  "createdAt": "2026-04-01T14:30:00Z",
  "acceptedAt": "2026-04-01T14:45:00Z",
  "startedAt": "2026-04-01T15:00:00Z",
  "completedAt": null,
  "stage": "In Progress",
  "totalCost": null,
  "notes": "Kitchen sink leak repair"
}
```

---

## 5. Cancel Booking

**Endpoint:** `DELETE /api/bookings/{bookingId}`

**Headers:** `Authorization: Bearer <token>`

### Response (200 OK)
```json
{
  "bookingId": "bk-ab12cd34ef56",
  "status": "Cancelled",
  "cancellationReason": "Customer cancelled",
  "refundAmount": 0,
  "cancelledAt": "2026-04-01T15:15:00Z"
}
```

### Errors
- **400 Bad Request** — Cannot cancel completed booking
- **404 Not Found** — Booking not found

---

## 6. List Bookings (Customer)

**Endpoint:** `GET /api/bookings?status=in_progress&limit=10&offset=0`

**Headers:** `Authorization: Bearer <token>`

### Query Parameters
```
status=all        (all, active, completed, cancelled)
limit=10          (max 50)
offset=0          (pagination)
sortBy=createdAt  (createdAt, status, updatedAt)
order=desc        (asc, desc)
```

### Response (200 OK)
```json
{
  "bookings": [
    {
      "bookingId": "bk-ab12cd34ef56",
      "status": "In Progress",
      "categoryId": "cat-plumbing",
      "createdAt": "2026-04-01T14:30:00Z",
      "provider": {
        "providerId": "prov-john123",
        "name": "John Plumber",
        "rating": 4.8
      }
    }
  ],
  "total": 15,
  "hasMore": true
}
```

---

## Error Responses

```json
{
  "error": {
    "code": "BOOKING_NOT_FOUND",
    "message": "The requested booking does not exist.",
    "bookingId": "bk-invalid"
  }
}
```

**Standard Codes:**
- `BOOKING_NOT_FOUND` — Booking doesn't exist
- `INVALID_CATEGORY` — Category not found
- `DUPLICATE_BOOKING` — Similar booking already exists
- `BOOKING_LIMIT_REACHED` — User has too many active bookings
- `UNAUTHORIZED` — User cannot access this booking

