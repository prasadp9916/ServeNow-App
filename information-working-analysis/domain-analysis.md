# Domain Analysis: ServeNow Platform

**Created:** April 1, 2026  
**Last Updated:** April 1, 2026  
**Status:** Version 1.0

---

## Executive Summary

ServeNow is an on-demand service aggregator platform connecting customers with qualified home service providers (plumbers, electricians, etc.) through intelligent matching and real-time coordination.

---

## Business Model

**Revenue Streams:**
1. **Platform Commission** — 15-20% of booking value
2. **Premium Provider Subscriptions** — Featured listings, priority matching
3. **Advertisement** — Category sponsorships

**Key Stakeholders:**
- Customers (demand side)
- Providers (supply side)
- Admins (platform operations)

---

## Core Business Processes

### 1. Demand Generation (Customer Flow)

```
Customer Browse Categories
    ↓
Customer Describes Problem
    ↓
AI Categorizes Problem
    ↓
Customer Confirms Location & Budget
    ↓
Customer Creates Booking
    ↓
Booking Status Tracking (Matching → Offered → Accepted → In Progress → Completed)
```

### 2. Supply Management (Provider Flow)

```
Provider Registers & Onboards
    ↓
Provider Sets Availability & Pricing
    ↓
Provider Receives Booking Offers
    ↓
Provider Accepts/Rejects Booking
    ↓
Provider Completes Service
    ↓
Provider Receives Payment & Feedback
```

### 3. Matching Algorithm

```
Customer Creates Booking
    ↓
System Extracts: Category, Location, Budget, Preferences
    ↓
Query Available Providers in Category
    ↓
Calculate Match Scores:
  - Distance score (closer = higher)
  - Rating score (higher rating = higher)
  - Availability score (available now = 1.0)
  - Price alignment (matches budget)
  - Certification match (has required certs)
    ↓
Rank Providers by Composite Score
    ↓
Notify Top 3 Providers (RabbitMQ)
    ↓
Wait for First Acceptance (within 30 seconds)
    ↓
Update Booking Status to "Accepted"
```

---

## Database Domain Model

### Entities

#### Customer
- `id` (UUID, PK)
- `phone` (string, unique)
- `name` (string)
- `email` (string, optional)
- `role` (enum: customer, provider, admin)
- `isNewUser` (bool)
- `language` (enum: en, te)
- `createdAt` (timestamp)
- `deletedAt` (timestamp, soft delete)

**Relationships:**
- Has many Bookings
- Has many Reviews
- Has one Wallet
- Has one Address

#### Booking
- `id` (UUID, PK)
- `customerId` (FK)
- `providerId` (FK, nullable until accepted)
- `categoryId` (string)
- `status` (enum: Created, Matching, Offered, Accepted, In Progress, Completed, Cancelled)
- `location` (Point, geospatial)
- `address` (string)
- `problemDescription` (text)
- `budget` (decimal)
- `currency` (string)
- `createdAt` (timestamp)
- `acceptedAt` (timestamp, nullable)
- `completedAt` (timestamp, nullable)
- `cost` (decimal, nullable)

**Relationships:**
- Belongs to Customer
- Belongs to Provider (accepted state)
- Has many BookingEvents (state transitions)
- Has one Rating (after completion)

#### Provider
- `id` (UUID, PK)
- `userId` (FK)
- `name` (string)
- `phone` (string)
- `email` (string)
- `rating` (decimal, 0-5)
- `reviewCount` (int)
- `completionRate` (decimal, 0-1)
- `yearsActive` (int)
- `locationLat` (decimal)
- `locationLng` (decimal)
- `serviceRadius` (int, km)
- `currentLoad` (int)
- `maxLoad` (int)
- `hourlyRate` (decimal)
- `kmRate` (decimal)
- `minimumCharge` (decimal)
- `verified` (bool)
- `availableFrom` (time)
- `availableUntil` (time)
- `createdAt` (timestamp)

**Relationships:**
- Has one User
- Has many Certifications
- Has many Bookings
- Has many AvailabilitySlots
- Has many Reviews

#### Category
- `id` (string, PK)
- `name` (string)
- `description` (text)
- `icon` (string, URL)
- `displayOrder` (int)
- `active` (bool)

**Relationships:**
- Has many Providers (many-to-many via ProviderCategory)
- Has many Bookings

#### Certification
- `id` (UUID, PK)
- `providerId` (FK)
- `name` (string)
- `code` (string)
- `issuer` (string)
- `issueDate` (date)
- `expiryDate` (date, nullable)
- `verifified` (bool)

---

## Bounded Contexts (DDD)

### 1. Authentication Context
**Responsibility:** User identity, authentication, role-based access

**Entities:** User, AuthSession, OtpToken  
**Aggregates:** User (with role)  
**Services:** OtpService, TokenService  
**Events:** UserRegistered, UserLoggedIn, RoleChanged

### 2. Booking Context
**Responsibility:** Booking lifecycle, status management

**Entities:** Booking, BookingEvent  
**Aggregates:** Booking  
**Services:** BookingService, BookingRepository  
**Events:** BookingCreated, BookingMatched, BookingAccepted, BookingCompleted

### 3. Matching Context
**Responsibility:** Intelligent provider matching, ranking algorithm

**Entities:** MatchScore, MatchResult  
**Aggregates:** MatchResult  
**Services:** MatchingService, ScoringEngine, AvailabilityService  
**Events:** ProvidersMatched, ProviderNotified, MatchAccepted

### 4. Provider Context
**Responsibility:** Provider profiles, available, pricing, certifications

**Entities:** Provider, Certification, AvailabilitySlot  
**Aggregates:** Provider (with certifications, availability)  
**Services:** ProviderService, ProviderRepository, AvailabilityService  
**Events:** ProviderRegistered, AvailabilityUpdated, PricingChanged

### 5. Notification Context
**Responsibility:** SMS, email, push notifications

**Services:** NotificationService, EmailService, SmsSender, PushService  
**Events:** BookingCreated → notify customer, ProvidersMatched → notify providers

---

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **Frontend** | Angular | 17 |
| **Backend** | .NET | 8 |
| **Database** | SQL Server | 2022 |
| **Cache** | Redis | 7.0 |
| **Messaging** | RabbitMQ | 3.13 |
| **AI** | Azure OpenAI | gpt-4-turbo |
| **Maps** | Azure Maps / Google Maps | Latest |

---

## Data Flow Diagrams

### Booking Creation Flow

```
┌─────────────┐
│ Customer    │
└──────┬──────┘
       │ 1. Browse Categories
       ↓
┌─────────────────────────────────────┐
│ Frontend: servenow-portal           │
│ - Category Selector                 │
│ - Problem Descriptor                │
│ - Location Confirmation             │
│ - Budget Input                      │
└──────┬──────────────────────────────┘
       │ 2. POST /api/bookings/categorize
       ↓
┌─────────────────────────────────────┐
│ BookingService                      │
│ - AI Categorization (Azure OpenAI)  │
│ - Validation                        │
└──────┬──────────────────────────────┘
       │ 3. POST /api/bookings
       ↓
┌─────────────────────────────────────┐
│ Booking DB                          │
│ - Create booking (status=Created)   │
└──────┬──────────────────────────────┘
       │ 4. Publish BookingCreated event
       ↓
┌─────────────────────────────────────┐
│ MatchingService (RabbitMQ)          │
│ - Rank providers                    │
│ - Select top 3                      │
└──────┬──────────────────────────────┘
       │ 5. Publish ProvidersMatched
       ↓
┌─────────────────────────────────────┐
│ NotificationService                 │
│ - Notify selected providers (SMS)   │
│ - Notify customer (email)           │
└─────────────────────────────────────┘
```

---

## Integration Points

| System | Protocol | Purpose |
|--------|----------|---------|
| Azure OpenAI | REST API | Problem categorization |
| Azure Maps | REST API | Geocoding, route calculation |
| Google Maps | SDK | Frontend map display |
| Twilio | REST API | SMS notifications |
| SendGrid | REST API | Email notifications |
| Firebase | REST API | Push notifications |
| Stripe | REST API | Payment processing |

---

## Scalability Considerations

### Load Bottlenecks

1. **Booking Creation Spike**
   - Solution: Message queue (RabbitMQ) + worker pool
   - Peak: 10,000 bookings/minute

2. **Matching Algorithm**
   - Solution: Elasticsearch for provider indexing, async computation
   - 1000 providers × geospatial queries

3. **Real-time Updates**
   - Solution: WebSocket + Redis pub/sub
   - 100,000 concurrent websocket connections

4. **Database Load**
   - Solution: Read replicas, caching layer, query optimization
   - Index on: (customer_id, created_at), (status), (location)

### Caching Strategy

```
Redis Cache:
- Categories: 1 hour TTL (cache hit: 95%)
- Provider availability: 5 minute TTL
- User sessions: 7 day TTL
- Match scores: 30 second TTL (per booking+providers combo)
```

---

## Future Enhancements

1. **Subscription Services** — Recurring bookings (weekly cleaning, etc.)
2. **Telemedicine Integration** — Video consultations before booking
3. **Payment Integration** — In-app payments vs. cash on delivery
4. **Analytics Dashboard** — Provider earnings, customer spending trends
5. **Provider Marketplace** — Providers can list packages/promotions
6. **Chatbot Support** — AI-powered customer service
7. **Bill Splitting** — Shared bookings with cost split

