# ZygoRIde Backend — Firebase Cloud Functions

**Runtime:** Node.js 18, TypeScript, Firebase Functions v1 (`firebase-functions`)
**Source:** `function/functions/src/`

---

## Middleware Stack

Every callable function passes through this middleware chain (applied via `withErrorHandler` + `withLogging` wrappers):

| Middleware | File | Purpose |
|---|---|---|
| Error Handler | `error-handler.middleware.ts` | Catches all errors; maps `FunctionError` to structured response |
| Logging | `logging.middleware.ts` | Logs function name, uid, duration |
| Auth | `auth.middleware.ts` | Verifies Firebase ID token; reads admin status from `admins/{uid}` Firestore doc; exposes `extendedAuth: { uid, email, emailVerified, isAdmin }` |
| CORS | `cors.middleware.ts` | CORS headers for HTTP endpoints |
| Validation | `validation.middleware.ts` | Zod schema validation; throws `INVALID_ARGUMENT` on failure |
| Rate Limit | `rate-limit.middleware.ts` | Per-user sliding window using RTDB `rateLimits/{uid}/{functionName}` (atomic transaction); throws `RESOURCE_EXHAUSTED` when exceeded |
| Idempotency | `idempotency.middleware.ts` | Prevents duplicate bookings and payments using a client-supplied idempotency key |

Admin identity is checked by reading `admins/{uid}` in Firestore — does **not** use custom claims.

---

## Rate Limits (per hour per user)

| Function | Limit |
|---|---|
| createProfile | 5 |
| updateProfile | 10 |
| uploadAvatar | 10 |
| removeAvatar | 10 |
| uploadID | 3 |
| createRide | 20 |
| updateRide | 30 |
| cancelRide | 10 |
| searchRides | 60 |
| createBooking | 30 |
| confirmBooking | 50 |
| cancelBooking | 20 |
| submitRating | 10 |
| createRideRequest | 10 |
| cancelRideRequest | 10 |
| getDirections (maps) | 100 |
| sendMessage | 60 |
| markRead | 120 |
| blockUser | 20 |
| deleteChat | 20 |
| startRide | 10 |
| verifyPassengerOtp | 50 |
| completeRide | 10 |
| completeBooking | 20 |
| mapsGeocode | 50 |
| mapsAutocomplete | 100 |
| submitSupportTicket | 5 |
| browseRideRequests | 60 |
| sendRideOffer | 20 |
| acceptRideOffer | 20 |
| rejectRideOffer | 30 |
| withdrawRideOffer | 20 |
| subscribeRequestAlerts | 10 |
| verifyIDDocument (admin) | 50 |
| getVerificationQueue (admin) | 60 |
| suspendUser (admin) | 20 |
| getSupportTickets (admin) | 60 |
| resolveTicket (admin) | 50 |
| adminSearchUsers | 60 |
| getDashboardStats | 30 |

---

## Functions by Module

### Auth — `functions/auth/`

#### `onCreate` (Firestore trigger)

**Trigger:** Firebase Auth `user.onCreate`
**What it does:** Automatically creates a skeleton `profiles/{uid}` document when a new Firebase Auth user is created. Sets default `accountStatus: active`, empty stats, empty vehicles array.
**Side effects:** None (no notifications).

---

### Profiles — `functions/profiles/`

#### `createProfile`

**Auth:** Required, email verified
**Inputs:** `firstName`, `lastName`, `phone?`, `preferences?`
**What it does:** Creates or completes the profile document in `profiles/{uid}`. Validates uniqueness constraints. Updates `driverStats`, `passengerStats`, `vehicles` with defaults.
**Side effects:** None.

#### `updateProfile`

**Auth:** Required, email verified
**Inputs:** Subset of profile fields (name, phone, language, notification preferences)
**What it does:** Partial-updates `profiles/{uid}`. Rejects attempts to change uid/email via this function.
**Side effects:** None.

#### `uploadAvatar`

**Auth:** Required
**Inputs:** `base64Image`, `mimeType`
**What it does:** Saves image to Storage `avatars/{uid}`, updates `profiles/{uid}.avatarUrl`.
**Side effects:** Deletes previous avatar from Storage if one existed.

#### `removeAvatar`

**Auth:** Required
**What it does:** Deletes avatar from Storage and clears `avatarUrl` on profile.

#### `uploadID`

**Auth:** Required, email verified
**Inputs:** `base64Image`, `mimeType`
**What it does:** Saves ID document to Storage `id-documents/{uid}` (encrypted blob path stored as `idStoragePath`). Sets `idVerificationStatus: PENDING`. Does **not** store the Aadhaar number.
**Side effects:** Sends `ID_VERIFICATION` notification to admin queue.

#### `addVehicle`

**Auth:** Required, email verified
**Inputs:** `{ make, model, plate, color, year? }`
**What it does:** Appends vehicle to `profiles/{uid}.vehicles` array. Max 4 vehicles per user.

#### `removeVehicle`

**Auth:** Required
**Inputs:** `plate`
**What it does:** Removes vehicle with matching plate from `profiles/{uid}.vehicles`.

#### `updateFcmToken`

**Auth:** Required
**Inputs:** `{ token, action: 'register' | 'unregister' }`
**What it does:** Adds or removes FCM token from `profiles/{uid}.fcmTokens` array.

---

### Rides — `functions/rides/`

#### `createRide`

**Auth:** Required, email verified
**Inputs:** `from`, `to` (Location), `rideDate`, `rideStartTime`, `price`, `seats`, `vehicleIndex`, `notes?`, `route` (RouteInfo), `plusCodes`
**What it does:**
- Validates profile completeness (firstName, lastName, at least 1 vehicle)
- Checks for time overlaps with existing OPEN/STARTED rides
- Encodes from/to coordinates to H3 cells at resolution 7 and 8 (`searchIndex`)
- Parses city names from Plus Code compound strings
- Creates `rides/{rideId}` with status `OPEN`
- Checks `rideRequests` for matching passengers (H3 cell overlap + date match) → triggers `onRideRequestMatched` side-effect
**Side effects:** Notifies matching ride request owners (`RIDE_MATCHED`), updates their request status to `MATCHED`.

#### `updateRide`

**Auth:** Required (ride owner)
**Inputs:** `rideId`, subset of updatable fields (price, seats, notes, time, route)
**What it does:** Validates ownership, checks for conflicts, updates ride doc.
**Side effects:** Sends `RIDE_UPDATED` notification to all confirmed passengers.

#### `cancelRide`

**Auth:** Required (ride owner)
**Inputs:** `rideId`, `reason?`
**What it does:** Sets ride to `CANCELLED`. Cancels all `PENDING` and `CONFIRMED` bookings. Refunds booked seats.
**Side effects:** Sends `RIDE_CANCELLED` notification to all affected passengers. Sends `ALTERNATE_RIDES_SUGGESTED` if `getSuggestedRides` finds nearby alternatives.

#### `searchRides`

**Auth:** Required
**Inputs:** `from`, `to` (with H3/plusCode), `date?`, `seats?`, `sortBy`, `cursor?`, `limit`
**What it does:** Queries `rides` Firestore collection by H3 cell overlap at resolution 7 (broad) then 8 (tight), filtered by date and available seats. Applies requested sort (relevance / price / date / seats). Returns paginated `SearchRideItem[]` with a `nextCursor`.
**Side effects:** None (read-only). Results cached 15 min client-side.

#### `getRideDetails`

**Auth:** Required
**Inputs:** `rideId`, `seatsRequested?`
**What it does:** Fetches full `Ride` doc plus booking count for the requesting user.

#### `startRide`

**Auth:** Required (ride owner), email verified
**Inputs:** `rideId`
**What it does:**
- Sets ride `status = STARTED`, `actualStartTime = now`
- Generates a 4-digit OTP for each confirmed booking; writes `pickupOtp` + `joinedAt` to each booking doc
- Writes `activeRides/{rideId}` to RTDB: `meta` (driverId, confirmedPassengerIds), `pickupStops` (sorted nearest-first using haversine greedy algorithm)
**Side effects:** Sends `RIDE_STARTED` FCM to all confirmed passengers.

#### `verifyPassengerOtp`

**Auth:** Required (ride owner)
**Inputs:** `rideId`, `passengerId`, `otp`
**What it does:** Matches OTP from `bookings/{id}.pickupOtp`. Updates RTDB `pickupStops/{passengerId}.status = PICKED_UP`.

#### `completeRide`

**Auth:** Required (ride owner)
**Inputs:** `rideId`
**What it does:**
- Sets ride `status = COMPLETED`, `actualEndTime = now`
- Sets all `CONFIRMED` bookings to `COMPLETED`
- Deletes RTDB `activeRides/{rideId}`
- Updates driver `driverStats.totalRides++`, `completionRate`
- Updates each passenger `passengerStats.totalRides++`
**Side effects:** Sends `RIDE_COMPLETED` to passengers. Sends `RATING_REMINDER` to both driver and each passenger.

#### `cancelRideRequest`

**Auth:** Required (request owner)
**Inputs:** `requestId`
**What it does:** Sets `rideRequests/{id}.status = CANCELLED`.

#### `createRideRequest`

**Auth:** Required, email verified
**Inputs:** `from`, `to`, `requestedDate`, `flexibleDates`, `dateRange?`, `seatsNeeded`, `maxPrice?`, `preferredDepartureTime?`
**What it does:** Creates `rideRequests/{id}`. Encodes H3 cells. Sets `expiresAt` 30 days out. Checks existing rides for immediate match.

#### `onRideRequestMatched` (internal)

Called by `createRide` when new ride overlaps existing pending requests. Updates matched request status to `MATCHED`. Sends `NEW_RIDE_REQUEST_NEARBY` to drivers subscribed via `requestAlertSubscriptions`.

#### `getSuggestedRides`

**Auth:** Required
**Inputs:** `from`, `to`, `date?`
**What it does:** Returns nearby alternate rides (looser H3 matching) for the "no results" empty state.

---

### Bookings — `functions/bookings/`

#### `createBooking`

**Auth:** Required, email verified
**Inputs:** `rideId`, `seatsRequested`, `idempotencyKey`
**What it does:**
- Idempotency check (prevents double-booking)
- Validates ride is OPEN, has enough seats, user is not already booked
- Creates `bookings/{bookingId}` with status `PENDING`
- Decrements `availableSeats` on ride
**Side effects:** Sends `BOOKING_REQUEST` notification to driver.

#### `confirmBooking`

**Auth:** Required (ride owner)
**Inputs:** `bookingId`
**What it does:** Sets booking to `CONFIRMED`. Adds passengerId to `ride.confirmedPassengerIds`.
**Side effects:** Sends `BOOKING_CONFIRMED` notification to passenger.

#### `cancelBooking`

**Auth:** Required (driver or passenger)
**Inputs:** `bookingId`, `reason?`
**What it does:** Sets booking to `CANCELLED`. Restores `availableSeats` on ride if within 2-hour window.
**Side effects:** Sends `BOOKING_CANCELLED` notification to the other party.

#### `completeBooking`

**Auth:** Required (ride owner)
**Inputs:** `bookingId`
**What it does:** Sets individual booking to `COMPLETED` (used if driver completes per-passenger).

---

### Ratings — `functions/ratings/`

#### `submitRating`

**Auth:** Required, email verified
**Inputs:** `rideId`, `bookingId`, `revieweeId`, `role` (DRIVER_TO_PASSENGER | PASSENGER_TO_DRIVER), `rating` (1–5), `review`, `tags?`
**What it does:**
- Validates rating window (7 days from ride completion)
- Creates `ratings/{ratingId}`
- Recalculates and updates target user's `driverStats.rating` / `passengerStats.rating` (weighted average over `reviewsCount`)
**Side effects:** Sends `RATING_RECEIVED` notification to reviewee.

---

### Maps — `functions/maps/`

#### `getDirections`

**Auth:** Required
**Inputs:** `origin`, `destination` (lat/lng)
**What it does:** Proxies Google Directions API. Caches result in `mapsCache/{hash}` Firestore collection (TTL enforced by Firestore TTL policy on `expireAt` field). Returns `{ distance, distanceValue, duration, durationValue, polyline }`.

---

### Messaging — `functions/messaging/`

#### `sendMessage`

**Auth:** Required
**Inputs:** `chatId`, `receiverId`, `text`, `rideId?`
**What it does:** Creates `chats/{chatId}/messages/{msgId}` doc. Creates or updates the `chats/{chatId}` parent doc (participants, lastMessage, lastMessageAt, unreadCounts). Validates message length ≤ 500 chars.
**Side effects:** Sends `NEW_MESSAGE` FCM to receiver (unless they have the chat open).

#### `markRead`

**Auth:** Required
**Inputs:** `chatId`
**What it does:** Zeroes `chats/{chatId}.unreadCounts[uid]`. (Can also be done client-side via direct Firestore write per security rules.)

#### `blockUser`

**Auth:** Required
**Inputs:** `targetUid`
**What it does:** Adds `targetUid` to `profiles/{uid}.blockedUsers`. Prevents future messages between the pair.

#### `deleteChat`

**Auth:** Required
**Inputs:** `chatId`
**What it does:** Soft-deletes chat for the requesting user by writing `chats/{chatId}.deletedBy[uid] = now`. Chat reappears if a new message arrives after the deletion timestamp.

---

### Offers — `functions/offers/`

#### `browseRideRequests`

**Auth:** Required
**Inputs:** `cursor?`, `limit?`, `maxDistanceKm?`
**What it does:** Returns paginated `rideRequests` with `PENDING` status. Filters by haversine distance from driver's location if `maxDistanceKm` supplied. Includes denormalized `passengerSnapshot`.

#### `sendRideOffer`

**Auth:** Required (driver), email verified
**Inputs:** `requestId`, `price`, `seatsOffered`, `message`
**What it does:** Creates `rideOffers/{offerId}`. Max 5 offers per request; max 10 active offers per driver. Sets `status = PENDING`.
**Side effects:** Sends `NEW_RIDE_OFFER` notification to passenger.

#### `acceptRideOffer`

**Auth:** Required (passenger / request owner)
**Inputs:** `offerId`
**What it does:** Sets offer to `ACCEPTED`. Auto-creates a `rides` doc for the driver and a `bookings` doc for the passenger. Auto-rejects all other pending offers on the same request with `AUTO_REJECTED`. Updates request status to `FULFILLED`.
**Side effects:** Sends `RIDE_OFFER_ACCEPTED` to driver. Sends `RIDE_OFFER_AUTO_REJECTED` to other drivers.

#### `rejectRideOffer`

**Auth:** Required (passenger)
**Inputs:** `offerId`, `reason?`
**What it does:** Sets offer to `REJECTED`.
**Side effects:** Sends `RIDE_OFFER_REJECTED` to driver.

#### `withdrawRideOffer`

**Auth:** Required (driver)
**Inputs:** `offerId`
**What it does:** Sets offer to `WITHDRAWN` if still `PENDING`.

#### `subscribeToRequestAlerts`

**Auth:** Required (driver)
**Inputs:** `h3Cells_7: string[]`, `enabled: boolean`
**What it does:** Creates or updates `requestAlertSubscriptions/{driverId}` with the driver's monitored H3 cells. Used to notify drivers of new ride requests in their area.

---

### Admin — `functions/admin/`

All admin functions require `isAdmin = true` (verified via `admins/{uid}` Firestore doc).

#### `getDashboardStats`

Returns: `totalUsers`, `suspendedUsers`, `openRides`, `activeRides`, `pendingVerifications`, `openTickets`, `bookingsLast7Days`.

#### `adminSearchUsers`

**Inputs:** `query`, `field` (email | phone | uid), `limit?`
**What it does:** Searches `profiles` collection by the specified field. Returns `UserResult[]`.

#### `getVerificationQueue`

**Inputs:** `status?` (pending | approved | rejected | all), `cursor?`, `limit?`
**What it does:** Returns paginated `VerificationItem[]` from profiles where `idVerificationStatus` matches.

#### `verifyIDDocument`

**Inputs:** `targetUid`, `action` (approve | reject), `reason?`
**What it does:** Sets `profiles/{targetUid}.idVerified = true/false`, `idVerificationStatus = APPROVED/REJECTED`. Logs action in `adminActions` collection.
**Side effects:** Sends `ID_VERIFICATION` notification to user.

#### `decryptIDDocument`

**Inputs:** `userId`, `adminPassword`
**What it does:** Verifies admin password, generates a short-lived signed GCS URL for the encrypted ID document.
**Returns:** `{ signedUrl, expiresAt, userName, idVerified }`

#### `suspendUser`

**Inputs:** `targetUid`, `action` (suspend | unsuspend | delete), `reason?`
**What it does:** Updates `profiles/{targetUid}.accountStatus`. On suspend: cancels all open rides and bookings. Logs to `adminActions`.
**Side effects:** Sends `ACCOUNT_SUSPENDED` or `ACCOUNT_RESTORED` notification.

#### `getSupportTickets`

**Inputs:** `status?`, `category?`, `cursor?`, `limit?`
**What it does:** Returns paginated `SupportTicket[]` from `supportTickets` collection.

#### `resolveTicket`

**Inputs:** `ticketId`, `resolution`, `adminNotes?`
**What it does:** Sets ticket `status = RESOLVED`, writes resolution + resolvedBy + resolvedAt.

---

### Support (mobile-facing) — `functions/support/`

#### `submitSupportTicket`

**Auth:** Required
**Inputs:** `category`, `description`, `screenshotUrls?`, `platform`, `appVersion?`
**What it does:** Creates `supportTickets/{ticketId}` with user snapshot, platform info, status `OPEN`.

---

### Scheduled Functions

| Function | Schedule | What it does |
|---|---|---|
| `scheduledCleanup` | Every 24 hours | Purges expired RTDB rate limit entries and idempotency keys from RTDB. Firestore `mapsCache` TTL is handled by Firestore's native TTL policy. |
| `expireRideRequests` | Every 12 hours | Batch-updates `PENDING` rideRequests past their `expiresAt` to `EXPIRED` status. |
| `expireStaleRides` | Every 12 hours | Batch-updates `OPEN` rides whose `rideStartTime` is in the past to `EXPIRED` status. |
| `rideReminders` | Every 30 minutes | Queries OPEN rides departing within the next hour; sends `RIDE_REMINDER` FCM to driver and all confirmed passengers. |
| `ratingReminders` | Every 24 hours | Queries completed rides with unrated bookings within the 7-day rating window; sends `RATING_REMINDER` FCM. |
| `archiveOldData` | Every Monday 02:00 | Moves COMPLETED/CANCELLED rides and bookings older than 60 days to archive subcollections. Deletes expired rideRequests and old notifications. |
| `autoCompleteStaleRides` | Every 12 hours | Auto-completes STARTED rides where `rideStartTime` is more than 24 hours past (handles driver forgot to complete). |

---

## Services

### `notification.service.ts`

`NotificationService.sendToUser(uid, title, body, data?, type?)`:
- Reads `profiles/{uid}.fcmTokens`
- Sends FCM multicast via `messaging.sendEachForMulticast`
- Prunes stale/invalid tokens from the profile after each batch send
- Sets Android `channelId` based on notification type; iOS `apns.payload.aps.sound = 'default'`

### `image.service.ts`

Image processing and signed URL generation for ID documents. Handles base64 decoding, GCS upload, and signed URL creation for admin review.

### `profile.service.ts`

`ProfileService.getProfile(uid)` — fetches and validates a profile document. Used internally by functions that need driver/passenger snapshots.

---

## Firestore Security Rules

| Collection | Who can read | Who can write | Notes |
|---|---|---|---|
| `profiles/{userId}` | Any authenticated user | Nobody (Cloud Functions only) | Broad read for ride bookings; all writes via CF |
| `rides/{rideId}` | Public (unauthenticated) | Nobody (CF only) | Public read for search |
| `bookings/{bookingId}` | Authenticated passenger or driver | Nobody (CF only) | Restricted to the two parties |
| `ratings/{ratingId}` | Public | Authenticated (reviewer == uid) | Reviewer field validated at create |
| `notifications/{notificationId}` | Owner (uid match) | Owner (mark read, delete) | Created by CF |
| `rideRequests/{requestId}` | Any authenticated user | Nobody (CF only) | Drivers browse passenger requests |
| `rideOffers/{offerId}` | Driver or passenger of offer | Nobody (CF only) | — |
| `chats/{chatId}` | Participants only | Participants (unreadCounts zero only) | Direct write allowed only to zero own unread count |
| `chats/{chatId}/messages/{messageId}` | Participants (or UID appears in chatId for new chats) | Nobody (CF only) | Allows empty-chat view before first message |

---

## Realtime Database Structure and Rules

```
activeRides/
  {rideId}/
    meta/              ← read: driver or confirmed passengers; write: CF only
      driverId
      confirmedPassengerIds/{uid}: true
    driverLocation/    ← read: driver or confirmed passengers; write: driver only
      latitude, longitude, heading, speed, timestamp, updatedAt
    passengerLocations/
      {passengerId}/   ← read: driver or this passenger; write: this passenger (if confirmed)
        latitude, longitude, heading, speed, timestamp
    pickupStops/       ← read: driver or confirmed passengers; write: driver (via CF)
      {passengerId}/
        status, address, order, seatsBooked, ...

rateLimits/            ← read: authenticated; write: CF only
onlineUsers/
  {uid}/              ← read: authenticated; write: own uid only
```

---

## Firebase Storage Paths

| Path | Contents | Access |
|---|---|---|
| `avatars/{uid}` | Profile photos | Public read (via `avatarUrl`) |
| `id-documents/{uid}` | Encrypted ID document blobs | Private; accessed via signed URL (admin only) |
| `dispute-evidence/{disputeId}/` | Evidence uploads for disputes | Private |
| `support-screenshots/{ticketId}/` | Support ticket screenshots | Private |
