## Booked Rides — Audit Findings

### Bugs (must fix)

- [BUG-01] `booked-ride-details.tsx` has 8 inline hex/rgba colors violating the "no inline hex" rule: `STATUS_COLORS` map uses `#f59e0b`/`#22c55e`/`#ef4444`/`#3b82f6` (all have `theme.colors.warning/success/error/info` equivalents); `"#fff"` used 3× in JSX; `"#A855F7"` destination icon; `"#FFD700"` star rating; `"#4ade80"` live-dot; `'rgba(255,255,255,0.85)'` in StyleSheet | `booked-ride-details.tsx:61-65,385,426,455,619,659,767,772,1020` | Severity: high
- [BUG-02] `completeBooking.fn.ts` uses read-modify-write (`filter()`) on `activeBookingIds` inside a transaction instead of `FieldValue.arrayRemove(bookingId)`. Every other CF (`cancelBooking`, `createBooking`) uses `arrayRemove`/`arrayUnion`. The filter approach unnecessarily reads and overwrites the full array, and would drop a concurrent addition to `activeBookingIds` if another transaction committed between the read and this write | `completeBooking.fn.ts:83-93` | Severity: medium
- [BUG-03] `BookedRides.tsx` `useEffect` for `onCountChange` is missing `onCountChange` in its deps array | `BookedRides.tsx:115` | Severity: low
- [BUG-04] `cancelBooking.fn.ts` comment typo: "Verify Permissios" | `cancelBooking.fn.ts:55` | Severity: low

### Improvements (should fix)

- [IMP-01] `booked-ride-details.tsx:767-772`: Status banner text uses `'#fff'` (inline) and `'rgba(255,255,255,0.85)'` (semi-transparent). Replace with `theme.colors.text.inverse` and the same color with `opacity: 0.85` in the style | `styles.statusBannerText / statusBannerSubtext` | Impact: code hygiene

### Notes
- chatId format (`[uid1, uid2].sort().join('_')`) is consistent between `booked-ride-details.tsx` and `posted-ride-details.tsx` — memory note is outdated but implementation is correct
- `createBooking` seats-on-PENDING model is correct: `availableSeats` decremented at PENDING, `bookedSeats` only incremented at CONFIRM — cancel logic mirrors this consistently
- `completeBooking` not exposed in FE — design decision for future (passenger mid-route exit)
- `capturedDriverId!` non-null assertion after transaction is safe: set inside transaction body, only reached if transaction succeeded
- 2-hour cancellation window applies to passengers only — intentional business rule
