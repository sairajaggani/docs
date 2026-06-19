# Browse Rides / Request Ride / My Requests — Bug Fixes + UX Improvements

**Date**: 2026-06-14  
**Scope**: `searchRides` · `browseRideRequests` · `createRideRequest` · `cancelRideRequest` · `expireRideRequests` · `need-ride.tsx` · `my-requests.tsx` · `request-details.tsx` · `request-details-passenger.tsx` · `BrowseRequestsList.tsx` · `post-ride.tsx` · `constants/theme.ts`

---

## Overview

Three-part delivery: backend data bugs first (one CF deploy), then FE data bugs (one PR), then FE UX polish (one PR).

---

## Part 1 — Backend Bug Fixes

### 1.1 `expireRideRequests.fn.ts` — expire MATCHED requests too

**Problem**: only `PENDING` requests are expired. MATCHED requests stay MATCHED forever after 7 days, showing stale data to passengers.

**Change**: query `where("status", "in", ["PENDING", "MATCHED"])`. All existing batch-expire and cascade-offers logic is unchanged.

**Deploy**: `firebase deploy --only functions:expireRideRequests`

---

### 1.2 `createRideRequest.fn.ts` — duplicate check covers MATCHED

**Problem**: a passenger with a MATCHED request for route X on date Y can create another PENDING request for the same route+date.

**Change**: duplicate check query changes from `where("status", "==", "PENDING")` to `where("status", "in", ["PENDING", "MATCHED"])`.

**Deploy**: `firebase deploy --only functions:createRideRequest`

---

### 1.3 `cancelRideRequest.fn.ts` — close TOCTOU on offer cancellation

**Problem**: pending offers are queried before the transaction. Offers added between that query and the commit are not cancelled.

**Change**: after the transaction commits, run a second `where("requestId", "==", requestId).where("status", "==", "PENDING")` sweep and cancel any remaining offers (idempotent). The transaction itself is unchanged (keeps its atomic read+write pattern).

**Deploy**: `firebase deploy --only functions:cancelRideRequest`

---

### 1.4 `browseRideRequests.fn.ts` — add `pendingOfferCount` to response

**Problem**: drivers browsing requests have no idea how many offers have already been sent, so they cannot price competitively.

**Change**: the CF already queries PENDING offers per page chunk for `driverOfferSet`. Extend this to also count pending offers per request and include `pendingOfferCount: number` in each response item.

**Deploy**: `firebase deploy --only functions:browseRideRequests`

---

### 1.5 `browseRideRequests.fn.ts` — accept `dateEnd` param for week filter

**Problem**: FE "This Week" filter sends no date, CF defaults to 14 days — identical to "All".

**Change**: add optional `dateEnd: string` to the CF Zod schema. When provided, use it as the `requestedDate <=` bound. FE passes `+7 days` for week filter.

**Deploy**: same deploy as 1.4.

---

## Part 2 — Frontend Data Bug Fixes

### 2.1 `my-requests.tsx` — guard stale "View Matched Ride"

**Problem**: MATCHED requests link to rides that may be COMPLETED/CANCELLED. Navigation leads to a confusing dead-end.

**Change**: for MATCHED cards, call `firestoreService.getDocument('rides', rideId)` on mount and check the ride's `status`. If not OPEN or STARTED, replace the button with a muted label: "Matched ride ended — waiting for re-match." Keep the cancel button active so the user can bail.

---

### 2.2 `need-ride.tsx` — fix pull-to-refresh spinner

**Problem**: `refreshing={false}` is hardcoded — the spinner never shows.

**Change**: add `const [isRefreshing, setIsRefreshing] = useState(false)`. Wrap `onRefresh` callback: `setIsRefreshing(true)` → `await refresh()` → `setIsRefreshing(false)` in finally. Pass `isRefreshing` to `<RefreshControl refreshing={isRefreshing}>`.

---

### 2.3 `request-details-passenger.tsx` — remove `as any` cast

**Problem**: `(item.driver as any).rating` suppresses a TypeScript error.

**Change**: check `DriverSnapshot` type in `src/types/models/`. Add `rating: number` if missing. Remove the cast.

---

### 2.4 `BrowseRequestsList.tsx` + `browseRideRequests` schema — fix "This Week" filter

**Problem**: `getDateParam('week')` returns `undefined`, identical to `'all'`.

**Change**: 
- FE: `getDateParam('week')` returns `{ date: todayStr, dateEnd: +7daysStr }`.
- CF: accept `dateEnd?: string` in schema; use it as the upper bound for `requestedDate`.
- Also update the label to "7 Days" to match what it actually shows.

---

## Part 3 — Frontend UX Polish

### 3.1 Swap button (`need-ride.tsx`)

Icon (`swap-vertical`) between from/to input fields. On press: swap both display strings and `SearchLocation` objects simultaneously. No API call needed.

---

### 3.2 Always-visible My Requests link (`need-ride.tsx`)

Remove `{!searching && ...}` wrapper from the `myRequestsLink` TouchableOpacity. Link stays visible during search and results.

---

### 3.3 Expires-in chip (`my-requests.tsx`)

Compute `daysLeft = Math.ceil((item.expiresAt.toMillis() - Date.now()) / DAY_MS)` for PENDING/MATCHED cards. Show a small chip: "Expires in Xd" (gray when > 1 day, orange `theme.colors.warning` when ≤ 1 day, red when 0).

---

### 3.4 Pull-to-refresh (`my-requests.tsx`)

Add `<RefreshControl>` to the FlatList. Handler: unsubscribes and re-subscribes the Firestore listener (forces a fresh snapshot).

---

### 3.5 Move FULFILLED out of Active tab (`my-requests.tsx`)

Active filter: `PENDING | MATCHED` only (remove FULFILLED). All tab: `PENDING | MATCHED | FULFILLED | EXPIRED` (remove only CANCELLED from this tab, or show everything). FULFILLED cards in All tab get a green ✓ "Fulfilled" chip instead of action buttons.

---

### 3.6 Resubmit for expired requests (`my-requests.tsx`)

EXPIRED cards: add a "Resubmit" button. On press: navigate to `/` (need-ride tab) with pre-filled params `fromAddress`, `fromLat`, `fromLng`, `toAddress`, `toLat`, `toLng`, `seatsNeeded`, `maxPrice` — same deep-link pattern used elsewhere. Pre-opens the request modal.

---

### 3.7 Status badge fix (`request-details.tsx`)

Replace the ad-hoc badge styles with a `STATUS_CONFIG` map (mirroring `my-requests.tsx`). Entries for PENDING, MATCHED, FULFILLED, EXPIRED, CANCELLED with correct bg/fg. Text color is taken from the config rather than hardcoded as primary.

---

### 3.8 Competing offer count chip (`request-details.tsx` + `BrowseRequestsList.tsx`)

- CF response now includes `pendingOfferCount` (from 1.4).
- Browse card: add a small chip "X offers" next to the distance badge.
- Request details screen: add a `pendingOfferCount` line in the Details section, e.g. "2 offers already sent".

---

### 3.9 Actual avatars (`request-details-passenger.tsx` + `BrowseRequestsList.tsx`)

Replace `<View style={avatarCircle}><Ionicons name="person"/></View>` with a helper `<AvatarImage uri={avatarUrl} size={44} />` that renders `<Image>` when uri is non-empty, falls back to the icon. Reusable across both screens.

---

### 3.10 Subscribe-to-alerts CTA (`BrowseRequestsList.tsx`)

Add a bell icon button in the filter row header. On press: call `functionsService.subscribeToRequestAlerts({ from: driverLocation, enabled: true })`. Show a filled bell when already subscribed (persist to AsyncStorage to avoid re-querying).

---

### 3.11 Cache invalidation on re-focus (`BrowseRequestsList.tsx`)

Add `useFocusEffect`. On focus: check a module-level `browseRequestsCacheInvalid` flag (set to `true` by `request-details` after a successful offer send). If set, call `fetchRequests(undefined, false, true)` (skipCache=true) and clear the flag.

---

### 3.12 Dead code removal (`post-ride.tsx`)

Delete: commented `isTimeRange` blocks, `endTime` state + handler, `showEndTimePicker` state + handler, `onEndTimeChange`, the `CheckBox` import comment, unused `isTimeRange`/`setIsTimeRange` state. No behaviour change.

---

### 3.13 Theme token for destination purple (`constants/theme.ts` + all files)

Add `destination: '#A855F7'` to `theme.colors`. Replace all 5+ hardcoded `#A855F7` occurrences in: `need-ride.tsx`, `my-requests.tsx`, `request-details.tsx`, `request-details-passenger.tsx`, `BrowseRequestsList.tsx`.

---

## Data Flow

No new Firestore collections or RTDB paths. No schema migrations. All changes are additive (new CF response field) or narrowing (stricter expiry query).

---

## Error Handling

- 2.1 stale ride check: if the Firestore read fails, fall back to showing the button normally (don't block the user).
- 3.10 subscribe: wrap in try/catch with `showError`; toast success on subscribe.
- 3.11 cache invalidation: if re-fetch fails, show stale data + refresh control so user can retry manually.

---

## Deploy Order

1. Deploy Part 1 CFs: `firebase deploy --only functions:expireRideRequests,functions:createRideRequest,functions:cancelRideRequest,functions:browseRideRequests`
2. Deploy Part 2 FE (after CF deploy clears Firestore of stale MATCHED requests — can be same day)
3. Deploy Part 3 FE (can be immediately after Part 2 or in same PR)

---

## Out of Scope

- Server-side SMS/WhatsApp for SOS (covered in backend.md)
- QR code OTP verification (covered in live-ride redesign)
- New Firestore indexes (not needed — all new queries use existing indexed fields)
- Maestro E2E tests (Phase 5)
