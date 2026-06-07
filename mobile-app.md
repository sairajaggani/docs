# ZygoRIde Mobile App

## Tech Stack

| Layer | Library / Version |
|---|---|
| Framework | React Native (Expo) |
| Router | Expo Router v3 (file-based, tabs + stack) |
| Firebase SDK | `@react-native-firebase` (native modules) |
| Auth | `@react-native-firebase/auth` |
| Firestore | `@react-native-firebase/firestore` |
| Realtime Database | `@react-native-firebase/database` |
| Storage | `@react-native-firebase/storage` |
| Cloud Functions | `@react-native-firebase/functions` |
| Crash reporting | `@react-native-firebase/crashlytics` |
| Push notifications | `@react-native-firebase/messaging` |
| App Check | `@react-native-firebase/app-check` |
| Maps / Autocomplete | Google Maps SDK + Places API |
| Location | `expo-location` (foreground + background) |
| Google Sign-In | `@react-native-google-signin/google-signin` |
| State management | React Context + hooks |
| Target SDK | Android 36 / iOS (static frameworks) |
| Min SDK | Android 24 |
| New Arch | Enabled |

Firebase is initialised via native config files (`google-services.json` / `GoogleService-Info.plist`). The JS entry point (`Car/firebase.ts`) exports: `auth`, `db` (Firestore), `rtdb` (RTDB), `storage`, `functions`.

---

## App Entry Point and Navigation

### Root layout — `app/_layout.tsx`

Wraps the entire app in three providers (outermost to innermost):

```
ProfileProvider
  MessagingProvider
    AlertProvider
      NavigationHandler        ← auth/routing guard
        EmailVerificationHandler
        StackLayout            ← Expo Router Stack
      Toast
```

**NavigationHandler** listens to `onIdTokenChanged` (catches email-verify status changes after `getIdToken(true)`). Routing rules:

| State | Redirect |
|---|---|
| Not logged in | `/login` |
| Logged in, email unverified | `/verify-email` |
| Logged in, verified, on auth screen | `/tabs/need-ride` |

A 3-second safety timeout forces `isReady = true` to prevent white screens on slow networks.

App Check and Crashlytics are initialised at module level before any component mounts.

### Stack screens

All screens use `headerShown: false`. Modal presentation for `confirm-ride` and `location-selector`.

| Screen | File |
|---|---|
| `tabs` (tab shell) | `app/tabs/_layout.tsx` |
| `login` | `app/login.tsx` |
| `signup` | `app/signup.tsx` |
| `verify-email` | `app/verify-email.tsx` |
| `confirm-ride` | `app/confirm-ride.tsx` (modal) |
| `location-selector` | `app/location-selector.tsx` (modal) |
| `ride-details` | `app/ride-details.tsx` |
| `posted-ride-details` | `app/posted-ride-details.tsx` |
| `booked-ride-details` | `app/booked-ride-details.tsx` |
| `update-profile` | `app/update-profile.tsx` |
| `update-auth` | `app/update-auth.tsx` |
| `live-ride` | `app/live-ride.tsx` |
| `rate-user` | `app/rate-user.tsx` |
| `public-profile` | `app/public-profile.tsx` |
| `chat` | `app/chat.tsx` |
| `messages` | `app/messages.tsx` |
| `support` | `app/support.tsx` |
| `settings` | `app/settings.tsx` |
| `terms-conditions` | `app/terms-conditions.tsx` |
| `my-requests` | `app/my-requests.tsx` |
| `request-details-passenger` | `app/request-details-passenger.tsx` |
| `request-details` | `app/request-details.tsx` |
| `send-offer` | `app/send-offer.tsx` |

### Tab shell — `app/tabs/_layout.tsx`

Five tabs. The **My Rides** tab shows a green live-badge dot when an active ride exists. The **Messages** tab shows an unread-count badge (from `MessagingContext`). iOS app icon badge is synced to total unread count via `messagingService.setBadgeCount`. A floating `LiveRideBanner` sits above the tab bar whenever a STARTED ride is detected.

| Tab | Screen | Icon |
|---|---|---|
| Offer Ride | `post-ride` | car |
| My Rides | `scheduled-rides` | calendar |
| Search | `need-ride` | search |
| Messages | `messages` | comment |
| Profile | `profile` | user |

---

## Screens

### `tabs/need-ride` — Find Ride

**Purpose:** Search for available rides and post ride requests when none are found.

**Key state:**
- `fromLocation` / `toLocation` — resolved `SearchLocation` with lat/lng/placeId/plusCode
- `selectedDate`, `passengers` — optional search filters
- `activeInput` — focused location field driving suggestion dropdown
- `suggestions` / `locationHistory` — Google Places autocomplete or local history
- `isMinimized` / `searchCardHeight` — animated collapse after search fires
- `myRideRequests` — live Firestore listener on own `PENDING`/`MATCHED` rideRequests

**Key actions:**
1. User selects From/To from autocomplete → `placesService.getPlaceDetails` resolves full coordinates
2. Tapping **Search** calls `useRideSearch.search()` which calls `searchRides` Cloud Function (H3 spatial search)
3. Sort modal changes `sortBy` and re-fires the search
4. **Load More** appends next page using cursor
5. **Request a Ride** modal → `functionsService.createRideRequest` with date, seats, maxPrice, preferredTime

**Navigation:** Ride card tap → `/ride-details`. "My ride requests" link → `/my-requests`.

**Firebase reads:** `rideRequests` (live listener, own records only).

---

### `tabs/post-ride` — Offer Ride + Browse Requests

**Purpose:** Dual-mode screen — drivers post rides or browse passenger requests.

**Mode toggle:** `offer` | `browse` (`BrowseRequestsList` component)

**Offer mode key state:**
- `from` / `to` — locations selected via `/location-selector` modal (returned via URL params)
- `date` / `startTime`
- `selectedVehicleIndex` — index into `profile.vehicles`
- `seats`, `price`, `notes`, `draftId` (edit mode)

**Key actions:**
- Vehicle management: add/edit/delete via `functionsService.addVehicle` / `removeVehicle`
- **Review Route & Post** validates fields and navigates to `/confirm-ride` modal

---

### `tabs/scheduled-rides` — My Rides

**Purpose:** Shows booked rides (passenger) and posted rides (driver) in two sub-tabs.

Components: `BookedRides` and `PostedRides` (from `components/rides/`). Badge counts bubble up via `onCountChange`. Initial tab driven by `tab` URL param.

---

### `tabs/messages` — Messages

**Purpose:** Lists all active chat conversations.

Reads from `MessagingContext`. Each row shows participant avatar/name, last message preview, unread badge, ride route snippet. Soft-deleted chats are hidden unless a newer message arrived after deletion. Taps navigate to `/chat`.

---

### `tabs/profile` — Profile

**Purpose:** View and manage own profile.

Shows: avatar, name, verification badges (ID Verified / Email / Phone), member since, stats row (rating/rides/reviews — taps to public profile), driver stats (collapsible), passenger stats (collapsible), contact info, language, vehicles. Warning banner if ID unverified.

---

### `live-ride` — Active Ride Tracking

Real-time tracking screen. Reads driver location from RTDB `activeRides/{rideId}/driverLocation`. Driver sees passenger pickup stops sorted nearest-first. OTP verification flow runs here via `verifyPassengerOtp` CF.

---

### `confirm-ride` — Review Route & Post

Modal showing computed route on a Google Maps polyline. Driver confirms and taps **Post Ride** → `functionsService.createRide`.

---

### `ride-details` — Public Ride View

Full ride details fetched via `getRideDetails` CF. Passenger can tap **Book** → `functionsService.createBooking`. Shows driver snapshot, vehicle, route map.

---

### `posted-ride-details` — Driver Ride Management

Driver view of a posted ride. Shows passenger booking list with statuses. Actions: edit, cancel, start ride. Start ride → `functionsService.startRide`.

---

### `booked-ride-details` — Passenger Booking

Booking status, OTP (once ride starts), cancel button, chat link.

---

### `chat` — Conversation

Reads `chats/{chatId}/messages` subcollection in real time. Sends via `functionsService.sendMessage`. Marks read by zeroing `unreadCounts[uid]` with a direct Firestore write (permitted by security rules).

---

### `rate-user` — Post-Ride Rating

Star rating + text review + tags submitted via `functionsService.submitRating`. Available within 7 days of ride completion.

---

### `my-requests` — Ride Requests List

Own ride requests with status chips (PENDING / MATCHED / FULFILLED / EXPIRED / CANCELLED). Tap → `request-details-passenger`.

---

### `send-offer` — Driver Sends Offer to Passenger

Driver sends a ride offer to a passenger request. Calls `functionsService.sendRideOffer`.

---

### `request-details` — Request Detail (Driver view)

Driver sees a passenger's ride request; can send an offer from here.

---

### `request-details-passenger` — Offer Detail (Passenger view)

Passenger views incoming ride offers; can accept or reject each.

---

### Auth screens

| Screen | Purpose |
|---|---|
| `login` | Email/password + Google Sign-In |
| `signup` | Email/password registration |
| `verify-email` | Polling loop + resend button |
| `update-profile` | Edit name, phone, avatar, ID upload |
| `update-auth` | Change email or password |

---

## Components

### `SplashAnimation`

Custom JS splash that plays before hiding the native splash. Fires `onFinish`; `NavigationHandler` waits for it before routing.

### `EmailVerificationHandler`

Background component that polls and listens for deep links to detect email verification completion. Forces navigation to tabs after success.

### `LiveRideBanner`

Floating sticky banner above the tab bar when a STARTED ride exists. Shows route summary and elapsed time. Tapping navigates to `/live-ride`.

**Props:** `ride` (id + addresses + duration), `role` (driver | passenger), `driverName`, `position`, `style`.

### `RideCard`

Single ride search result. Shows from/to, date, price/seat, available seats, driver avatar + rating, vehicle info.

**Props:** `ride: SearchRideItem`, `onPress`.

### `BrowseRequestsList`

Driver browse-mode component. Lists passenger `rideRequests`. Shows distance filter, passenger snapshot, date, seats, max price. Tapping → `/request-details`.

### `RideRouteMap`

Renders a polyline route on Google Maps. Used in `confirm-ride` and `ride-details`.

### `EditRideModal`

Inline modal for editing price, seats, notes, or time on a posted ride.

### `DatePicker`

Cross-platform date picker wrapper. Props: `value`, `onChange`, `placeholder`, `minimumDate`, `showClearButton`.

### `SkeletonLoader`

Animated shimmer placeholders while search results load. `count` prop controls number of cards.

### `SuggestedRides`

Displays alternative ride suggestions (from `getSuggestedRides` CF) when no exact match found.

### `rides/BookedRides` and `rides/PostedRides`

Sub-components of the My Rides tab. Each manages its own Firestore listener and bubbles up count via `onCountChange`.

---

## Hooks

### `useActiveRide`

Detects a STARTED ride for the current user in either role.

- Driver: Firestore listener on `rides` where `driverId == uid && status == STARTED` (limit 1)
- Passenger: Firestore listener on `bookings` where `passengerId == uid && status == CONFIRMED`, then listeners on each ride doc to check `status == STARTED`

**Returns:** `{ activeRide: ActiveRideInfo | null, role: 'driver' | 'passenger' | null, loading: boolean }`

### `useRideSearch`

Manages paginated ride search via the `searchRides` Cloud Function. Caches results for 15 minutes.

**Returns:** `{ rides, loading, searching, error, hasMore, noResults, sortBy, search, updateSort, loadMore, refresh, clear }`

`search(filters)` fires the CF and stores results. `loadMore` appends the next cursor page. `refresh` bypasses cache.

### `useFcmToken`

Initialises FCM handlers on mount. Registers the FCM token with the backend only if notification permission was already granted. Listens for `onTokenRefresh` and re-registers.

### `useAlert`

Wraps `AlertContext` with convenience methods `showAlert`, `showError`, `showConfirm`.

### `useDebounce` / `useDebouncedValue`

Debounces a value (default 400ms). Used in search inputs to delay Places API calls.

### `useNotificationPermission`

Requests notification permission on `ensurePermission()` call. Used after booking confirmation to prompt once non-intrusively.

### `usePassengerLocation`

Publishes passenger GPS to RTDB `activeRides/{rideId}/passengerLocations/{uid}` during an active ride.

### `useRideTracking`

Publishes driver GPS to RTDB `activeRides/{rideId}/driverLocation` during an active ride.

---

## Context

### `ProfileContext`

**Provides:** `profile: Profile | null`, `loading`, `user`, `isOnline`, `refreshProfile()`, `updateProfileLocally(updates)`

Live Firestore `onSnapshot` on `profiles/{uid}`. Tracks network state via `NetInfo`. Tears down and re-creates listener on auth state change.

### `MessagingContext`

**Provides:** `chats: Chat[]`, `totalUnread: number`, `loading`

Single Firestore listener on `chats` where `participants array-contains uid`, ordered by `lastMessageAt` desc, limit 25. Filters soft-deleted chats.

### `AlertContext`

**Provides:** `showAlert(config: AlertConfig)`

Custom animated modal alert. Types: `default | success | error | warning`. Registers a global imperative handler so utility code can show alerts without hooks.

---

## Services

### `src/services/firebase/auth.service.ts`
Wraps Firebase Auth: `signIn`, `signUp`, `signOut`, `getCurrentUser`, `sendEmailVerification`, `sendPasswordReset`.

### `src/services/firebase/firestore.service.ts`
Generic Firestore wrapper: `onCollectionSnapshot`, `onDocumentSnapshot`, `getDocument`, `setDocument`, `updateDocument`.

### `src/services/firebase/functions.service.ts`
Wraps every Cloud Function call. See Functions table in Backend doc for the full list.

### `src/services/firebase/messaging.service.ts`
Wraps FCM: foreground/background handlers, Android notification channels, iOS badge count, token retrieval, `onTokenRefresh`, `areNotificationsEnabled`.

### `src/services/maps/places.service.ts`
Wraps Google Places API. `autocomplete(query)` → `AutocompletePrediction[]`. `getPlaceDetails(placeId)` → `{ lat, lng, placeId, plusCode, plusCodeCompound }`.

### `src/services/location/`
`expo-location` wrapper for foreground + background GPS. Used by `usePassengerLocation` and `useRideTracking`.

### `src/services/monitoring/`
Crashlytics wrapper. `crashlyticsService.initialize()` called at app start. `GlobalErrorHandler` captures unhandled JS exceptions.

### `src/services/cache/cache.service.ts`
In-memory TTL cache. Used by `useRideSearch` (15-minute TTL per search query).

### `src/services/storage/storage.service.ts`
AsyncStorage wrapper for: recent search pairs (`SearchPair[]`), per-field location history.

---

## Data Flow: Booking a Ride

1. **Search** — user fills From/To on `need-ride`. Tapping **Search** calls `searchRides` CF (H3 spatial search).
2. **Results** — CF returns paginated `SearchRideItem[]`. Cards rendered by `RideCard`.
3. **Details** — tapping a card → `/ride-details`. `getRideDetails` CF fetches full ride doc + booking count.
4. **Book** — passenger taps **Book**. `createBooking` CF:
   - Creates `bookings` doc with status `PENDING`.
   - Sends `BOOKING_REQUEST` push to driver.
   - Decrements `availableSeats` on ride.
5. **Confirm** — driver opens `posted-ride-details`, taps **Confirm**. `confirmBooking` CF sets booking to `CONFIRMED`, notifies passenger (`BOOKING_CONFIRMED`).
6. **Start** — driver taps **Start Ride**. `startRide` CF:
   - Sets ride `status = STARTED`.
   - Generates 4-digit OTP per confirmed passenger, stores in `bookings/{id}.pickupOtp`.
   - Writes `activeRides/{rideId}` to RTDB (meta + pickup stops sorted nearest-first).
   - Sends `RIDE_STARTED` to all confirmed passengers.
7. **OTP verify** — passenger shows OTP; driver enters it in `live-ride`. `verifyPassengerOtp` CF marks stop `PICKED_UP`.
8. **Complete** — driver taps **Complete**. `completeRide` CF sets ride to `COMPLETED`, bookings to `COMPLETED`, deletes RTDB node, triggers `RATING_REMINDER` notifications.
9. **Rate** — `rate-user` screen calls `submitRating` CF. Window: 7 days.

---

## Firebase Collections per Screen

| Screen | Reads | Writes (via CF) |
|---|---|---|
| need-ride | `rideRequests` (own, live) | `rideRequests` (createRideRequest) |
| post-ride | `profiles/{uid}` (via context) | `rides` (createRide/updateRide), `profiles` (addVehicle/removeVehicle) |
| scheduled-rides | `bookings` (own, live), `rides` (own, live) | `bookings` (cancel), `rides` (cancel) |
| messages | `chats` (own, live via context) | `chats` (markRead direct write, deleteChat CF) |
| profile | `profiles/{uid}` (live via context) | `profiles` (updateProfile, uploadAvatar, uploadID) |
| ride-details | `rides/{id}` | `bookings` (createBooking) |
| posted-ride-details | `rides/{id}`, `bookings` (for ride) | `bookings` (confirm/cancel), `rides` (startRide, cancel) |
| booked-ride-details | `bookings/{id}`, `rides/{id}` | `bookings` (cancel) |
| live-ride | RTDB `activeRides/{id}` (live) | RTDB `driverLocation` / `passengerLocations` |
| chat | `chats/{id}/messages` (live) | `chats` (sendMessage CF, markRead direct) |
| rate-user | — | `ratings` (submitRating CF) |
| my-requests | `rideRequests` (own, live) | `rideRequests` (cancel CF) |
| request-details-passenger | `rideOffers` (on request) | `rideOffers` (accept/reject CF) |
| request-details | `rideRequests/{id}` | `rideOffers` (sendRideOffer CF) |

---

## RTDB Paths Used by the App

| Path | Purpose | Who writes |
|---|---|---|
| `activeRides/{rideId}/meta` | Ride metadata during live ride | `startRide` CF |
| `activeRides/{rideId}/driverLocation` | Real-time driver GPS | Driver device |
| `activeRides/{rideId}/passengerLocations/{uid}` | Real-time passenger GPS | Passenger device |
| `activeRides/{rideId}/pickupStops` | Pickup stop statuses | `startRide` CF (create), `verifyPassengerOtp` CF (update) |
| `rateLimits/{uid}/{functionName}` | Per-user rate limit counters | Backend rate limiter |
| `onlineUsers/{uid}` | Online presence | Client |

---

## Permissions Required

| Permission | Platform | Reason |
|---|---|---|
| `ACCESS_FINE_LOCATION` | Android | Live ride GPS |
| `ACCESS_COARSE_LOCATION` | Android | Fallback location |
| `ACCESS_BACKGROUND_LOCATION` | Android | Driver background tracking |
| `FOREGROUND_SERVICE` | Android | Background location service |
| `POST_NOTIFICATIONS` | Android 13+ | Push notifications |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | iOS | Ride tracking |
| `NSLocationWhenInUseUsageDescription` | iOS | Active ride display |
| `UIBackgroundModes: location, remote-notification` | iOS | Background GPS + FCM |
