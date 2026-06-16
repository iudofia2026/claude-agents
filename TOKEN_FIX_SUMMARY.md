# GitHub Issue 137 Fix: Invalid Acceptance Token Error

## Problem Summary
Customers get "Invalid acceptance token" error when clicking the Accept Proposed Time button, even though the token appears in the URL.

## Root Causes Identified

1. **Insufficient Logging**: No visibility into token generation, storage, and validation process
2. **Potential URL Encoding Issues**: Tokens could be corrupted during URL transmission
3. **Database Transaction Issues**: Tokens might not be saved correctly due to transaction rollbacks
4. **Missing Verification**: No confirmation that tokens were actually saved to database
5. **Scheduled Date Calculation Bug**: Using wrong date field for final schedule calculation

## Fixes Implemented

### 1. Enhanced Reply Endpoint (`src/app/api/admin/tickets/[id]/reply/route.ts`)

**Added Comprehensive Logging:**
- Token generation with full token logged for debugging
- Alternate time logic comparison details
- Database preparation and execution logging
- Token verification after database save
- Error handling with detailed error codes

**Key Improvements:**
```typescript
// Token generation logging
console.log('🔄 [ALTERNATE TIME] Generated acceptance token for alternate time proposal:', {
  acceptanceTokenFull: acceptanceToken, // Full token for debugging
  acceptanceUrl: `${baseUrl}/api/accept-alternate-time?token=${acceptanceToken}`,
});

// Database verification
const verificationResult = await db.select(...).where(eq(inspections.id, ticketId));
const tokenMatches = verified.token === acceptanceToken;
console.log('🔍 [DATABASE VERIFICATION] Token verification results:', {
  tokenMatches,
  verificationPassed: tokenMatches && tokenNotExpired,
});
```

### 2. Enhanced Acceptance Endpoint (`src/app/api/accept-alternate-time/route.ts`)

**Added URL Encoding/Decoding Support:**
- Automatic detection and handling of URL-encoded tokens
- Fallback to original token if decoding fails
- Try both original and decoded tokens in database query

**Enhanced Database Query:**
```typescript
// Try both decoded and original tokens
let result = await db.select(...).where(eq(inspections.alternateTimeAcceptanceToken, decodedToken));
if (result.length === 0 && token !== decodedToken) {
  result = await db.select(...).where(eq(inspections.alternateTimeAcceptanceToken, token));
}
```

**Comprehensive Token Validation:**
- Character-by-character comparison for mismatch debugging
- Detailed logging of all validation steps
- Fixed scheduled date calculation to use alternateTimeProposedDate

**Better Error Handling:**
- Graceful error handling with detailed logging
- Error redirection with metadata
- Activity logging for both success and failure cases

### 3. Fixed Scheduled Date Calculation

**Before:**
```typescript
const scheduledDate = new Date(`${inspection.preferredDate} ${inspection.alternateTimeProposedTime}`);
```

**After:**
```typescript
const dateToUse = inspection.alternateTimeProposedDate || inspection.preferredDate;
const scheduledDate = new Date(`${dateToUse} ${inspection.alternateTimeProposedTime}`);
```

### 4. Added Debug Endpoint

Created `/api/debug/alternate-time-tokens` (admin-only) for:
- Database connection testing
- Schema verification
- Token inspection
- Performance testing
- URL encoding scenario testing

## Complete Flow Now Includes:

1. **Token Generation** (Reply Endpoint)
   - ✅ Generate secure random token
   - ✅ Log full token for debugging
   - ✅ Set 7-day expiry
   - ✅ Prepare alternate time data

2. **Token Storage** (Reply Endpoint)
   - ✅ Update database with token and metadata
   - ✅ Verify token was saved correctly
   - ✅ Log detailed verification results
   - ✅ Handle database errors gracefully

3. **Email Generation** (Templates)
   - ✅ Include acceptance URL with token
   - ✅ Token properly encoded for URL

4. **Token Validation** (Acceptance Endpoint)
   - ✅ Handle URL encoding/decoding
   - ✅ Try both original and decoded tokens
   - ✅ Comprehensive token matching
   - ✅ Expiration checking
   - ✅ Duplicate acceptance prevention

5. **Schedule Confirmation** (Acceptance Endpoint)
   - ✅ Use correct date for scheduledDate calculation
   - ✅ Update inspection status
   - ✅ Clear token after use
   - ✅ Log acceptance activity
   - ✅ Redirect to success page

## Testing Steps

1. **Test Debug Endpoint:**
   ```bash
   GET /api/debug/alternate-time-tokens
   # (Requires admin authentication)
   ```

2. **Test Complete Flow:**
   - Create inspection with preferred time
   - Admin replies with alternate time
   - Check logs for token generation and storage
   - Click acceptance link in email
   - Check logs for token validation
   - Verify inspection status updated

3. **Monitor Logs For:**
   - 🔍 [ALTERNATE TIME DEBUG] - Alternate time logic
   - 🔄 [DATABASE] - Database operations
   - 🔍 [DATABASE VERIFICATION] - Token verification
   - 🔗 [URL DECODING] - URL encoding issues
   - 🔍 [TOKEN VALIDATION] - Token validation steps
   - 📅 [SCHEDULE CALCULATION] - Date calculation

## Error Scenarios Now Handled

1. **Token Not Found:**
   - Logs all existing tokens for comparison
   - Provides detailed debugging information

2. **URL Encoding Issues:**
   - Automatically detects and decodes encoded tokens
   - Falls back to original token if needed

3. **Database Transaction Failures:**
   - Comprehensive error logging
   - Continues processing even if secondary operations fail

4. **Date Calculation Errors:**
   - Validates calculated date before use
   - Uses correct date field for final schedule

5. **Expired or Already Used Tokens:**
   - Clear error messages
   - Appropriate HTTP status codes
   - Graceful user handling

## Files Modified

- `src/app/api/admin/tickets/[id]/reply/route.ts` - Enhanced logging and verification
- `src/app/api/accept-alternate-time/route.ts` - URL decoding and enhanced validation
- `src/app/api/debug/alternate-time-tokens/route.ts` - New debug endpoint

## Expected Outcome

The acceptance token workflow should now work end-to-end with comprehensive logging to identify any remaining issues. Customers should be able to successfully accept alternate time proposals, and administrators will have full visibility into the process through detailed logs and the debug endpoint.