# Backend Audit Fixes — Implementation Plan

**Source audit:** `/Users/sairajaggani/Desktop/Space/Project/backend.md`
**Branch:** `fix/backend-audit-p0-p1` (off `CarpoolBackend/main`)
**Date:** 2026-06-17

---

## Global Constraints

- Runtime: Firebase Cloud Functions, Node.js 20, TypeScript, ESM (`import`/`export`)
- Backend root: `/Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions/`
- Source tree: `src/functions/`, `src/config/`, `src/middleware/`, `src/utils/`
- Error pattern: `throw Errors.INVALID_INPUT({ message: '...' })` (from `src/utils/errors.ts`)
- Logging: `logger.info(...)`, `logger.error(...)`, `logger.warn(...)` from `@utils/logger`
- Timestamps: Firestore `Timestamp` from `firebase-admin/firestore`
- IST offset: UTC+5:30 = 19800 seconds. When deriving local date from a UTC ms value for India: `new Date(ms + 19800000).toISOString().split("T")[0]`
- Tests: `jest` in `tests/unit/` and `tests/integration/`. Run with `cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npx jest`
- NEVER commit directly to `main` — all work is on `fix/backend-audit-p0-p1`
- NEVER skip tests — run relevant tests after each change
- Severity legend: P0 = data corruption/security, P1 = wrong behaviour/silent failure, P2 = perf/reliability

---

## Task 1: P0 — Currency Fixes in sendRideOffer

**Issues:** B-P0-3, B-P0-4
**File:** `src/functions/offers/sendRideOffer.fn.ts`

### B-P0-3 — Currency symbol in notification
**Location:** Line ~192

Current code:
```ts
`${profile.firstName} offered a ride for ₹${validated.price}`
```

**Fix:** Pick the symbol from `validated.currency`:
```ts
const currencySymbol = validated.currency === 'INR' ? '₹' : validated.currency === 'USD' ? '$' : validated.currency;
const notificationBody = `${profile.firstName} offered a ride for ${currencySymbol}${validated.price}`;
```

### B-P0-4 — Enforce currency match
**Location:** Lines ~118–129 (after reading the ride request doc)

**Fix:** After reading `request` from Firestore, add:
```ts
if (validated.currency !== request.currency) {
  throw Errors.INVALID_INPUT({ message: `Offer currency (${validated.currency}) must match request currency (${request.currency})` });
}
```
Add this check before the price cap math.

**Tests to write:** Unit test in `tests/unit/functions/offers/sendRideOffer.test.ts` (if exists, add cases; if not, create it):
1. `should reject offer with mismatched currency`
2. `should use correct currency symbol in notification for USD offer`
3. `should use ₹ symbol for INR offer`

---

## Task 2: P0 — preferredDepartureTime Validation

**Issues:** B-P0-1, B-P0-2
**Files:**
- `src/functions/rides/createRideRequest.fn.ts` (B-P0-2)
- `src/functions/offers/acceptRideOffer.fn.ts` (B-P0-1)

### B-P0-2 — Validate on createRideRequest
**Location:** `createRideRequest.fn.ts` lines ~117–119 (after schema parsing)

**Fix:** After `validated` is set, add validation:
```ts
if (validated.preferredDepartureTime !== undefined) {
  const ms = validated.preferredDepartureTime;
  // Derive local date in IST (UTC+5:30)
  const localDateStr = new Date(ms + 19800000).toISOString().split("T")[0];
  const isOnRequestedDate = localDateStr === validated.requestedDate;
  const isInFlexRange = validated.flexibleDates &&
    validated.dateRange &&
    localDateStr >= validated.dateRange.startDate &&
    localDateStr <= validated.dateRange.endDate;

  if (!isOnRequestedDate && !isInFlexRange) {
    throw Errors.INVALID_INPUT({
      message: 'preferredDepartureTime must fall on requestedDate (or within dateRange for flexible requests)',
    });
  }
}
```

### B-P0-1 — Validate and sanitize on acceptRideOffer
**Location:** `acceptRideOffer.fn.ts` lines ~84–101

The field `request.preferredDepartureTime` arrives as a Firestore Timestamp. Derive `rideStartMs` safely:

```ts
// Validate preferredDepartureTime is on requestedDate
const rideStartMs = (() => {
  const rawMs = request.preferredDepartureTime?.toMillis?.() ?? null;
  if (rawMs !== null) {
    const localDateStr = new Date(rawMs + 19800000).toISOString().split("T")[0];
    const isOnDate = localDateStr === request.requestedDate;
    const isInRange = request.flexibleDates &&
      request.dateRange &&
      localDateStr >= request.dateRange.startDate &&
      localDateStr <= request.dateRange.endDate;
    if (isOnDate || isInRange) return rawMs;
  }
  // Default: requestedDate at 08:00 IST
  const [y, m, d] = request.requestedDate.split('-').map(Number);
  // 08:00 IST = 02:30 UTC
  return Date.UTC(y, m - 1, d, 2, 30);
})();
```

**Tests to write/extend:**
1. `createRideRequest`: rejects `preferredDepartureTime` not on `requestedDate`
2. `createRideRequest`: accepts `preferredDepartureTime` within `dateRange` for flexible request
3. `acceptRideOffer`: uses requestedDate@08:00 IST when `preferredDepartureTime` is on wrong date
4. `acceptRideOffer`: uses valid `preferredDepartureTime` when it matches `requestedDate`

---

## Task 3: P0 — browseRideRequests Cursor Fix

**Issue:** B-P0-5
**File:** `src/functions/offers/browseRideRequests.fn.ts` lines ~136–174

**Problem:** The cursor `step` field is encoded back unchanged — each page re-runs every H3 step.

**Fix (option a — document all-scan model, remove step from cursor):**
1. Remove `step` from the cursor state type/encoding.
2. Change the cursor to only carry `lastRequestId` (and any other fields it currently uses except `step`).
3. The pagination slices from `lastRequestId` onwards in the merged result.
4. Add a comment: `// All H3 steps are always scanned; cursor only tracks position within merged results.`

If the cursor type has a TypeScript interface, update it to remove the `step` field. Search for `cursorState` and `CursorState` in the file.

**Tests to write:**
1. `should return consistent second page when using cursor from first page`
2. `should not re-emit items from first page on second call`

---

## Task 4: P1 — browseRideRequests Functional Fixes

**Issues:** B-P1-3, B-P1-4
**File:** `src/functions/offers/browseRideRequests.fn.ts`

### B-P1-4 — Fix UTC date (use IST)
**Location:** Line ~93

Current:
```ts
const todayStr = now.toISOString().split("T")[0];
```

Fix:
```ts
// Use IST (UTC+5:30) to avoid midnight-UTC window where today in India ≠ today in UTC
const todayStr = new Date(now.getTime() + 19800000).toISOString().split("T")[0];
```

### B-P1-3 — Filter unfulfillable requests
**Location:** Lines ~313–344 (result filtering / enrichment section)

Add server-side filters before returning results:
1. Skip requests where `request.pendingOfferCount >= MAX_PER_REQUEST` (constant is 5).
   - Read `pendingOfferCount` from the request doc (it exists as of the `pendingOfferCount` feature commit).
   - If `pendingOfferCount` field is absent, treat as 0 (backwards compat).
2. Add optional `maxSeatsAvailable` param: if driver sends it, skip requests with `request.seatsNeeded > maxSeatsAvailable`.
3. Add `offerLimitReached` boolean to the response if the *calling driver* has `MAX_ACTIVE_PER_DRIVER` (10) active offers — derive from the driver's active-offer count query already performed in this function.

**Tests to write:**
1. `should exclude requests where pendingOfferCount >= 5`
2. `should filter by maxSeatsAvailable when provided`
3. `should set offerLimitReached flag when driver has 10 active offers`
4. `today uses IST date not UTC` (unit test with a mocked now at 00:30 UTC which is 06:00 IST)

---

## Task 5: P1 — Notification Fixes (expire + cancel)

**Issues:** B-P1-1, B-P1-2
**Files:**
- `src/functions/scheduled/expireRideRequests.fn.ts` (B-P1-1)
- `src/functions/rides/cancelRideRequest.fn.ts` (B-P1-2)

### B-P1-1 — Notify passengers on request expiry
**Location:** `expireRideRequests.fn.ts` — after marking a request `EXPIRED`

After each request is expired, send FCM notification to the passenger:
```ts
await NotificationService.sendToUser(request.userId, {
  title: 'Ride Request Expired',
  body: 'Your ride request has expired. Tap to resubmit.',
  data: {
    type: 'RIDE_REQUEST_EXPIRED',
    requestId: doc.id,
    deepLink: 'carpool://my-requests',
  },
});
```

Look at how `NotificationService.sendToUser` is called elsewhere in the codebase for the exact signature. Use `.catch(err => logger.error('Failed to send expiry notification', { err }))` to avoid blocking the batch.

### B-P1-2 — Notify drivers when offers cancelled
**Location:** `cancelRideRequest.fn.ts` — after the transaction commits and the post-commit sweep

After the offer-cancellation sweep, fan-out `RIDE_OFFER_CANCELLED` to unique driver UIDs:

```ts
// Collect unique driverIds from cancelled offers
const cancelledDriverIds = [...new Set(cancelledOffers.map(o => o.driverId))];
await Promise.all(
  cancelledDriverIds.map(driverId =>
    NotificationService.sendToUser(driverId, {
      title: 'Offer Cancelled',
      body: 'A passenger cancelled their ride request. Your offer has been withdrawn.',
      data: { type: 'RIDE_OFFER_CANCELLED', requestId },
    }).catch(err => logger.error('Failed to notify driver of offer cancellation', { err, driverId }))
  )
);
```

**Tests to write:**
1. `expireRideRequests should send RIDE_REQUEST_EXPIRED FCM to passenger`
2. `cancelRideRequest should notify each unique driver whose offer was cancelled`
3. `cancelRideRequest notification failure should not throw`

---

## Task 6: P1 — subscribeToRequestAlerts + expireRideRequests Batch Fix

**Issues:** B-P1-9, B-P1-10
**Files:**
- `src/functions/scheduled/expireRideRequests.fn.ts` (B-P1-9)
- `src/functions/offers/subscribeToRequestAlerts.fn.ts` (B-P1-10)

### B-P1-9 — Batch commit: use explicit counter flag
**Location:** `expireRideRequests.fn.ts` — the batch loop (lines ~31–49)

Replace `count % 500 !== 0` guard with an explicit `batchCount` counter (same pattern already used in offers helper):

```ts
let batch = db.batch();
let batchCount = 0;

for (const doc of docs) {
  // ... set fields on batch
  batchCount++;
  if (batchCount === 500) {
    await batch.commit();
    batch = db.batch();
    batchCount = 0;
  }
}

if (batchCount > 0) {
  await batch.commit();
}
```

### B-P1-10 — Clear h3Cells when disabling alerts
**Location:** `subscribeToRequestAlerts.fn.ts` lines ~34–39

When `enabled === false`, overwrite the entire doc without the H3 cells:
```ts
if (validated.enabled === false) {
  await subRef.set({
    driverId: uid,
    enabled: false,
    updatedAt: Timestamp.now(),
  }); // merge: false (default) — clears h3Cells_7 and other fields
} else {
  // existing enable logic with h3Cells_7
}
```

**Tests to write:**
1. `subscribeToRequestAlerts: disabling should clear h3Cells_7 from the doc`
2. `expireRideRequests: batch counter uses explicit batchCount > 0 check`

---

## Task 7: P1 — acceptRideOffer Robustness Fixes

**Issues:** B-P1-13, B-P1-14, B-P1-15
**File:** `src/functions/offers/acceptRideOffer.fn.ts`

### B-P1-13 — Catch notification errors
**Location:** Lines ~326–332

Current:
```ts
NotificationService.sendToUser(...);  // no await, no .catch
```

Fix:
```ts
NotificationService.sendToUser(...).catch(err =>
  logger.error('Failed to send acceptRideOffer notification', { err })
);
```

### B-P1-14 — Add date filter to overlap queries
**Location:** Lines ~138–185

In the queries for driver and passenger overlap checks, add `where("rideDate", ">=", todayDateStr)` to limit historical reads. You need `todayDateStr` (IST date string) — derive it the same way as Task 4's fix: `new Date(Date.now() + 19800000).toISOString().split("T")[0]`.

If there's no existing composite index for `(status, rideDate)`, add a note in the commit message that an index may be needed and which one.

### B-P1-15 — Runtime guard for rideStartTime
**Location:** Lines ~145–152

Current:
```ts
(r.rideStartTime as Timestamp).toMillis()
```

Fix:
```ts
if (!r.rideStartTime?.toMillis) continue;
const startMs = r.rideStartTime.toMillis();
```

**Tests to write:**
1. `acceptRideOffer: notification send failure should not throw`
2. `acceptRideOffer: skips ride docs missing rideStartTime in overlap check`

---

## Task 8: P1 — sendRideOffer Read Optimisation

**Issues:** B-P1-11
**File:** `src/functions/offers/sendRideOffer.fn.ts`

### B-P1-11 — Merge two sequential offer reads into one
**Location:** Lines ~78–109

Current:
1. First query: `existingOffersOnRequest` — `requestId == X AND status == PENDING`
2. Second query: `driverActiveOffers` — `driverId == uid AND status == PENDING`

**Fix:** Run only the driver's pending offers query first, then derive what you need:
```ts
// Single query for driver's pending offers
const driverOffersSnap = await db.collection('rideOffers')
  .where('driverId', '==', uid)
  .where('status', '==', 'PENDING')
  .get();

const activeCount = driverOffersSnap.size;
if (activeCount >= MAX_ACTIVE_PER_DRIVER) {
  throw Errors.LIMIT_EXCEEDED({ message: 'You have reached the maximum number of active offers' });
}

// Check if driver already has an offer on this request
const existingOffer = driverOffersSnap.docs.find(d => d.data().requestId === requestId);
if (existingOffer) {
  throw Errors.ALREADY_EXISTS({ message: 'You already have a pending offer on this request' });
}
```

Skip the original `existingOffersOnRequest` query entirely — it was only needed for the duplicate check, which is now covered by `driverOffersSnap`.

**Note:** The request-level `pendingOfferCount >= MAX_PER_REQUEST` check should still read from the request doc itself (which has the counter as of the recent feature commit). Do not re-query all offers on the request.

**Tests to write:**
1. `sendRideOffer: should reject when driver already has an offer on this request (uses driver query)`
2. `sendRideOffer: should reject when driver has >= 10 active offers`
3. `sendRideOffer: should make only one offer collection query`

---

## Task 9: P2 — browseRideRequests Performance

**Issues:** B-P2-1, B-P2-2, B-P2-3, B-P2-4
**File:** `src/functions/offers/browseRideRequests.fn.ts`

### B-P2-1 — Skip city fallback when enough results
**Location:** Lines ~148–153

Before running city fallback, add:
```ts
const limit = validated.limit ?? 20;
if (scoredResults.length >= limit * 3) {
  // Skip city fallback — enough results from H3 steps
} else {
  // existing city fallback
}
```

### B-P2-2 — Order H3 step queries by requestedDate ASC
**Location:** Lines ~119–145

Add `.orderBy('requestedDate', 'asc')` to each H3 step query. This requires the relevant Firestore index to already exist (check `firestore.indexes.json` — the `(fromH3_8, status, requestedDate)` and `(fromH3_7, status, requestedDate)` indexes should already be present per the memory notes).

### B-P2-3 — Single offer query instead of two
**Location:** Lines ~189–215

Replace the two-query pattern with a single query of all pending offers on the request chunk, then derive `hasMyOffer` by `doc.data().driverId === uid`:

```ts
const offerSnap = await db.collection('rideOffers')
  .where('requestId', 'in', requestIdChunk)
  .where('status', '==', 'PENDING')
  .get();

const pendingCountByRequest: Record<string, number> = {};
const hasMyOfferByRequest: Record<string, boolean> = {};

for (const doc of offerSnap.docs) {
  const { requestId, driverId } = doc.data();
  pendingCountByRequest[requestId] = (pendingCountByRequest[requestId] ?? 0) + 1;
  if (driverId === uid) hasMyOfferByRequest[requestId] = true;
}
```

### B-P2-4 — Tertiary sort key for deterministic ordering
**Location:** Lines ~156–159

Add `requestId` as a tie-breaker:
```ts
scoredResults.sort((a, b) => {
  if (b.relevanceScore !== a.relevanceScore) return b.relevanceScore - a.relevanceScore;
  const dateCmp = a.request.requestedDate.localeCompare(b.request.requestedDate);
  if (dateCmp !== 0) return dateCmp;
  return a.request.requestId.localeCompare(b.request.requestId);
});
```

**Tests to write:**
1. `should skip city fallback when scoredResults.length >= limit * 3`
2. `should use requestId as tertiary sort key for equal score+date`
3. `should derive hasMyOffer from single offer query`

---

## Task 10: Security + Observability + P3 Polish

**Issues:** B-S-3, B-P3-8, B-O-1, B-O-2
**Files:**
- `src/config/constants.ts` (B-S-3, B-P3-8)
- `src/functions/offers/browseRideRequests.fn.ts` (B-O-1)
- `src/functions/scheduled/expireRideRequests.fn.ts` (B-O-2)
- `src/middleware/rate-limit.middleware.ts` (B-S-3)

### B-S-3 — Lower subscribeToRequestAlerts rate limit
**Location:** `constants.ts` line ~66

Change `SUBSCRIBE_REQUEST_ALERTS` rate from `10/min` to `3/min`.

Find where the rate limit is defined and set it to 3 per minute. Check `rate-limit.middleware.ts` for the specific configuration and update it there if that's where the limit is applied.

### B-P3-8 — Move EXPIRY_DAYS to constants
**Location:** `createRideRequest.fn.ts` — wherever "7" appears as the expiry days magic number

Add to `constants.ts`:
```ts
RIDE_REQUEST: {
  EXPIRY_DAYS: 7,
},
```

Replace the magic `7` in `createRideRequest.fn.ts` with `CONSTANTS.RIDE_REQUEST.EXPIRY_DAYS`.

### B-O-1 — Metrics on browseRideRequests step distribution
**Location:** `browseRideRequests.fn.ts` — at the end of the function, before returning

Add:
```ts
logger.info('browseRideRequests completed', {
  stepsScanned: maxStep + 1,
  resultsReturned: results.length,
  cityFallbackHit: ranCityFallback, // boolean flag set when city fallback runs
});
```

Track `ranCityFallback` as a boolean in the function body.

### B-O-2 — Wrap expireRideRequests in try/catch
**Location:** `expireRideRequests.fn.ts` — the top-level handler body

Wrap the main logic in try/catch:
```ts
try {
  // ... existing logic
} catch (err) {
  logger.error('expireRideRequests failed', { err });
  throw err; // re-throw so Cloud Scheduler sees the failure
}
```

**Tests to write:**
1. `subscribeToRequestAlerts rate limit is 3/min`
2. `expireRideRequests logs structured error on failure`
