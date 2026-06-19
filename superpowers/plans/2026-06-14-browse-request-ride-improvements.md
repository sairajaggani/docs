# Browse Rides / Request Ride / My Requests — Bug Fixes + UX Improvements

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix backend data bugs (stale MATCHED requests, duplicate requests, offer TOCTOU), add `pendingOfferCount` + `dateEnd` to browse CF, then fix FE data bugs, then ship UX polish (swap button, expiry chips, avatars, FULFILLED tab split, subscribe CTA, theme token).

**Architecture:** Three-part delivery — Part 1: CF fixes deployed independently; Part 2: FE data bugs (no new APIs needed); Part 3: FE UX polish (consumes new CF fields from Part 1). All changes are additive or narrowing — no schema migrations, no new Firestore collections.

**Tech Stack:** Firebase Cloud Functions (Node 20, TypeScript, ESM) · Firestore · React Native + Expo SDK 54 + Expo Router v6 · TypeScript strict

---

## File Map

**Modified — Backend**
- `CarpoolBackend/functions/src/functions/scheduled/expireRideRequests.fn.ts` — add MATCHED to expiry query
- `CarpoolBackend/functions/src/functions/rides/createRideRequest.fn.ts` — duplicate check includes MATCHED
- `CarpoolBackend/functions/src/functions/rides/cancelRideRequest.fn.ts` — post-commit offer sweep
- `CarpoolBackend/functions/src/functions/offers/browseRideRequests.fn.ts` — add `dateEnd` param + `pendingOfferCount` per item
- `CarpoolBackend/functions/src/middleware/validation.middleware.ts` — add `dateEnd` to `browseRideRequests` schema

**Modified — Frontend services/types**
- `Carpool/src/services/firebase/functions.service.ts` — add `dateEnd` to `browseRideRequests` signature

**Created — Frontend**
- `Carpool/components/AvatarImage.tsx` — reusable avatar with uri/fallback

**Modified — Frontend screens & components**
- `Carpool/constants/theme.ts` — add `colors.destination`
- `Carpool/app/tabs/need-ride.tsx` — pull-to-refresh fix, swap button, always-visible link, resubmit params
- `Carpool/app/my-requests.tsx` — stale ride guard, expires chip, pull-to-refresh, FULFILLED tab fix, resubmit button
- `Carpool/app/request-details.tsx` — STATUS_CONFIG map, offer count chip, destination token
- `Carpool/app/request-details-passenger.tsx` — remove `as any` cast, AvatarImage
- `Carpool/components/BrowseRequestsList.tsx` — week filter fix, useFocusEffect, offer count chip, AvatarImage, subscribe CTA, destination token
- `Carpool/app/tabs/post-ride.tsx` — remove dead commented-out code

---

## Part 1 — Backend Bug Fixes

---

### Task 1: expireRideRequests — expire MATCHED requests too

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/scheduled/expireRideRequests.fn.ts:18`

- [ ] **Step 1: Open the file and locate the Firestore query**

Line 18 currently reads:
```ts
      .where("status", "==", "PENDING")
```

- [ ] **Step 2: Change query to include MATCHED**

Replace the single `where("status", "==", "PENDING")` with an `in` query:

```ts
// Before
const snapshot = await db
  .collection(CONSTANTS.COLLECTIONS.RIDE_REQUESTS)
  .where("status", "==", "PENDING")
  .where("expiresAt", "<", now)
  .get();
```

```ts
// After
const snapshot = await db
  .collection(CONSTANTS.COLLECTIONS.RIDE_REQUESTS)
  .where("status", "in", ["PENDING", "MATCHED"])
  .where("expiresAt", "<", now)
  .get();
```

No other changes needed — the batch update, commit loop, and `expirePendingOffersForRequests` call below it are all unchanged and already handle MATCHED requests correctly.

- [ ] **Step 3: Type-check**

```bash
cd CarpoolBackend/functions && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add functions/src/functions/scheduled/expireRideRequests.fn.ts
git commit -m "fix(scheduled): expire MATCHED ride requests after 7 days

MATCHED requests were never swept — passengers saw stale 'View Matched
Ride' buttons pointing to completed/cancelled rides indefinitely."
```

---

### Task 2: createRideRequest — duplicate check covers MATCHED status

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/createRideRequest.fn.ts:43`

- [ ] **Step 1: Locate the duplicate check query (~line 37-46)**

```ts
const existingRequests = await db
  .collection(CONSTANTS.COLLECTIONS.RIDE_REQUESTS)
  .where("userId", "==", uid)
  .where("fromH3_7", "==", fromH3Check)
  .where("toH3_7", "==", toH3Check)
  .where("requestedDate", "==", validated.requestedDate)
  .where("status", "==", "PENDING")
  .limit(1)
  .get();
```

- [ ] **Step 2: Change the status filter to include MATCHED**

```ts
const existingRequests = await db
  .collection(CONSTANTS.COLLECTIONS.RIDE_REQUESTS)
  .where("userId", "==", uid)
  .where("fromH3_7", "==", fromH3Check)
  .where("toH3_7", "==", toH3Check)
  .where("requestedDate", "==", validated.requestedDate)
  .where("status", "in", ["PENDING", "MATCHED"])
  .limit(1)
  .get();
```

- [ ] **Step 3: Update the error message to be accurate**

```ts
if (!existingRequests.empty) {
  throw new functions.https.HttpsError(
    "already-exists",
    "You already have an active request for this route and date."
  );
}
```

- [ ] **Step 4: Type-check**

```bash
cd CarpoolBackend/functions && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add functions/src/functions/rides/createRideRequest.fn.ts
git commit -m "fix(createRideRequest): block duplicate requests when existing is MATCHED"
```

---

### Task 3: cancelRideRequest — post-commit offer sweep closes TOCTOU gap

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/cancelRideRequest.fn.ts`

- [ ] **Step 1: Locate the section after the transaction (around line 127)**

The transaction block ends with `});`. Immediately after, the logger call and `return ResponseFormatter.success(...)` appear.

- [ ] **Step 2: Add a post-commit sweep between the transaction and the return**

Insert after the closing `});` of the transaction and before the `logger.info`:

```ts
      // Post-commit sweep: cancel any PENDING offers that were created between
      // the pre-transaction read and the commit (closes the TOCTOU window).
      const remainingPendingOffers = await db
        .collection(CONSTANTS.COLLECTIONS.RIDE_OFFERS)
        .where("requestId", "==", requestId)
        .where("status", "==", "PENDING")
        .get();

      if (!remainingPendingOffers.empty) {
        const sweepBatch = db.batch();
        for (const doc of remainingPendingOffers.docs) {
          sweepBatch.update(doc.ref, {
            status: "CANCELLED",
            cancelledBy: uid,
            cancelledAt: now,
            updatedAt: now,
          });
        }
        await sweepBatch.commit();
        logger.info("Post-commit offer sweep cancelled additional offers", {
          requestId,
          count: remainingPendingOffers.size,
        });
      }
```

- [ ] **Step 3: Update the `cancelledOffers` count in the return to include the sweep**

```ts
      return ResponseFormatter.success({
        requestId,
        cancelledOffers: pendingOffersSnap.size + remainingPendingOffers.size,
        message: "Ride request cancelled successfully.",
      });
```

- [ ] **Step 4: Type-check**

```bash
cd CarpoolBackend/functions && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add functions/src/functions/rides/cancelRideRequest.fn.ts
git commit -m "fix(cancelRideRequest): post-commit sweep closes offer TOCTOU gap"
```

---

### Task 4: browseRideRequests — add dateEnd schema param + pendingOfferCount per item

**Files:**
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts:368-374`
- Modify: `CarpoolBackend/functions/src/functions/offers/browseRideRequests.fn.ts`

- [ ] **Step 1: Add `dateEnd` to the browseRideRequests Zod schema**

In `validation.middleware.ts`, find the `browseRideRequests` schema (~line 368) and add `dateEnd`:

```ts
  browseRideRequests: z.object({
    from: commonSchemas.location,
    date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
    dateEnd: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
    cursor: z.string().optional(),
    limit: z.number().int().min(1).max(50).default(10),
    maxStep: z.number().int().min(0).max(4).optional(),
  }),
```

- [ ] **Step 2: Update `browseRideRequests.fn.ts` to destructure `dateEnd` and use it as the upper date bound**

Find the destructure line (~line 57):
```ts
      const { from, date, cursor } = validated;
```
Change to:
```ts
      const { from, date, cursor } = validated;
      const dateEndParam = validated.dateEnd;
```

Find the date range block (~lines 93-103):
```ts
      if (date) {
        dateStart = date;
        dateEnd = date;
      } else {
        dateStart = todayStr;
        const endDate = new Date(now.getTime() + 14 * CONSTANTS.TIME.DAY_MS);
        dateEnd = endDate.toISOString().split("T")[0];
      }
```

Note: the local variable is already named `dateEnd`. Rename the local `dateEnd` variable to `dateEndStr` throughout the function to avoid shadowing the destructured param, then apply the override:

```ts
      let dateStart: string;
      let dateEndStr: string;

      if (date) {
        dateStart = date;
        dateEndStr = dateEndParam ?? date;
      } else {
        dateStart = todayStr;
        const defaultEnd = new Date(now.getTime() + 14 * CONSTANTS.TIME.DAY_MS);
        dateEndStr = dateEndParam ?? defaultEnd.toISOString().split("T")[0];
      }
```

Then rename every use of `dateEnd` (the local variable) to `dateEndStr` in the step definitions and query calls. There are occurrences in `queryRequests` calls and `queryCityFallback` call — update all of them:

```ts
      const steps = [
        () => queryRequests({
          h3Field: "fromH3_8", h3Values: [driverH3_8],
          dateStart, dateEnd: dateEndStr, score: 100,
          // ...
        }),
        () => queryRequests({
          h3Field: "fromH3_8", h3Values: h3_8_neighbors,
          dateStart, dateEnd: dateEndStr, score: 80,
          // ...
        }),
        () => queryRequests({
          h3Field: "fromH3_7", h3Values: h3_7_ring1,
          dateStart, dateEnd: dateEndStr, score: 60,
          // ...
        }),
      ];

      // ...
      if (maxStep >= 3 && h3_7_ring2.length > 0 && startStep <= 3) {
        await queryRequests({
          h3Field: "fromH3_7", h3Values: h3_7_ring2,
          dateStart, dateEnd: dateEndStr, score: 45,
          // ...
        });
      }

      if (maxStep >= 4 && driverCity && startStep <= 4) {
        await queryCityFallback({
          city: driverCity, dateStart, dateEnd: dateEndStr, score: 30,
          // ...
        });
      }
```

- [ ] **Step 3: Add `pendingOfferCount` alongside the existing `driverOfferSet` query**

Find the existing offer-set query block (~lines 176-199). After building `driverOfferSet`, add a parallel count map:

```ts
      const driverOfferSet = new Set<string>();
      const offerCountMap = new Map<string, number>();

      if (pageRequestIds.length > 0) {
        const chunks: string[][] = [];
        for (let i = 0; i < pageRequestIds.length; i += 30) {
          chunks.push(pageRequestIds.slice(i, i + 30));
        }
        await Promise.all(
          chunks.map(async (chunk) => {
            // Driver's own pending offers (for hasMyOffer badge)
            const driverOfferSnap = await db
              .collection(CONSTANTS.COLLECTIONS.RIDE_OFFERS)
              .where("driverId", "==", uid)
              .where("requestId", "in", chunk)
              .where("status", "==", "PENDING")
              .select("requestId")
              .get();
            for (const doc of driverOfferSnap.docs) {
              driverOfferSet.add(doc.data().requestId as string);
            }

            // All pending offers per request (for competition count)
            const allOffersSnap = await db
              .collection(CONSTANTS.COLLECTIONS.RIDE_OFFERS)
              .where("requestId", "in", chunk)
              .where("status", "==", "PENDING")
              .select("requestId")
              .get();
            for (const doc of allOffersSnap.docs) {
              const reqId = doc.data().requestId as string;
              offerCountMap.set(reqId, (offerCountMap.get(reqId) ?? 0) + 1);
            }
          })
        );
      }
```

- [ ] **Step 4: Include `pendingOfferCount` in the response items**

Find the `items` mapping block (~line 232) and add the new field:

```ts
      const items = pageItems.map(({ request, relevanceScore, distanceKm }) => {
        const passenger: PassengerInfo | null =
          request.passengerSnapshot ?? profileMap.get(request.userId) ?? null;
        return {
          requestId: request.requestId,
          userId: request.userId,
          from: { ... },
          to: { ... },
          requestedDate: request.requestedDate,
          flexibleDates: request.flexibleDates,
          dateRange: request.dateRange,
          seatsNeeded: request.seatsNeeded,
          maxPrice: request.maxPrice,
          currency: request.currency,
          status: request.status,
          preferredDepartureTime: request.preferredDepartureTime?.toMillis() ?? null,
          distanceKm,
          relevanceScore,
          hasMyOffer: driverOfferSet.has(request.requestId),
          pendingOfferCount: offerCountMap.get(request.requestId) ?? 0,
          passenger: passenger ? { ... } : null,
        };
      });
```

- [ ] **Step 5: Type-check**

```bash
cd CarpoolBackend/functions && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add functions/src/middleware/validation.middleware.ts \
        functions/src/functions/offers/browseRideRequests.fn.ts
git commit -m "feat(browseRideRequests): add dateEnd param and pendingOfferCount per item"
```

---

### Task 5: Deploy all Part 1 backend changes

- [ ] **Step 1: Final type-check**

```bash
cd CarpoolBackend/functions && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 2: Deploy**

```bash
firebase deploy --only \
  functions:expireRideRequests,\
  functions:createRideRequest,\
  functions:cancelRideRequest,\
  functions:browseRideRequests
```
Expected: all 4 functions deployed successfully.

---

## Part 2 — Frontend Data Bug Fixes

---

### Task 6: functions.service.ts — add dateEnd to browseRideRequests signature

**Files:**
- Modify: `Carpool/src/services/firebase/functions.service.ts:508-516`

- [ ] **Step 1: Add `dateEnd` to the method signature**

```ts
  async browseRideRequests(data: {
    from: { latitude: number; longitude: number; address: string; placeId: string; plusCode?: string; plusCodeCompound?: string };
    date?: string;
    dateEnd?: string;
    cursor?: string;
    limit: number;
    maxStep?: number;
  }): Promise<any> {
    return this.callFunction('browseRideRequests', data, { timeout: 20000 });
  }
```

- [ ] **Step 2: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add src/services/firebase/functions.service.ts
git commit -m "feat(functions.service): add dateEnd to browseRideRequests signature"
```

---

### Task 7: BrowseRequestsList — fix "This Week" filter + useFocusEffect refresh

**Files:**
- Modify: `Carpool/components/BrowseRequestsList.tsx`

- [ ] **Step 1: Change `getDateParam` return type and week case**

Find `getDateParam` (~line 328) and rewrite it:

```ts
function getDateParam(filter: DateFilter): { date?: string; dateEnd?: string } {
  const now = new Date();
  const pad = (n: number) => n.toString().padStart(2, '0');
  const toDateStr = (d: Date) => `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;

  switch (filter) {
    case 'today':
      return { date: toDateStr(now) };
    case 'tomorrow': {
      const tm = new Date(now);
      tm.setDate(tm.getDate() + 1);
      return { date: toDateStr(tm) };
    }
    case 'week': {
      const end = new Date(now);
      end.setDate(end.getDate() + 7);
      return { dateEnd: toDateStr(end) };
    }
    case 'all':
    default:
      return {};
  }
}
```

- [ ] **Step 2: Update `fetchRequests` to destructure the new return value**

Find the line inside `fetchRequests`:
```ts
      const dateParam = getDateParam(dateFilter);
```
Replace with:
```ts
      const { date: dateParam, dateEnd: dateEndParam } = getDateParam(dateFilter);
```

Then update the `functionsService.browseRideRequests` call:
```ts
      const result = await functionsService.browseRideRequests({
        from: driverLocation,
        date: dateParam,
        dateEnd: dateEndParam,
        cursor: cursorParam,
        limit: 10,
        maxStep: radiusTier,
      });
```

Also update the cache key to include `dateEndParam`:
```ts
      const cacheKey = `browseRequests:${dateFilter}:${radiusTier}:${driverLocation.latitude.toFixed(3)}:${driverLocation.longitude.toFixed(3)}`;
```
(No change needed — `dateFilter` already distinguishes `'week'`.)

- [ ] **Step 3: Add `useFocusEffect` to quietly refresh on tab re-focus**

Add import at the top:
```ts
import { useFocusEffect, useRouter } from 'expo-router';
```

Inside `BrowseRequestsList` component, after the existing `useEffect` hooks, add:
```ts
  // Quietly refresh cached data whenever the browse tab gains focus.
  // Ensures hasMyOffer badges and offer counts are fresh after navigating away.
  useFocusEffect(
    useCallback(() => {
      if (driverLocation) {
        fetchRequests(undefined, false, true);
      }
    }, [driverLocation, fetchRequests]),
  );
```

- [ ] **Step 4: Update the filter label from "This Week" to "7 Days"**

Find the filter pill map (~line 671):
```tsx
{f === 'all' ? 'All' : f === 'today' ? 'Today' : f === 'tomorrow' ? 'Tomorrow' : 'This Week'}
```
Change to:
```tsx
{f === 'all' ? 'All' : f === 'today' ? 'Today' : f === 'tomorrow' ? 'Tomorrow' : '7 Days'}
```

- [ ] **Step 5: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add components/BrowseRequestsList.tsx
git commit -m "fix(BrowseRequestsList): fix 7-day filter and refresh on tab focus"
```

---

### Task 8: my-requests — guard stale "View Matched Ride" button

**Files:**
- Modify: `Carpool/app/my-requests.tsx`

- [ ] **Step 1: Add state to track which matched ride IDs are stale**

Add inside the `MyRequests` component (after existing state declarations):
```ts
  const [staleRideIds, setStaleRideIds] = useState<Set<string>>(new Set());
```

- [ ] **Step 2: Add an effect that checks matched ride statuses on load**

After the existing offer-count `useEffect`, add:
```ts
  useEffect(() => {
    const matchedRequests = requests.filter(
      r => r.status === 'MATCHED' && r.matchedRideIds.length > 0
    );
    if (matchedRequests.length === 0) return;

    const rideIds = [...new Set(matchedRequests.flatMap(r => r.matchedRideIds))];

    Promise.all(
      rideIds.map(rideId =>
        firestoreService.getDocument<{ status: string }>('rides', rideId)
          .then(ride => ({ rideId, status: ride?.status ?? 'UNKNOWN' }))
          .catch(() => ({ rideId, status: 'UNKNOWN' }))
      )
    ).then(results => {
      const stale = new Set(
        results
          .filter(r => r.status !== 'OPEN' && r.status !== 'STARTED')
          .map(r => r.rideId)
      );
      setStaleRideIds(stale);
    });
  }, [requests]);
```

Note: `firestoreService.getDocument` — verify this method exists. Check `src/services/firebase/firestore.service.ts` for a one-shot document read. If the method is named differently (e.g. `getDoc`), use the correct name. Alternatively use `firestoreService.onDocumentSnapshot` and immediately unsubscribe.

- [ ] **Step 3: Replace the "View Matched Ride" button with a stale-aware version**

Find the matched ride button block (~line 184-199):
```tsx
              {item.status === 'MATCHED' && item.matchedRideIds.length > 0 && (
                <TouchableOpacity
                  style={styles.viewRideButton}
                  onPress={() => router.push({
                    pathname: '/ride-details',
                    params: {
                      rideId: item.matchedRideIds[0],
                      seatsRequested: String(item.seatsNeeded),
                    },
                  })}
                  activeOpacity={0.8}
                >
                  <Ionicons name="car-outline" size={16} color={theme.colors.text.inverse} />
                  <Text style={styles.viewRideText}>View Matched Ride</Text>
                </TouchableOpacity>
              )}
```

Replace with:
```tsx
              {item.status === 'MATCHED' && item.matchedRideIds.length > 0 && (
                staleRideIds.has(item.matchedRideIds[0]) ? (
                  <View style={styles.staleRideNote}>
                    <Ionicons name="alert-circle-outline" size={15} color={theme.colors.text.tertiary} />
                    <Text style={styles.staleRideNoteText}>Matched ride ended — waiting for new match</Text>
                  </View>
                ) : (
                  <TouchableOpacity
                    style={styles.viewRideButton}
                    onPress={() => router.push({
                      pathname: '/ride-details',
                      params: {
                        rideId: item.matchedRideIds[0],
                        seatsRequested: String(item.seatsNeeded),
                      },
                    })}
                    activeOpacity={0.8}
                  >
                    <Ionicons name="car-outline" size={16} color={theme.colors.text.inverse} />
                    <Text style={styles.viewRideText}>View Matched Ride</Text>
                  </TouchableOpacity>
                )
              )}
```

- [ ] **Step 4: Add the stale note styles**

Add to `StyleSheet.create(...)` at the bottom of the file:
```ts
  staleRideNote: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 6,
    paddingVertical: 8,
    paddingHorizontal: 4,
  },
  staleRideNoteText: {
    fontSize: theme.fontSize.sm,
    color: theme.colors.text.tertiary,
    flex: 1,
    lineHeight: 18,
  },
```

- [ ] **Step 5: Verify `firestoreService.getDocument` exists**

```bash
grep -n "getDocument\|getDoc\b" Carpool/src/services/firebase/firestore.service.ts | head -10
```

If the method doesn't exist, add a one-shot helper. If it's called `getDoc`, update the import in step 2. If it doesn't exist at all, use this alternative inside the effect:

```ts
      firestoreService.onDocumentSnapshot<{ status: string }>(
        'rides', rideId,
        (doc) => { unsub(); resolve({ rideId, status: doc?.status ?? 'UNKNOWN' }); },
        () => { unsub(); resolve({ rideId, status: 'UNKNOWN' }); }
      )
```
where `unsub` is the returned unsubscribe function.

- [ ] **Step 6: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add app/my-requests.tsx
git commit -m "fix(my-requests): guard stale matched ride links with live status check"
```

---

### Task 9: need-ride — fix pull-to-refresh spinner

**Files:**
- Modify: `Carpool/app/tabs/need-ride.tsx`

- [ ] **Step 1: Add `isRefreshing` state**

Find the `const [showSortModal, ...` state block and add below it:
```ts
  const [isRefreshing, setIsRefreshing] = useState(false);
```

- [ ] **Step 2: Wrap the `onRefresh` callback to set the flag**

Find the existing `onRefresh` (~line 444):
```ts
  const onRefresh = useCallback(async () => {
    await refresh();
  }, [refresh]);
```

Replace with:
```ts
  const onRefresh = useCallback(async () => {
    setIsRefreshing(true);
    try {
      await refresh();
    } finally {
      setIsRefreshing(false);
    }
  }, [refresh]);
```

- [ ] **Step 3: Pass `isRefreshing` to RefreshControl**

Find the `<RefreshControl` (~line 1046):
```tsx
          refreshControl={
            <RefreshControl
              refreshing={false}
              onRefresh={onRefresh}
```
Replace `refreshing={false}` with `refreshing={isRefreshing}`:
```tsx
          refreshControl={
            <RefreshControl
              refreshing={isRefreshing}
              onRefresh={onRefresh}
```

- [ ] **Step 4: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add app/tabs/need-ride.tsx
git commit -m "fix(need-ride): show pull-to-refresh spinner (was hardcoded false)"
```

---

### Task 10: request-details-passenger — remove `as any` cast

**Files:**
- Modify: `Carpool/app/request-details-passenger.tsx:175`

- [ ] **Step 1: Confirm DriverSnapshot has `rating`**

```bash
grep -n "DriverSnapshot" Carpool/src/types/models/profile.types.ts | head -5
grep -A 10 "export interface DriverSnapshot" Carpool/src/types/models/profile.types.ts
```
Expected: `rating: number` is present in the interface.

- [ ] **Step 2: Remove the `as any` cast**

Find (~line 175):
```tsx
              {(item.driver as any).rating > 0 && (
                <View style={styles.ratingRow}>
                  <Ionicons name="star" size={13} color={theme.colors.warning} />
                  <Text style={styles.ratingText}>{(item.driver as any).rating.toFixed(1)}</Text>
                </View>
              )}
```

Replace with:
```tsx
              {item.driver.rating > 0 && (
                <View style={styles.ratingRow}>
                  <Ionicons name="star" size={13} color={theme.colors.warning} />
                  <Text style={styles.ratingText}>{item.driver.rating.toFixed(1)}</Text>
                </View>
              )}
```

- [ ] **Step 3: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors. If TypeScript complains that `rating` doesn't exist on `DriverSnapshot`, add `rating: number` to the interface in `src/types/models/profile.types.ts`.

- [ ] **Step 4: Commit**

```bash
git add app/request-details-passenger.tsx
git commit -m "fix(request-details-passenger): remove as-any cast for driver rating"
```

---

## Part 3 — Frontend UX Polish

---

### Task 11: theme.ts — add destination color token + replace all hardcoded #A855F7

**Files:**
- Modify: `Carpool/constants/theme.ts`
- Modify: `Carpool/app/tabs/need-ride.tsx`
- Modify: `Carpool/app/my-requests.tsx`
- Modify: `Carpool/app/request-details.tsx`
- Modify: `Carpool/app/request-details-passenger.tsx`
- Modify: `Carpool/components/BrowseRequestsList.tsx`

- [ ] **Step 1: Add the destination token to theme**

Open `constants/theme.ts`. In the `colors` object, after `primary`, add:
```ts
    destination: '#A855F7',
```

- [ ] **Step 2: Replace all occurrences across files**

Run this to find every occurrence:
```bash
grep -rn "#A855F7\|'#A855F7'" Carpool/app Carpool/components
```

For each file found, replace the string `'#A855F7'` (and `"#A855F7"`) with `theme.colors.destination`. Ensure `theme` is already imported in each file (it is in all the files listed above).

Occurrences to replace:
- `need-ride.tsx`: destination icon color + `recentSearchDotTo` style background + `routeDotTo` background
- `my-requests.tsx`: `dotTo` style background
- `request-details.tsx`: `dotTo` style background
- `request-details-passenger.tsx`: `dotTo` style background
- `BrowseRequestsList.tsx`: `dotTo` style background

- [ ] **Step 3: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add constants/theme.ts \
        app/tabs/need-ride.tsx \
        app/my-requests.tsx \
        app/request-details.tsx \
        app/request-details-passenger.tsx \
        components/BrowseRequestsList.tsx
git commit -m "style: add theme.colors.destination token, replace all hardcoded #A855F7"
```

---

### Task 12: need-ride — add swap button + always-visible My Requests link

**Files:**
- Modify: `Carpool/app/tabs/need-ride.tsx`

- [ ] **Step 1: Add a `handleSwap` function**

After `handleClearSearch`:
```ts
  const handleSwap = () => {
    const tmpInput = fromInput;
    const tmpLocation = fromLocation;
    setFromInput(toInput);
    setFromLocation(toLocation);
    setToInput(tmpInput);
    setToLocation(tmpLocation);
    setSuggestions([]);
    setLocationHistory([]);
    setActiveInput(null);
  };
```

- [ ] **Step 2: Insert the swap button between the from and to input wrappers**

In the expanded search view, between the closing `</View>` of the from `inputWrapper` and the opening `<View style={styles.inputWrapper}>` of the to field, add:

```tsx
            {/* Swap button */}
            <TouchableOpacity
              style={styles.swapButton}
              onPress={handleSwap}
              activeOpacity={0.7}
              hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}
            >
              <Ionicons name="swap-vertical" size={18} color={theme.colors.primary} />
            </TouchableOpacity>
```

- [ ] **Step 3: Add swap button style**

In `StyleSheet.create`:
```ts
  swapButton: {
    alignSelf: 'flex-end',
    marginRight: theme.spacing.lg,
    marginVertical: -4,
    width: 32,
    height: 32,
    borderRadius: 16,
    backgroundColor: theme.colors.primaryLight,
    alignItems: 'center',
    justifyContent: 'center',
    borderWidth: 1,
    borderColor: theme.colors.border,
    zIndex: 2,
  },
```

- [ ] **Step 4: Remove the `!searching` guard from the My Requests link**

Find (~line 1001):
```tsx
      {/* My Ride Requests link */}
      {!searching && (
        <TouchableOpacity
          style={styles.myRequestsLink}
          onPress={() => router.push('/my-requests')}
          activeOpacity={0.7}
        >
```

Remove `{!searching && (` and the corresponding closing `)}`:
```tsx
      {/* My Ride Requests link */}
      <TouchableOpacity
        style={styles.myRequestsLink}
        onPress={() => router.push('/my-requests')}
        activeOpacity={0.7}
      >
        <Ionicons name="notifications-outline" size={14} color={theme.colors.primary} />
        <Text style={styles.myRequestsLinkText}>
          My ride requests{myRideRequests.length > 0 ? `  ·  ${myRideRequests.length} active` : ''}
        </Text>
        <Ionicons name="chevron-forward" size={14} color={theme.colors.primary} />
      </TouchableOpacity>
```

- [ ] **Step 5: Read resubmit params from URL and pre-fill search state**

The Resubmit button (Task 13) will navigate here with params. Add `useLocalSearchParams` import if not present, then read the params at the top of `NeedRide`:

```ts
  const params = useLocalSearchParams<{
    resubmitFromAddress?: string;
    resubmitFromLat?: string;
    resubmitFromLng?: string;
    resubmitFromPlaceId?: string;
    resubmitToAddress?: string;
    resubmitToLat?: string;
    resubmitToLng?: string;
    resubmitToPlaceId?: string;
    resubmitSeats?: string;
    resubmitMaxPrice?: string;
  }>();
```

Add a one-time effect to pre-fill from resubmit params:
```ts
  useEffect(() => {
    if (!params.resubmitFromAddress || !params.resubmitToAddress) return;
    const fromLat = parseFloat(params.resubmitFromLat ?? '0');
    const fromLng = parseFloat(params.resubmitFromLng ?? '0');
    const toLat = parseFloat(params.resubmitToLat ?? '0');
    const toLng = parseFloat(params.resubmitToLng ?? '0');
    setFromInput(params.resubmitFromAddress);
    setToInput(params.resubmitToAddress);
    setFromLocation({ latitude: fromLat, longitude: fromLng, address: params.resubmitFromAddress, placeId: params.resubmitFromPlaceId });
    setToLocation({ latitude: toLat, longitude: toLng, address: params.resubmitToAddress, placeId: params.resubmitToPlaceId });
    if (params.resubmitSeats) setPassengers(params.resubmitSeats);
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []); // run once on mount only
```

- [ ] **Step 6: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add app/tabs/need-ride.tsx
git commit -m "feat(need-ride): add swap button, always-visible requests link, resubmit prefill"
```

---

### Task 13: my-requests — expires chip, pull-to-refresh, FULFILLED tab fix, resubmit button

**Files:**
- Modify: `Carpool/app/my-requests.tsx`

- [ ] **Step 1: Fix the active tab filter — remove FULFILLED**

Find (~line 104):
```ts
  const visibleRequests = filter === 'active'
    ? requests.filter(r => r.status === 'PENDING' || r.status === 'MATCHED' || r.status === 'FULFILLED')
    : requests.filter(r => r.status !== 'CANCELLED');
```

Replace with:
```ts
  const visibleRequests = filter === 'active'
    ? requests.filter(r => r.status === 'PENDING' || r.status === 'MATCHED')
    : requests;
```

Update the active count accordingly:
```ts
  const activeCount = requests.filter(r => r.status === 'PENDING' || r.status === 'MATCHED').length;
```

- [ ] **Step 2: Add the "Expires in X days" chip inside `renderItem`**

Find the meta row block inside `renderItem` (lines ~163-178). After the existing `<View style={styles.metaRow}>` block (closing `</View>`), add the expires chip before the actions section:

```tsx
          {/* Expiry chip — only for active requests */}
          {(item.status === 'PENDING' || item.status === 'MATCHED') && (() => {
            const daysLeft = Math.ceil(
              (item.expiresAt.toMillis() - Date.now()) / (1000 * 60 * 60 * 24)
            );
            const urgent = daysLeft <= 1;
            return (
              <View style={[styles.expiryChip, urgent && styles.expiryChipUrgent]}>
                <Ionicons
                  name="time-outline"
                  size={11}
                  color={urgent ? theme.colors.error : theme.colors.text.tertiary}
                />
                <Text style={[styles.expiryChipText, urgent && styles.expiryChipTextUrgent]}>
                  {daysLeft <= 0 ? 'Expires today' : `Expires in ${daysLeft}d`}
                </Text>
              </View>
            );
          })()}
```

- [ ] **Step 3: Add Resubmit button for EXPIRED cards**

Find the actions section inside `renderItem` (~line 181):
```tsx
          {(item.status === 'PENDING' || item.status === 'MATCHED' || item.status === 'FULFILLED') && (
            <View style={styles.actions}>
```

After the closing `</View>` of the actions block (after the cancel button block), add a sibling block for EXPIRED:
```tsx
          {item.status === 'EXPIRED' && (
            <View style={styles.actions}>
              <TouchableOpacity
                style={styles.resubmitButton}
                onPress={() => router.push({
                  pathname: '/tabs/need-ride',
                  params: {
                    resubmitFromAddress: item.from.address,
                    resubmitFromLat: String(item.from.latitude),
                    resubmitFromLng: String(item.from.longitude),
                    resubmitFromPlaceId: item.from.placeId ?? '',
                    resubmitToAddress: item.to.address,
                    resubmitToLat: String(item.to.latitude),
                    resubmitToLng: String(item.to.longitude),
                    resubmitToPlaceId: item.to.placeId ?? '',
                    resubmitSeats: String(item.seatsNeeded),
                    resubmitMaxPrice: item.maxPrice ? String(item.maxPrice) : '',
                  },
                })}
                activeOpacity={0.8}
              >
                <Ionicons name="refresh-outline" size={16} color={theme.colors.primary} />
                <Text style={styles.resubmitText}>Resubmit</Text>
              </TouchableOpacity>
            </View>
          )}
```

- [ ] **Step 4: Add pull-to-refresh to the FlatList**

The component uses a Firestore listener, not a one-shot fetch, so "refresh" means unsubscribe + re-subscribe. Add state and handler:

```ts
  const [refreshing, setRefreshing] = useState(false);
  const unsubscribeRef = useRef<(() => void) | null>(null);
```

Modify the listener `useEffect` to store the unsubscribe fn:
```ts
  useEffect(() => {
    if (!user?.uid) return;
    setLoading(true);
    const unsubscribe = firestoreService.onCollectionSnapshot<RideRequest>(
      'rideRequests',
      [{ field: 'userId', operator: '==', value: user.uid }],
      undefined,
      undefined,
      (data) => {
        const sorted = [...data].sort(
          (a, b) => b.createdAt.toMillis() - a.createdAt.toMillis()
        );
        setRequests(sorted);
        setLoading(false);
        setRefreshing(false);
      },
      (_err) => {
        setLoading(false);
        setRefreshing(false);
      },
    );
    unsubscribeRef.current = unsubscribe;
    return unsubscribe;
  }, [user?.uid]);
```

Add a refresh handler:
```ts
  const handleRefresh = useCallback(() => {
    if (!user?.uid) return;
    setRefreshing(true);
    unsubscribeRef.current?.();
    const unsubscribe = firestoreService.onCollectionSnapshot<RideRequest>(
      'rideRequests',
      [{ field: 'userId', operator: '==', value: user.uid }],
      undefined,
      undefined,
      (data) => {
        const sorted = [...data].sort((a, b) => b.createdAt.toMillis() - a.createdAt.toMillis());
        setRequests(sorted);
        setLoading(false);
        setRefreshing(false);
      },
      () => { setRefreshing(false); }
    );
    unsubscribeRef.current = unsubscribe;
  }, [user?.uid]);
```

Add `RefreshControl` import at the top (add to existing RN imports):
```ts
import { ..., RefreshControl, useRef } from 'react-native';
```

Add `<RefreshControl>` to the FlatList:
```tsx
          <FlatList
            data={visibleRequests}
            keyExtractor={(item) => item.requestId}
            renderItem={renderItem}
            contentContainerStyle={styles.listContent}
            showsVerticalScrollIndicator={false}
            ListEmptyComponent={renderEmpty}
            refreshControl={
              <RefreshControl
                refreshing={refreshing}
                onRefresh={handleRefresh}
                colors={[theme.colors.primary]}
                tintColor={theme.colors.primary}
              />
            }
          />
```

- [ ] **Step 5: Add new styles**

```ts
  expiryChip: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 4,
    alignSelf: 'flex-start',
  },
  expiryChipUrgent: {},
  expiryChipText: {
    fontSize: 11,
    color: theme.colors.text.tertiary,
  },
  expiryChipTextUrgent: {
    color: theme.colors.error,
    fontWeight: theme.fontWeight.semibold as '600',
  },
  resubmitButton: {
    flex: 1,
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    gap: theme.spacing.sm,
    backgroundColor: theme.colors.primaryLight,
    paddingVertical: 10,
    borderRadius: theme.borderRadius.lg,
    borderWidth: 1,
    borderColor: theme.colors.primary,
  },
  resubmitText: {
    fontSize: theme.fontSize.base,
    fontWeight: theme.fontWeight.semibold as '600',
    color: theme.colors.primary,
  },
```

- [ ] **Step 6: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add app/my-requests.tsx
git commit -m "feat(my-requests): expires chip, pull-to-refresh, resubmit, FULFILLED out of Active"
```

---

### Task 14: request-details — STATUS_CONFIG + offer count chip

**Files:**
- Modify: `Carpool/app/request-details.tsx`

- [ ] **Step 1: Add a STATUS_CONFIG map at the top of the file (after imports)**

```ts
const STATUS_CONFIG: Record<string, { label: string; bg: string; fg: string }> = {
  PENDING:   { label: 'Open for Offers',     bg: '#dcfce7',                    fg: '#15803d' },
  MATCHED:   { label: 'Matched (Still Open)', bg: theme.colors.primaryLight,    fg: theme.colors.primary },
  FULFILLED: { label: 'Fulfilled',            bg: '#dcfce7',                    fg: '#15803d' },
  EXPIRED:   { label: 'Expired',              bg: '#f3f4f6',                    fg: '#9ca3af' },
  CANCELLED: { label: 'Cancelled',            bg: '#fef2f2',                    fg: '#ef4444' },
};
```

- [ ] **Step 2: Replace the ad-hoc status badge with the config-driven one**

Find the status row (~line 214):
```tsx
        {/* Status badge */}
        <View style={styles.statusRow}>
          <View style={[styles.statusBadge, request.status === 'FULFILLED' && styles.statusFulfilled]}>
            <Text style={styles.statusText}>
              {request.status === 'PENDING' ? 'Open for Offers' :
                request.status === 'MATCHED' ? 'Matched (Still Open)' :
                  request.status === 'FULFILLED' ? 'Fulfilled' : request.status}
            </Text>
          </View>
        </View>
```

Replace with:
```tsx
        {/* Status badge */}
        <View style={styles.statusRow}>
          {(() => {
            const cfg = STATUS_CONFIG[request.status] ?? { label: request.status, bg: '#f3f4f6', fg: '#9ca3af' };
            return (
              <View style={[styles.statusBadge, { backgroundColor: cfg.bg }]}>
                <Text style={[styles.statusText, { color: cfg.fg }]}>{cfg.label}</Text>
              </View>
            );
          })()}
        </View>
```

Remove the now-unused `statusFulfilled` style if present.

- [ ] **Step 3: Show the pending offer count on the request-details screen**

The request-details screen is the driver view — drivers see a "Send Offer" CTA. Add a pending offer count display below the Details section. Add state:
```ts
  const [pendingOfferCount, setPendingOfferCount] = useState<number>(0);
```

Add an effect to count offers after request loads:
```ts
  useEffect(() => {
    if (!requestId || !request) return;
    const unsub = firestoreService.onCollectionSnapshot<{ status: string }>(
      'rideOffers',
      [
        { field: 'requestId', operator: '==', value: requestId },
        { field: 'status', operator: '==', value: 'PENDING' },
      ],
      undefined,
      undefined,
      (data) => setPendingOfferCount(data.length),
      () => {},
    );
    return unsub;
  }, [requestId, request]);
```

Add a detail row to the Details section (after the existing detailRows):
```tsx
          {pendingOfferCount > 0 && (
            <View style={styles.detailRow}>
              <Ionicons name="people-outline" size={18} color={theme.colors.text.tertiary} />
              <Text style={styles.detailText}>
                {pendingOfferCount} offer{pendingOfferCount !== 1 ? 's' : ''} already sent
              </Text>
            </View>
          )}
```

- [ ] **Step 4: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add app/request-details.tsx
git commit -m "feat(request-details): STATUS_CONFIG badge fix + live pending offer count"
```

---

### Task 15: Create reusable AvatarImage component

**Files:**
- Create: `Carpool/components/AvatarImage.tsx`

- [ ] **Step 1: Create the component**

```tsx
// components/AvatarImage.tsx
import { Ionicons } from '@expo/vector-icons';
import { Image, StyleSheet, View } from 'react-native';
import { theme } from '../constants/theme';

interface AvatarImageProps {
  uri?: string | null;
  size?: number;
  iconSize?: number;
}

export default function AvatarImage({ uri, size = 44, iconSize = 20 }: AvatarImageProps) {
  const radius = size / 2;

  if (uri) {
    return (
      <Image
        source={{ uri }}
        style={[styles.avatar, { width: size, height: size, borderRadius: radius }]}
      />
    );
  }

  return (
    <View
      style={[
        styles.avatarPlaceholder,
        { width: size, height: size, borderRadius: radius },
      ]}
    >
      <Ionicons name="person" size={iconSize} color={theme.colors.text.tertiary} />
    </View>
  );
}

const styles = StyleSheet.create({
  avatar: {
    backgroundColor: theme.colors.primaryLight,
  },
  avatarPlaceholder: {
    backgroundColor: theme.colors.primaryLight,
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

- [ ] **Step 2: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add components/AvatarImage.tsx
git commit -m "feat(components): add reusable AvatarImage with uri/fallback"
```

---

### Task 16: request-details-passenger — use AvatarImage for driver photo

**Files:**
- Modify: `Carpool/app/request-details-passenger.tsx`

- [ ] **Step 1: Import AvatarImage**

Add to imports:
```ts
import AvatarImage from '../components/AvatarImage';
```

- [ ] **Step 2: Replace the generic avatar circle with AvatarImage**

Find the driver info row inside `renderOffer` (~line 168):
```tsx
            <View style={styles.avatarCircle}>
              <Ionicons name="person" size={20} color={theme.colors.text.tertiary} />
            </View>
```

Replace with:
```tsx
            <AvatarImage uri={item.driver.avatarUrl} size={44} iconSize={20} />
```

- [ ] **Step 3: Remove the now-unused `avatarCircle` style** (check it's not used elsewhere in the file first with a quick search).

- [ ] **Step 4: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add app/request-details-passenger.tsx
git commit -m "feat(request-details-passenger): render driver avatar photo"
```

---

### Task 17: BrowseRequestsList — offer count chip, AvatarImage, subscribe CTA

**Files:**
- Modify: `Carpool/components/BrowseRequestsList.tsx`

- [ ] **Step 1: Update `BrowseRequestItem` interface to include `pendingOfferCount`**

Find the `interface BrowseRequestItem` (~line 291) and add:
```ts
  pendingOfferCount: number;
```

- [ ] **Step 2: Import AvatarImage**

Add to imports:
```ts
import AvatarImage from './AvatarImage';
```

- [ ] **Step 3: Replace the avatar placeholder with AvatarImage in `renderItem`**

Find (~line 561):
```tsx
          <View style={styles.avatarPlaceholder}>
            <Ionicons name="person" size={14} color={theme.colors.primary} />
          </View>
```
Replace with:
```tsx
          <AvatarImage uri={item.passenger?.avatarUrl} size={28} iconSize={14} />
```
Remove the now-unused `avatarPlaceholder` style if not used elsewhere.

- [ ] **Step 4: Add `pendingOfferCount` chip to the card**

Inside `renderItem`, in the `badgeCol` View (which already contains `distanceBadge` and `offerSentBadge`), add below `offerSentBadge`:
```tsx
            {item.pendingOfferCount > 0 && (
              <View style={styles.offerCountBadge}>
                <Text style={styles.offerCountText}>
                  {item.pendingOfferCount} offer{item.pendingOfferCount !== 1 ? 's' : ''}
                </Text>
              </View>
            )}
```

Add styles:
```ts
  offerCountBadge: {
    backgroundColor: '#fef3c7',
    paddingHorizontal: 8,
    paddingVertical: 3,
    borderRadius: theme.borderRadius.full,
  },
  offerCountText: {
    fontSize: theme.fontSize.xs,
    fontWeight: '600' as const,
    color: '#92400e',
  },
```

- [ ] **Step 5: Add subscribe-to-alerts CTA in the filter row**

Add subscription state at the top of the component:
```ts
  const [subscribed, setSubscribed] = useState(false);
  const [subscribing, setSubscribing] = useState(false);
```

Load persisted subscription state on mount:
```ts
  useEffect(() => {
    storageService.getString('browseRequestsSubscribed').then(v => {
      setSubscribed(v === 'true');
    });
  }, []);
```

Note: `storageService.getString` — check if this method exists. If not, use `AsyncStorage.getItem` directly (import `AsyncStorage` from `@react-native-async-storage/async-storage`).

Add the subscribe toggle handler:
```ts
  const handleSubscribeToggle = useCallback(async () => {
    if (!driverLocation || subscribing) return;
    setSubscribing(true);
    try {
      const next = !subscribed;
      await functionsService.subscribeToRequestAlerts({
        enabled: next,
        latitude: driverLocation.latitude,
        longitude: driverLocation.longitude,
      });
      setSubscribed(next);
      await AsyncStorage.setItem('browseRequestsSubscribed', String(next));
    } catch {
      // fail silently — don't block the UI
    } finally {
      setSubscribing(false);
    }
  }, [driverLocation, subscribed, subscribing]);
```

In the `return` JSX, inside the filter row (`<View style={styles.filterRow}>`), add the bell button at the end of the filter pills:
```tsx
        <TouchableOpacity
          style={styles.subscribeBtn}
          onPress={handleSubscribeToggle}
          disabled={subscribing || !driverLocation}
          activeOpacity={0.7}
          accessibilityLabel={subscribed ? 'Unsubscribe from new request alerts' : 'Subscribe to new request alerts'}
        >
          <Ionicons
            name={subscribed ? 'notifications' : 'notifications-outline'}
            size={18}
            color={subscribed ? theme.colors.primary : theme.colors.text.tertiary}
          />
        </TouchableOpacity>
```

Add style:
```ts
  subscribeBtn: {
    marginLeft: 'auto',
    padding: theme.spacing.sm,
    justifyContent: 'center',
    alignItems: 'center',
  },
```

- [ ] **Step 6: Add AsyncStorage import if not present**

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';
```

- [ ] **Step 7: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors. If `storageService.getString` doesn't exist, use `AsyncStorage.getItem` directly.

- [ ] **Step 8: Commit**

```bash
git add components/BrowseRequestsList.tsx
git commit -m "feat(BrowseRequestsList): offer count chip, passenger avatar, subscribe CTA"
```

---

### Task 18: post-ride — remove dead commented-out code

**Files:**
- Modify: `Carpool/app/tabs/post-ride.tsx`

- [ ] **Step 1: Delete all commented-out code blocks**

Remove the following from `post-ride.tsx`:

1. The commented `import CheckBox from 'expo-checkbox'` line (~line 3)
2. The entire commented `{isTimeRange && (<View ...>...</View>)}` end-time picker block (~lines 505-517)
3. The commented `{showEndTimePicker && ...}` block (~lines 528-535)
4. The commented `<TouchableOpacity style={styles.checkboxContainer} ...>` block (~lines 537-544)

- [ ] **Step 2: Remove unused state and handlers**

Delete these declarations:
- `const [isTimeRange, setIsTimeRange] = useState(false);` (~line 87)
- `const [endTime, setEndTime] = useState<Date | null>(null);` (~line 89 — check usage)
- `const [showEndTimePicker, setShowEndTimePicker] = useState(false);` (~line 90)
- `const onEndTimeChange = ...` function body (~lines 275-288)

Check each one: if `endTime` or `showEndTimePicker` are referenced anywhere in the active (non-commented) code, keep them. If not, delete.

Also remove from `handleViewMap` the `isTimeRange` check and `endTime` references if they only appear in the deleted blocks.

- [ ] **Step 3: Remove unused styles**

Delete styles only used by the deleted code: `checkboxContainer`, `checkboxLabel`. Run:
```bash
grep -n "checkboxContainer\|checkboxLabel" Carpool/app/tabs/post-ride.tsx
```
If only in the style definition (no JSX references), delete them.

- [ ] **Step 4: Type-check**

```bash
cd Carpool && npx tsc --noEmit
```
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add app/tabs/post-ride.tsx
git commit -m "cleanup(post-ride): remove commented-out dead code and unused state"
```

---

## Self-Review

**Spec coverage:**
- ✅ 1.1 expireRideRequests MATCHED → Task 1
- ✅ 1.2 createRideRequest duplicate MATCHED → Task 2
- ✅ 1.3 cancelRideRequest TOCTOU sweep → Task 3
- ✅ 1.4 browseRideRequests pendingOfferCount → Task 4
- ✅ 1.5 browseRideRequests dateEnd param → Task 4
- ✅ functions.service.ts dateEnd → Task 6
- ✅ 2.1 stale ride guard → Task 8
- ✅ 2.2 pull-to-refresh fix → Task 9
- ✅ 2.3 as any cast → Task 10
- ✅ 2.4 week filter → Task 7
- ✅ 3.1 swap button → Task 12
- ✅ 3.2 always-visible link → Task 12
- ✅ 3.3 expires chip → Task 13
- ✅ 3.4 pull-to-refresh my-requests → Task 13
- ✅ 3.5 FULFILLED out of Active → Task 13
- ✅ 3.6 resubmit button → Tasks 12 + 13
- ✅ 3.7 status badge fix → Task 14
- ✅ 3.8 offer count chip → Tasks 4 + 14 + 17
- ✅ 3.9 avatars → Tasks 15 + 16 + 17
- ✅ 3.10 subscribe CTA → Task 17
- ✅ 3.11 cache invalidation → Task 7 (useFocusEffect)
- ✅ 3.12 dead code removal → Task 18
- ✅ 3.13 theme token → Task 11

**No gaps found.**

**Type consistency:** All references to `pendingOfferCount` added in Task 4 (CF) flow through to Task 6 (service type) and Task 17 (FE interface + render). `AvatarImage` created in Task 15, imported in Tasks 16 + 17. `resubmit*` params set in Task 13, read in Task 12.
