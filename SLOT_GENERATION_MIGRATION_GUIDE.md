# Slot Generation Pipeline Migration Guide

## Overview

The slot generation system has been completely rebuilt with production-ready features including comprehensive error handling, validation, monitoring, rate limiting, and scalability improvements.

## What Changed

### Old System
```typescript
// Simple implementation with basic error handling
export async function autoGenerateAvailabilitySlots(
  inspectionHours: InspectionHours,
  daysAhead: number = 30
): Promise<number> {
  try {
    // Fetch existing slots
    // Generate new slots
    // Insert in bulk (single operation)
    // Return count
  } catch (error) {
    console.error('Failed:', error);
    throw error;
  }
}
```

**Issues:**
- ❌ No retry logic for database failures
- ❌ All-or-nothing inserts (one failure = complete failure)
- ❌ No validation of inputs
- ❌ No rate limiting
- ❌ Limited error information
- ❌ No monitoring or metrics
- ❌ No handling of edge cases
- ❌ No timeout protection
- ❌ No memory management for large datasets

### New System
```typescript
// Production-ready with comprehensive features
export async function generateAvailabilitySlots(
  inspectionHours: InspectionHours,
  options: GenerateAvailabilitySlotsOptions
): Promise<SlotGenerationResult> {
  // ✅ Database connection validation with retry
  // ✅ Comprehensive input validation
  // ✅ Rate limiting
  // ✅ Batch processing with retry logic
  // ✅ Partial success handling
  // ✅ Detailed metrics and monitoring
  // ✅ Edge case handling (DST, leap years, holidays)
  // ✅ Timeout protection
  // ✅ Memory monitoring
}
```

## Migration Steps

### Step 1: No Changes Required (Backward Compatible)

The existing API continues to work without any code changes:

```typescript
// This still works exactly as before
import { autoGenerateAvailabilitySlots } from '@/lib/availability';

const slotsCreated = await autoGenerateAvailabilitySlots(inspectionHours, 30);
```

**What happens internally:**
- Uses new robust service automatically
- Provides same return value (number of slots)
- Throws errors for complete failures
- Allows partial success

### Step 2: Optional - Use New API for Enhanced Features

```typescript
import { generateAvailabilitySlots } from '@/lib/slot-generation';

const result = await generateAvailabilitySlots(inspectionHours, {
  daysAhead: 30,
  skipExistingDates: true,
  enableTransactions: false,
});

// Access detailed results
console.log('Success:', result.success);
console.log('Slots created:', result.slotsCreated);
console.log('Warnings:', result.warnings);
console.log('Metrics:', result.metrics);
```

### Step 3: Update API Consumers

The inspection hours API now returns more detailed information:

**Old Response:**
```json
{
  "success": true,
  "hours": { ... }
}
```

**New Response:**
```json
{
  "success": true,
  "hours": { ... },
  "slotGeneration": {
    "success": true,
    "slotsCreated": 450,
    "slotsSkipped": 50,
    "warnings": [],
    "metrics": {
      "operationId": "slot-gen-1234567890-abc123",
      "durationMs": 2500,
      "status": "completed"
    }
  }
}
```

**Frontend Update (Optional):**
```typescript
// Before
const response = await fetch('/api/admin/settings/inspection-hours', {
  method: 'PUT',
  body: JSON.stringify({ hours }),
});
const { success, hours } = await response.json();

// After (with slot generation info)
const response = await fetch('/api/admin/settings/inspection-hours', {
  method: 'PUT',
  body: JSON.stringify({ hours }),
});
const { success, hours, slotGeneration } = await response.json();

if (slotGeneration.success) {
  console.log(`Generated ${slotGeneration.slotsCreated} slots`);
} else {
  console.warn('Slot generation had issues:', slotGeneration.error);
}
```

## New Features Available

### 1. Rate Limiting

Prevents abuse and ensures system stability:

```typescript
// Automatically enforced in API routes
// Max 10 requests/minute
// Max 100 requests/hour
// Max 3 concurrent operations
```

**Error Response:**
```json
{
  "success": true,
  "hours": { ... },
  "slotGeneration": {
    "success": false,
    "error": "Rate limit exceeded. Slots will be generated later."
  }
}
```
**HTTP Status:** 429 (Too Many Requests)

### 2. Validation

Comprehensive validation before processing:

```typescript
import { validateInspectionHours } from '@/lib/slot-generation';

const validation = validateInspectionHours(hours);

if (!validation.valid) {
  console.error('Errors:', validation.errors);
  // ["monday: Start time 25:00 must be before finish time 20:00"]
}

if (validation.warnings.length > 0) {
  console.warn('Warnings:', validation.warnings);
  // ["Only 2 day(s) open for inspections. Consider opening more days"]
}
```

### 3. Health Monitoring

New health check endpoint:

```http
GET /api/admin/slot-generation/health
```

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-01-20T12:00:00Z",
  "checks": {
    "slotGeneration": {
      "status": "healthy",
      "lastOperationId": "slot-gen-1234567890-abc123",
      "lastOperationStatus": "completed",
      "errorRate": 0
    },
    "database": {
      "status": "healthy",
      "connected": true
    }
  },
  "admin": {
    "databaseStats": {
      "totalSlots": 5000,
      "availableSlots": 4500,
      "oldestSlotDate": "2025-01-20",
      "newestSlotDate": "2025-04-20"
    }
  }
}
```

### 4. Advanced Configuration

```typescript
import { generateAvailabilitySlots } from '@/lib/slot-generation';

const result = await generateAvailabilitySlots(inspectionHours, {
  daysAhead: 60,
  slotDuration: 60,
  skipExistingDates: true,
  regenerate: false, // Set true to delete and regenerate all slots
  enableTransactions: false, // Set true for all-or-nothing
  batchSize: 500,
  config: {
    maxRetries: 5,
    operationTimeoutMs: 60000,
    skipHolidays: true,
    holidayDates: ['2025-12-25', '2025-01-01', '2025-07-04'],
  },
});
```

### 5. Metrics and Monitoring

Every operation provides detailed metrics:

```typescript
const result = await generateAvailabilitySlots(inspectionHours, { ... });

console.log('Metrics:', {
  operationId: result.metrics.operationId,
  duration: result.metrics.durationMs,
  slotsGenerated: result.metrics.slotsGenerated,
  slotsInserted: result.metrics.slotsInserted,
  slotsFailed: result.metrics.slotsFailed,
  batchesProcessed: result.metrics.batchesProcessed,
  memoryUsedMB: result.metrics.memoryUsedMB,
  status: result.metrics.status,
});
```

### 6. Error Classification

Detailed error types for better debugging:

```typescript
import { SlotGenerationErrorCode } from '@/lib/slot-generation';

if (!result.success) {
  switch (result.error?.code) {
    case SlotGenerationErrorCode.DB_CONNECTION_FAILED:
      // Database connection issue
      break;
    case SlotGenerationErrorCode.RATE_LIMIT_EXCEEDED:
      // Too many requests
      break;
    case SlotGenerationErrorCode.INVALID_DATE_RANGE:
      // Invalid configuration
      break;
    // ... more error types
  }
}
```

## Breaking Changes

### None for Existing Code

The migration is **100% backward compatible**. All existing code continues to work.

### Optional Breaking Changes (If Upgrading to New API)

If you choose to use the new `generateAvailabilitySlots` API:

1. **Return Type Changed**
   ```typescript
   // Old
   const count: number = await autoGenerateAvailabilitySlots(hours, 30);

   // New
   const result: SlotGenerationResult = await generateAvailabilitySlots(hours, { ... });
   const count = result.slotsCreated;
   ```

2. **Error Handling Changed**
   ```typescript
   // Old - throws on any error
   try {
     const count = await autoGenerateAvailabilitySlots(hours, 30);
   } catch (error) {
     // Handle error
   }

   // New - returns error info, may have partial success
   const result = await generateAvailabilitySlots(hours, { ... });
   if (!result.success) {
     if (result.slotsCreated > 0) {
       // Partial success
     } else {
       // Complete failure
     }
   }
   ```

## Configuration Updates

### Environment Variables

No new environment variables required. Uses existing:
- `POSTGRES_URL` - Database connection string
- `NODE_ENV` - Environment (development/production)

### Rate Limiting Configuration

Default limits (can be customized in code):

```typescript
// src/lib/slot-generation/config.ts
export const RATE_LIMIT_CONFIG: RateLimitConfig = {
  maxRequestsPerMinute: 10,
  maxRequestsPerHour: 100,
  maxConcurrentGenerations: 3,
};
```

### Slot Generation Configuration

Default configuration (can be overridden per request):

```typescript
// src/lib/slot-generation/config.ts
export const DEFAULT_CONFIG: SlotGenerationConfig = {
  maxDaysAhead: 90,
  defaultDaysAhead: 30,
  batchSize: 500,
  maxSlotsPerGeneration: 10000,
  maxRetries: 3,
  operationTimeoutMs: 30000,
  // ... more options
};
```

## Testing Checklist

- [ ] Test existing slot generation still works
- [ ] Test inspection hours update in admin panel
- [ ] Test rate limiting (rapid updates)
- [ ] Test large date ranges (60+ days)
- [ ] Test validation errors (invalid hours)
- [ ] Test database connection failures (temporarily break connection)
- [ ] Test health check endpoint
- [ ] Monitor logs for structured JSON output
- [ ] Verify no memory issues with large generations
- [ ] Test concurrent updates from multiple admins

## Monitoring and Alerts

### Recommended Monitoring

1. **Health Check**
   - Monitor `/api/admin/slot-generation/health` every 5 minutes
   - Alert if status is "unhealthy" for > 2 checks

2. **Error Rate**
   - Monitor error rate in health endpoint
   - Alert if error rate > 25%

3. **Performance**
   - Monitor average operation duration
   - Alert if duration > 10 seconds

4. **Rate Limits**
   - Monitor rate limit hits in logs
   - Review if frequent rate limiting occurs

### Log Examples

**Success:**
```json
{
  "level": "info",
  "message": "Slot generation completed successfully",
  "timestamp": "2025-01-20T12:00:00Z",
  "operationId": "slot-gen-1234567890-abc123",
  "durationMs": 2500,
  "slotsCreated": 450
}
```

**Failure:**
```json
{
  "level": "error",
  "message": "Slot generation failed",
  "timestamp": "2025-01-20T12:00:00Z",
  "operationId": "slot-gen-1234567890-abc123",
  "errorCode": "DB_TIMEOUT",
  "errorMessage": "Database operation timed out"
}
```

## Rollback Plan

If issues occur, rollback is simple:

1. **Revert Code Changes**
   ```bash
   git revert <commit-hash>
   ```

2. **System Continues Working**
   - Database schema unchanged
   - API endpoints unchanged (just enhanced)
   - Old implementation still available

3. **No Data Migration Needed**
   - All changes are in application layer
   - No database migrations required

## Support and Troubleshooting

### Common Issues

1. **"Rate limit exceeded" errors**
   - **Cause:** Too many rapid updates
   - **Solution:** Wait 60 seconds or adjust rate limits

2. **Slow performance**
   - **Cause:** Large date range or database latency
   - **Solution:** Reduce `daysAhead` or increase `batchSize`

3. **Partial failures**
   - **Cause:** Database intermittent issues
   - **Solution:** System handles gracefully, review logs for details

4. **Validation errors**
   - **Cause:** Invalid inspection hours configuration
   - **Solution:** Check error details, fix configuration

### Getting Help

1. Check health endpoint: `/api/admin/slot-generation/health`
2. Review application logs for operation IDs
3. Check database connectivity
4. Review error codes and messages
5. Consult README: `/src/lib/slot-generation/README.md`

## Summary

✅ **Backward Compatible** - No code changes required
✅ **Enhanced Reliability** - Better error handling and retry logic
✅ **Better Monitoring** - Detailed metrics and health checks
✅ **Improved Performance** - Batch processing and optimization
✅ **Production Ready** - Rate limiting, validation, edge case handling
✅ **Well Documented** - Comprehensive documentation and examples

The system is ready for production use with no breaking changes to existing functionality.
