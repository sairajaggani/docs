## Post Ride — Audit Findings

### Bugs (must fix)

- [BUG-01] Currency hardcoded to 'INR' in confirm-ride.tsx | `Carpool/app/confirm-ride.tsx:97` | Severity: medium
  - post-ride.tsx correctly derives currency from pickup coordinates (India bbox → INR, else → USD) but never passes it as a nav param.
  - confirm-ride.tsx always sends `currency: 'INR'` to the CF, bypassing the FE cap and sending wrong currency to the backend for non-Indian users.
  - Fix: pass `currency` in navParams from post-ride → confirm-ride.

- [BUG-02] Vehicle edit is non-atomic (remove → add): data-loss risk | `Carpool/app/tabs/post-ride.tsx:197-208` | Severity: high
  - handleSaveVehicle calls removeVehicle() then addVehicle() sequentially. If addVehicle fails, the vehicle is permanently deleted.
  - Fix: wrap in try/catch — if addVehicle throws, re-add the original vehicle as best-effort recovery before re-throwing.

- [BUG-03] selectedVehicleIndex points to wrong vehicle after edit | `Carpool/app/tabs/post-ride.tsx:208-211` | Severity: medium
  - After remove+add, the edited vehicle lands at the end of the array (index = length - 1). The selectedVehicleIndex is never updated after an edit, so if the edited vehicle was selected it now silently points to a different vehicle.
  - Fix: after edit path, set selectedVehicleIndex to (profile.vehicles.length - 1) when the edited vehicle was selected.

- [BUG-04] Seats field allows "0" on FE, bypassing schema min:1 | `Carpool/app/tabs/post-ride.tsx:557-561` | Severity: medium
  - onChangeText strips non-digits but allows "0". The default `seats || '1'` in handleViewMap is falsy-safe for empty string but NOT for "0" (truthy). parseInt("0", 10) = 0 is sent to the CF, which rejects it with a confusing validation error.
  - Fix: add FE guard in handleViewMap to show an alert if parsed seats < 1.

- [BUG-05] updateRide notifies only confirmedPassengerIds on critical changes | `CarpoolBackend/functions/src/functions/rides/updateRide.fn.ts:181-188` | Severity: medium
  - When price or time changes, only confirmed passengers are notified. Pending-booking passengers won't know about the change until they check the app, and might board expecting a different price/time.
  - Fix: use passengerIds (all passengers) instead of confirmedPassengerIds.

### Improvements (should fix)

- [IMP-01] confirm-ride.tsx uses inline hex colors throughout | `Carpool/app/confirm-ride.tsx` | Impact: low
  - Colors like `#6f42c1`, `#f8f9fd`, `#333`, `#1a1a1a`, `#666`, `#888` violate the theme consistency rule.
  - These should use `theme.colors.*` values.

### Notes

- rideDate vs rideStartTime consistency: The BE receives both `rideDate: "YYYY-MM-DD"` and `rideStartTime: ms`. A consistency check (ensuring the rideDate matches the local date derived from rideStartTime) would be timezone-sensitive and is not practical without transmitting the user's timezone. The FE always generates rideDate from the local device clock, so in practice they are consistent.
- isTimeRange is always 'false' in navParams (hardcoded at post-ride.tsx:354) — the end-time picker was removed from the UI but state/logic remain. This is dead code, not a bug; a cleanup task for later.
- The vehicle modal has no length limit on Make/Model/Color fields on the FE, but the BE schema enforces max:50 / max:30. Low-impact since the error message surfaces correctly.
