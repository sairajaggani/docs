# Quality Audit — Design Spec
**Date:** 2026-06-14  
**Approach:** Sequential Deep Dive (audit → fix → next feature)

## Scope

Audit all core features for bugs, UX gaps, missing edge cases, validation holes, and security issues. Each feature covers both frontend (React Native/Expo) and backend (Cloud Functions + Firestore rules).

**In scope:**
- Post Ride
- Search Rides
- Booked Rides
- Posted Rides
- Profile
- Ratings
- Permissions (Firestore rules + auth middleware)

**Out of scope (already reviewed/in-progress):**
- Live Ride
- SOS
- Request Rides
- Browse Rides

## Audit Order

| # | Feature | FE Files | BE Functions |
|---|---|---|---|
| 1 | Post Ride | `app/tabs/post-ride.tsx` | `createRide`, `updateRide` CFs |
| 2 | Search Rides | `app/tabs/need-ride.tsx`, `components/SuggestedRides.tsx`, `hooks/useRideSearch.ts` | `searchRides`, `getSuggestedRides` CFs |
| 3 | Booked Rides | `components/rides/BookedRides.tsx`, `app/booked-ride-details.tsx` | `createBooking`, `confirmBooking`, `cancelBooking`, `completeBooking` CFs |
| 4 | Posted Rides | `app/tabs/scheduled-rides.tsx`, `components/rides/PostedRides.tsx`, `app/posted-ride-details.tsx`, `components/EditRideModal.tsx` | `cancelRide`, `startRide`, `completeRide`, `markPassengerNoShow`, `verifyPassengerOtp` CFs |
| 5 | Profile | `app/tabs/profile.tsx`, `app/profile/`, `app/update-profile.tsx`, `app/public-profile.tsx` | `createProfile`, `updateProfile`, `addVehicle`, `removeVehicle`, `uploadAvatar`, `removeAvatar`, `uploadID` CFs |
| 6 | Ratings | `app/rate-user.tsx` | `submitRating` CF |
| 7 | Permissions | `firestore.rules`, `database.rules.json`, `middleware/auth.middleware.ts`, `middleware/validation.middleware.ts` | — |

## Per-Feature Audit Checklist

For each feature, the auditing agent checks:

**Frontend:**
- [ ] Loading states present for all async operations
- [ ] Error handling covers all failure paths (network, auth, validation)
- [ ] Empty states handled (no results, no data)
- [ ] Input validation on forms matches backend expectations
- [ ] Navigation flows correct (back, success, error paths)
- [ ] Theme consistency (`theme.colors.*`, `theme.spacing.*`, no inline hex)
- [ ] No direct Firestore writes from screens
- [ ] Offline queue used where appropriate
- [ ] Race conditions (double-tap, rapid navigation)
- [ ] Missing field handling (null/undefined guards)

**Backend:**
- [ ] Input validation comprehensive (schema matches what FE sends)
- [ ] Auth checks on every CF that needs it
- [ ] Idempotency on write operations
- [ ] Rate limiting in place
- [ ] Error messages safe (no internal info leaked)
- [ ] Firestore reads/writes correct (no stale data, correct transaction usage)
- [ ] Edge cases handled (empty arrays, missing optional fields, boundary values)
- [ ] Logging adequate for debugging

**Cross-cutting:**
- [ ] FE and BE agree on field names and types
- [ ] Status transitions enforced on both sides
- [ ] Data consistency (no orphaned documents)

## Output Format Per Feature

Each audit produces a findings report at:
`docs/quality-audit/<feature>-findings.md`

Format:
```
## <Feature> — Audit Findings

### Bugs (must fix)
- [BUG-xx] Description | File:line | Severity: critical/high/medium/low

### Improvements (should fix)  
- [IMP-xx] Description | File | Impact

### Notes
- Anything notable but not actionable
```

After findings are written, the same session fixes all Bugs and high-impact Improvements, then commits.

## Session Management

- Each feature = one Claude Code session (to keep context clean)
- Start each session by reading this file + the findings file for context
- Commit after each feature is fully audited + fixed
- Update `docs/quality-audit/progress.md` after each feature

## Success Criteria

- All Bugs (critical + high) fixed across all 7 features
- All medium/low bugs documented and triaged
- Improvements implemented where feasible in-session
- All changes committed (no push)
