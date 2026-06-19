## Posted Rides — Audit Findings

### Bugs (must fix)

- [BUG-01] Tab badge passes `filteredRides.length` to `onCountChange` — badge in scheduled-rides.tsx shows however many rides are in the currently-selected filter category (e.g. 15 on "All"), not the meaningful pending-request count | `PostedRides.tsx:123` | Severity: medium

- [BUG-02] `startRide` BE has no guard against 0 confirmed passengers — FE gates `canStart` on `confirmedBookings.length > 0` but the Cloud Function doesn't; a direct SDK call can start a ride with no passengers, leaving a zombie STARTED ride with empty RTDB pickupStops | `startRide.fn.ts` after line 244 | Severity: high

- [BUG-03] `expiredBanner` is positioned in normal flow but rendered as a sibling of the absolute-positioned footer — in RN's flex layout the ScrollView takes all remaining space, pushing the banner out of the visible area; when `canCancelExpired` is true the banner is never seen | `posted-ride-details.tsx:780–850` | Severity: medium

### Improvements (should fix)

- [IMP-01] `markPassengerNoShow` never notifies the passenger — driver marks them no-show silently; passenger sees the ride complete with no explanation | `markPassengerNoShow.fn.ts` | Impact: medium

- [IMP-02] `handleCompleteRide` in `posted-ride-details.tsx` is dead code — it is defined but never wired to any JSX button (completion happens in `live-ride.tsx`) | `posted-ride-details.tsx:320` | Impact: low (code hygiene, leave for future)

### Notes

- `chatId` format in `posted-ride-details.tsx:387` (`[uid1, uid2].sort().join('_')`) matches `sendMessage` CF exactly — memory doc was outdated; no bug here.
- `verifyPassengerOtp` charges wrong-OTP attempts to lowest-createdAt booking across the ride, not the specific passenger being scanned. Acceptable for current scale but worth revisiting if OTP abuse becomes an issue.
- `onRefresh` in `PostedRides.tsx` is cosmetic-only (doesn't force a Firestore re-fetch). Acceptable because the real-time listener keeps data fresh; only an issue if the listener drops silently.
- `updateRide` CF correctly guards: `seats < bookedSeats` → error. No client can reduce seats below confirmed count.
- `rideCompletion.util.ts` correctly handles no-shows: `noShow: !hasJoined` set on each CONFIRMED booking at completion time.
