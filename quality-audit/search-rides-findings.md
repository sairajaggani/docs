## Search Rides — Audit Findings

### Bugs (must fix)

- [BUG-01] `handleSuggestionSelect` / `handleHistorySelect` set address text but leave `fromLocation`/`toLocation` null when `getPlaceDetails` throws. User sees a filled field, then gets a confusing "select from suggestions" alert on Search tap | `need-ride.tsx:260-349` | Severity: medium
- [BUG-02] No client-side max validation for the Passengers field — user can type "99", backend rejects with a raw zod validation error with no friendly message | `need-ride.tsx:1028-1032` | Severity: medium
- [BUG-03] `SuggestedRides.tsx` `useEffect` dep array only has `[excludeRideId]` but the effect also uses `from`, `to`, `date`, `seats`. Suggestions go stale if parent passes different coordinates without changing `excludeRideId` | `SuggestedRides.tsx:62` | Severity: medium
- [BUG-04] `RideCard.tsx:83` uses inline hex `"#A855F7"` for the destination icon instead of `theme.colors.destination` | `RideCard.tsx:83` | Severity: low
- [BUG-05] `searchRides.fn.ts` JSDoc on `queryAndCollect` and `queryInAndFilter` claims "When seats provided, adds availableSeats >= seats server-side" — seats are actually filtered in-memory via `filterRide()`, not at the query level | `searchRides.fn.ts` query helper JSDoc | Severity: low
- [BUG-06] Four `console.warn` calls in committed code violate app standards | `need-ride.tsx:222,228,302,343` | Severity: low
- [BUG-07] Resubmit lat/lng params default to `'0'` when absent — `parseFloat('0')` produces (0, 0) (Atlantic Ocean) without detection or guard | `need-ride.tsx:237-252` | Severity: low

### Improvements (should fix)

- [IMP-01] Recent Searches "to" address row uses `recentSearchFromText` style instead of `recentSearchToText` — both rows look identical (same weight, same color), should be visually distinct | `need-ride.tsx:716` | Impact: UX clarity
- [IMP-02] `SuggestedRides.tsx:85` uses string URL `router.push(\`/ride-details?rideId=...\`)` vs the object form used everywhere else — fragile with special chars in rideId | `SuggestedRides.tsx:85` | Impact: consistency/safety
- [IMP-03] Dead style definitions never referenced in JSX: 13 `routeAlerts*` styles and `recentSearchToText` | `need-ride.tsx:2099-2235, 1748` | Impact: code hygiene

### Notes
- Rate limiting (60 req/user) and auth + email verification enforced on both CFs — correct
- H3 spatial matching with progressive fallback is sound
- 15-minute client-side cache for search results is appropriate
- `getSuggestedRides` response omits `vehicleDetails`, `totalSeats`, `notes`, `status`, `relevanceScore` vs `SearchRideItem` type — not a runtime issue since `RideCard` doesn't render these fields, but `useState<SearchRideItem[]>` accepts the `any[]` silently
