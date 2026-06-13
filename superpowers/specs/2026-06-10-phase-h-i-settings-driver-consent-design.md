# Phase H + I — Safety Settings & Driver Consent
**Date:** 2026-06-10  
**Status:** Approved — ready for implementation plan

---

## Scope

**Phase H:** Safety toggles in settings + one-time permission onboarding screen shown before first ride.  
**Phase I:** Driver Consent modal + `acceptDriverSafetyConsent` CF + `createRide` gate.

**Out of scope (dropped):** Auto-share toggle. Rationale: `ShareLiveRideButton` (Phase G) already covers manual sharing in one tap. Auto-share adds RTDB cost, battery drain, and contact notification fatigue on every ride without proportional safety gain. SOS (Phase F) covers the emergency case. No `autoShare` field anywhere.

---

## Data Model Changes

### `Carpool/src/types/models/profile.types.ts`

Add two new interfaces and extend `Profile`:

```ts
export interface SafetySettings {
  sosEnabled: boolean;   // default: true (absent = true)
}

export interface DriverSafetyConsent {
  acceptedAt: FirebaseFirestoreTypes.Timestamp;
  version: string;       // '1.0'
}
```

Add to `Profile` interface (both optional — no Firestore migration needed):
```ts
safetySettings?: SafetySettings;
driverSafetyConsent?: DriverSafetyConsent;
```

**Absence = defaults:** Code always reads `profile.safetySettings?.sosEnabled ?? true`. Never assume presence.

### AsyncStorage

Key `safety_permissions_onboarded` (string `'1'` when seen). Device-local only — it's a UI gate, not safety-critical data. Reinstall resets it; that's acceptable (user sees the educational screen again).

---

## Phase H: Safety Settings Toggles

### `updateSafetySettings` CF — new callable

**File:** `CarpoolBackend/functions/src/functions/safety/updateSafetySettings.fn.ts`

- **Input:** `{ sosEnabled: boolean }`
- **Auth:** required (`context.auth.uid`)
- **Action:** writes `safetySettings: { sosEnabled }` to `profiles/{uid}`
- **Idempotent:** yes — safe to call with same value
- **Rate limit:** none needed (preference write, low frequency)
- **Validation schema** (add to `validation.middleware.ts`):
  ```ts
  updateSafetySettings: z.object({
    sosEnabled: z.boolean(),
  })
  ```

### `app/settings.tsx` — Safety card changes

Extend the existing Safety card inline (Option A — no sub-screen). Below the "Trusted Contacts" row, add:

1. **Toggle row:** "Show SOS button during rides" / subtitle "Long-press to trigger emergency alert"  
   - Icon: `alert-circle-outline`, red tint background  
   - Value: `profile.safetySettings?.sosEnabled ?? true`  
   - On toggle: optimistic local state flip → call `updateSafetySettings` → revert on error via `showError`
   - No separate loading spinner — optimistic is standard mobile toggle UX

2. **Disclaimer text** below the card (not inside it):  
   `"SOS requires cellular signal. In dead zones, dial emergency services from any nearby phone."`  
   Style: `itemSub`-sized, tertiary text color, left-padded to align with card content.

The `SettingsItem` type cannot accommodate a toggle (it's nav-only). Add a separate `ToggleItem` type local to `settings.tsx` and a `renderToggleItem` function — do not mutate the existing `renderItem` function.

### `hooks/useSafetySettings.ts` — new hook

```ts
export function useSafetySettings(): { sosEnabled: boolean } {
  const { profile } = useProfile();
  return { sosEnabled: profile?.safetySettings?.sosEnabled ?? true };
}
```

**Consumed by:** `SosButton.tsx` — gate rendering on `sosEnabled`. If false, `SosButton` returns `null`.

### Permission Onboarding Screen

**File:** `Carpool/app/onboarding/safety-permissions.tsx`  
**Route:** `/onboarding/safety-permissions`  
**Register in:** `app/_layout.tsx` as a stack screen (no header — screen manages its own)

**Content (static, no OS permission requests here):**
- Shield icon + "Your safety, your control" heading
- Three permission cards: Location (Always Allow), Contacts, Notifications — copy verbatim from `todo.md §10`
- Warning banner: "SOS requires cellular signal. In dead zones, use any nearby phone."
- "Got it, continue" button → `AsyncStorage.setItem('safety_permissions_onboarded', '1')` → `router.back()`

**Cannot be dismissed** without tapping the button. Back gesture blocked via `usePreventRemove`.

### `hooks/useSafetyOnboarding.ts` — new hook

```ts
export function useSafetyOnboarding(): { onboardingComplete: boolean; loaded: boolean }
```

- Reads `AsyncStorage.getItem('safety_permissions_onboarded')` on mount
- Returns `{ loaded: false, onboardingComplete: false }` until AsyncStorage resolves — callers must not act until `loaded` is true
- After resolving: if absent → `router.push('/onboarding/safety-permissions')` and return `{ loaded: true, onboardingComplete: false }`
- If present → return `{ loaded: true, onboardingComplete: true }`
- AsyncStorage error → treat as complete (fail open: do not block the user on a storage error)

**Mount points:**
- `app/tabs/post-ride.tsx` — top of component; render nothing (or a spinner) until `loaded`
- `app/ride-details.tsx` — top of `RideDetails` component; inside `handleBookRide` check `onboardingComplete` as a closure variable and bail early if false (do not call the hook inside the function — hooks must be at component top level)

---

## Phase I: Driver Consent

### `acceptDriverSafetyConsent` CF — new callable

**File:** `CarpoolBackend/functions/src/functions/safety/acceptDriverSafetyConsent.fn.ts`

- **Input:** none
- **Auth:** required
- **Action:** writes `driverSafetyConsent: { acceptedAt: FieldValue.serverTimestamp(), version: '1.0' }` to `profiles/{uid}`
- **Idempotent:** yes — re-accepting is safe (overwrites with same data)
- **No validation schema** needed (no input)
- **No rate limit** needed

### `hooks/useDriverConsent.ts` — new hook

```ts
export function useDriverConsent(): { needsConsent: boolean }
```

- `needsConsent = true` when: profile has at least one vehicle AND `profile.driverSafetyConsent` is absent
- Reads live from `ProfileContext` — updates automatically after CF resolves and profile refreshes

### `components/DriverSafetyConsentModal.tsx` — new component

- `presentationStyle="pageSheet"` — iOS native sheet sliding up; on Android renders as full modal
- **Cannot be dismissed** without accepting: no X, back gesture blocked via `usePreventRemove`
- **Content:**
  - Title: "Driver Safety Agreement"
  - The full clause from `todo.md §11` verbatim
  - Link: "Read full Terms & Conditions" → `router.push('/terms-conditions')`
  - "I Agree" button → calls `acceptDriverSafetyConsent` CF → loading state → closes on success
  - Error: `showError(err)` + re-enable button
- **Props:** `visible: boolean` — always `true` when rendered, but retained so the modal can play its dismiss animation before unmounting (set `visible` to `false` briefly on success before unmounting)

**Mount point:** `app/tabs/post-ride.tsx` — rendered at the top:
```tsx
const { needsConsent } = useDriverConsent();
const [consentVisible, setConsentVisible] = useState(true);
// ...
{needsConsent && (
  <DriverSafetyConsentModal
    visible={consentVisible}
    onAccepted={() => setConsentVisible(false)}
  />
)}
```

The modal blocks the Post Ride tab completely until accepted. Profile refresh from `ProfileContext` after CF resolves will flip `needsConsent` to `false` and unmount the modal. No separate route needed.

### `createRide.fn.ts` — consent gate

Add after the existing vehicle check:
```ts
if (!profile.driverSafetyConsent?.acceptedAt) {
  throw new HttpsError(
    'failed-precondition',
    'Driver safety consent required.',
    { code: 'DRIVER_CONSENT_REQUIRED' }
  );
}
```

The FE modal prevents reaching this in normal flow. This is the API-level safety net.

### `app/terms-conditions.tsx` — clause addition

Add a "Safety Features" section with the clause from `todo.md §11`. No new screen or route.

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| `updateSafetySettings` CF fails | Revert optimistic toggle, `showError(err)` |
| `acceptDriverSafetyConsent` CF fails | Re-enable "I Agree" button, `showError(err)` |
| AsyncStorage read fails | Treat as `onboardingComplete: true` — fail open, do not block user on a storage error |
| `createRide` returns `DRIVER_CONSENT_REQUIRED` | FE shows `DriverSafetyConsentModal` — but this shouldn't be reachable via UI |

---

## File List

### Frontend (Carpool/)
| File | Action |
|---|---|
| `src/types/models/profile.types.ts` | Add `SafetySettings`, `DriverSafetyConsent`, extend `Profile` |
| `app/settings.tsx` | Add `ToggleItem` type + `renderToggleItem` + SOS toggle row + disclaimer |
| `app/onboarding/safety-permissions.tsx` | New screen |
| `app/terms-conditions.tsx` | Add Safety Features clause |
| `app/_layout.tsx` | Register `safety-permissions` stack screen |
| `app/tabs/post-ride.tsx` | Mount `useSafetyOnboarding` + `useDriverConsent` + `DriverSafetyConsentModal` |
| `app/ride-details.tsx` | Call `useSafetyOnboarding` in `handleBookRide` |
| `hooks/useSafetySettings.ts` | New hook |
| `hooks/useSafetyOnboarding.ts` | New hook |
| `hooks/useDriverConsent.ts` | New hook |
| `components/DriverSafetyConsentModal.tsx` | New component |
| `components/SosButton.tsx` | Gate render on `useSafetySettings().sosEnabled` |
| `src/services/firebase/functions.service.ts` | Add `updateSafetySettings`, `acceptDriverSafetyConsent` |

### Backend (CarpoolBackend/)
| File | Action |
|---|---|
| `functions/src/functions/safety/updateSafetySettings.fn.ts` | New CF |
| `functions/src/functions/safety/acceptDriverSafetyConsent.fn.ts` | New CF |
| `functions/src/functions/safety/index.ts` | Export both new CFs |
| `functions/src/index.ts` | Export both new CFs |
| `functions/src/functions/rides/createRide.fn.ts` | Add consent gate after vehicle check |
| `functions/src/middleware/validation.middleware.ts` | Add `updateSafetySettings` schema |

---

## Deploy Command

```bash
firebase deploy --only functions:updateSafetySettings,functions:acceptDriverSafetyConsent
```

`createRide` change requires redeploying that CF too:
```bash
firebase deploy --only functions:updateSafetySettings,functions:acceptDriverSafetyConsent,functions:createRide
```

---

## Testing

### Unit tests
- `useSafetySettings`: absent profile field → defaults to `true`
- `useDriverConsent`: no vehicles → `needsConsent: false`; vehicles + no consent → `true`; vehicles + consent → `false`
- `useSafetyOnboarding`: AsyncStorage absent → returns false + pushes route; present → returns true
- `DriverSafetyConsentModal`: renders, I Agree calls CF, error re-enables button, back gesture blocked
- `safety-permissions.tsx`: renders all 3 permission cards, button sets AsyncStorage + calls router.back

### Backend integration tests
- `updateSafetySettings`: valid input → writes to profile; unauthenticated → 401
- `acceptDriverSafetyConsent`: writes consent doc; idempotent (call twice = no error)
- `createRide`: missing consent + vehicles → `DRIVER_CONSENT_REQUIRED`; consent present → proceeds normally

---

## Acceptance Criteria

- [ ] SOS toggle in settings persists across app restart
- [ ] `SosButton` hidden when `sosEnabled = false`
- [ ] Safety permissions onboarding shown once before first ride booking/posting, never again
- [ ] `DriverSafetyConsentModal` shown on Post Ride tab when driver has vehicles but no consent
- [ ] Modal cannot be dismissed without accepting
- [ ] `createRide` CF rejects if consent missing (test via emulator)
- [ ] Terms & Conditions screen contains the new Safety Features clause
- [ ] Both new CFs deployed and visible in Firebase Console
