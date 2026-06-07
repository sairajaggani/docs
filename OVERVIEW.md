# ZygoRIde — System Overview

---

## System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          Firebase Platform                         │
│                                                                    │
│  ┌─────────────┐  ┌─────────────────┐  ┌──────────────────────┐  │
│  │  Firestore  │  │ Realtime Database│  │   Firebase Storage   │  │
│  │  (primary   │  │  (live tracking, │  │  (avatars, ID docs,  │  │
│  │   store)    │  │   rate limits,   │  │   screenshots)       │  │
│  │             │  │   online users)  │  │                      │  │
│  └──────┬──────┘  └────────┬────────┘  └──────────────────────┘  │
│         │                  │                                       │
│  ┌──────▼──────────────────▼─────────────────────────────────┐   │
│  │               Cloud Functions (Node 18, TS)                │   │
│  │  Auth · Profiles · Rides · Bookings · Ratings · Maps       │   │
│  │  Messaging · Offers · Admin · Support · Scheduled          │   │
│  └──────┬───────────────────────────────────┬─────────────────┘   │
│         │                                   │                      │
│  ┌──────▼──────┐                 ┌──────────▼──────┐              │
│  │  Firebase   │                 │  Firebase Cloud  │              │
│  │    Auth     │                 │   Messaging (FCM)│              │
│  └─────────────┘                 └─────────────────┘              │
└────────────────────────────────────────────────────────────────────┘
         ▲                    ▲                    ▲
         │                    │                    │
 ┌───────┴──────┐   ┌─────────┴──────┐   ┌────────┴───────┐
 │ Mobile App   │   │  Admin Panel   │   │  Landing Page  │
 │ React Native │   │  Next.js 14    │   │  Pure HTML/CSS │
 │ Expo         │   │  (web only)    │   │  Formspree     │
 └──────────────┘   └────────────────┘   └────────────────┘
```

---

## How the Four Parts Connect

| Part | Connects to Firebase via | Direct Firestore? | Direct RTDB? |
|---|---|---|---|
| Mobile app | `@react-native-firebase` native SDK + Cloud Functions | Yes (reads, limited writes) | Yes (live tracking only) |
| Admin panel | Firebase JS SDK — Cloud Functions only | No | No |
| Landing page | None (static) — Formspree for contact form | No | No |
| Cloud Functions | Firebase Admin SDK | Yes (full access) | Yes (full access) |

The mobile app reads Firestore directly (with security rules enforcement) for live listeners but funnels all writes through Cloud Functions. The admin panel never touches Firestore directly.

---

## Firestore Collection Schema

### `profiles/{uid}`

| Field | Type | Notes |
|---|---|---|
| uid | string | Mirrors Auth UID |
| email | string | |
| phone | string | |
| firstName | string | |
| lastName | string | |
| avatarUrl | string | GCS public URL |
| idUrl | string | (legacy) public thumbnail URL |
| idStoragePath | string? | Private GCS path for encrypted blob |
| idVerified | boolean | |
| idVerificationStatus | string | `PENDING` \| `APPROVED` \| `REJECTED` |
| idRejectionReason | string? | |
| accountStatus | string | `active` \| `suspended` \| `deleted` |
| fcmTokens | string[] | Push token array (multicast) |
| driverStats | object | `{ rating, totalRides, totalCancellations, reviewsCount, completionRate, cancellationRate }` |
| passengerStats | object | `{ rating, totalRides, totalCancellations, reviewsCount, cancellationRate }` |
| vehicles | Vehicle[] | `[{ make, model, year?, color, plate, capacity? }]` |
| preferences | object | `{ currency, language, notifications: { bookingRequests, rideUpdates, marketing } }` |
| activeRideIds | string[] | Currently active ride IDs |
| activeBookingIds | string[] | Currently active booking IDs |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `rides/{rideId}`

| Field | Type | Notes |
|---|---|---|
| rideId | string | |
| driverId | string | |
| driver | DriverSnapshot | `{ uid, firstName, lastName, avatarUrl, rating, totalRides, phone }` |
| vehicleDetails | VehicleSnapshot | `{ make, model, year?, color, plate }` |
| from | Location | `{ latitude, longitude, address, placeId, city, state?, plusCode?, plusCodeCompound? }` |
| to | Location | same shape as `from` |
| rideDate | string | `YYYY-MM-DD` |
| rideStartTime | Timestamp | |
| rideEndTime | Timestamp | |
| isTimeRange | boolean | |
| route | RouteInfo | `{ distanceValue, durationValue, distance, duration, polyline }` |
| price | number | Per seat |
| currency | string | `INR` \| `USD` |
| seats | number | Total offered |
| bookedSeats | number | |
| availableSeats | number | |
| passengerIds | string[] | All who booked (including cancelled) |
| confirmedPassengerIds | string[] | Only CONFIRMED bookings |
| notes | string | |
| status | string | `OPEN` \| `FULL` \| `STARTED` \| `COMPLETED` \| `CANCELLED` \| `EXPIRED` |
| actualStartTime | Timestamp? | |
| actualEndTime | Timestamp? | |
| flagged | boolean | |
| flagReason | string? | |
| searchIndex | object | `{ fromH3_7, fromH3_8, toH3_7, toH3_8, fromCity, toCity, date, status, availableSeats }` |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `bookings/{bookingId}`

| Field | Type | Notes |
|---|---|---|
| bookingId | string | |
| rideId | string | |
| driverId | string | |
| passengerId | string | |
| passenger | PassengerSnapshot | `{ uid, firstName, lastName, avatarUrl, rating, phone }` |
| driver | DriverSnapshot | same shape |
| seatsBooked | number | |
| priceAtBooking | number | Per seat at time of booking |
| totalPrice | number | `seatsBooked × priceAtBooking` |
| status | string | `PENDING` \| `CONFIRMED` \| `CANCELLED` \| `COMPLETED` |
| cancellationReason | string? | |
| cancelledBy | string? | uid |
| cancelledAt | Timestamp? | |
| paymentStatus | string | `PENDING` \| `PAID` |
| routeSnapshot | object | `{ fromAddress, toAddress, rideStartTime, rideEndTime }` |
| idempotencyKey | string | Prevents duplicate bookings |
| completedAt | Timestamp? | |
| pickupOtp | string? | 4-digit OTP written at ride start |
| joinedAt | Timestamp? | Written at ride start |
| noShow | boolean? | |
| bookedForFriend | object? | `{ name, phone }` |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `ratings/{ratingId}`

| Field | Type | Notes |
|---|---|---|
| ratingId | string | |
| rideId | string | |
| bookingId | string | |
| reviewerId | string | uid |
| revieweeId | string | uid |
| role | string | `DRIVER_TO_PASSENGER` \| `PASSENGER_TO_DRIVER` |
| rating | number | 1–5 |
| review | string | Max 500 chars |
| tags | string[]? | |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `rideRequests/{requestId}`

| Field | Type | Notes |
|---|---|---|
| requestId | string | |
| userId | string | |
| from | Location | |
| to | Location | |
| fromH3_7, fromH3_8, toH3_7, toH3_8 | string | H3 index cells |
| requestedDate | string | `YYYY-MM-DD` |
| flexibleDates | boolean | |
| dateRange | object? | `{ start, end }` |
| seatsNeeded | number | |
| maxPrice | number? | |
| currency | string | |
| preferredDepartureTime | Timestamp? | |
| status | string | `PENDING` \| `MATCHED` \| `FULFILLED` \| `EXPIRED` \| `CANCELLED` |
| matchedRideIds | string[] | |
| expiresAt | Timestamp | 30 days from creation |
| passengerSnapshot | object? | `{ firstName, lastName, avatarUrl, rating }` — denormalized |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `rideOffers/{offerId}`

| Field | Type | Notes |
|---|---|---|
| offerId | string | |
| requestId | string | |
| driverId | string | |
| passengerId | string | |
| driver | DriverSnapshot | |
| vehicleDetails | VehicleSnapshot | |
| price | number | |
| currency | string | |
| message | string | Max 300 chars |
| haversineDistanceKm | number | |
| seatsOffered | number | |
| status | string | `PENDING` \| `ACCEPTED` \| `REJECTED` \| `WITHDRAWN` \| `EXPIRED` \| `AUTO_REJECTED` |
| fromAddress | string | |
| toAddress | string | |
| requestedDate | string | |
| resultRideId | string? | Set on accept |
| resultBookingId | string? | Set on accept |
| rejectedAt, acceptedAt, withdrawnAt | Timestamp? | |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `chats/{chatId}`

| Field | Type | Notes |
|---|---|---|
| chatId | string | `[uid1, uid2].sort().join('_')` |
| participants | string[] | `[uid1, uid2]` |
| rideId | string? | |
| rideSnapshot | object? | `{ fromAddress, toAddress }` |
| lastMessage | string? | |
| lastSenderId | string? | |
| lastMessageAt | Timestamp? | |
| unreadCounts | object | `{ [uid]: number }` |
| deletedBy | object? | `{ [uid]: Timestamp }` — soft delete |

#### `chats/{chatId}/messages/{messageId}`

| Field | Type |
|---|---|
| senderId | string |
| text | string |
| createdAt | Timestamp |

### `supportTickets/{ticketId}`

| Field | Type | Notes |
|---|---|---|
| ticketId | string | |
| uid | string | |
| userName | string | |
| userEmail | string | |
| userPhone | string? | |
| category | string | |
| description | string | |
| screenshotUrls | string[] | |
| platform | string | ios \| android |
| appVersion | string? | |
| status | string | `OPEN` \| `RESOLVED` |
| adminNotes | string? | |
| resolution | string? | |
| resolvedBy | string? | admin uid |
| resolvedAt | Timestamp? | |
| createdAt | Timestamp | |
| updatedAt | Timestamp | |

### `requestAlertSubscriptions/{driverId}`

| Field | Type | Notes |
|---|---|---|
| driverId | string | |
| enabled | boolean | |
| h3Cells_7 | string[] | H3 resolution-7 cells driver wants alerts for |
| updatedAt | Timestamp | |

### Other collections

| Collection | Purpose |
|---|---|
| `notifications/{id}` | FCM notification log per user |
| `mapsCache/{hash}` | Google Directions API cache (TTL via Firestore `expireAt`) |
| `adminActions/{id}` | Audit log of admin actions (verify, suspend, resolve) |
| `admins/{uid}` | Admin identity gate — existence = admin access |
| `disputes/{id}` | Dispute evidence (not yet fully implemented) |

---

## Realtime Database Schema

```
activeRides/
  {rideId}/
    meta/
      driverId: string
      confirmedPassengerIds/
        {uid}: true
    driverLocation/
      latitude: number
      longitude: number
      heading: number
      speed: number
      timestamp: number (ms)
      updatedAt: number (ms)
    passengerLocations/
      {passengerId}/
        latitude: number
        longitude: number
        heading: number
        speed: number
        timestamp: number (ms)
    pickupStops/
      {passengerId}/
        passengerId: string
        passengerName: string
        latitude: number
        longitude: number
        address: string
        plusCode: string
        plusCodeCompound: string
        placeId: string
        seatsBooked: number
        status: "PENDING" | "PICKED_UP" | "NO_SHOW"
        notifiedAt: number | null
        order: number   ← nearest-first sort

rateLimits/
  {uid}/
    {functionName}/
      count: number
      resetAt: number (ms)

onlineUsers/
  {uid}/
    online: boolean
    lastSeen: number (ms)
```

---

## Firebase Storage Paths

| Path | Contents | Access |
|---|---|---|
| `avatars/{uid}` | Profile photo | Public URL via `avatarUrl` field |
| `id-documents/{uid}` | Encrypted ID document blob | Private; signed URL via `decryptIDDocument` CF (admin only) |
| `dispute-evidence/{disputeId}/` | Evidence uploads | Private |
| `support-screenshots/{ticketId}/` | Support ticket screenshots | Private |

---

## Auth Flow

```
1. User opens app
   ├── No session → /login
   └── Session exists
         ├── Email unverified → /verify-email
         │     └── User clicks email link → reload() → emailVerified = true → /tabs/need-ride
         └── Email verified → /tabs/need-ride

2. On /tabs/need-ride, ProfileContext checks profiles/{uid}
   ├── Profile exists → app ready
   └── Profile missing → onCreate CF already created skeleton; createProfile CF completes it

3. ID verification flow
   ├── User uploads ID via uploadID CF → idVerificationStatus = PENDING
   ├── Admin reviews in admin panel → verifyIDDocument CF
   │     ├── Approve → idVerified = true, idVerificationStatus = APPROVED
   │     │     └── Notification: ID_VERIFICATION (approved)
   │     └── Reject → idVerified = false, idVerificationStatus = REJECTED, reason set
   │           └── Notification: ID_VERIFICATION (rejected)
   └── Profile screen shows ID Verified badge once idVerified = true
```

---

## Core User Flows

### 1. Rider Books a Ride

```
Rider: need-ride tab
  → fills From/To (Google Places autocomplete → lat/lng/H3)
  → taps Search
  → searchRides CF (H3 spatial match, paginated)
  → taps ride card → ride-details screen (getRideDetails CF)
  → taps Book → createBooking CF
      ├── Firestore: bookings/{id} created (PENDING)
      ├── rides/{id}.availableSeats decremented
      └── FCM → driver: BOOKING_REQUEST

Driver: posted-ride-details
  → taps Confirm → confirmBooking CF
      ├── Firestore: bookings/{id} status = CONFIRMED
      ├── rides/{id}.confirmedPassengerIds updated
      └── FCM → rider: BOOKING_CONFIRMED

Rider: booked-ride-details
  → sees confirmed booking; chat link available
```

### 2. Driver Posts a Ride and Gets a Booking

```
Driver: post-ride tab (offer mode)
  → taps From/To → location-selector modal (Google Maps)
  → sets date, time, vehicle, seats, price, notes
  → taps "Review Route & Post" → confirm-ride modal
      ├── route computed via getDirections CF (cached in mapsCache)
      └── taps "Post Ride" → createRide CF
            ├── validates profile + vehicles + no time overlap
            ├── encodes H3 cells (res 7 + 8)
            ├── creates rides/{id} (OPEN)
            └── matches vs existing rideRequests → MATCHED notifications

Passenger books (see flow 1 above)
```

### 3. Live Ride (Start → OTP → Complete → Rate)

```
Driver: posted-ride-details
  → taps "Start Ride" → startRide CF
      ├── rides/{id}.status = STARTED
      ├── bookings/{id}.pickupOtp = random 4 digits (per passenger)
      ├── RTDB activeRides/{id} written (meta + pickupStops sorted nearest-first)
      └── FCM → all confirmed passengers: RIDE_STARTED

Both parties: app shows LiveRideBanner → taps to live-ride screen
  ├── Driver sees map + passenger pickup stops
  └── Passengers see driver location moving on map

Driver reaches first stop → passenger shows OTP
  → driver enters OTP → verifyPassengerOtp CF
      └── RTDB pickupStops/{passengerId}.status = PICKED_UP

[Repeat for each passenger]

Driver: taps "Complete Ride" → completeRide CF
  ├── rides/{id}.status = COMPLETED
  ├── all CONFIRMED bookings → COMPLETED
  ├── RTDB activeRides/{id} deleted
  ├── driverStats.totalRides++ / completionRate updated
  ├── each passengerStats.totalRides++
  └── FCM: RIDE_COMPLETED + RATING_REMINDER to all parties

Both parties: rate-user screen (within 7 days)
  → submitRating CF
      ├── ratings/{id} created
      ├── reviewee's rating recalculated (weighted average)
      └── FCM: RATING_RECEIVED
```

### 4. Ride Request → Match → Offer → Accept

```
Passenger: need-ride tab, no results found
  → taps "Request a Ride" → modal
  → fills date (or date range), seats, optional maxPrice + preferred time
  → taps "Submit Request" → createRideRequest CF
      ├── rideRequests/{id} created (PENDING, expiresAt 30 days)
      └── if existing rides match → MATCHED status + FCM to passenger

Driver: post-ride tab (browse mode) OR receives NEW_RIDE_REQUEST_NEARBY FCM
  → browseRideRequests CF → list of PENDING requests
  → taps request → request-details screen
  → taps "Send Offer" → send-offer screen
      ├── fills price, seats, message
      └── sendRideOffer CF
            ├── rideOffers/{id} created (PENDING)
            └── FCM → passenger: NEW_RIDE_OFFER

Passenger: request-details-passenger screen
  → sees incoming offers
  → taps "Accept" → acceptRideOffer CF
      ├── rideOffers/{id}.status = ACCEPTED
      ├── auto-creates rides/{id} (for driver) + bookings/{id} (CONFIRMED)
      ├── rideRequests/{id}.status = FULFILLED
      ├── all other PENDING offers → AUTO_REJECTED
      ├── FCM → driver: RIDE_OFFER_ACCEPTED
      └── FCM → other drivers: RIDE_OFFER_AUTO_REJECTED
```

---

## Notification Map

| Event | Notification Type | Recipient(s) |
|---|---|---|
| Passenger books a ride | `BOOKING_REQUEST` | Driver |
| Driver confirms booking | `BOOKING_CONFIRMED` | Passenger |
| Driver or passenger cancels booking | `BOOKING_CANCELLED` | Other party |
| Driver cancels ride | `RIDE_CANCELLED` | All confirmed passengers |
| Driver updates ride details | `RIDE_UPDATED` | All confirmed passengers |
| Driver starts ride | `RIDE_STARTED` | All confirmed passengers |
| Ride completed | `RIDE_COMPLETED` | All confirmed passengers |
| Ride complete / rating window open | `RATING_REMINDER` | Driver + each passenger |
| Rating submitted | `RATING_RECEIVED` | Reviewee |
| Ride request matches new ride | `RIDE_MATCHED` | Passenger (request owner) |
| New ride in driver's alert area | `NEW_RIDE_REQUEST_NEARBY` | Subscribed drivers |
| Passenger's request fulfilled | `RIDE_REQUEST_FULFILLED` | Passenger |
| Driver sends offer | `NEW_RIDE_OFFER` | Passenger |
| Passenger accepts offer | `RIDE_OFFER_ACCEPTED` | Driver |
| Passenger rejects offer | `RIDE_OFFER_REJECTED` | Driver |
| Other offer auto-rejected | `RIDE_OFFER_AUTO_REJECTED` | Other drivers |
| Upcoming ride (< 1 hour) | `RIDE_REMINDER` | Driver + confirmed passengers |
| New chat message | `NEW_MESSAGE` | Receiver |
| ID verification result | `ID_VERIFICATION` | User |
| Account suspended | `ACCOUNT_SUSPENDED` | User |
| Account restored | `ACCOUNT_RESTORED` | User |
| Alternate rides found after cancellation | `ALTERNATE_RIDES_SUGGESTED` | Affected passengers |

---

## Scheduled Jobs Summary

| Job | Schedule | Purpose |
|---|---|---|
| `scheduledCleanup` | Every 24 hours | Purge expired RTDB rate limit entries and idempotency keys |
| `expireRideRequests` | Every 12 hours | Expire `PENDING` rideRequests past `expiresAt` |
| `expireStaleRides` | Every 12 hours | Expire `OPEN` rides past their `rideStartTime` |
| `rideReminders` | Every 30 minutes | Send `RIDE_REMINDER` for rides departing within 1 hour |
| `ratingReminders` | Every 24 hours | Send `RATING_REMINDER` for unrated completed rides within 7 days |
| `archiveOldData` | Every Monday 02:00 | Archive rides/bookings > 60 days old; delete expired requests and old notifications |
| `autoCompleteStaleRides` | Every 12 hours | Auto-complete STARTED rides > 24 hours past start time |
