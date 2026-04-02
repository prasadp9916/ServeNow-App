# ServeNow API V1 — Matching Service Contracts

## Overview

Matching endpoints rank and recommend providers based on availability, location, skill match, and customer preferences.

---

## 1. Rank Providers

**Endpoint:** `POST /api/matching/rank`

**Headers:** `Authorization: Bearer <token>`

### Request
```json
{
  "bookingId": "bk-ab12cd34ef56",
  "categoryId": "cat-plumbing",
  "location": {
    "latitude": 17.3850,
    "longitude": 78.4867
  },
  "budget": {
    "minAmount": 500,
    "maxAmount": 2000,
    "currency": "INR"
  },
  "preferences": {
    "maxDistance": 15,
    "minRating": 3.5,
    "certificationsRequired": ["licensed-plumber"]
  }
}
```

### Response (200 OK)
```json
{
  "providers": [
    {
      "providerId": "prov-john123",
      "name": "John Plumber",
      "rating": 4.8,
      "reviewCount": 142,
      "phone": "+919876543210",
      "distance": 2.5,
      "distanceUnit": "km",
      "eta": "12 minutes",
      "availability": {
        "available": true,
        "currentLoad": 1,
        "maxLoad": 3
      },
      "certifications": ["licensed-plumber", "master-plumber"],
      "languages": ["en", "te"],
      "hourlyRate": 500,
      "kmRate": 10,
      "matchScore": 0.94,
      "scoreBreakdown": {
        "distance": 0.95,
        "rating": 0.92,
        "availability": 1.0,
        "priceAlignment": 0.90,
        "certifications": 1.0
      },
      "avatar": "https://cdn.servenow.local/avatars/john-123.jpg"
    },
    {
      "providerId": "prov-ramu456",
      "name": "Ramu Plumber",
      "rating": 4.6,
      "reviewCount": 98,
      "phone": "+918765432109",
      "distance": 5.2,
      "distanceUnit": "km",
      "eta": "18 minutes",
      "availability": {
        "available": true,
        "currentLoad": 2,
        "maxLoad": 3
      },
      "certifications": ["licensed-plumber"],
      "languages": ["en", "te"],
      "hourlyRate": 450,
      "kmRate": 8,
      "matchScore": 0.88,
      "scoreBreakdown": {
        "distance": 0.82,
        "rating": 0.90,
        "availability": 0.95,
        "priceAlignment": 0.88,
        "certifications": 0.85
      },
      "avatar": "https://cdn.servenow.local/avatars/ramu-456.jpg"
    }
  ],
  "radiusUsedKm": 10,
  "expandedSearch": false,
  "totalMatches": 2,
  "searchDurationMs": 450
}
```

**Match Score Components:**
- **Distance** (0-1) — Lower distance = higher score
- **Rating** (0-1) — Higher rating = higher score
- **Availability** (0-1) — Currently free = 1.0
- **Price Alignment** (0-1) — Budget match
- **Certifications** (0-1) — Required certifications present
- **Overall Score** — Weighted average of above

---

## 2. Get Provider Details

**Endpoint:** `GET /api/providers/{providerId}`

**Headers:** `Authorization: Bearer <token>`

### Response (200 OK)
```json
{
  "providerId": "prov-john123",
  "name": "John Plumber",
  "rating": 4.8,
  "reviewCount": 142,
  "phone": "+919876543210",
  "email": "john@plumber.local",
  "certifications": [
    {
      "name": "Licensed Plumber",
      "code": "licensed-plumber",
      "issuer": "Telangana Regulation Board",
      "expiresAt": "2027-12-31"
    },
    {
      "name": "Master Plumber",
      "code": "master-plumber",
      "issuer": "Plumbers Guild",
      "expiresAt": null
    }
  ],
  "experience": {
    "yearsActive": 8,
    "totalJobs": 142,
    "completionRate": 0.98
  },
  "pricing": {
    "hourlyRate": 500,
    "kmRate": 10,
    "minimumChargeAmount": 500,
    "currency": "INR"
  },
  "availability": {
    "available": true,
    "currentLoad": 1,
    "maxLoad": 3,
    "workingHours": {
      "startTime": "08:00",
      "endTime": "20:00",
      "daysOff": ["Sunday"]
    }
  },
  "location": {
    "latitude": 17.3750,
    "longitude": 78.4950,
    "serviceRadius": 15,
    "address": "Hyderabad, Telangana"
  },
  "languages": ["en", "te", "hi"],
  "specializations": ["plumbing", "water-purification"],
  "reviews": [
    {
      "reviewId": "rev-001",
      "rating": 5,
      "title": "Excellent work!",
      "comment": "Fixed the leak perfectly. Very professional.",
      "customerId": "cust-alice",
      "createdAt": "2026-03-28T10:00:00Z"
    }
  ],
  "averageReviewRating": 4.8,
  "avatar": "https://cdn.servenow.local/avatars/john-123.jpg",
  "verified": true
}
```

---

## 3. Search Providers

**Endpoint:** `GET /api/providers/search?category=plumbing&location=17.3850,78.4867&distance=10`

**Query Parameters:**
```
category=plumbing           (required)
location=lat,lng            (required)
distance=10                 (km, optional)
minRating=3.5               (optional)
skills=skill1,skill2        (optional)
limit=20                    (max 50)
offset=0                    (pagination)
```

### Response (200 OK)
```json
{
  "providers": [...],
  "total": 45,
  "hasMore": true,
  "location": {
    "latitude": 17.3850,
    "longitude": 78.4867
  }
}
```

---

## 4. Match Acceptance Webhook

When a provider accepts a booking, the matching service can notify via webhook:

**Webhook Event:** `booking.match.accepted`

```json
{
  "event": "booking.match.accepted",
  "timestamp": "2026-04-01T15:00:00Z",
  "bookingId": "bk-ab12cd34ef56",
  "providerId": "prov-john123",
  "provider": {
    "name": "John Plumber",
    "phone": "+919876543210",
    "eta": "12 minutes",
    "rating": 4.8
  },
  "updatedStatus": "Accepted"
}
```

---

## Error Responses

```json
{
  "error": {
    "code": "NO_PROVIDERS_FOUND",
    "message": "No providers available matching your criteria.",
    "expandedRadius": 20
  }
}
```

**Standard Codes:**
- `NO_PROVIDERS_FOUND` — No matching providers in area
- `INVALID_LOCATION` — Location coordinates invalid
- `INVALID_CATEGORY` — Category doesn't exist
- `SEARCH_TIMEOUT` — Matching engine timeout
- `PROVIDER_NOT_FOUND` — Provider doesn't exist

