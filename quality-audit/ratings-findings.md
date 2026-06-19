## Ratings — Audit Findings

### Bugs (must fix)

- [BUG-01] 5× inline `'#fff'` + `'#FFD700'` star color violating "no inline hex" rule. `'#fff'` → `theme.colors.text.inverse`; `'#FFD700'` → `theme.colors.warning` (consistent with `booked-ride-details.tsx` fix from previous audit) | `Carpool/app/rate-user.tsx:66,298,402,441,571,621` | Severity: high

- [BUG-02] Ratings Firestore rule: `allow read: if true;` exposes `review` text and `reviewerId`/`revieweeId` to unauthenticated clients — ratings should require at least an authenticated session to read | `CarpoolBackend/firestore.rules:51` | Severity: medium

- [BUG-03] `TAG_ICONS` key mismatch: map has `'Safe Driver'` but `DRIVER_TAGS` uses `'Safe Ride'` — the shield icon is never resolved and falls back to `'pricetag-outline'` | `Carpool/app/rate-user.tsx:40,73` | Severity: low

- [BUG-04] Notification body for `submitRating` includes the raw review text (up to 500 chars) — FCM best practice is ≤100 chars for notification body; long reviews produce truncated/broken push notifications | `CarpoolBackend/functions/src/functions/ratings/submitRating.fn.ts:196` | Severity: low

### Improvements (should fix)

- [IMP-01] `functionsService.submitRating` types `review: string` (required) but the BE schema marks it optional; change to `review?: string` so callers can omit it cleanly | `Carpool/src/services/firebase/functions.service.ts:361` | Impact: type accuracy

- [IMP-02] `getUserRatingsForRide` returns `Promise<any[]>` — should return a typed array using the `Rating` model type | `Carpool/src/services/firebase/firestore.service.ts:433` | Impact: type safety

### Notes

- Backend logic is solid: transaction re-checks for duplicate ratings to prevent race conditions; role is determined server-side (cannot be spoofed by client); 7-day rating window enforced server-side.
- `ratedPassengerIds` in `posted-ride-details.tsx` is correctly populated via a `useEffect` (line 210) — no bug.
- `rideDate` passed as a string through navigation params; `formatDate('')` safely returns `''` and the JSX guards with `{params.rideDate && ...}` — no bug.
- `ratingReminders` scheduled function batches correctly with the 500-doc batch limit.
