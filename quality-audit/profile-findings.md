## Profile — Audit Findings

### Bugs (must fix)

- [BUG-01] `public-profile.tsx:139` — rating rendered as `{item.rating}.0` — for a non-integer rating like 4.5 this displays "4.5.0" | `public-profile.tsx:139` | Severity: high

- [BUG-02] `public-profile.tsx:102` — `overallRating` uses `||` shortcut (`driverRating || passengerRating`) — wrong when driver rating is 0.0 with actual reviews; should use the same weighted-average formula as `profile.tsx:67-77` | `public-profile.tsx:102` | Severity: medium

- [BUG-03] `update-profile.tsx:133` — ID image base64 header hardcoded to `image/jpeg` regardless of actual image type — a PNG upload would be stored with wrong MIME metadata | `update-profile.tsx:133` | Severity: medium

- [BUG-04] `addVehicle.fn.ts` / `removeVehicle.fn.ts` — no rate limiting; both CFs lack `await rateLimiters.addVehicle/removeVehicle` calls and `ADD_VEHICLE`/`REMOVE_VEHICLE` aren't in RATE_LIMITS constants | `addVehicle.fn.ts`, `removeVehicle.fn.ts` | Severity: low

### Improvements (should fix)

- [IMP-01] Vehicle edit is non-atomic — when editing, code does `removeVehicle` then `addVehicle`; if the add fails after remove the vehicle is permanently lost. Fix: for same-plate edit keep current order (remove-then-add, required to pass duplicate check); for different-plate add new first then remove old | `update-profile.tsx:169-175` | Impact: medium

- [IMP-02] `console.warn`/`console.error` calls in committed code: lines 108, 141, 257 | `update-profile.tsx` | Impact: low

### Notes

- `uploadAvatar` CF validates URL is from Firebase Storage but not that it's the caller's own path — low risk since avatars are public anyway.
- `updateProfile` CF allows updating the `email` field in the Firestore profile doc, but the FE never sends it. Low priority since the FE doesn't exercise this path.
- Large commented-out block in `profile.tsx:229-248` (old Reviews & Ratings button) — harmless but clutters the file.
- `addVehicle.fn.ts` indentation is inconsistent (mixed indentation inside the withLogging callback) — cosmetic only.
