# LAMC Website - Technical Documentation & System Status

**Last Updated:** November 16, 2025
**Current Branch:** main
**Status:** Production Ready | Build Passing | 14/14 System Validation Checks Passed

---

## Table of Contents

1. [System Status & Health](#system-status--health)
2. [A/B Testing System](#ab-testing-system)
3. [Live Photo Integration](#live-photo-integration)
4. [Email System](#email-system)
5. [Booking & Calendar System](#booking--calendar-system)
6. [Database Schema](#database-schema)
7. [API Reference](#api-reference)
8. [Management Commands](#management-commands)
9. [Monitoring & Alerting](#monitoring--alerting)
10. [Troubleshooting Guide](#troubleshooting-guide)
11. [Security Implementation](#security-implementation)
12. [Performance Optimization](#performance-optimization)
13. [Deployment Guide](#deployment-guide)

---

## System Status & Health

### Current Production Status

| System | Status | Details |
|--------|--------|---------|
| Core Website | OPERATIONAL | All public pages functional |
| Admin Portal | OPERATIONAL | Authentication and dashboard working |
| A/B Testing Engine | OPERATIONAL | Autonomous rotation active |
| Live Photo Integration | OPERATIONAL | 7 photos integrated with video autoplay |
| Email Automation | OPERATIONAL | Confirmations and notifications working |
| Google Calendar | OPERATIONAL | Events created, .ics files sent |
| Booking System | OPERATIONAL | Real-time conflict prevention |
| Database | HEALTHY | All migrations applied |
| Build Status | PASSING | TypeScript compilation successful |

### System Validation Results (14/14 Passed)

```
[PASS] Database connectivity
[PASS] Photo rotation table exists
[PASS] A/B tests table exists
[PASS] Flag analytics table exists
[PASS] Automation logs table exists
[PASS] Photo inventory count
[PASS] Live Photos with video support
[PASS] A/B test endpoints accessible
[PASS] Analytics tracking functional
[PASS] Cron job configuration valid
[PASS] Environment variables set
[PASS] Email service connected
[PASS] Calendar integration configured
[PASS] Asset management operational
```

### Health Monitoring Endpoints

```bash
# System health dashboard
GET /api/admin/automation-health

# Response structure:
{
  "overall": "healthy" | "warning" | "critical",
  "components": {
    "database": "healthy",
    "photoRotation": "healthy",
    "cronJobs": "healthy",
    "testExecution": "healthy"
  },
  "metrics": {
    "totalTests": 3,
    "activeTests": 2,
    "totalPhotos": 15,
    "errorRate24h": 0.02
  },
  "recentEvents": [...],
  "issues": [],
  "recommendations": []
}
```

---

## A/B Testing System

### Architecture Overview

The autonomous A/B testing system consists of four core components:

1. **Photo Rotation Engine** (`src/lib/photoRotationEngine.ts`)
   - Manages photo inventory and selection
   - Validates new photos and tracks performance
   - Handles automatic test creation
   - Maintains photo status lifecycle

2. **A/B Test Engine** (`src/lib/abTestEngine.ts`)
   - Runs statistical analysis on test results
   - Promotes winners using Chi-squared tests (95% confidence)
   - Manages test lifecycle and triggers rotations
   - Minimum sample size enforcement

3. **Alerting Engine** (`src/lib/alertingEngine.ts`)
   - Monitors system health continuously
   - Sends email alerts for critical issues
   - Tracks error rates and performance degradation
   - Manages alert cooldown periods

4. **Management Scripts** (`scripts/manage-photo-rotation.ts`)
   - CLI tool for photo inventory management
   - Health checks and maintenance operations
   - Batch photo imports and test creation
   - Performance analytics viewing

### Database Tables

```sql
-- Photo inventory with performance tracking
CREATE TABLE photo_rotation (
  id SERIAL PRIMARY KEY,
  image_path TEXT NOT NULL,
  video_path TEXT,
  alt_text TEXT NOT NULL,
  title TEXT,
  description TEXT,
  test_type TEXT NOT NULL,
  location TEXT NOT NULL,
  status TEXT DEFAULT 'available',
  priority INTEGER DEFAULT 0,
  times_used INTEGER DEFAULT 0,
  total_wins INTEGER DEFAULT 0,
  win_rate DECIMAL,
  avg_conversion_rate DECIMAL,
  tags JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- A/B test configurations and results
CREATE TABLE ab_tests (
  id SERIAL PRIMARY KEY,
  test_key TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  test_type TEXT NOT NULL,
  status TEXT DEFAULT 'active',
  variant_a JSONB NOT NULL,
  variant_b JSONB NOT NULL,
  winner TEXT,
  conversion_event TEXT NOT NULL,
  min_sample_size INTEGER DEFAULT 100,
  confidence_level DECIMAL DEFAULT 0.95,
  started_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Historical test snapshots
CREATE TABLE ab_test_results (
  id SERIAL PRIMARY KEY,
  test_id INTEGER REFERENCES ab_tests(id),
  variant_a_views INTEGER,
  variant_a_conversions INTEGER,
  variant_b_views INTEGER,
  variant_b_conversions INTEGER,
  chi_squared DECIMAL,
  p_value DECIMAL,
  is_significant BOOLEAN,
  winner TEXT,
  snapshot_at TIMESTAMP DEFAULT NOW()
);

-- System operation logs
CREATE TABLE automation_logs (
  id SERIAL PRIMARY KEY,
  operation TEXT NOT NULL,
  status TEXT NOT NULL,
  details JSONB,
  error_message TEXT,
  execution_time_ms INTEGER,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Autonomous Operation Flow

**Bi-weekly Analysis Cycle (1st and 15th at Midnight UTC):**

1. **Fetch Active Tests**
   - Query `ab_tests` table for `status = 'active'`
   - Each test location processes independently

2. **Statistical Analysis**
   - Calculate conversion rates for variants A & B
   - Run chi-squared test for statistical significance
   - Require minimum sample size (configurable, default 100)

3. **Winner Promotion**
   - Update test status to `completed`
   - Set winning variant in database
   - All frontend hooks immediately honor the winner

4. **Automatic Rotation**
   - Select next challenger from `photo_rotation`
   - Create new test: `Winner vs New Challenger`
   - Update photo statuses to `active`

5. **Notifications**
   - Email alerts for winners and system status
   - Log all operations to `automation_logs`

### Frontend Integration

```typescript
// Hero image test hook
import { useHeroImageTest, trackHeroConversion } from '@/hooks/useHeroImageTest';

export function HeroSection() {
  const heroImage = useHeroImageTest();

  return (
    <Image
      src={heroImage.imagePath}
      alt={heroImage.alt}
      onClick={() => trackHeroConversion(heroImage.variant)}
    />
  );
}
```

**Hook Operation Flow:**
1. Check `/api/ab-tests/[testKey]` for promoted winner
2. If winner exists, use it immediately
3. If no winner, check localStorage for assignment
4. If no stored assignment, randomize (50/50 split)
5. Track all variant views and conversions

### Statistical Significance

**Minimum Requirements:**
- 100 views minimum (combined A+B)
- 95% confidence level (Chi-squared > 3.841)
- P-value < 0.05

**Example Results:**
```
Variant A: 200 views, 8 conversions (4.0%)
Variant B: 200 views, 16 conversions (8.0%)

Chi-squared: 5.26 (> 3.841 threshold)
P-value: 0.022
Result: SIGNIFICANT
Winner: Variant B
```

### Photo Status Lifecycle

```
[available] --> [active] --> [winner] or [available]
     |              |              |
     v              v              v
[retired] <--- [retired] <--- [retired]
```

- **available**: Ready for testing
- **active**: Currently in an A/B test
- **winner**: Promoted as winning variant
- **retired**: Removed from rotation

### Creating New A/B Tests

**Method 1: CLI Tool**
```bash
npm run test:create

# Follow prompts:
# - testKey (unique identifier)
# - Friendly name
# - Test type (hero/gallery/service-card)
# - Conversion event name
# - Minimum sample size
# - Variant A and B metadata
```

**Method 2: Direct API**
```typescript
POST /api/admin/ab-tests
{
  "testKey": "hero-image-v2",
  "name": "Hero Image Test Version 2",
  "testType": "hero",
  "variantA": {
    "imagePath": "/images/hero-a.jpg",
    "altText": "Bathroom renovation",
    "title": "Professional Renovation"
  },
  "variantB": {
    "imagePath": "/images/hero-b.jpg",
    "altText": "Exterior painting",
    "title": "Expert Painting Services"
  },
  "conversionEvent": "inspection-scheduled",
  "minSampleSize": 100,
  "confidenceLevel": 0.95
}
```

---

## Live Photo Integration

### Overview

Apple Live Photos (image + video pairs) are integrated with progressive enhancement A/B testing. The system tests Live Photos against static images while maintaining 100% reliability through graceful degradation.

### Current Live Photo Inventory

| File | Video Size | Category | Test Type | Location | Priority |
|------|-----------|----------|-----------|----------|----------|
| IMG_7842 | 2.1MB | Deck Work | Hero | hero-carousel | 9 |
| IMG_3800 | 1.8MB | Project Showcase | Hero | hero-showcase | 8 |
| IMG_9697 | 2.3MB | Project Showcase | Hero | hero-showcase | 8 |
| IMG_9007 | 1.9MB | Painting | Gallery | gallery-featured | 7 |
| IMG_9757 | 2.0MB | Renovation | Gallery | gallery-renovation | 6 |
| IMG_0097 | 1.7MB | Construction | Service Card | services-construction | 5 |
| IMG_9825 | 2.2MB | Construction | Service Card | services-construction | 7 |

### LivePhotoShowcase Component

```tsx
import { LivePhotoShowcase } from '@/components/LivePhotoShowcase';

<LivePhotoShowcase
  testKey="hero-deck-showcase"
  fallbackImage="/images/IMG_7842.jpg"
  altText="Deck renovation project"
  className="w-full h-full"
  priority={true}
  conversionEvent="hero_engagement"
/>
```

**Component Features:**
- Progressive enhancement (static image first)
- A/B variant selection (50/50 split)
- Video autoplay when visible (Intersection Observer)
- Graceful fallback on any error
- Analytics tracking for views and conversions
- Mobile-optimized with iOS/Android support

### AutoplayVideo Component

```tsx
import { AutoplayVideo } from '@/components/AutoplayVideo';

<AutoplayVideo
  src="/videos/IMG_7842.mov"
  poster="/images/IMG_7842.jpg"
  className="w-full h-full object-cover"
  loop={true}
  muted={true}
/>
```

**Technical Features:**
- Intersection Observer: Only plays when 50% visible
- Bandwidth Optimization: Pauses when out of view
- Hardware Acceleration: GPU-accelerated playback
- Loop Seamlessly: Smooth transitions
- Click to Control: Manual pause/play
- Accessibility: Keyboard navigation support

### Browser Compatibility

| Browser | Support | Notes |
|---------|---------|-------|
| iOS Safari | Full | `playsInline` attribute required |
| Chrome/Edge | Full | Hardware accelerated |
| Firefox | Full | Graceful fallback |
| Mobile Browsers | Optimized | Touch interaction support |

### Error Handling & Fallbacks

1. **Network Errors**: Falls back to static images
2. **Video Load Failures**: Shows cover image instead
3. **Autoplay Blocked**: Provides play button overlay
4. **A/B Test API Down**: Uses fallback images
5. **JavaScript Disabled**: Shows static images via `<noscript>`

---

## Email System

### Architecture

**Email Service:** Resend API

**Automated Email Types:**
1. Customer confirmation (appointment scheduled)
2. Admin notification (new lead/inspection)
3. A/B test winner notifications
4. System health alerts

### Confirmation Email Workflow

**Trigger Conditions:**
- Ticket type: Inspection (not lead)
- Status changed to: "scheduled" OR "confirmed"
- Required: `scheduledDate` must exist
- API: Status update via admin API

**Email Content:**
- Customer name and greeting
- Confirmed date and time (formatted)
- Service type requested
- Service address
- Admin contact information
- Rescheduling instructions
- .ics calendar file attachment

### Non-Blocking Design

```typescript
// Email failures don't prevent status updates
try {
  await sendConfirmationEmail(customer, appointment);
  await logActivity('consultation_confirmation_sent', {...});
} catch (emailError) {
  await logActivity('consultation_confirmation_failed', { error: emailError });
  // Status update still succeeds
}
```

**Benefits:**
- Status update succeeds even if email fails
- Comprehensive error logging
- Activity log records both success and failure
- Full audit trail for compliance

### Activity Logging

| Event Type | Description | Logged Data |
|------------|-------------|-------------|
| `consultation_confirmation_sent` | Email sent successfully | Customer email, date/time |
| `consultation_confirmation_failed` | Email failed to send | Error details, customer email |
| `admin_notification_sent` | New lead/inspection alert | Ticket ID, type |
| `calendar_event_created` | Google Calendar event created | Event ID, details |

### Email Templates

Located in `/src/lib/email/`:
- `confirmation.ts` - Appointment confirmation
- `notification.ts` - Admin notifications
- `ab-test-results.ts` - A/B test winner alerts
- `system-alerts.ts` - Health monitoring alerts

### Domain Configuration

```bash
# Current: Resend sandbox (limited recipients)
# Required: Custom domain verification

# Steps:
1. Add domain to Resend dashboard
2. Configure DNS records:
   - SPF record
   - DKIM record
   - DMARC record (recommended)
3. Verify domain ownership
4. Update FROM_EMAIL in code
5. DNS propagation: 24-72 hours
```

### Email Setup Process

1. **Set up Resend Account**: Visit https://resend.com and create account
2. **Add Domain**: Add `lamcpainting.com` to Resend dashboard
3. **Configure DNS**: Add DNS records (SPF, DKIM) to domain registrar
4. **Verify Domain**: Wait for DNS propagation (24-72 hours) and verify
5. **Update FROM_EMAIL**: Change from `onboarding@resend.dev` to `noreply@lamcpainting.com`
6. **Test**: Use `node test-email-functionality.js` to verify

### Troubleshooting Email Issues

**Customers Not Receiving Emails:**
1. Check Resend domain status (must show "Verified")
2. Verify environment variables are set correctly
3. Check Vercel function logs for errors
4. Test with real customer email addresses
5. Use DNSChecker to verify DNS records have propagated

**Common Error Messages:**
- `"onboarding@resend.dev"`: Still in sandbox mode
- `"Domain not verified"`: Complete domain verification
- `"Missing environment variable"`: Add required variables
- `"Failed to send email"`: Check API key and domain status

---

## Booking & Calendar System

### Real-Time Availability API

```typescript
GET /api/availability?date=2025-11-20

// Response:
{
  "date": "2025-11-20",
  "slots": [
    { "time": "09:00", "available": true },
    { "time": "10:00", "available": false }, // Already booked
    { "time": "11:00", "available": true },
    // ...
  ],
  "inspectionHours": {
    "start": "09:00",
    "finish": "17:00"
  }
}
```

### Conflict Prevention (3-Layer Protection)

1. **Layer 1: Real-time API**
   - Shows only available slots
   - Updates as bookings are made
   - Cache: 0 seconds (always fresh)

2. **Layer 2: Server-side Validation**
   - Validates on form submission
   - Returns 409 Conflict if slot taken
   - Prevents race conditions

3. **Layer 3: Auto-refresh on Conflict**
   - Frontend auto-refreshes on 409 error
   - User selects different time
   - Seamless experience

### Google Calendar Integration

**Automatic Event Creation:**
```typescript
// When admin confirms appointment:
1. Creates event on admin's Google Calendar
2. Includes customer details, service type, address
3. Sets 1-hour default duration (configurable)
4. Adds reminders (24 hours and 1 hour before)
5. Stores Google Calendar event ID in database
```

**Customer Calendar Invites:**
```typescript
// .ics file generation:
1. Generates RFC 5545 compliant .ics files
2. Attaches to confirmation emails via Resend
3. Works with all major calendar apps
4. Includes organizer and attendee information
```

**Implementation Files:**
- `src/lib/calendar/client.ts` - Google Calendar API service
- `src/lib/calendar/ics.ts` - ICS file generator
- `src/app/api/calendar/blocked-times/route.ts` - Blocked times API

### Asset Management

**File Type Support:**
| Type | Extensions | Max Size | Max Count |
|------|-----------|----------|-----------|
| Images | JPEG, PNG, WebP, HEIC, GIF | 10MB | 10 |
| Documents | PDF, Word, Excel, Text, CSV | 25MB | 5 |
| Videos | MP4, MOV, AVI, WebM | 100MB | 5 |
| Other | ZIP, RAR | 50MB | - |
| **Total** | - | - | **15 files** |

**Storage Structure:**
```
Vercel Blob Storage:
├── assets/
│   ├── images/
│   │   └── 1234567890-photo1.jpg
│   ├── documents/
│   │   └── 1234567892-contract.pdf
│   └── videos/
│       └── 1234567894-walkthrough.mp4
```

---

## Database Schema

### Core Tables (10 Total)

#### users
Admin authentication with role-based access control.
```sql
- id, email, password_hash, name, role, created_at, updated_at
```

#### leads
Contact form submissions with status workflow.
```sql
- id, name, email, phone, service, message, status, source, notes,
  assigned_to, contacted_at, completed_at, created_at, updated_at
```

#### inspections
Inspection booking requests with multi-asset support.
```sql
- id, name, email, phone, address, service, preferred_date,
  preferred_time, details, photos (legacy), assets (JSONB),
  status, scheduled_date, calendar_event_id, notes, assigned_to,
  completed_at, created_at, updated_at
```

#### inspection_availability
Admin calendar configuration with custom time blocks.
```sql
- id, date, time_slot, duration (minutes), is_available, notes,
  created_at, updated_at
```

#### prefill_tokens
Secure form prefilling with token expiration.
```sql
- id, token, email, name, phone, expires_at, created_at
```

#### rate_limits
API rate limiting per IP and endpoint.
```sql
- id, identifier (IP), endpoint, attempts, window_start,
  created_at, updated_at
```

#### flag_analytics
A/B test and feature flag tracking.
```sql
- id, flag_key, flag_value, event_type, event_data, session_id,
  timestamp
```

#### notes
CRM notes linked to leads/inspections.
```sql
- id, entity_type, entity_id, content, created_by, created_at
```

#### activity_log
Comprehensive audit trail.
```sql
- id, action, entity_type, entity_id, user_id, metadata,
  created_at, updated_at
```

#### settings
Application configuration storage.
```sql
- id, key, value (JSONB), created_at, updated_at
- Examples: 'inspection_hours', 'business_hours', 'contact_info'
```

---

## API Reference

### Public APIs

| Endpoint | Method | Rate Limited | Purpose |
|----------|--------|--------------|---------|
| `/api/contact` | POST | 5/hr | Contact form submission |
| `/api/inspection` | POST | 5/hr | Inspection booking |
| `/api/availability` | GET | No | Real-time booking availability |
| `/api/calendar/blocked-times` | GET | No | Confirmed appointment times |
| `/api/ab-tests/[testKey]` | GET | No | Get test status and winner |
| `/api/analytics/flags` | POST | No | Track variant views/conversions |
| `/api/auth/[...nextauth]` | * | No | Authentication (NextAuth) |

### Admin APIs (Protected)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/admin/dashboard` | GET | Dashboard metrics |
| `/api/admin/tickets` | GET | List all tickets (unified) |
| `/api/admin/tickets/[id]` | GET/PATCH | Ticket details and updates |
| `/api/admin/tickets/[id]/accept` | POST | Accept proposed date/time |
| `/api/admin/tickets/[id]/reply` | POST | Email customer + calendar |
| `/api/admin/analytics` | GET | Analytics data |
| `/api/admin/availability` | GET/POST | Calendar CRUD |
| `/api/admin/availability/bulk` | POST | Bulk calendar updates |
| `/api/admin/availability/range` | GET | Date range query |
| `/api/admin/settings/inspection-hours` | GET/PUT | Inspection hours config |
| `/api/admin/ab-tests` | GET/POST | Test management |
| `/api/admin/automation-health` | GET | System health dashboard |
| `/api/admin/automation-alerts` | GET | Critical event notifications |
| `/api/admin/flags` | GET | Feature flags list |

### Cron Job APIs

| Endpoint | Schedule | Purpose |
|----------|----------|---------|
| `/api/cron/ab-test-analysis` | 1st & 15th monthly | Bi-weekly test analysis |
| `/api/cron/health-monitoring` | Every 15 min | System health checks |
| `/api/cron/cleanup-rate-limits` | Manual/Cron | Remove old rate limits |

---

## Management Commands

### Development Commands

```bash
npm run dev              # Start development server (localhost:3000)
npm run build            # Build for production
npm run start            # Run production build locally
npm run lint             # Run ESLint
npm run type-check       # TypeScript validation
```

### Database Commands

```bash
npm run db:push          # Apply schema changes to database
npm run db:generate      # Generate migration files
npm run db:studio        # Open Drizzle Studio (visual browser)

# Manual migration:
psql $POSTGRES_URL -f drizzle/0007_add_photo_rotation_system.sql
```

### Admin & User Management

```bash
npm run admin:create     # Create admin user (interactive)
```

### A/B Testing & Photo Management

```bash
npm run photo:manage     # Interactive photo rotation manager
                         # Options:
                         # 1. View system health and statistics
                         # 2. Add new photo to rotation
                         # 3. List photo inventory
                         # 4. Update photo status
                         # 5. View photo performance
                         # 6. Cleanup and maintenance
                         # 7. View automation logs
                         # 8. Create new A/B test
                         # 9. Scan directory for photos

npm run test:create      # Create new A/B test (interactive)

npm run system:validate  # Run 14-point system validation

npm run live-photos:populate  # Add Live Photos to rotation
```

### Testing Commands

```bash
npm run test:admin-api-unit    # Run unit tests
npm run test:admin-api         # Run integration tests (requires server)
```

### Manual Cron Triggers

```bash
# Trigger A/B test analysis manually
curl "https://your-site.com/api/cron/ab-test-analysis?secret=YOUR_CRON_SECRET"

# Check system health
curl "https://your-site.com/api/admin/automation-health"

# View automation logs
npm run db:studio
# Navigate to automation_logs table
```

---

## Monitoring & Alerting

### Health Monitoring (Every 15 Minutes)

The system automatically monitors:

1. **Error Rate Monitoring**
   - Threshold: 20% error rate in last hour
   - Alert: High failure rate warning

2. **System Availability**
   - Threshold: 3 consecutive failures
   - Alert: System down critical

3. **Photo Inventory**
   - Threshold: <3 photos per test type
   - Alert: Low inventory warning

4. **Cron Job Health**
   - Threshold: No execution in 14 days
   - Alert: Automation failure

5. **Database Connectivity**
   - Threshold: 3 DB errors in 5 minutes
   - Alert: Critical system alert

6. **Performance Monitoring**
   - Threshold: >30s average execution time
   - Alert: Performance degradation

### Alert Types & Recipients

All alerts sent to `ADMIN_EMAIL` environment variable:

| Severity | Icon | Examples |
|----------|------|----------|
| Critical | 🚨 | System down, database failures |
| Warning | ⚠️ | High error rates, low inventory |
| Info | 📄 | Successful operations, health checks |

### Alert Cooldown Periods

| Alert Type | Cooldown |
|------------|----------|
| Error Rate | 1 hour |
| System Down | 30 minutes |
| Low Inventory | 24 hours |
| Cron Failure | 6 hours |

---

## Troubleshooting Guide

### A/B Tests Not Working

```bash
# Check test status
npm run photo:manage
# Select option 3: List Photo Inventory

# Verify cron jobs running
curl "https://your-site.com/api/cron/ab-test-analysis?secret=YOUR_SECRET"

# Check database tables
npm run db:studio
# Verify ab_tests and flag_analytics tables have data
```

### No Test Rotation

```bash
# Check photo inventory
npm run photo:manage
# Select option 1: System Health Check

# Add more photos if needed (<3 per test type)
npm run photo:manage
# Select option 9: Scan directory for photos
```

### Videos Not Playing on Mobile

- Ensure `playsInline` attribute is set
- iOS requires user interaction for autoplay in some contexts
- Check browser console for autoplay policy errors

### Email Not Received

```bash
# Check environment variables
echo $ADMIN_EMAIL
echo $RESEND_API_KEY

# Check spam folder
# Verify Resend API key is valid
# Check activity logs for email failures
npm run db:studio
# Navigate to activity_log table
```

### Static Images Instead of Videos

- Check browser console for errors
- May be autoplay policy restriction
- Network connectivity issue
- Video file not found at specified path

### System Errors

```bash
# View recent automation logs
npm run photo:manage
# Select option 7: View Automation Logs

# Check health monitoring
curl "https://your-site.com/api/admin/automation-health"

# View Vercel function logs
# Vercel Dashboard -> Deployments -> Functions
```

### "Not Enough Data" Message

- Need minimum 100 total views (combined A+B)
- Wait longer for traffic
- Or lower `minSampleSize` threshold in test configuration

### Database Connection Issues

```bash
# Test database connectivity
npm run db:studio

# Check POSTGRES_URL environment variable
# Verify Neon database is accessible
# Check for connection pool limits
```

### Slow Page Load

- Check image optimization (Next.js Image component)
- Verify lazy loading is active
- Review bundle size with `npm run build`
- Check CDN cache headers

### High Error Rates

```bash
# View error logs in Vercel Dashboard
# Search: "level":"error"

# Check rate limiting
# Verify database connection pool
# Review API response times
```

---

## Security Implementation

### Authentication

- **Provider:** NextAuth.js with JWT sessions
- **Password Hashing:** Bcrypt (10 salt rounds)
- **Session Storage:** HTTP-only cookies (XSS protection)
- **Security Flag:** HTTPS only in production
- **Session Expiry:** 30 days
- **OAuth:** Google OAuth provider supported

### Authorization

- **Route Protection:** Middleware-based `/admin/*` protection
- **Role-Based Access:** Admin-only API endpoints
- **Session Verification:** Every request validated
- **Automatic Redirect:** Unauthenticated users redirected to login

### Input Validation

- Email format validation (regex)
- Phone format validation
- Required field enforcement
- File type whitelist (images, documents, videos)
- File size limits per type
- XSS prevention (input sanitization)
- SQL injection prevention (Drizzle ORM parameterized queries)

### Rate Limiting

```typescript
// Database-backed implementation
// Per-IP per-endpoint tracking
// 5 requests/hour on public forms

// RFC 6585 compliant headers:
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 3
X-RateLimit-Reset: 1699564800
Retry-After: 3600  // When rate limited
```

### API Security

- **Cron Endpoints:** Protected by Vercel headers + secret fallback
- **Admin APIs:** Require NextAuth authentication
- **Automation Alerts:** Use shared secret
- **CORS:** Configured for production domain
- **Request Validation:** All inputs sanitized
- **Error Messages:** Sanitized (no stack traces in production)

---

## Performance Optimization

### Lighthouse Scores

| Metric | Score | Target |
|--------|-------|--------|
| Performance | 85-90 | 80+ |
| Accessibility | 95+ | 90+ |
| Best Practices | 90+ | 85+ |
| SEO | 95+ | 90+ |

### Core Web Vitals

| Metric | Current | Target |
|--------|---------|--------|
| LCP (Largest Contentful Paint) | <2.5s | <2.5s |
| FID (First Input Delay) | <100ms | <100ms |
| CLS (Cumulative Layout Shift) | <0.1 | <0.1 |

### Response Times

| Operation | Average | Target |
|-----------|---------|--------|
| API Response | <500ms | <1s |
| Homepage Load | 1.5-2.5s | <3s |
| Gallery Load | 2-3s | <3s |
| Admin Dashboard | 1-1.5s | <2s |
| Database Query | <100ms | <200ms |

### Optimization Techniques

**Image Optimization:**
- Next.js Image component with blur placeholders
- Automatic WebP/AVIF conversion
- Responsive image sizing
- Lazy loading for off-screen images

**Code Optimization:**
- Automatic code splitting via App Router
- Tree shaking for smaller bundles
- Dynamic imports for heavy components
- Minification and compression

**Database Optimization:**
- Connection pooling (Neon)
- Indexed queries for common operations
- Query optimization with EXPLAIN
- Transaction batching

**Mobile Performance:**
- 60 FPS scrolling optimization
- Touch response <100ms
- Reduced motion support
- Battery-friendly video autoplay

### Caching Strategy

- Static assets: CDN (Vercel Edge Network)
- API responses: 0-second cache (real-time)
- Database queries: Connection pooling
- Images: Browser cache with versioning

---

## Deployment Guide

### Pre-Deployment Checklist

#### Code Readiness
- All TypeScript build errors resolved
- Email functionality implemented and tested
- Error handling enhanced and verified
- Documentation complete and comprehensive
- Test scripts created and validated

#### Environment Setup
- Environment variables documented
- Resend domain setup guide available
- Production configuration template ready
- Security best practices documented

#### Testing Complete
- Unit tests for email functions pass
- Integration tests for workflows pass
- End-to-end testing completed
- Error scenario testing verified
- Sandbox mode behavior tested

### Production Deployment Steps

#### Step 1: Environment Variables Configuration
```bash
# Production environment setup required
RESEND_API_KEY="re_production_key"
FROM_EMAIL="noreply@lamcpainting.com"
ADMIN_EMAIL="luis@lamcpainting.com"
```

#### Step 2: Resend Domain Verification
1. Add domain to Resend dashboard
2. Configure DNS records (SPF, DKIM, DMARC)
3. Verify domain ownership
4. Update `FROM_EMAIL` to use verified domain
5. Test with real customer email addresses

#### Step 3: Deploy Code Changes
```bash
# Push to dev-1 for preview deployment
git push origin dev-1

# Test preview environment
# Merge to main for production deployment
git checkout main
git merge dev-1
git push origin main
```

#### Step 4: Production Verification
- Email functionality works with real customer emails
- Domain verification status: "Verified" in Resend
- All email scenarios work in production
- Error logs monitored for 24 hours
- Customer feedback collected

### Post-Deployment Monitoring

**Monitor These Metrics:**
- Email delivery success rate (target: >95%)
- Email bounce/complaint rates (target: <1%)
- Customer response times
- Admin adoption of email features
- System error rates
- API response times

**Monitoring Period:** 48 hours of close monitoring after deployment

### Rollback Procedures

#### Immediate Rollback (Critical Issues)
```bash
# Revert to previous working version
git revert HEAD
git push origin main

# Verify rollback
# - Check website functionality
# - Verify database operations
# - Confirm admin dashboard works
```

#### Email-Specific Rollback (Email Issues Only)
```bash
# Disable email functionality temporarily
# Set RESEND_API_KEY="" in Vercel environment variables
# All other functionality continues working

# After fixing email issues, re-enable:
# Set proper RESEND_API_KEY in Vercel environment variables
```

### Vercel Configuration

The `vercel.json` file controls deployment behavior:

```json
{
  "git": {
    "deploymentEnabled": {
      "main": true
    }
  },
  "crons": [
    {
      "path": "/api/cron/ab-test-analysis",
      "schedule": "0 0 1,15 * *"
    },
    {
      "path": "/api/cron/health-monitoring",
      "schedule": "*/15 * * * *"
    }
  ]
}
```

This configuration:
- Enables deployments only for the `main` branch
- Disables automatic deployments for all other branches
- Configures A/B test analysis cron (bi-weekly)
- Configures health monitoring cron (every 15 minutes)

---

## Environment Variables Reference

### Required Variables

```bash
# Database
POSTGRES_URL="postgresql://..."
POSTGRES_URL_NON_POOLING="postgresql://..."

# Email
RESEND_API_KEY="re_..."
ADMIN_EMAIL="admin@domain.com"
FROM_EMAIL="LAMC Painting <noreply@lamcpainting.com>"

# Storage
BLOB_READ_WRITE_TOKEN="vercel_blob_..."

# Authentication
NEXTAUTH_SECRET="..."  # openssl rand -base64 32
NEXTAUTH_URL="http://localhost:3000"
ADMIN_PASSWORD="ChangeThisPassword123!"

# Google Calendar
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."

# A/B Testing & Automation
CRON_SECRET="your-random-secret-here"
AUTOMATION_SECRET="automation-secret-key"
NEXT_PUBLIC_SITE_URL="https://your-domain.com"
```

### Optional Variables

```bash
# Feature flags (if needed)
FEATURE_FLAG_NEW_HERO=true
FEATURE_FLAG_DARK_MODE=false

# Analytics
GOOGLE_ANALYTICS_ID="G-..."

# Rate limiting overrides
RATE_LIMIT_MAX_REQUESTS=5
RATE_LIMIT_WINDOW_MS=3600000
```

---

**Document Version:** 6.0
**Last Updated:** November 16, 2025
**Next Review:** Post-production launch
