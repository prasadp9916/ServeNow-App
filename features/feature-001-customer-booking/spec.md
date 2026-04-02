# Feature-001: Customer Booking Flow

**Feature ID:** F-001  
**Status:** In Development  
**Priority:** P0 (Critical)  
**Target Release:** Q2 2026

---

## Overview

Customers request on-demand home services by selecting a problem category, describing their issue, and confirming location. The system intelligently categorizes the issue, matches qualified providers, and coordinates the engagement.

---

## User Stories

### Story 1.1: Browse Service Categories

**As a** customer  
**I want to** see available service categories  
**So that** I can choose what service I need

**Acceptance Criteria:**
- [ ] GET `/api/services/categories` returns all categories with icons
- [ ] Categories display name, description, and icon
- [ ] Categories are displayed in defined order
- [ ] Multi-language support (English, Telugu)
- [ ] Icon CDN has 99.9% uptime

**Estimation:** 3 points  
**Owner:** Frontend Lead

---

### Story 1.2: Describe Booking Problem

**As a** customer  
**I want to** describe my problem in natural language  
**So that** the system can automatically categorize it

**Acceptance Criteria:**
- [ ] Text input accepts 20-500 characters
- [ ] Real-time character count displayed
- [ ] Problem description is required field
- [ ] Submit button disabled until description provided
- [ ] Loading spinner shows during AI processing

**Estimation:** 2 points  
**Owner:** Frontend Engineer

---

### Story 1.3: AI-Powered Problem Categorization

**As a** system  
**I want to** use Azure OpenAI to categorize problems  
**So that** accurate service matching occurs

**Acceptance Criteria:**
- [ ] POST `/api/bookings/categorize` calls Azure OpenAI
- [ ] Returns confidence scores for top 3 categories
- [ ] Processes within 2 seconds (p95)
- [ ] Fallback to manual category if AI fails
- [ ] Logs categorization attempts for analytics
- [ ] Respects AI API rate limits

**Estimation:** 5 points  
**Owner:** Backend Engineer

---

### Story 1.4: Confirm Location & Budget

**As a** customer  
**I want to** provide my location and budget range  
**So that** providers can provide accurate quotes

**Acceptance Criteria:**
- [ ] GPS location auto-populated (with permission)
- [ ] Manual address entry available
- [ ] Address geocoded and validated
- [ ] Budget range has min/max inputs
- [ ] Flexible/fixed budget options available
- [ ] Validates service availability at location

**Estimation:** 4 points  
**Owner:** Frontend/Backend Engineers

---

### Story 1.5: Create Booking Request

**As a** customer  
**I want to** submit a booking request  
**So that** providers can respond with quotes

**Acceptance Criteria:**
- [ ] POST `/api/bookings` creates booking atomically
- [ ] Booking ID returned immediately
- [ ] Status set to "Created"
- [ ] Confirmation SMS sent to customer
- [ ] Confirmation shown in UI
- [ ] Load test: 1000 concurrent bookings per minute

**Estimation:** 4 points  
**Owner:** Backend Lead

---

### Story 1.6: Track Booking Status

**As a** customer  
**I want to** see real-time booking status  
**So that** I know when provider is arriving

**Acceptance Criteria:**
- [ ] GET `/api/bookings/{id}` returns current status
- [ ] WebSocket connection for live updates
- [ ] Status transitions: Created → Matching → Offered → Accepted → In Progress → Completed
- [ ] Provider details shown when accepted
- [ ] ETA countdown timer displayed
- [ ] Cancel button available before provider arrival

**Estimation:** 6 points  
**Owner:** Full Stack Team

---

## Implementation Details

### Database Schema

**bookings table:**
```sql
CREATE TABLE bookings (
  booking_id UUID PRIMARY KEY,
  customer_id UUID NOT NULL REFERENCES customers(id),
  category_id VARCHAR(50) NOT NULL,
  status VARCHAR(20) NOT NULL,
  location_lat DECIMAL(10,8),
  location_lng DECIMAL(11,8),
  address VARCHAR(255),
  problem_description TEXT,
  budget_min INT,
  budget_max INT,
  currency VARCHAR(3),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP NULL
);

CREATE INDEX idx_bookings_customer ON bookings(customer_id);
CREATE INDEX idx_bookings_status ON bookings(status);
CREATE INDEX idx_bookings_created ON bookings(created_at DESC);
```

### API Endpoints

| Method | Endpoint | Controller | Status |
|--------|----------|-----------|--------|
| GET | `/api/services/categories` | ServiceController.GetCategories | ✅ Implemented |
| POST | `/api/bookings/categorize` | BookingController.Categorize | ✅ Implemented |
| POST | `/api/bookings` | BookingController.CreateBooking | ✅ Implemented |
| GET | `/api/bookings/{id}` | BookingController.GetBooking | ✅ Implemented |
| GET | `/api/bookings` | BookingController.ListBookings | ✅ Implemented |
| DELETE | `/api/bookings/{id}` | BookingController.CancelBooking | 🔄 In Progress |

### Frontend Components

- [ ] `BookingFlow` — Main feature component
- [ ] `CategorySelector` — Category selection UI
- [ ] `ProblemDescriptor` — Text input component
- [ ] `LocationConfirm` — Map + address input
- [ ] `BudgetInput` — Budget range selector
- [ ] `BookingConfirmation` — Confirmation dialog
- [ ] `BookingTracker` — Status tracking UI
- [ ] `ProviderCard` — Provider info display

---

## Success Metrics

- **Conversion Rate:** ≥70% of category selections → booking creation
- **Time to Book:** <2 minutes average
- **Cancellation Rate:** <5% within first hour
- **Customer Satisfaction:** ≥4.5/5 stars post-booking

---

## Non-Functional Requirements

- **Performance:** All API calls <500ms p95
- **Availability:** 99.95% uptime
- **Scalability:** Support 10,000 concurrent bookings
- **Data Privacy:** GDPR compliant, encrypted at rest

---

## Dependencies

- Azure OpenAI API for categorization
- Google Maps API for geocoding
- RabbitMQ for event publishing
- SQL Server for data storage
- Redis for caching

---

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|-----------|
| OpenAI API latency | Medium | Fallback to manual categorization |
| High booking volume spike | High | Auto-scaling + queue system |
| Location accuracy | Medium | Manual address entry option |
| Budget estimation errors | Low | Show confidence scores |

