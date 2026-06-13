# Phase H + I — Safety Settings & Driver Consent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an SOS visibility toggle to settings, a one-time safety-permissions onboarding screen, and a driver consent modal + backend gate that blocks ride creation until consent is accepted.

**Architecture:** Two new callable Cloud Functions (`updateSafetySettings`, `acceptDriverSafetyConsent`) write to `profiles/{uid}`. Three thin hooks (`useSafetySettings`, `useSafetyOnboarding`, `useDriverConsent`) read from `ProfileContext` or `AsyncStorage`. The `createRide` CF gains a consent guard. The `DriverSafetyConsentModal` mounts inside the Post Ride tab and blocks it until the driver accepts. The permission onboarding screen is a full-screen route pushed once before the user's first ride action.

**Tech Stack:** React Native + Expo Router v6 + TypeScript + Firebase Functions v2 (Node 20 ESM) + Firestore + `@testing-library/react-native` + `firebase-functions-test` + Jest

---

## File Map

### New files — Backend
| File | Purpose |
|---|---|
| `CarpoolBackend/functions/src/functions/safety/updateSafetySettings.fn.ts` | Callable CF — writes `safetySettings` to profile |
| `CarpoolBackend/functions/src/functions/safety/acceptDriverSafetyConsent.fn.ts` | Callable CF — writes `driverSafetyConsent` to profile |
| `CarpoolBackend/functions/tests/integration/safety/safetySettings.test.ts` | Integration tests for both new CFs + createRide gate |

### Modified files — Backend
| File | Change |
|---|---|
| `CarpoolBackend/functions/src/utils/errors.ts` | Add `DRIVER_CONSENT_REQUIRED` error |
| `CarpoolBackend/functions/src/middleware/validation.middleware.ts` | Add `updateSafetySettings` schema |
| `CarpoolBackend/functions/src/functions/safety/index.ts` | Export both new CFs |
| `CarpoolBackend/functions/src/index.ts` | Export both new CFs |
| `CarpoolBackend/functions/src/functions/rides/createRide.fn.ts` | Add consent guard after vehicle check |

### New files — Frontend
| File | Purpose |
|---|---|
| `Carpool/hooks/useSafetySettings.ts` | Returns `{ sosEnabled }` from ProfileContext |
| `Carpool/hooks/useSafetyOnboarding.ts` | AsyncStorage gate — pushes to onboarding screen if not yet seen |
| `Carpool/hooks/useDriverConsent.ts` | Returns `{ needsConsent }` for drivers without consent |
| `Carpool/components/DriverSafetyConsentModal.tsx` | Full-screen modal blocking Post Ride tab until consent |
| `Carpool/app/onboarding/safety-permissions.tsx` | One-time educational screen before first ride |
| `Carpool/src/__tests__/hooks/useSafetySettings.test.ts` | Unit tests |
| `Carpool/src/__tests__/hooks/useSafetyOnboarding.test.ts` | Unit tests |
| `Carpool/src/__tests__/hooks/useDriverConsent.test.ts` | Unit tests |
| `Carpool/src/__tests__/screens/safety-permissions.test.tsx` | Component tests |
| `Carpool/src/__tests__/components/DriverSafetyConsentModal.test.tsx` | Component tests |

### Modified files — Frontend
| File | Change |
|---|---|
| `Carpool/src/types/models/profile.types.ts` | Add `SafetySettings`, `DriverSafetyConsent`, extend `Profile` |
| `Carpool/src/services/firebase/functions.service.ts` | Add `updateSafetySettings`, `acceptDriverSafetyConsent` wrappers |
| `Carpool/app/settings.tsx` | Add `ToggleItem` type + `renderToggleItem` + SOS toggle + disclaimer |
| `Carpool/app/terms-conditions.tsx` | Add Safety Features clause |
| `Carpool/app/_layout.tsx` | Register `onboarding/safety-permissions` stack screen |
| `Carpool/app/tabs/post-ride.tsx` | Mount `useSafetyOnboarding` + `useDriverConsent` + `DriverSafetyConsentModal` |
| `Carpool/app/ride-details.tsx` | Mount `useSafetyOnboarding`, bail in `handleBookRide` if not complete |
| `Carpool/components/SosButton.tsx` | Gate render on `useSafetySettings().sosEnabled` |

---

## Task 1: Data model — profile types + errors.ts

**Files:**
- Modify: `Carpool/src/types/models/profile.types.ts`
- Modify: `CarpoolBackend/functions/src/utils/errors.ts`

- [ ] **Step 1: Add SafetySettings + DriverSafetyConsent to profile.types.ts**

Open `Carpool/src/types/models/profile.types.ts`. After the `NotificationPreferences` interface and before `UserPreferences`, add:

```ts
export interface SafetySettings {
  sosEnabled: boolean;
}

export interface DriverSafetyConsent {
  acceptedAt: FirebaseFirestoreTypes.Timestamp;
  version: string;
}
```

Then in the `Profile` interface, after the `preferences` field, add:

```ts
  safetySettings?: SafetySettings;
  driverSafetyConsent?: DriverSafetyConsent;
```

Both are optional — existing profile docs without these fields are valid.

- [ ] **Step 2: Add DRIVER_CONSENT_REQUIRED to errors.ts**

Open `CarpoolBackend/functions/src/utils/errors.ts`. After the `DUPLICATE_RATING` entry, add:

```ts
  DRIVER_CONSENT_REQUIRED: () => new FunctionError(
    "DRIVER_CONSENT_REQUIRED",
    "Driver safety consent is required before posting rides.",
    428
  ),
```

- [ ] **Step 3: Type-check frontend**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add src/types/models/profile.types.ts
git commit -m "feat(types): add SafetySettings and DriverSafetyConsent to Profile"

cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/utils/errors.ts
git commit -m "feat(errors): add DRIVER_CONSENT_REQUIRED error"
```

---

## Task 2: `updateSafetySettings` CF + validation schema + integration test

**Files:**
- Create: `CarpoolBackend/functions/src/functions/safety/updateSafetySettings.fn.ts`
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts`
- Create: `CarpoolBackend/functions/tests/integration/safety/safetySettings.test.ts`

- [ ] **Step 1: Add validation schema**

Open `CarpoolBackend/functions/src/middleware/validation.middleware.ts`. After the `resolveSos` schema entry, add:

```ts
  updateSafetySettings: z.object({
    sosEnabled: z.boolean(),
  }),
```

- [ ] **Step 2: Write the failing integration test**

Create `CarpoolBackend/functions/tests/integration/safety/safetySettings.test.ts`:

```ts
import { describe, it, expect, beforeAll, afterEach } from '@jest/globals';
import firebaseTest from 'firebase-functions-test';
import {
  initializeTestFirebase,
  clearAllTestData,
  createTestUser,
  createTestProfile,
} from '../../helpers/firebase-test.js';
import { db } from '../../../src/config/firebase-admin.js';
import { CONSTANTS } from '../../../src/config/constants.js';

import { updateSafetySettings as updateSafetySettingsRaw } from '../../../src/functions/safety/updateSafetySettings.fn.js';
import { acceptDriverSafetyConsent as acceptDriverSafetyConsentRaw } from '../../../src/functions/safety/acceptDriverSafetyConsent.fn.js';

const testEnv = firebaseTest();
const updateSafetySettings = testEnv.wrap(updateSafetySettingsRaw);
const acceptDriverSafetyConsent = testEnv.wrap(acceptDriverSafetyConsentRaw);

const ctx = (uid: string, email = 'test@example.com') => ({
  auth: { uid, token: { email, email_verified: true } },
});

async function makeUser(email = 'user@example.com') {
  const user = await createTestUser(email);
  await createTestProfile(user.uid, user.email!, {
    firstName: 'Test',
    lastName: 'User',
    phone: '+14155550100',
  });
  return { user, ctx: ctx(user.uid, user.email!) };
}

describe('updateSafetySettings', () => {
  beforeAll(() => { initializeTestFirebase(); });
  afterEach(async () => { await clearAllTestData(); });

  it('writes sosEnabled: false to the profile', async () => {
    const { user, ctx: userCtx } = await makeUser();

    await updateSafetySettings({ sosEnabled: false }, userCtx);

    const doc = await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(user.uid).get();
    expect(doc.data()?.safetySettings?.sosEnabled).toBe(false);
  });

  it('writes sosEnabled: true to the profile', async () => {
    const { user, ctx: userCtx } = await makeUser();

    await updateSafetySettings({ sosEnabled: true }, userCtx);

    const doc = await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(user.uid).get();
    expect(doc.data()?.safetySettings?.sosEnabled).toBe(true);
  });

  it('is idempotent — calling twice with same value does not error', async () => {
    const { ctx: userCtx } = await makeUser();

    await updateSafetySettings({ sosEnabled: false }, userCtx);
    await expect(updateSafetySettings({ sosEnabled: false }, userCtx)).resolves.not.toThrow();
  });

  it('rejects unauthenticated calls', async () => {
    await expect(
      updateSafetySettings({ sosEnabled: false }, { auth: null })
    ).rejects.toThrow();
  });

  it('rejects missing sosEnabled field', async () => {
    const { ctx: userCtx } = await makeUser();
    await expect(
      updateSafetySettings({}, userCtx)
    ).rejects.toThrow();
  });
});
```

- [ ] **Step 3: Run test — verify it fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings
```

Expected: FAIL — `updateSafetySettings.fn.js` module not found.

- [ ] **Step 4: Implement `updateSafetySettings.fn.ts`**

Create `CarpoolBackend/functions/src/functions/safety/updateSafetySettings.fn.ts`:

```ts
import * as functions from "firebase-functions";
import { db } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { ResponseFormatter } from "../../utils/response.js";
import { authenticateUser, requireEmailVerification } from "../../middleware/auth.middleware.js";
import { validateData, schemas } from "../../middleware/validation.middleware.js";
import { withErrorHandler } from "../../middleware/error-handler.middleware.js";
import { withLogging } from "../../middleware/logging.middleware.js";

export const updateSafetySettings = functions.https.onCall(
  withErrorHandler(async (data, context) => {
    return withLogging("updateSafetySettings", async (data, authContext) => {
      requireEmailVerification(authContext);

      const { sosEnabled } = validateData(schemas.updateSafetySettings, data);
      const { uid } = authContext.extendedAuth;

      await db
        .collection(CONSTANTS.COLLECTIONS.PROFILES)
        .doc(uid)
        .update({ safetySettings: { sosEnabled } });

      return ResponseFormatter.success({});
    })(data, context);
  })
);
```

- [ ] **Step 5: Run test — verify it passes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings --testNamePattern="updateSafetySettings"
```

Expected: 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/safety/updateSafetySettings.fn.ts \
        functions/src/middleware/validation.middleware.ts \
        functions/tests/integration/safety/safetySettings.test.ts
git commit -m "feat(safety): add updateSafetySettings CF with integration tests"
```

---

## Task 3: `acceptDriverSafetyConsent` CF + integration tests

**Files:**
- Create: `CarpoolBackend/functions/src/functions/safety/acceptDriverSafetyConsent.fn.ts`
- Modify: `CarpoolBackend/functions/tests/integration/safety/safetySettings.test.ts` (add describe block)

- [ ] **Step 1: Add acceptDriverSafetyConsent describe block to the test file**

Append to `CarpoolBackend/functions/tests/integration/safety/safetySettings.test.ts` (after the closing `});` of the updateSafetySettings describe):

```ts
describe('acceptDriverSafetyConsent', () => {
  beforeAll(() => { initializeTestFirebase(); });
  afterEach(async () => { await clearAllTestData(); });

  it('writes driverSafetyConsent with version 1.0 to the profile', async () => {
    const { user, ctx: userCtx } = await makeUser();

    await acceptDriverSafetyConsent({}, userCtx);

    const doc = await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(user.uid).get();
    const consent = doc.data()?.driverSafetyConsent;
    expect(consent).toBeDefined();
    expect(consent.version).toBe('1.0');
    expect(consent.acceptedAt).toBeDefined();
  });

  it('is idempotent — calling twice does not error', async () => {
    const { ctx: userCtx } = await makeUser();

    await acceptDriverSafetyConsent({}, userCtx);
    await expect(acceptDriverSafetyConsent({}, userCtx)).resolves.not.toThrow();
  });

  it('rejects unauthenticated calls', async () => {
    await expect(
      acceptDriverSafetyConsent({}, { auth: null })
    ).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Run test — verify new tests fail**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings --testNamePattern="acceptDriverSafetyConsent"
```

Expected: FAIL — `acceptDriverSafetyConsent.fn.js` not found.

- [ ] **Step 3: Implement `acceptDriverSafetyConsent.fn.ts`**

Create `CarpoolBackend/functions/src/functions/safety/acceptDriverSafetyConsent.fn.ts`:

```ts
import * as functions from "firebase-functions";
import { Timestamp } from "firebase-admin/firestore";
import { db } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { ResponseFormatter } from "../../utils/response.js";
import { requireEmailVerification } from "../../middleware/auth.middleware.js";
import { withErrorHandler } from "../../middleware/error-handler.middleware.js";
import { withLogging } from "../../middleware/logging.middleware.js";

export const acceptDriverSafetyConsent = functions.https.onCall(
  withErrorHandler(async (data, context) => {
    return withLogging("acceptDriverSafetyConsent", async (_data, authContext) => {
      requireEmailVerification(authContext);

      const { uid } = authContext.extendedAuth;

      await db
        .collection(CONSTANTS.COLLECTIONS.PROFILES)
        .doc(uid)
        .update({
          driverSafetyConsent: {
            acceptedAt: Timestamp.now(),
            version: '1.0',
          },
        });

      return ResponseFormatter.success({});
    })(data, context);
  })
);
```

- [ ] **Step 4: Run all safetySettings tests — verify they pass**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings
```

Expected: all 8 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/safety/acceptDriverSafetyConsent.fn.ts \
        functions/tests/integration/safety/safetySettings.test.ts
git commit -m "feat(safety): add acceptDriverSafetyConsent CF with integration tests"
```

---

## Task 4: `createRide` consent gate + integration test

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/createRide.fn.ts`
- Modify: `CarpoolBackend/functions/tests/integration/safety/safetySettings.test.ts` (add describe block)

- [ ] **Step 1: Add createRide consent gate tests**

You need to import `createRide` in the test file. Add at the top of `safetySettings.test.ts` with the other imports:

```ts
import { createRide as createRideRaw } from '../../../src/functions/rides/createRide.fn.js';
const createRide = testEnv.wrap(createRideRaw);
```

Then append to the test file a new describe block:

```ts
describe('createRide — driver consent gate', () => {
  beforeAll(() => { initializeTestFirebase(); });
  afterEach(async () => { await clearAllTestData(); });

  async function makeDriverWithVehicle(email = 'driver@example.com') {
    const user = await createTestUser(email);
    await createTestProfile(user.uid, user.email!, {
      firstName: 'Driver',
      lastName: 'User',
      phone: '+14155550200',
      vehicles: [{
        make: 'Toyota',
        model: 'Camry',
        color: 'White',
        plate: 'TEST123',
        year: 2020,
        capacity: 3,
      }],
    });
    return { user, ctx: ctx(user.uid, user.email!) };
  }

  const validRideInput = {
    from: { address: 'Mumbai', lat: 19.076, lng: 72.877, plusCode: 'ABC' },
    to: { address: 'Pune', lat: 18.52, lng: 73.856, plusCode: 'DEF' },
    rideDate: '2026-12-01',
    rideStartTime: 1764547200000,
    rideEndTime: 1764554400000,
    isTimeRange: false,
    price: 300,
    currency: 'INR',
    seats: 2,
    notes: '',
    vehicleIndex: 0,
  };

  it('throws DRIVER_CONSENT_REQUIRED when driver has no consent', async () => {
    const { ctx: driverCtx } = await makeDriverWithVehicle();

    await expect(
      createRide(validRideInput, driverCtx)
    ).rejects.toMatchObject({ code: 'DRIVER_CONSENT_REQUIRED' });
  });

  it('proceeds normally when driver has consent', async () => {
    const { user, ctx: driverCtx } = await makeDriverWithVehicle('driver2@example.com');

    await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(user.uid).update({
      driverSafetyConsent: { acceptedAt: Timestamp.now(), version: '1.0' },
    });

    await expect(createRide(validRideInput, driverCtx)).resolves.not.toThrow();
  });
});
```

Also add `import { Timestamp } from 'firebase-admin/firestore';` to the test file imports.

- [ ] **Step 2: Run test — verify consent gate test fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings --testNamePattern="createRide"
```

Expected: FAIL — `DRIVER_CONSENT_REQUIRED` not thrown (gate not yet added).

- [ ] **Step 3: Add consent gate to createRide.fn.ts**

Open `CarpoolBackend/functions/src/functions/rides/createRide.fn.ts`. At the top, add `Errors` to the existing import (it's already imported — just verify). Then after the vehicle existence check (line ~59, after `throw Errors.INVALID_INPUT({ message: "Please add at least one vehicle..." })`), insert:

```ts
      // Validate driver has accepted safety consent (Phase I)
      if (!profile.driverSafetyConsent?.acceptedAt) {
        throw Errors.DRIVER_CONSENT_REQUIRED();
      }
```

The surrounding context will look like:

```ts
      // Validate user has vehicles
      if (profile.vehicles.length === 0) {
        throw Errors.INVALID_INPUT({
          message: "Please add at least one vehicle before posting a ride",
        });
      }

      // Validate driver has accepted safety consent (Phase I)
      if (!profile.driverSafetyConsent?.acceptedAt) {
        throw Errors.DRIVER_CONSENT_REQUIRED();
      }

      // Validate vehicle index
      if (validatedData.vehicleIndex >= profile.vehicles.length) {
```

- [ ] **Step 4: Run all safetySettings tests — verify all pass**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings
```

Expected: all 11 tests PASS.

- [ ] **Step 5: Type-check backend**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 6: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/rides/createRide.fn.ts \
        functions/tests/integration/safety/safetySettings.test.ts
git commit -m "feat(safety): gate createRide on driver safety consent"
```

---

## Task 5: Export new CFs + wire `functions.service.ts`

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/index.ts`
- Modify: `CarpoolBackend/functions/src/index.ts`
- Modify: `Carpool/src/services/firebase/functions.service.ts`

- [ ] **Step 1: Export from safety/index.ts**

Open `CarpoolBackend/functions/src/functions/safety/index.ts`. Add two exports:

```ts
export { updateSafetySettings } from "./updateSafetySettings.fn.js";
export { acceptDriverSafetyConsent } from "./acceptDriverSafetyConsent.fn.js";
```

- [ ] **Step 2: Export from main index.ts**

Open `CarpoolBackend/functions/src/index.ts`. Find where the safety functions are exported (search for `triggerSos` export). Add alongside them:

```ts
export { updateSafetySettings } from "./functions/safety/index.js";
export { acceptDriverSafetyConsent } from "./functions/safety/index.js";
```

- [ ] **Step 3: Add CF wrappers to functions.service.ts**

Open `Carpool/src/services/firebase/functions.service.ts`. After the `resolveSos` method (around line 625), add:

```ts
  async updateSafetySettings(data: { sosEnabled: boolean }): Promise<void> {
    return this.callFunction('updateSafetySettings', data, { retry: false });
  }

  async acceptDriverSafetyConsent(): Promise<void> {
    return this.callFunction('acceptDriverSafetyConsent', {}, { retry: false });
  }
```

- [ ] **Step 4: Type-check both repos**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npx tsc --noEmit
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: no errors in either.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/safety/index.ts functions/src/index.ts
git commit -m "feat(safety): export updateSafetySettings and acceptDriverSafetyConsent"

cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add src/services/firebase/functions.service.ts
git commit -m "feat(functions): add updateSafetySettings and acceptDriverSafetyConsent wrappers"
```

---

## Task 6: `useSafetySettings` hook + unit test

**Files:**
- Create: `Carpool/hooks/useSafetySettings.ts`
- Create: `Carpool/src/__tests__/hooks/useSafetySettings.test.ts`

- [ ] **Step 1: Write the failing test**

Create `Carpool/src/__tests__/hooks/useSafetySettings.test.ts`:

```ts
import { renderHook } from '@testing-library/react-native';
import { useSafetySettings } from '../../../hooks/useSafetySettings';

jest.mock('../../../context/ProfileContext', () => ({
  useProfile: jest.fn(),
}));

import { useProfile } from '../../../context/ProfileContext';

describe('useSafetySettings', () => {
  beforeEach(() => { jest.clearAllMocks(); });

  it('returns sosEnabled: true when profile has no safetySettings (default)', () => {
    (useProfile as jest.Mock).mockReturnValue({ profile: { uid: 'u1' } });
    const { result } = renderHook(() => useSafetySettings());
    expect(result.current.sosEnabled).toBe(true);
  });

  it('returns sosEnabled: false when profile.safetySettings.sosEnabled is false', () => {
    (useProfile as jest.Mock).mockReturnValue({
      profile: { uid: 'u1', safetySettings: { sosEnabled: false } },
    });
    const { result } = renderHook(() => useSafetySettings());
    expect(result.current.sosEnabled).toBe(false);
  });

  it('returns sosEnabled: true when profile.safetySettings.sosEnabled is true', () => {
    (useProfile as jest.Mock).mockReturnValue({
      profile: { uid: 'u1', safetySettings: { sosEnabled: true } },
    });
    const { result } = renderHook(() => useSafetySettings());
    expect(result.current.sosEnabled).toBe(true);
  });

  it('returns sosEnabled: true when profile is null', () => {
    (useProfile as jest.Mock).mockReturnValue({ profile: null });
    const { result } = renderHook(() => useSafetySettings());
    expect(result.current.sosEnabled).toBe(true);
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/hooks/useSafetySettings.test.ts
```

Expected: FAIL — `useSafetySettings` module not found.

- [ ] **Step 3: Implement the hook**

Create `Carpool/hooks/useSafetySettings.ts`:

```ts
import { useProfile } from '../context/ProfileContext';

export function useSafetySettings(): { sosEnabled: boolean } {
  const { profile } = useProfile();
  return { sosEnabled: profile?.safetySettings?.sosEnabled ?? true };
}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/hooks/useSafetySettings.test.ts
```

Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add hooks/useSafetySettings.ts src/__tests__/hooks/useSafetySettings.test.ts
git commit -m "feat(hooks): add useSafetySettings hook"
```

---

## Task 7: `useSafetyOnboarding` hook + unit test

**Files:**
- Create: `Carpool/hooks/useSafetyOnboarding.ts`
- Create: `Carpool/src/__tests__/hooks/useSafetyOnboarding.test.ts`

- [ ] **Step 1: Write the failing test**

Create `Carpool/src/__tests__/hooks/useSafetyOnboarding.test.ts`:

```ts
import { renderHook, act, waitFor } from '@testing-library/react-native';
import { useSafetyOnboarding } from '../../../hooks/useSafetyOnboarding';

const mockPush = jest.fn();
jest.mock('expo-router', () => ({ useRouter: () => ({ push: mockPush }) }));

const mockGetItem = jest.fn();
jest.mock('@react-native-async-storage/async-storage', () => ({
  getItem: (...args: unknown[]) => mockGetItem(...args),
}));

describe('useSafetyOnboarding', () => {
  beforeEach(() => { jest.clearAllMocks(); });

  it('returns loaded:false, onboardingComplete:false initially', () => {
    mockGetItem.mockReturnValue(new Promise(() => {})); // never resolves
    const { result } = renderHook(() => useSafetyOnboarding());
    expect(result.current.loaded).toBe(false);
    expect(result.current.onboardingComplete).toBe(false);
  });

  it('returns loaded:true, onboardingComplete:true and does NOT push when flag is set', async () => {
    mockGetItem.mockResolvedValue('1');
    const { result } = renderHook(() => useSafetyOnboarding());
    await waitFor(() => expect(result.current.loaded).toBe(true));
    expect(result.current.onboardingComplete).toBe(true);
    expect(mockPush).not.toHaveBeenCalled();
  });

  it('returns loaded:true, onboardingComplete:false and pushes onboarding when flag absent', async () => {
    mockGetItem.mockResolvedValue(null);
    const { result } = renderHook(() => useSafetyOnboarding());
    await waitFor(() => expect(result.current.loaded).toBe(true));
    expect(result.current.onboardingComplete).toBe(false);
    expect(mockPush).toHaveBeenCalledWith('/onboarding/safety-permissions');
  });

  it('fails open on AsyncStorage error (returns onboardingComplete:true, no push)', async () => {
    mockGetItem.mockRejectedValue(new Error('Storage error'));
    const { result } = renderHook(() => useSafetyOnboarding());
    await waitFor(() => expect(result.current.loaded).toBe(true));
    expect(result.current.onboardingComplete).toBe(true);
    expect(mockPush).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/hooks/useSafetyOnboarding.test.ts
```

Expected: FAIL — `useSafetyOnboarding` module not found.

- [ ] **Step 3: Implement the hook**

Create `Carpool/hooks/useSafetyOnboarding.ts`:

```ts
import { useState, useEffect } from 'react';
import { useRouter } from 'expo-router';
import AsyncStorage from '@react-native-async-storage/async-storage';

const STORAGE_KEY = 'safety_permissions_onboarded';

export function useSafetyOnboarding(): { loaded: boolean; onboardingComplete: boolean } {
  const [state, setState] = useState({ loaded: false, onboardingComplete: false });
  const router = useRouter();

  useEffect(() => {
    AsyncStorage.getItem(STORAGE_KEY)
      .then((value) => {
        if (value === '1') {
          setState({ loaded: true, onboardingComplete: true });
        } else {
          setState({ loaded: true, onboardingComplete: false });
          router.push('/onboarding/safety-permissions');
        }
      })
      .catch(() => {
        setState({ loaded: true, onboardingComplete: true });
      });
  }, []);

  return state;
}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/hooks/useSafetyOnboarding.test.ts
```

Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add hooks/useSafetyOnboarding.ts src/__tests__/hooks/useSafetyOnboarding.test.ts
git commit -m "feat(hooks): add useSafetyOnboarding hook"
```

---

## Task 8: `useDriverConsent` hook + unit test

**Files:**
- Create: `Carpool/hooks/useDriverConsent.ts`
- Create: `Carpool/src/__tests__/hooks/useDriverConsent.test.ts`

- [ ] **Step 1: Write the failing test**

Create `Carpool/src/__tests__/hooks/useDriverConsent.test.ts`:

```ts
import { renderHook } from '@testing-library/react-native';
import { useDriverConsent } from '../../../hooks/useDriverConsent';

jest.mock('../../../context/ProfileContext', () => ({
  useProfile: jest.fn(),
}));
import { useProfile } from '../../../context/ProfileContext';

const vehicleStub = { make: 'Toyota', model: 'Camry', color: 'White', plate: 'ABC123' };

describe('useDriverConsent', () => {
  beforeEach(() => { jest.clearAllMocks(); });

  it('needsConsent: false when profile has no vehicles', () => {
    (useProfile as jest.Mock).mockReturnValue({ profile: { uid: 'u1', vehicles: [] } });
    const { result } = renderHook(() => useDriverConsent());
    expect(result.current.needsConsent).toBe(false);
  });

  it('needsConsent: true when profile has vehicles but no driverSafetyConsent', () => {
    (useProfile as jest.Mock).mockReturnValue({
      profile: { uid: 'u1', vehicles: [vehicleStub] },
    });
    const { result } = renderHook(() => useDriverConsent());
    expect(result.current.needsConsent).toBe(true);
  });

  it('needsConsent: false when profile has vehicles AND driverSafetyConsent', () => {
    (useProfile as jest.Mock).mockReturnValue({
      profile: {
        uid: 'u1',
        vehicles: [vehicleStub],
        driverSafetyConsent: { acceptedAt: { seconds: 1700000000, nanoseconds: 0 }, version: '1.0' },
      },
    });
    const { result } = renderHook(() => useDriverConsent());
    expect(result.current.needsConsent).toBe(false);
  });

  it('needsConsent: false when profile is null', () => {
    (useProfile as jest.Mock).mockReturnValue({ profile: null });
    const { result } = renderHook(() => useDriverConsent());
    expect(result.current.needsConsent).toBe(false);
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/hooks/useDriverConsent.test.ts
```

Expected: FAIL — `useDriverConsent` module not found.

- [ ] **Step 3: Implement the hook**

Create `Carpool/hooks/useDriverConsent.ts`:

```ts
import { useProfile } from '../context/ProfileContext';

export function useDriverConsent(): { needsConsent: boolean } {
  const { profile } = useProfile();
  const needsConsent = Boolean(
    profile?.vehicles?.length &&
    !profile?.driverSafetyConsent?.acceptedAt
  );
  return { needsConsent };
}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/hooks/useDriverConsent.test.ts
```

Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add hooks/useDriverConsent.ts src/__tests__/hooks/useDriverConsent.test.ts
git commit -m "feat(hooks): add useDriverConsent hook"
```

---

## Task 9: `settings.tsx` SOS toggle

**Files:**
- Modify: `Carpool/app/settings.tsx`

- [ ] **Step 1: Add ToggleItem type and renderToggleItem**

Open `Carpool/app/settings.tsx`. After the `SettingsItem` type definition, add:

```ts
type ToggleItem = {
    icon: keyof typeof Ionicons.glyphMap;
    label: string;
    sub?: string;
    value: boolean;
    onValueChange: (value: boolean) => void;
    iconBg?: string;
};
```

After the `renderItem` function, add:

```ts
    const renderToggleItem = (item: ToggleItem, index: number, isLast: boolean) => (
        <View
            key={index}
            style={[styles.item, !isLast && styles.itemBorder]}
        >
            <View style={[styles.iconWrap, item.iconBg ? { backgroundColor: item.iconBg } : null]}>
                <Ionicons name={item.icon} size={20} color={theme.colors.error} />
            </View>
            <View style={styles.itemContent}>
                <Text style={styles.itemLabel}>{item.label}</Text>
                {item.sub && <Text style={styles.itemSub}>{item.sub}</Text>}
            </View>
            <Switch
                value={item.value}
                onValueChange={item.onValueChange}
                trackColor={{ false: theme.colors.border, true: theme.colors.primary }}
                thumbColor={theme.colors.surface}
                ios_backgroundColor={theme.colors.border}
            />
        </View>
    );
```

- [ ] **Step 2: Add Switch import and state + CF call**

At the top of the file, add `Switch` to the React Native imports and add these imports:

```ts
import { useSafetySettings } from '../hooks/useSafetySettings';
import { functionsService } from '../src/services/firebase/functions.service';
```

Inside `SettingsScreen`, add state and handler after the existing `useState(false)` for `loggingOut`:

```ts
    const { sosEnabled } = useSafetySettings();
    const [localSosEnabled, setLocalSosEnabled] = useState(sosEnabled);
    const [savingSos, setSavingSos] = useState(false);

    const handleSosToggle = async (value: boolean) => {
        setLocalSosEnabled(value);
        setSavingSos(true);
        try {
            await functionsService.updateSafetySettings({ sosEnabled: value });
        } catch (err) {
            setLocalSosEnabled(!value);
            showError('Failed to save setting', String(err));
        } finally {
            setSavingSos(false);
        }
    };
```

- [ ] **Step 3: Wire toggle into Safety card**

In `SettingsScreen`, replace the entire Safety section render (the `safetyItems` map inside the Safety card) with:

```tsx
                {/* Safety Section */}
                <Text style={styles.sectionLabel}>SAFETY</Text>
                <View style={styles.card}>
                    {safetyItems.map((item, i) =>
                        renderItem(item, i, false),
                    )}
                    {renderToggleItem(
                        {
                            icon: 'alert-circle-outline',
                            label: 'Show SOS button during rides',
                            sub: 'Long-press to trigger emergency alert',
                            value: localSosEnabled,
                            onValueChange: handleSosToggle,
                            iconBg: theme.colors.error + '15',
                        },
                        0,
                        true,
                    )}
                </View>
                <Text style={[styles.itemSub, styles.safetyDisclaimer]}>
                    SOS requires cellular signal. In dead zones, dial emergency services from any nearby phone.
                </Text>
```

- [ ] **Step 4: Add disclaimer style**

In the `StyleSheet.create` at the bottom, add:

```ts
    safetyDisclaimer: {
        marginHorizontal: theme.spacing.lg,
        marginTop: theme.spacing.xs,
        marginBottom: theme.spacing.sm,
    },
```

- [ ] **Step 5: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 6: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add app/settings.tsx
git commit -m "feat(settings): add SOS toggle and safety disclaimer to Safety section"
```

---

## Task 10: `safety-permissions.tsx` screen + `_layout.tsx` registration

**Files:**
- Create: `Carpool/app/onboarding/safety-permissions.tsx`
- Modify: `Carpool/app/_layout.tsx`
- Create: `Carpool/src/__tests__/screens/safety-permissions.test.tsx`

- [ ] **Step 1: Write the failing test**

Create `Carpool/src/__tests__/screens/safety-permissions.test.tsx`:

```tsx
import React from 'react';
import { render, fireEvent, waitFor } from '@testing-library/react-native';
import SafetyPermissionsScreen from '../../../../app/onboarding/safety-permissions';

const mockBack = jest.fn();
const mockSetItem = jest.fn();

jest.mock('expo-router', () => ({ useRouter: () => ({ back: mockBack }) }));
jest.mock('@react-native-async-storage/async-storage', () => ({
  setItem: (...args: unknown[]) => mockSetItem(...args),
}));

describe('SafetyPermissionsScreen', () => {
  beforeEach(() => { jest.clearAllMocks(); });

  it('renders location permission card', () => {
    const { getByText } = render(<SafetyPermissionsScreen />);
    expect(getByText(/Location/i)).toBeTruthy();
  });

  it('renders contacts permission card', () => {
    const { getByText } = render(<SafetyPermissionsScreen />);
    expect(getByText(/Contacts/i)).toBeTruthy();
  });

  it('renders notifications permission card', () => {
    const { getByText } = render(<SafetyPermissionsScreen />);
    expect(getByText(/Notifications/i)).toBeTruthy();
  });

  it('renders the cellular signal warning', () => {
    const { getByText } = render(<SafetyPermissionsScreen />);
    expect(getByText(/cellular signal/i)).toBeTruthy();
  });

  it('tapping "Got it, continue" sets AsyncStorage flag and calls router.back', async () => {
    mockSetItem.mockResolvedValue(undefined);
    const { getByText } = render(<SafetyPermissionsScreen />);

    fireEvent.press(getByText(/Got it, continue/i));

    await waitFor(() => {
      expect(mockSetItem).toHaveBeenCalledWith('safety_permissions_onboarded', '1');
      expect(mockBack).toHaveBeenCalled();
    });
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/screens/safety-permissions.test.tsx
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement the screen**

Create `Carpool/app/onboarding/safety-permissions.tsx`:

```tsx
import { Ionicons } from '@expo/vector-icons';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { useRouter } from 'expo-router';
import { ScrollView, StyleSheet, Text, TouchableOpacity, View } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';

import { theme } from '../../constants/theme';

const STORAGE_KEY = 'safety_permissions_onboarded';

export default function SafetyPermissionsScreen() {
    const router = useRouter();

    const handleContinue = async () => {
        await AsyncStorage.setItem(STORAGE_KEY, '1');
        router.back();
    };

    return (
        <SafeAreaView style={styles.safeArea} edges={['top', 'bottom']}>
            <ScrollView contentContainerStyle={styles.content} showsVerticalScrollIndicator={false}>
                <View style={styles.hero}>
                    <Text style={styles.heroIcon}>🛡️</Text>
                    <Text style={styles.title}>Your safety, your control</Text>
                    <Text style={styles.subtitle}>
                        Carpool uses a few permissions to keep you safe during rides
                    </Text>
                </View>

                <View style={styles.cards}>
                    <View style={styles.card}>
                        <Text style={styles.cardIcon}>📍</Text>
                        <View style={styles.cardContent}>
                            <Text style={styles.cardTitle}>Location (Always Allow)</Text>
                            <Text style={styles.cardBody}>
                                Required to send your live location to your trusted contacts during SOS
                                or ride share. We never share location outside an active SOS or share session.
                            </Text>
                        </View>
                    </View>

                    <View style={styles.card}>
                        <Text style={styles.cardIcon}>👥</Text>
                        <View style={styles.cardContent}>
                            <Text style={styles.cardTitle}>Contacts</Text>
                            <Text style={styles.cardBody}>
                                Used only when you tap "Add trusted contact". Nothing is uploaded automatically.
                            </Text>
                        </View>
                    </View>

                    <View style={styles.card}>
                        <Text style={styles.cardIcon}>🔔</Text>
                        <View style={styles.cardContent}>
                            <Text style={styles.cardTitle}>Notifications</Text>
                            <Text style={styles.cardBody}>
                                So your trusted contacts get alerted instantly when you trigger SOS.
                            </Text>
                        </View>
                    </View>
                </View>

                <View style={styles.warning}>
                    <Ionicons name="warning-outline" size={16} color={theme.colors.warning} />
                    <Text style={styles.warningText}>
                        SOS requires cellular signal. In dead zones, use any nearby phone or pre-arrange check-ins.
                    </Text>
                </View>

                <TouchableOpacity
                    style={styles.continueBtn}
                    onPress={handleContinue}
                    activeOpacity={0.8}
                    accessibilityRole="button"
                    accessibilityLabel="Got it, continue"
                >
                    <Text style={styles.continueBtnText}>Got it, continue</Text>
                </TouchableOpacity>
            </ScrollView>
        </SafeAreaView>
    );
}

const styles = StyleSheet.create({
    safeArea: {
        flex: 1,
        backgroundColor: theme.colors.background,
    },
    content: {
        padding: theme.spacing.lg,
        paddingBottom: theme.spacing.xxl,
    },
    hero: {
        alignItems: 'center',
        marginBottom: theme.spacing.xl,
        paddingTop: theme.spacing.xl,
    },
    heroIcon: {
        fontSize: 56,
        marginBottom: theme.spacing.md,
    },
    title: {
        fontSize: theme.fontSize.xl,
        fontWeight: theme.fontWeight.bold as '700',
        color: theme.colors.text.primary,
        textAlign: 'center',
        marginBottom: theme.spacing.sm,
    },
    subtitle: {
        fontSize: theme.fontSize.sm,
        color: theme.colors.text.secondary,
        textAlign: 'center',
        lineHeight: 20,
    },
    cards: {
        gap: theme.spacing.md,
        marginBottom: theme.spacing.lg,
    },
    card: {
        flexDirection: 'row',
        alignItems: 'flex-start',
        backgroundColor: theme.colors.surface,
        borderRadius: theme.borderRadius.xl,
        padding: theme.spacing.lg,
        gap: theme.spacing.md,
        ...theme.shadows.sm,
    },
    cardIcon: {
        fontSize: 28,
    },
    cardContent: {
        flex: 1,
    },
    cardTitle: {
        fontSize: theme.fontSize.base,
        fontWeight: theme.fontWeight.semibold as '600',
        color: theme.colors.text.primary,
        marginBottom: theme.spacing.xs,
    },
    cardBody: {
        fontSize: theme.fontSize.sm,
        color: theme.colors.text.secondary,
        lineHeight: 20,
    },
    warning: {
        flexDirection: 'row',
        alignItems: 'flex-start',
        backgroundColor: theme.colors.warningLight ?? '#fef9c3',
        borderRadius: theme.borderRadius.lg,
        padding: theme.spacing.md,
        gap: theme.spacing.sm,
        marginBottom: theme.spacing.xl,
    },
    warningText: {
        flex: 1,
        fontSize: theme.fontSize.xs,
        color: theme.colors.text.secondary,
        lineHeight: 18,
    },
    continueBtn: {
        backgroundColor: theme.colors.primary,
        borderRadius: theme.borderRadius.xl,
        paddingVertical: 16,
        alignItems: 'center',
    },
    continueBtnText: {
        fontSize: theme.fontSize.md,
        fontWeight: theme.fontWeight.semibold as '600',
        color: '#ffffff',
    },
});
```

**Note on `theme.colors.warning` / `theme.colors.warningLight`:** check `Carpool/constants/theme.ts`. If `warning` or `warningLight` aren't defined, substitute `'#f59e0b'` and `'#fef3c7'` respectively and add them to the theme file.

- [ ] **Step 4: Register the screen in _layout.tsx**

Open `Carpool/app/_layout.tsx`. After the line `<Stack.Screen name="profile/trusted-contacts" options={{ headerShown: false }} />`, add:

```tsx
            <Stack.Screen name="onboarding/safety-permissions" options={{ headerShown: false, gestureEnabled: false }} />
```

- [ ] **Step 5: Run test — verify it passes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/screens/safety-permissions.test.tsx
```

Expected: 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add app/onboarding/safety-permissions.tsx app/_layout.tsx \
        src/__tests__/screens/safety-permissions.test.tsx
git commit -m "feat(screens): add safety-permissions onboarding screen"
```

---

## Task 11: `DriverSafetyConsentModal` component + test

**Files:**
- Create: `Carpool/components/DriverSafetyConsentModal.tsx`
- Create: `Carpool/src/__tests__/components/DriverSafetyConsentModal.test.tsx`

- [ ] **Step 1: Write the failing test**

Create `Carpool/src/__tests__/components/DriverSafetyConsentModal.test.tsx`:

```tsx
import React from 'react';
import { render, fireEvent, waitFor } from '@testing-library/react-native';
import { DriverSafetyConsentModal } from '../../../components/DriverSafetyConsentModal';

const mockAcceptDriverSafetyConsent = jest.fn();
const mockPush = jest.fn();
const mockShowError = jest.fn();

jest.mock('../../../src/services/firebase/functions.service', () => ({
  functionsService: {
    acceptDriverSafetyConsent: (...args: unknown[]) => mockAcceptDriverSafetyConsent(...args),
  },
}));
jest.mock('expo-router', () => ({ useRouter: () => ({ push: mockPush }) }));
jest.mock('../../../hooks/useAlert', () => ({ useAlert: () => ({ showError: mockShowError }) }));

describe('DriverSafetyConsentModal', () => {
  const onAccepted = jest.fn();

  beforeEach(() => { jest.clearAllMocks(); });

  it('renders when visible=true', () => {
    const { getByText } = render(
      <DriverSafetyConsentModal visible={true} onAccepted={onAccepted} />
    );
    expect(getByText(/Driver Safety Agreement/i)).toBeTruthy();
  });

  it('renders the TOS clause text', () => {
    const { getByText } = render(
      <DriverSafetyConsentModal visible={true} onAccepted={onAccepted} />
    );
    expect(getByText(/vehicle plate/i)).toBeTruthy();
  });

  it('renders the I Agree button', () => {
    const { getByText } = render(
      <DriverSafetyConsentModal visible={true} onAccepted={onAccepted} />
    );
    expect(getByText(/I Agree/i)).toBeTruthy();
  });

  it('calls acceptDriverSafetyConsent and onAccepted when I Agree is pressed', async () => {
    mockAcceptDriverSafetyConsent.mockResolvedValue(undefined);
    const { getByText } = render(
      <DriverSafetyConsentModal visible={true} onAccepted={onAccepted} />
    );

    fireEvent.press(getByText(/I Agree/i));

    await waitFor(() => {
      expect(mockAcceptDriverSafetyConsent).toHaveBeenCalled();
      expect(onAccepted).toHaveBeenCalled();
    });
  });

  it('calls showError and does NOT call onAccepted when CF fails', async () => {
    mockAcceptDriverSafetyConsent.mockRejectedValue(new Error('network error'));
    const { getByText } = render(
      <DriverSafetyConsentModal visible={true} onAccepted={onAccepted} />
    );

    fireEvent.press(getByText(/I Agree/i));

    await waitFor(() => {
      expect(mockShowError).toHaveBeenCalled();
      expect(onAccepted).not.toHaveBeenCalled();
    });
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/components/DriverSafetyConsentModal.test.tsx
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement the component**

Create `Carpool/components/DriverSafetyConsentModal.tsx`:

```tsx
import { useRouter } from 'expo-router';
import { useState } from 'react';
import {
    ActivityIndicator,
    Modal,
    ScrollView,
    StyleSheet,
    Text,
    TouchableOpacity,
    View,
} from 'react-native';

import { theme } from '../constants/theme';
import { useAlert } from '../hooks/useAlert';
import { functionsService } from '../src/services/firebase/functions.service';

type Props = {
    visible: boolean;
    onAccepted: () => void;
};

export function DriverSafetyConsentModal({ visible, onAccepted }: Props) {
    const router = useRouter();
    const { showError } = useAlert();
    const [loading, setLoading] = useState(false);

    const handleAgree = async () => {
        setLoading(true);
        try {
            await functionsService.acceptDriverSafetyConsent();
            onAccepted();
        } catch (err) {
            showError('Could not save consent', String(err));
        } finally {
            setLoading(false);
        }
    };

    return (
        <Modal
            visible={visible}
            animationType="slide"
            presentationStyle="pageSheet"
            onRequestClose={() => {}}
        >
            <ScrollView contentContainerStyle={styles.content} showsVerticalScrollIndicator={false}>
                <View style={styles.handle} />
                <Text style={styles.title}>Driver Safety Agreement</Text>

                <View style={styles.clauseBox}>
                    <Text style={styles.clauseHeading}>Safety features</Text>
                    <Text style={styles.clauseText}>
                        As a driver, you acknowledge that passengers may share live ride information
                        — including your vehicle plate, vehicle model, and your name as it appears on
                        your driver profile — with their emergency contacts via SMS, WhatsApp, or a
                        public tracking link. This is a non-optional condition of providing rides on
                        Carpool and is necessary to provide passenger safety protection.
                    </Text>
                </View>

                <TouchableOpacity
                    onPress={() => router.push('/terms-conditions')}
                    style={styles.tosLink}
                >
                    <Text style={styles.tosLinkText}>Read full Terms &amp; Conditions →</Text>
                </TouchableOpacity>

                <TouchableOpacity
                    style={[styles.agreeBtn, loading && styles.agreeBtnDisabled]}
                    onPress={handleAgree}
                    disabled={loading}
                    activeOpacity={0.8}
                    accessibilityRole="button"
                    accessibilityLabel="I Agree to the driver safety conditions"
                >
                    {loading ? (
                        <ActivityIndicator size="small" color="#ffffff" />
                    ) : (
                        <Text style={styles.agreeBtnText}>I Agree</Text>
                    )}
                </TouchableOpacity>
            </ScrollView>
        </Modal>
    );
}

const styles = StyleSheet.create({
    content: {
        padding: theme.spacing.lg,
        paddingBottom: theme.spacing.xxl,
    },
    handle: {
        width: 40,
        height: 4,
        borderRadius: 2,
        backgroundColor: theme.colors.border,
        alignSelf: 'center',
        marginBottom: theme.spacing.lg,
        marginTop: theme.spacing.sm,
    },
    title: {
        fontSize: theme.fontSize.xl,
        fontWeight: theme.fontWeight.bold as '700',
        color: theme.colors.text.primary,
        marginBottom: theme.spacing.lg,
    },
    clauseBox: {
        backgroundColor: theme.colors.surface,
        borderRadius: theme.borderRadius.xl,
        padding: theme.spacing.lg,
        marginBottom: theme.spacing.md,
        ...theme.shadows.sm,
    },
    clauseHeading: {
        fontSize: theme.fontSize.base,
        fontWeight: theme.fontWeight.semibold as '600',
        color: theme.colors.text.primary,
        marginBottom: theme.spacing.sm,
    },
    clauseText: {
        fontSize: theme.fontSize.sm,
        color: theme.colors.text.secondary,
        lineHeight: 22,
    },
    tosLink: {
        alignSelf: 'flex-start',
        marginBottom: theme.spacing.xl,
    },
    tosLinkText: {
        fontSize: theme.fontSize.sm,
        color: theme.colors.primary,
    },
    agreeBtn: {
        backgroundColor: theme.colors.primary,
        borderRadius: theme.borderRadius.xl,
        paddingVertical: 16,
        alignItems: 'center',
    },
    agreeBtnDisabled: {
        opacity: 0.6,
    },
    agreeBtnText: {
        fontSize: theme.fontSize.md,
        fontWeight: theme.fontWeight.semibold as '600',
        color: '#ffffff',
    },
});
```

- [ ] **Step 4: Run test — verify it passes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest src/__tests__/components/DriverSafetyConsentModal.test.tsx
```

Expected: 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/DriverSafetyConsentModal.tsx \
        src/__tests__/components/DriverSafetyConsentModal.test.tsx
git commit -m "feat(components): add DriverSafetyConsentModal"
```

---

## Task 12: Wire everything up

**Files:**
- Modify: `Carpool/components/SosButton.tsx`
- Modify: `Carpool/app/tabs/post-ride.tsx`
- Modify: `Carpool/app/ride-details.tsx`
- Modify: `Carpool/app/terms-conditions.tsx`

- [ ] **Step 1: Gate SosButton on sosEnabled**

Open `Carpool/components/SosButton.tsx`. Add these imports at the top:

```ts
import { useSafetySettings } from '../hooks/useSafetySettings';
```

Inside the component, at the very top before any JSX, add:

```ts
  const { sosEnabled } = useSafetySettings();
  if (!sosEnabled) return null;
```

- [ ] **Step 2: Wire useSafetyOnboarding + useDriverConsent into post-ride.tsx**

Open `Carpool/app/tabs/post-ride.tsx`. Add imports:

```ts
import { useState } from 'react';  // ensure useState is imported
import { useSafetyOnboarding } from '../../hooks/useSafetyOnboarding';
import { useDriverConsent } from '../../hooks/useDriverConsent';
import { DriverSafetyConsentModal } from '../../components/DriverSafetyConsentModal';
```

Inside the component, near the top after existing hooks, add:

```ts
  const { loaded: onboardingLoaded } = useSafetyOnboarding();
  const { needsConsent } = useDriverConsent();
  const [consentVisible, setConsentVisible] = useState(true);
```

The component has two return points: an early loading guard (lines ~397-402) and the main `return (<KeyboardAvoidingView ...>)` below it.

Add this block **between** the loading guard and the main return:

```tsx
  // Phase H: block tab until safety onboarding is seen
  if (!onboardingLoaded) return null;
```

Then, inside the main `return (...)`, directly after the opening `<KeyboardAvoidingView ...>` tag, add:

```tsx
      {needsConsent && (
        <DriverSafetyConsentModal
          visible={consentVisible}
          onAccepted={() => setConsentVisible(false)}
        />
      )}
```

This renders the modal on top of the tab content without restructuring the existing JSX tree.

- [ ] **Step 3: Wire useSafetyOnboarding into ride-details.tsx**

Open `Carpool/app/ride-details.tsx`. Add import:

```ts
import { useSafetyOnboarding } from '../hooks/useSafetyOnboarding';
```

Inside the `RideDetails` component, near the top after existing hooks, add:

```ts
  const { onboardingComplete } = useSafetyOnboarding();
```

Inside `handleBookRide` (around line 218), add as the first check:

```ts
  const handleBookRide = async () => {
    if (!onboardingComplete) return;
    // ... rest of existing booking logic
```

- [ ] **Step 4: Add Safety Features clause to terms-conditions.tsx**

Open `Carpool/app/terms-conditions.tsx`. The terms content lives in the `SECTIONS` array at the top of the file. After the last entry (`'12. Contact'`), add two new entries:

```ts
    {
        title: '13. Safety Features',
        body: 'As a driver, you acknowledge that passengers may share live ride information — including your vehicle plate, vehicle model, and your name as it appears on your driver profile — with their emergency contacts via SMS, WhatsApp, or a public tracking link. This is a non-optional condition of providing rides on Carpool and is necessary to provide passenger safety protection.',
    },
    {
        title: '14. Emergency Services Disclaimer',
        body: 'Carpool is NOT a replacement for emergency services. The SOS feature is a notification tool — always call local emergency services directly when in danger. SOS requires cellular signal. In dead zones, use any nearby phone.',
    },
```

No style changes needed — the `SECTIONS` map renders them automatically using existing `styles.sectionTitle` and `styles.sectionBody`.

- [ ] **Step 5: Type-check everything**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit
```

Expected: no errors. Fix any TypeScript errors before proceeding.

- [ ] **Step 6: Run all frontend tests**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx jest --testPathPattern="useSafetySettings|useSafetyOnboarding|useDriverConsent|safety-permissions|DriverSafetyConsentModal"
```

Expected: all 22 tests PASS.

- [ ] **Step 7: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/SosButton.tsx app/tabs/post-ride.tsx \
        app/ride-details.tsx app/terms-conditions.tsx
git commit -m "feat(safety): wire SOS visibility, consent modal, and onboarding into screens"
```

---

## Task 13: Final checks + deploy

- [ ] **Step 1: Run backend unit tests**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:unit
```

Expected: all tests PASS. Fix any regressions.

- [ ] **Step 2: Run backend integration tests (all safety)**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run test:integration -- --testPathPattern=safetySettings
```

Expected: 11 tests PASS.

- [ ] **Step 3: Run backend linter**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npm run lint
```

Expected: no errors.

- [ ] **Step 4: Type-check backend**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 5: Deploy new CFs and updated createRide**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
firebase deploy --only functions:updateSafetySettings,functions:acceptDriverSafetyConsent,functions:createRide
```

Expected: 3 functions deployed successfully. Verify all three appear in Firebase Console → Functions.

- [ ] **Step 6: Smoke-test from dev account**

Using a dev test account in the app:
1. Open Settings → Safety — confirm SOS toggle appears, toggle off and back on
2. Log in on a fresh install (or clear AsyncStorage) and tap into the Post Ride tab — confirm safety-permissions screen appears and "Got it, continue" dismisses it
3. With a driver account that has no `driverSafetyConsent`, open Post Ride tab — confirm consent modal appears and blocks the tab
4. Tap "I Agree" — confirm modal dismisses and Post Ride tab is usable
5. Toggle SOS button off in Settings — confirm SOS button disappears from live-ride screen

- [ ] **Step 7: Final commit + tag**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add -p  # stage any missed files
git commit -m "chore: phase H+I complete — safety settings, onboarding, driver consent"

cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add -p
git commit -m "chore: phase H+I complete — updateSafetySettings, acceptDriverSafetyConsent, createRide consent gate"
```
