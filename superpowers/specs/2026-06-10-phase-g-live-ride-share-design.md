# Phase G — Live Ride Share Design

**Date:** 2026-06-10  
**Status:** Approved  
**Depends on:** Phase B (liveTracking service), Phase C (getTrackingData CF + public tracking page)

---

## 1. Goal

Let any user on an active ride (driver or confirmed passenger) share a public live-tracking link with anyone — family, friends — via the native share sheet. The recipient opens the link in a browser and sees the sharer's live location on the map built in Phase C.

---

## 2. User-Facing Behaviour

### Share flow
1. Ride status transitions to `STARTED` → "Share Live Ride" button appears (green) on `live-ride.tsx` (driver) and `booked-ride-details.tsx` (passenger).
2. User taps → `createRideShare({ rideId })` CF called.
3. CF returns `{ shareToken, trackingUrl }`.
4. FE starts `liveTrackingService.startTracking(shareToken, 'share')`.
5. Native share sheet opens with pre-built message:
   ```
   🚗 Track my ride live: https://carpool-app-1e668.web.app/track/{token}
   Driver: {driverName} · {vehiclePlate}
   ```
6. Button label changes to "Stop Sharing" (filled green).

### Stop Sharing flow
1. User taps "Stop Sharing" → `revokeRideShare({ shareToken })` CF called.
2. FE calls `liveTrackingService.stopTracking(shareToken)`.
3. Button returns to "Share Live Ride".

### Auto-revoke
- Firestore trigger fires when `rides/{rideId}.status` → `COMPLETED` or `CANCELLED`.
- All active `rideShares` for that ride are batch-revoked.
- RTDB `liveTracking/{token}` nodes are removed 60 s later.

---

## 3. Architecture

### Frontend

| File | Responsibility |
|---|---|
| `Carpool/components/ShareLiveRideButton.tsx` | Toggle button: idle → loading → sharing → stopping. Derives state from `useRideShare`. |
| `Carpool/hooks/useRideShare.ts` | Firestore listener on active share for `(rideId, userId)`. Exposes `{ activeShare, createShare, revokeShare, isLoading }`. |
| `Carpool/src/types/models/rideShare.types.ts` | `RideShare` type matching Firestore doc shape. |
| `Carpool/src/services/firebase/functions.service.ts` | Add `createRideShare` + `revokeRideShare` callable wrappers. |
| `Carpool/app/live-ride.tsx` | Mount `ShareLiveRideButton` when `rideStatus === 'STARTED'` (driver view). |
| `Carpool/app/booked-ride-details.tsx` | Mount `ShareLiveRideButton` when `booking.status === 'CONFIRMED' && rideStatus === 'STARTED'` (passenger view). |

### Backend

| File | Responsibility |
|---|---|
| `createRideShare.fn.ts` | Callable CF — idempotent create, cap enforcement, token generation. |
| `revokeRideShare.fn.ts` | Callable CF — status update, RTDB cleanup. |
| `onRideComplete.fn.ts` | Firestore onCreate/onUpdate trigger — auto-revoke on ride COMPLETED/CANCELLED. |
| `rideShareCleanup.fn.ts` | Hourly scheduled — delete stale RTDB `liveTracking` nodes. |

---

## 4. Data Model

### Firestore: `rideShares/{shareToken}`

```ts
{
  shareToken: string,          // 24-byte base64url random, used as doc ID
  rideId: string,
  userId: string,              // sharer's UID
  type: 'share',
  status: 'ACTIVE' | 'REVOKED' | 'EXPIRED',
  driverName: string,          // first name + last initial
  vehiclePlate: string,
  vehicleModel: string,
  passengerName: string,       // sharer's display name (first + last initial)
  createdAt: Timestamp,
  expiresAt: Timestamp,        // createdAt + 6 h (hard ceiling; trigger updates on COMPLETED)
  viewerCount: number,         // incremented by getTrackingData CF (Phase C)
  revokedAt?: Timestamp,
}
```

### RTDB: `rideSharesByToken/{shareToken}`
Index node written by `createRideShare`, read by RTDB rules to authorise `liveTracking` writes:
```json
{ "userId": "<uid>" }
```

---

## 5. `createRideShare` CF

**Type:** Callable (requires auth)  
**Input:** `{ rideId: string }`

**Logic:**
1. Authenticate + rate-limit (`CREATE_RIDE_SHARE: 20/hour`).
2. Read ride doc — verify `status === 'STARTED'` and caller is driver or confirmed passenger.
3. Check for existing active share: `rideShares where rideId == rideId && userId == uid && status == 'ACTIVE'` — if found, return it (idempotent).
4. Firestore transaction: count active shares for this user (`rideShares where userId == uid && status == 'ACTIVE'`). If ≥ 3, throw `LIMIT_EXCEEDED`.
5. Generate `shareToken = crypto.randomBytes(24).toString('base64url')`.
6. Write `rideShares/{shareToken}` with `status: 'ACTIVE'`, `expiresAt: now + 6h`, driver/passenger name snapshot.
7. Write RTDB `rideSharesByToken/{shareToken} = { userId: uid }`.
8. Return `{ shareToken, trackingUrl: 'https://carpool-app-1e668.web.app/track/{shareToken}', driverName, vehiclePlate, passengerName }`.

---

## 6. `revokeRideShare` CF

**Type:** Callable (requires auth)  
**Input:** `{ shareToken: string }`

**Logic:**
1. Authenticate + rate-limit (`REVOKE_RIDE_SHARE: 50/hour`).
2. Read `rideShares/{shareToken}` — verify `userId === caller.uid` and `status === 'ACTIVE'`.
3. Update `status: 'REVOKED'`, `revokedAt: now`.
4. Schedule RTDB cleanup: after 60 s grace, delete `liveTracking/{shareToken}` and `rideSharesByToken/{shareToken}` (use a `setTimeout` inside the CF — acceptable for a single cleanup write).

---

## 7. `onRideComplete` Firestore Trigger

**Trigger:** `onDocumentUpdated('rides/{rideId}')` — fires when `status` changes to `COMPLETED` or `CANCELLED`.

**Logic:**
1. Check `before.status !== after.status` and `after.status ∈ ['COMPLETED', 'CANCELLED']`.
2. Query `rideShares where rideId == rideId && status == 'ACTIVE'`.
3. Batch-update all to `status: 'REVOKED', revokedAt: now`.
4. For each revoked token: delete `rideSharesByToken/{token}` and `liveTracking/{token}` from RTDB.
5. **Idempotent** — check `status !== 'ACTIVE'` before updating to avoid double-processing.

---

## 8. `rideShareCleanup` Scheduled Function

**Schedule:** Every hour (`every 60 minutes`).

**Logic:**
1. Query `rideShares where expiresAt < now && status == 'ACTIVE'` (any that slipped past the trigger).
2. Batch-update to `status: 'EXPIRED'`.
3. Delete corresponding `liveTracking/{token}` and `rideSharesByToken/{token}` RTDB nodes.

---

## 9. `ShareLiveRideButton` Component

**States:**
- `idle` — ride not STARTED or no share active → green outlined "Share Live Ride" button
- `loading` — CF call in flight → spinner, disabled
- `sharing` — active share exists → green filled "Stop Sharing" button
- `stopping` — revoke in flight → spinner, disabled

**Behaviour:**
- Tapping `idle` → call `createShare()` → open `Share.share({ message })` if successful
- Tapping `sharing` → call `revokeShare()` with `activeShare.shareToken`
- If `createShare` throws `LIMIT_EXCEEDED` → `showError("You already have 3 active ride shares. Stop one before sharing again.")`
- Location permission: if `liveTrackingService.startTracking` throws → `showError(err)`, share token already created (user can retry or revoke)

**Accessibility:**
- `accessibilityRole="button"`
- `accessibilityLabel` changes with state: "Share live ride", "Stop sharing live ride"

---

## 10. `useRideShare` Hook

```ts
interface UseRideShareReturn {
  activeShare: RideShare | null;
  isLoading: boolean;
  createShare: () => Promise<void>;
  revokeShare: () => Promise<void>;
}
```

- Firestore listener: `rideShares where rideId == rideId && userId == uid && status == 'ACTIVE'` (direct read, no CF)
- `createShare()`: calls `functionsService.createRideShare({ rideId })` → starts `liveTrackingService.startTracking(token, 'share')` → opens `Share.share`
- `revokeShare()`: calls `functionsService.revokeRideShare({ shareToken })` → calls `liveTrackingService.stopTracking(token)`
- Unmount guard: cancel Firestore listener on unmount

---

## 11. Firestore Rules

```
match /rideShares/{shareToken} {
  allow read: if isAuthenticated() && resource.data.userId == request.auth.uid;
  allow write: if false; // createRideShare / revokeRideShare CFs only
}
```

---

## 12. Constants + Validation

**Rate limits (add to `RATE_LIMITS`):**
```ts
CREATE_RIDE_SHARE: 20,
REVOKE_RIDE_SHARE: 50,
```

**Validation schemas:**
```ts
createRideShare: z.object({ rideId: z.string().min(1) }),
revokeRideShare: z.object({ shareToken: z.string().min(1) }),
```

**Cap constant (add to `CONSTANTS`):**
```ts
RIDE_SHARE: {
  MAX_ACTIVE_PER_USER: 3,
  MAX_PER_DAY: 20,
  EXPIRY_MS: 6 * 60 * 60 * 1000,  // 6 hours hard ceiling
  RTDB_CLEANUP_GRACE_MS: 60_000,
},
```

---

## 13. Tests

### Backend unit tests
- `createRideShare`: ride not STARTED → error; caller not driver/passenger → error; idempotent return on existing active share; cap-3 enforcement under concurrent calls; success path returns correct shape
- `revokeRideShare`: wrong owner → error; already revoked → error; success path
- `onRideComplete`: ride COMPLETED → all active shares revoked; CANCELLED → same; already revoked → no double update
- `rideShareCleanup`: expired shares batch-revoked, RTDB nodes deleted

### Frontend unit tests
- `ShareLiveRideButton`: renders idle when no active share; renders sharing when share active; cap exceeded error shown; loading state during CF call
- `useRideShare`: Firestore listener wired; createShare calls service + tracking + share sheet; revokeShare calls service + stops tracking

---

## 14. Deploy

```bash
firebase deploy --only functions:createRideShare,functions:revokeRideShare,functions:onRideComplete,functions:rideShareCleanup,firestore:rules
```
