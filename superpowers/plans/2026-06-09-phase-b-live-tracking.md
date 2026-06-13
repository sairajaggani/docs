# Phase B — Live Tracking Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a unified location-core service that streams device GPS + battery to RTDB via a sink registry, so Phase F (SOS) and Phase G (ride share) can both publish to `liveTracking/{shareToken}` without colliding with the existing driver-ride GPS task.

**Architecture:** Single `expo-task-manager` task and single `expo-location.startLocationUpdatesAsync` subscription, owned by a new `locationCoreService`. Each consumer registers a sink (driver-ride, sos, share) that handles its own RTDB path and throttle policy. The existing `rideLocationService` becomes a thin facade that registers a `driver-ride` sink — zero call-site churn. Battery saver is sample-then-throttle in the sink callback (no task restart). Stop is server-driven via a status flag at `liveTracking/{token}/status` listened to by each sink.

**Tech Stack:** React Native (Expo SDK 54) + TypeScript strict, `expo-location`, `expo-task-manager`, `expo-battery` (new), `@notifee/react-native`, `@react-native-async-storage/async-storage`, Firebase RTDB, Jest, `@firebase/rules-unit-testing`.

**Spec reference:** `docs/superpowers/specs/2026-06-09-phase-b-live-tracking-design.md`

---

## Working directories

- **Frontend:** `/Users/sairajaggani/Desktop/Space/Project/Carpool/`
- **Backend (rules + rules tests):** `/Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions/`

All `cd` commands below are absolute paths; do not chain `cd` with git commands.

## File map

| Path | Action | Responsibility |
|---|---|---|
| `Carpool/src/services/location/location-core.service.ts` | **create** | GPS task owner, sink registry, fan-out, AsyncStorage persistence, foreground notification reconciler |
| `Carpool/src/services/location/sinks/types.ts` | **create** | `LocationSink`, `LocationSample`, `SinkMode`, `NotificationCopy` types |
| `Carpool/src/services/location/sinks/driver-ride.sink.ts` | **create** | Wraps `realtimeService.setDriverLocation`; 25m+30min throttle preserved from current service |
| `Carpool/src/services/location/sinks/live-tracking.sink.ts` | **create** | Writes `liveTracking/{token}`; listens to `liveTracking/{token}/status`; degraded callback |
| `Carpool/src/services/location/ride-location.service.ts` | **modify** | Strip internals; keep export name + method signatures; delegate to `locationCoreService` |
| `Carpool/src/services/location/index.ts` | **modify** | Add `locationCoreService` re-export |
| `Carpool/src/services/tracking/liveTracking.service.ts` | **create** | Public API: `startTracking(token, mode)`, `stopTracking(token)` |
| `Carpool/src/services/tracking/locationPermission.ts` | **create** | Foreground + background permission orchestration with `canAskAgain` fallback |
| `Carpool/src/services/tracking/battery.service.ts` | **create** | Wraps `expo-battery`; exposes `getThrottleFactor()` |
| `Carpool/src/services/firebase/realtime.service.ts` | **modify** | Add `setLiveTracking(token, payload)` and `subscribeLiveTrackingStatus(token, cb)` |
| `Carpool/app.config.js` | **modify** | Add `FOREGROUND_SERVICE_LOCATION` to `android.permissions` |
| `Carpool/package.json` | **modify** | Add `expo-battery` (via `npx expo install`) |
| `CarpoolBackend/database.rules.json` | **modify** | Add `liveTracking`, `sosAlertsByToken`, `rideSharesByToken` rule blocks |
| `CarpoolBackend/functions/tests/security/rules.test.ts` | **modify** | Extend with `describe('Realtime DB — liveTracking rules')` |
| `Carpool/src/__tests__/services/location/location-core.test.ts` | **create** | Sink registry, fan-out, notification reconciler, kill-recovery |
| `Carpool/src/__tests__/services/location/sinks/driver-ride.sink.test.ts` | **create** | Regression guard on 25m/30min throttle |
| `Carpool/src/__tests__/services/location/sinks/live-tracking.sink.test.ts` | **create** | Throttle factor 1/2/4, stop-flag listener, degraded callback |
| `Carpool/src/__tests__/services/location/ride-location.facade.test.ts` | **create** | Facade compat for the legacy public API |
| `Carpool/src/__tests__/services/tracking/battery.service.test.ts` | **create** | Threshold mapping + error fallback |
| `Carpool/src/__tests__/services/tracking/liveTracking.test.ts` | **create** | Public API: register/unregister/idempotency |
| `Carpool/src/__tests__/services/tracking/locationPermission.test.ts` | **create** | Permission flows incl. `canAskAgain === false` |

---

## Task 1: Setup — install expo-battery, add Android permission, baseline checks

**Files:**
- Modify: `Carpool/package.json` (via `expo install`)
- Modify: `Carpool/app.config.js`

- [ ] **Step 1: Confirm baseline tests pass before any change**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest --listTests | wc -l && npx jest src/__tests__/services/location 2>&1 | tail -20
```
Expected: list of test files prints; current location tests pass.

- [ ] **Step 2: Install expo-battery**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx expo install expo-battery
```
Expected: `package.json` gains an `expo-battery` dependency entry. Stdout includes `added` or `installed expo-battery`.

- [ ] **Step 3: Add FOREGROUND_SERVICE_LOCATION to android.permissions**

Edit `Carpool/app.config.js`. Find the `android.permissions` array (currently ends with `"POST_NOTIFICATIONS"`). Add `"FOREGROUND_SERVICE_LOCATION"` immediately after `"FOREGROUND_SERVICE"`. Resulting array:

```js
permissions: [
  "ACCESS_FINE_LOCATION",
  "ACCESS_COARSE_LOCATION",
  "ACCESS_BACKGROUND_LOCATION",
  "FOREGROUND_SERVICE",
  "FOREGROUND_SERVICE_LOCATION",
  "INTERNET",
  "WAKE_LOCK",
  "RECEIVE_BOOT_COMPLETED",
  "POST_NOTIFICATIONS",
],
```

- [ ] **Step 4: Verify TypeScript still compiles**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```
Expected: exit 0, no errors.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add package.json package-lock.json app.config.js && git commit -m "chore(safety): install expo-battery and add FOREGROUND_SERVICE_LOCATION permission"
```

---

## Task 2: RTDB rules — `liveTracking` + index nodes

**Files:**
- Modify: `CarpoolBackend/database.rules.json`
- Modify: `CarpoolBackend/functions/tests/security/rules.test.ts`

- [ ] **Step 1: Write failing rules tests**

Append a new `describe` block to the end of `CarpoolBackend/functions/tests/security/rules.test.ts` (inside the file's existing top-level scope, after the existing Firestore + RTDB describes):

```ts
describe('Realtime DB — liveTracking rules', () => {
    const TOKEN = 'tok_test_abc123';
    const OWNER_UID = 'owner_uid';
    const OTHER_UID = 'other_uid';

    beforeAll(async () => {
        if (rtdbTestEnv) await rtdbTestEnv.cleanup();
        const rulesPath = path.resolve(process.cwd(), '../database.rules.json');
        const rules = fs.readFileSync(rulesPath, 'utf8');
        rtdbTestEnv = await initializeTestEnvironment({
            projectId: PROJECT_ID,
            database: { rules, host: '127.0.0.1', port: 9000 },
        });
    });

    beforeEach(async () => {
        await rtdbTestEnv.clearDatabase();
        // Seed the index node as the admin (mimics backend CF)
        await rtdbTestEnv.withSecurityRulesDisabled(async (ctx) => {
            await ctx.database().ref(`sosAlertsByToken/${TOKEN}`).set({ userId: OWNER_UID, alertId: 'a1' });
        });
    });

    it('allows anonymous read of liveTracking/{token}', async () => {
        const anon = rtdbTestEnv.unauthenticatedContext();
        await assertSucceeds(anon.database().ref(`liveTracking/${TOKEN}`).once('value'));
    });

    it('allows owner to write liveTracking/{token}', async () => {
        const owner = rtdbTestEnv.authenticatedContext(OWNER_UID);
        await assertSucceeds(owner.database().ref(`liveTracking/${TOKEN}`).set({ lat: 1, lng: 2, updatedAt: Date.now() }));
    });

    it('denies non-owner write to liveTracking/{token}', async () => {
        const stranger = rtdbTestEnv.authenticatedContext(OTHER_UID);
        await assertFails(stranger.database().ref(`liveTracking/${TOKEN}`).set({ lat: 1, lng: 2, updatedAt: Date.now() }));
    });

    it('denies write to a token with no index node', async () => {
        const owner = rtdbTestEnv.authenticatedContext(OWNER_UID);
        await assertFails(owner.database().ref(`liveTracking/tok_no_index`).set({ lat: 1, lng: 2, updatedAt: Date.now() }));
    });

    it('denies FE read of sosAlertsByToken index node', async () => {
        const anyone = rtdbTestEnv.authenticatedContext(OWNER_UID);
        await assertFails(anyone.database().ref(`sosAlertsByToken/${TOKEN}`).once('value'));
    });

    it('denies FE write of sosAlertsByToken index node', async () => {
        const anyone = rtdbTestEnv.authenticatedContext(OWNER_UID);
        await assertFails(anyone.database().ref(`sosAlertsByToken/${TOKEN}`).set({ userId: OWNER_UID, alertId: 'a1' }));
    });

    it('allows owner via rideSharesByToken index node', async () => {
        await rtdbTestEnv.withSecurityRulesDisabled(async (ctx) => {
            await ctx.database().ref(`rideSharesByToken/tok_share`).set({ userId: OWNER_UID, rideId: 'r1' });
        });
        const owner = rtdbTestEnv.authenticatedContext(OWNER_UID);
        await assertSucceeds(owner.database().ref(`liveTracking/tok_share`).set({ lat: 1, lng: 2, updatedAt: Date.now() }));
    });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npm run test:security 2>&1 | tail -40
```
Expected: 7 new tests fail — assertions about `liveTracking` (rules don't exist yet) — typically permission-denied where success was expected, success where denied was expected.

If `npm run test:security` is not defined, fall back to:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npx jest tests/security/rules.test.ts 2>&1 | tail -40
```

- [ ] **Step 3: Add rules**

Edit `CarpoolBackend/database.rules.json`. Inside the top-level `"rules": { ... }` object, **alongside** (not nested under) the existing `"activeRides"` block, add three sibling blocks:

```json
"liveTracking": {
  "$token": {
    ".read": true,
    ".write": "auth != null && (root.child('sosAlertsByToken/' + $token + '/userId').val() == auth.uid || root.child('rideSharesByToken/' + $token + '/userId').val() == auth.uid)"
  }
},
"sosAlertsByToken": {
  "$token": {
    ".read": false,
    ".write": false
  }
},
"rideSharesByToken": {
  "$token": {
    ".read": false,
    ".write": false
  }
}
```

Make sure commas between sibling rule blocks are correct (the existing `"activeRides"` block needs a trailing comma before the new ones).

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npx jest tests/security/rules.test.ts 2>&1 | tail -40
```
Expected: all tests pass, including the 7 new `liveTracking` rule tests and all pre-existing tests still green.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend && git add database.rules.json functions/tests/security/rules.test.ts && git commit -m "feat(safety): add liveTracking + token-index RTDB rules with tests"
```

---

## Task 3: Battery service

**Files:**
- Create: `Carpool/src/services/tracking/battery.service.ts`
- Create: `Carpool/src/__tests__/services/tracking/battery.service.test.ts`

- [ ] **Step 1: Write failing test**

Create `Carpool/src/__tests__/services/tracking/battery.service.test.ts`:

```ts
jest.mock('expo-battery', () => ({
  getBatteryLevelAsync: jest.fn(),
  addBatteryLevelListener: jest.fn(),
}));

import * as ExpoBattery from 'expo-battery';
import { batteryService } from '../../../services/tracking/battery.service';

const mockedBattery = ExpoBattery as jest.Mocked<typeof ExpoBattery>;

describe('batteryService', () => {
  let listenerCb: ((evt: { batteryLevel: number }) => void) | null = null;
  const remove = jest.fn();

  beforeEach(() => {
    listenerCb = null;
    remove.mockClear();
    mockedBattery.addBatteryLevelListener.mockImplementation((cb) => {
      listenerCb = cb;
      return { remove } as any;
    });
    mockedBattery.getBatteryLevelAsync.mockResolvedValue(0.85);
    batteryService.shutdown(); // reset between tests
  });

  it('returns factor 1 before first reading (safe default)', () => {
    expect(batteryService.getThrottleFactor()).toBe(1);
  });

  it('returns factor 1 at ≥20%', async () => {
    await batteryService.start();
    listenerCb?.({ batteryLevel: 0.5 });
    expect(batteryService.getThrottleFactor()).toBe(1);
    listenerCb?.({ batteryLevel: 0.2 });
    expect(batteryService.getThrottleFactor()).toBe(1);
  });

  it('returns factor 2 at 10–20%', async () => {
    await batteryService.start();
    listenerCb?.({ batteryLevel: 0.19 });
    expect(batteryService.getThrottleFactor()).toBe(2);
    listenerCb?.({ batteryLevel: 0.1 });
    expect(batteryService.getThrottleFactor()).toBe(2);
  });

  it('returns factor 4 at <10%', async () => {
    await batteryService.start();
    listenerCb?.({ batteryLevel: 0.09 });
    expect(batteryService.getThrottleFactor()).toBe(4);
    listenerCb?.({ batteryLevel: 0.0 });
    expect(batteryService.getThrottleFactor()).toBe(4);
  });

  it('falls back to factor 1 if expo-battery throws on start', async () => {
    mockedBattery.getBatteryLevelAsync.mockRejectedValueOnce(new Error('no battery API'));
    await batteryService.start();
    expect(batteryService.getThrottleFactor()).toBe(1);
  });

  it('unsubscribes on shutdown', async () => {
    await batteryService.start();
    batteryService.shutdown();
    expect(remove).toHaveBeenCalled();
  });

  it('start() is idempotent — second call does not double-subscribe', async () => {
    await batteryService.start();
    await batteryService.start();
    expect(mockedBattery.addBatteryLevelListener).toHaveBeenCalledTimes(1);
  });

  it('returns current battery percentage (0–100) for sink payloads', async () => {
    mockedBattery.getBatteryLevelAsync.mockResolvedValueOnce(0.42);
    await batteryService.start();
    expect(batteryService.getBatteryPct()).toBe(42);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/tracking/battery.service.test.ts 2>&1 | tail -20
```
Expected: FAIL — `Cannot find module '../../../services/tracking/battery.service'`.

- [ ] **Step 3: Implement battery service**

Create `Carpool/src/services/tracking/battery.service.ts`:

```ts
import * as ExpoBattery from 'expo-battery';
import { logger } from '../../utils/logger';

type ThrottleFactor = 1 | 2 | 4;

class BatteryService {
  private subscription: { remove: () => void } | null = null;
  private currentLevel: number | null = null;
  private starting: Promise<void> | null = null;

  async start(): Promise<void> {
    if (this.subscription) return;
    if (this.starting) return this.starting;

    this.starting = (async () => {
      try {
        const level = await ExpoBattery.getBatteryLevelAsync();
        this.currentLevel = typeof level === 'number' && level >= 0 ? level : null;
      } catch (err) {
        logger.warn('batteryService: getBatteryLevelAsync failed, defaulting throttle to 1', err);
        this.currentLevel = null;
      }

      try {
        this.subscription = ExpoBattery.addBatteryLevelListener(({ batteryLevel }) => {
          if (typeof batteryLevel === 'number' && batteryLevel >= 0) {
            this.currentLevel = batteryLevel;
          }
        });
      } catch (err) {
        logger.warn('batteryService: addBatteryLevelListener failed', err);
      }
    })();

    await this.starting;
    this.starting = null;
  }

  shutdown(): void {
    if (this.subscription) {
      try {
        this.subscription.remove();
      } catch {
        // best-effort
      }
      this.subscription = null;
    }
    this.currentLevel = null;
  }

  getThrottleFactor(): ThrottleFactor {
    const lvl = this.currentLevel;
    if (lvl === null) return 1; // unknown → safe default
    if (lvl < 0.10) return 4;
    if (lvl < 0.20) return 2;
    return 1;
  }

  getBatteryPct(): number | null {
    return this.currentLevel === null ? null : Math.round(this.currentLevel * 100);
  }
}

export const batteryService = new BatteryService();
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/tracking/battery.service.test.ts 2>&1 | tail -20
```
Expected: all 8 tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/tracking/battery.service.ts src/__tests__/services/tracking/battery.service.test.ts && git commit -m "feat(safety): batteryService with throttle factor mapping"
```

---

## Task 4: Location permission service

**Files:**
- Create: `Carpool/src/services/tracking/locationPermission.ts`
- Create: `Carpool/src/__tests__/services/tracking/locationPermission.test.ts`

- [ ] **Step 1: Write failing test**

Create `Carpool/src/__tests__/services/tracking/locationPermission.test.ts`:

```ts
jest.mock('expo-location', () => ({
  requestForegroundPermissionsAsync: jest.fn(),
  requestBackgroundPermissionsAsync: jest.fn(),
  getForegroundPermissionsAsync: jest.fn(),
  getBackgroundPermissionsAsync: jest.fn(),
}));

jest.mock('react-native', () => ({
  Linking: { openSettings: jest.fn() },
  Platform: { OS: 'ios' },
}));

import * as ExpoLocation from 'expo-location';
import { Linking } from 'react-native';
import { locationPermission } from '../../../services/tracking/locationPermission';

const mockedLoc = ExpoLocation as jest.Mocked<typeof ExpoLocation>;
const mockedLinking = Linking as jest.Mocked<typeof Linking>;

describe('locationPermission', () => {
  beforeEach(() => jest.clearAllMocks());

  it('returns { foreground: true, background: true } when both granted', async () => {
    mockedLoc.requestForegroundPermissionsAsync.mockResolvedValue({ status: 'granted', canAskAgain: true } as any);
    mockedLoc.requestBackgroundPermissionsAsync.mockResolvedValue({ status: 'granted', canAskAgain: true } as any);
    const result = await locationPermission.ensure();
    expect(result).toEqual({ foreground: true, background: true });
  });

  it('returns { foreground: true, background: false } when only foreground granted', async () => {
    mockedLoc.requestForegroundPermissionsAsync.mockResolvedValue({ status: 'granted', canAskAgain: true } as any);
    mockedLoc.requestBackgroundPermissionsAsync.mockResolvedValue({ status: 'denied', canAskAgain: true } as any);
    const result = await locationPermission.ensure();
    expect(result).toEqual({ foreground: true, background: false });
  });

  it('throws when foreground denied', async () => {
    mockedLoc.requestForegroundPermissionsAsync.mockResolvedValue({ status: 'denied', canAskAgain: true } as any);
    await expect(locationPermission.ensure()).rejects.toThrow(/foreground location permission/i);
  });

  it('opens settings when canAskAgain is false on foreground', async () => {
    mockedLoc.getForegroundPermissionsAsync.mockResolvedValue({ status: 'denied', canAskAgain: false } as any);
    await expect(locationPermission.ensure()).rejects.toThrow();
    expect(mockedLinking.openSettings).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/tracking/locationPermission.test.ts 2>&1 | tail -20
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement locationPermission**

Create `Carpool/src/services/tracking/locationPermission.ts`:

```ts
import * as ExpoLocation from 'expo-location';
import { Linking } from 'react-native';
import { logger } from '../../utils/logger';

export interface PermissionResult {
  foreground: boolean;
  background: boolean;
}

class LocationPermission {
  async ensure(): Promise<PermissionResult> {
    const existing = await ExpoLocation.getForegroundPermissionsAsync().catch(() => null);
    if (existing && existing.status === 'denied' && existing.canAskAgain === false) {
      try {
        await Linking.openSettings();
      } catch (err) {
        logger.warn('locationPermission: openSettings failed', err);
      }
      throw new Error('Foreground location permission denied (cannot re-prompt). Opened Settings.');
    }

    const fg = await ExpoLocation.requestForegroundPermissionsAsync();
    if (fg.status !== 'granted') {
      throw new Error('Foreground location permission denied.');
    }

    let backgroundGranted = false;
    try {
      const bg = await ExpoLocation.requestBackgroundPermissionsAsync();
      backgroundGranted = bg.status === 'granted';
    } catch (err) {
      logger.warn('locationPermission: background request errored', err);
    }

    return { foreground: true, background: backgroundGranted };
  }
}

export const locationPermission = new LocationPermission();
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/tracking/locationPermission.test.ts 2>&1 | tail -20
```
Expected: all 4 tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/tracking/locationPermission.ts src/__tests__/services/tracking/locationPermission.test.ts && git commit -m "feat(safety): locationPermission with canAskAgain settings fallback"
```

---

## Task 5: Sink contract types

**Files:**
- Create: `Carpool/src/services/location/sinks/types.ts`

- [ ] **Step 1: Create types file**

Create `Carpool/src/services/location/sinks/types.ts`:

```ts
export type SinkMode = 'driver-ride' | 'sos' | 'share';

export interface LocationSample {
  latitude: number;
  longitude: number;
  accuracy: number | null;
  speed: number | null;
  bearing: number | null;
  timestamp: number; // ms since epoch
}

export interface NotificationCopy {
  title: string;
  body: string;
  channelId: string;
  importance: 'low' | 'high';
  colorHex: string;
}

export interface LocationSink {
  id: string;                                    // 'driver-ride' or `live-${shareToken}`
  mode: SinkMode;
  notificationPriority: 0 | 1 | 2;               // 0=ride, 1=share, 2=sos
  attach(): Promise<void>;
  detach(): Promise<void>;
  onSample(sample: LocationSample, throttleFactor: 1 | 2 | 4): Promise<void>;
  notificationCopy(): NotificationCopy;
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```
Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/location/sinks/types.ts && git commit -m "feat(safety): sink contract types for location core"
```

---

## Task 6: realtimeService extensions for liveTracking writes + status subscribe

**Files:**
- Modify: `Carpool/src/services/firebase/realtime.service.ts`
- Modify: `Carpool/src/__tests__/services/realtime.service.test.ts`

- [ ] **Step 1: Read existing realtime.service.ts to find insertion point**

Open `Carpool/src/services/firebase/realtime.service.ts`. Locate the `setDriverLocation` method (around line 77) and `removeDriverLocation` (around line 113). Add the new methods immediately after `removeDriverLocation`.

- [ ] **Step 2: Write failing tests**

Append to `Carpool/src/__tests__/services/realtime.service.test.ts` inside its existing top-level `describe`:

```ts
describe('liveTracking', () => {
  it('writes to liveTracking/{token} via setLiveTracking', async () => {
    const setMock = jest.fn().mockResolvedValue(undefined);
    const refMock = jest.fn(() => ({ set: setMock, on: jest.fn(), off: jest.fn() }));
    // @ts-expect-error: assign mocked rtdb
    realtimeService.__rtdb = { ref: refMock };
    await realtimeService.setLiveTracking('tok_abc', {
      lat: 12.34, lng: 56.78, accuracy: 5, speed: 0, bearing: 0,
      batteryPct: 80, updatedAt: 1700000000000,
    });
    expect(refMock).toHaveBeenCalledWith('liveTracking/tok_abc');
    expect(setMock).toHaveBeenCalledWith(expect.objectContaining({
      lat: 12.34, lng: 56.78, batteryPct: 80,
    }));
  });

  it('subscribes to liveTracking/{token}/status and invokes callback on value', () => {
    const onMock = jest.fn();
    const offMock = jest.fn();
    const refMock = jest.fn(() => ({ on: onMock, off: offMock }));
    // @ts-expect-error: assign mocked rtdb
    realtimeService.__rtdb = { ref: refMock };
    const cb = jest.fn();
    const unsub = realtimeService.subscribeLiveTrackingStatus('tok_abc', cb);
    expect(refMock).toHaveBeenCalledWith('liveTracking/tok_abc/status');
    expect(onMock).toHaveBeenCalledWith('value', expect.any(Function));
    // simulate snapshot
    const handler = onMock.mock.calls[0][1];
    handler({ val: () => 'STOPPED' });
    expect(cb).toHaveBeenCalledWith('STOPPED');
    unsub();
    expect(offMock).toHaveBeenCalled();
  });
});
```

> Note: the existing realtime.service.test.ts already mocks the rtdb instance via a similar pattern. If `realtimeService.__rtdb` is not the actual injection point used in existing tests, mirror the existing pattern (likely `jest.mock('@react-native-firebase/database')`). Read the top of the existing test file and replicate. The principle (mock the database ref, assert ref + write call) is what matters.

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/realtime.service.test.ts 2>&1 | tail -20
```
Expected: 2 new tests fail — `setLiveTracking is not a function` / `subscribeLiveTrackingStatus is not a function`.

- [ ] **Step 4: Add the two methods**

In `Carpool/src/services/firebase/realtime.service.ts`, immediately after `removeDriverLocation`, insert:

```ts
  async setLiveTracking(
    token: string,
    payload: {
      lat: number;
      lng: number;
      accuracy: number | null;
      speed: number | null;
      bearing: number | null;
      batteryPct: number | null;
      updatedAt: number;
    },
  ): Promise<void> {
    try {
      await this.rtdb.ref(`liveTracking/${token}`).set(payload);
    } catch (error) {
      logger.warn('setLiveTracking failed (will retry on next sample):', error);
      throw error;
    }
  }

  subscribeLiveTrackingStatus(
    token: string,
    onStatus: (status: string | null) => void,
  ): () => void {
    const ref = this.rtdb.ref(`liveTracking/${token}/status`);
    const handler = (snap: { val: () => unknown }) => {
      const v = snap.val();
      onStatus(typeof v === 'string' ? v : null);
    };
    ref.on('value', handler as any);
    return () => {
      try {
        ref.off('value', handler as any);
      } catch {
        // best-effort
      }
    };
  }
```

> If the field is `this.rtdb` rather than something else in the existing class, match it. Read the top of the existing file to find the private rtdb reference name.

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/realtime.service.test.ts 2>&1 | tail -20
```
Expected: all tests pass including 2 new.

- [ ] **Step 6: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/firebase/realtime.service.ts src/__tests__/services/realtime.service.test.ts && git commit -m "feat(safety): realtimeService.setLiveTracking + subscribeLiveTrackingStatus"
```

---

## Task 7: locationCoreService — sink registry + persistence (no GPS yet)

**Files:**
- Create: `Carpool/src/services/location/location-core.service.ts`
- Create: `Carpool/src/__tests__/services/location/location-core.test.ts`

- [ ] **Step 1: Write failing test (registry + persistence only — no GPS)**

Create `Carpool/src/__tests__/services/location/location-core.test.ts`:

```ts
jest.mock('@react-native-async-storage/async-storage', () => ({
  setItem: jest.fn().mockResolvedValue(undefined),
  getItem: jest.fn().mockResolvedValue(null),
  removeItem: jest.fn().mockResolvedValue(undefined),
}));

jest.mock('expo-task-manager', () => ({
  defineTask: jest.fn(),
  isTaskRegisteredAsync: jest.fn().mockResolvedValue(false),
}));

jest.mock('expo-location', () => ({
  startLocationUpdatesAsync: jest.fn().mockResolvedValue(undefined),
  stopLocationUpdatesAsync: jest.fn().mockResolvedValue(undefined),
  Accuracy: { BestForNavigation: 6 },
  ActivityType: { AutomotiveNavigation: 3 },
}));

jest.mock('@notifee/react-native', () => ({
  __esModule: true,
  default: {
    createChannel: jest.fn().mockResolvedValue(undefined),
    displayNotification: jest.fn().mockResolvedValue(undefined),
    cancelNotification: jest.fn().mockResolvedValue(undefined),
    stopForegroundService: jest.fn().mockResolvedValue(undefined),
  },
  AndroidImportance: { LOW: 2, HIGH: 4 },
  AndroidProgress: {},
}));

jest.mock('../../../services/tracking/battery.service', () => ({
  batteryService: {
    start: jest.fn().mockResolvedValue(undefined),
    shutdown: jest.fn(),
    getThrottleFactor: jest.fn(() => 1),
    getBatteryPct: jest.fn(() => 80),
  },
}));

jest.mock('../../../services/tracking/locationPermission', () => ({
  locationPermission: {
    ensure: jest.fn().mockResolvedValue({ foreground: true, background: true }),
  },
}));

import AsyncStorage from '@react-native-async-storage/async-storage';
import { locationCoreService } from '../../../services/location/location-core.service';
import type { LocationSink } from '../../../services/location/sinks/types';

function makeSink(id: string, priority: 0 | 1 | 2 = 0): LocationSink {
  return {
    id,
    mode: priority === 2 ? 'sos' : priority === 1 ? 'share' : 'driver-ride',
    notificationPriority: priority,
    attach: jest.fn().mockResolvedValue(undefined),
    detach: jest.fn().mockResolvedValue(undefined),
    onSample: jest.fn().mockResolvedValue(undefined),
    notificationCopy: () => ({
      title: 't', body: 'b', channelId: 'ride-tracking', importance: 'low', colorHex: '#000',
    }),
  };
}

describe('locationCoreService — registry', () => {
  beforeEach(async () => {
    await locationCoreService.__resetForTests();
    jest.clearAllMocks();
  });

  it('registers a sink and persists to AsyncStorage', async () => {
    const sink = makeSink('s1');
    await locationCoreService.registerSink(sink);
    expect(sink.attach).toHaveBeenCalled();
    expect(locationCoreService.getActiveSinks().map((s) => s.id)).toEqual(['s1']);
    expect(AsyncStorage.setItem).toHaveBeenCalledWith(
      '@location_core_active_sinks',
      expect.stringContaining('"s1"'),
    );
  });

  it('register is idempotent — second register with same id is a no-op', async () => {
    const sink = makeSink('s1');
    await locationCoreService.registerSink(sink);
    await locationCoreService.registerSink(sink);
    expect(sink.attach).toHaveBeenCalledTimes(1);
    expect(locationCoreService.getActiveSinks()).toHaveLength(1);
  });

  it('unregisters a sink and persists', async () => {
    const sink = makeSink('s1');
    await locationCoreService.registerSink(sink);
    await locationCoreService.unregisterSink('s1');
    expect(sink.detach).toHaveBeenCalled();
    expect(locationCoreService.getActiveSinks()).toHaveLength(0);
  });

  it('fan-out invokes onSample for every active sink with battery throttle factor', async () => {
    const s1 = makeSink('s1');
    const s2 = makeSink('s2', 2);
    await locationCoreService.registerSink(s1);
    await locationCoreService.registerSink(s2);
    await locationCoreService.__fanOutForTests({
      latitude: 1, longitude: 2, accuracy: 5, speed: 0, bearing: 0, timestamp: 100,
    });
    expect(s1.onSample).toHaveBeenCalledWith(expect.anything(), 1);
    expect(s2.onSample).toHaveBeenCalledWith(expect.anything(), 1);
  });

  it('notification reconciler picks highest priority sink', async () => {
    const ride = makeSink('driver-ride', 0);
    const sos = makeSink('live-tok', 2);
    await locationCoreService.registerSink(ride);
    await locationCoreService.registerSink(sos);
    const top = locationCoreService.getTopPrioritySink();
    expect(top?.id).toBe('live-tok');
  });

  it('rehydrates activeSinks from AsyncStorage on bootstrap', async () => {
    (AsyncStorage.getItem as jest.Mock).mockResolvedValueOnce(
      JSON.stringify([{ id: 'live-tok', mode: 'sos', registeredAt: 1700000000000 }]),
    );
    const rebuilders = {
      sos: jest.fn((stored) => makeSink(stored.id, 2)),
      share: jest.fn(),
      'driver-ride': jest.fn(),
    };
    await locationCoreService.bootstrap(rebuilders as any);
    expect(rebuilders.sos).toHaveBeenCalled();
    expect(locationCoreService.getActiveSinks()).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/location-core.test.ts 2>&1 | tail -20
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement registry portion (no GPS, no notifee logic yet)**

Create `Carpool/src/services/location/location-core.service.ts`:

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { logger } from '../../utils/logger';
import { batteryService } from '../tracking/battery.service';
import type { LocationSample, LocationSink, SinkMode } from './sinks/types';

const STORAGE_KEY_ACTIVE_SINKS = '@location_core_active_sinks';

interface PersistedSink {
  id: string;
  mode: SinkMode;
  registeredAt: number;
  payload?: Record<string, unknown>; // sink-specific recovery data (e.g., shareToken, rideId)
}

type SinkRebuilders = Partial<Record<SinkMode, (stored: PersistedSink) => LocationSink>>;

class LocationCoreService {
  private sinks = new Map<string, LocationSink>();

  async registerSink(sink: LocationSink, recoveryPayload?: Record<string, unknown>): Promise<void> {
    if (this.sinks.has(sink.id)) {
      logger.debug(`locationCore: sink ${sink.id} already registered, skipping`);
      return;
    }
    await sink.attach();
    this.sinks.set(sink.id, sink);
    await this.persist(recoveryPayload);
    await batteryService.start();
  }

  async unregisterSink(id: string): Promise<void> {
    const sink = this.sinks.get(id);
    if (!sink) return;
    try {
      await sink.detach();
    } catch (err) {
      logger.warn(`locationCore: detach failed for ${id}`, err);
    }
    this.sinks.delete(id);
    await this.persist();
    if (this.sinks.size === 0) {
      batteryService.shutdown();
    }
  }

  getActiveSinks(): LocationSink[] {
    return Array.from(this.sinks.values());
  }

  getTopPrioritySink(): LocationSink | null {
    let top: LocationSink | null = null;
    for (const s of this.sinks.values()) {
      if (!top || s.notificationPriority > top.notificationPriority) top = s;
    }
    return top;
  }

  async bootstrap(rebuilders: SinkRebuilders): Promise<void> {
    const raw = await AsyncStorage.getItem(STORAGE_KEY_ACTIVE_SINKS).catch(() => null);
    if (!raw) return;
    let entries: PersistedSink[] = [];
    try {
      entries = JSON.parse(raw) as PersistedSink[];
    } catch {
      return;
    }
    for (const stored of entries) {
      const rebuild = rebuilders[stored.mode];
      if (!rebuild) continue;
      try {
        const sink = rebuild(stored);
        await sink.attach();
        this.sinks.set(sink.id, sink);
      } catch (err) {
        logger.warn(`locationCore: bootstrap rebuild failed for ${stored.id}`, err);
      }
    }
  }

  private async persist(_recoveryPayload?: Record<string, unknown>): Promise<void> {
    const entries: PersistedSink[] = Array.from(this.sinks.values()).map((s) => ({
      id: s.id,
      mode: s.mode,
      registeredAt: Date.now(),
    }));
    try {
      await AsyncStorage.setItem(STORAGE_KEY_ACTIVE_SINKS, JSON.stringify(entries));
    } catch (err) {
      logger.warn('locationCore: persist failed', err);
    }
  }

  // ── Test hooks ──────────────────────────────────────────────────────────────
  async __resetForTests(): Promise<void> {
    for (const id of Array.from(this.sinks.keys())) {
      await this.unregisterSink(id);
    }
    this.sinks.clear();
  }

  async __fanOutForTests(sample: LocationSample): Promise<void> {
    const throttle = batteryService.getThrottleFactor();
    for (const sink of this.sinks.values()) {
      try {
        await sink.onSample(sample, throttle);
      } catch (err) {
        logger.warn(`locationCore: sink ${sink.id} onSample threw`, err);
      }
    }
  }
}

export const locationCoreService = new LocationCoreService();
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/location-core.test.ts 2>&1 | tail -20
```
Expected: all 6 tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/location/location-core.service.ts src/__tests__/services/location/location-core.test.ts && git commit -m "feat(safety): locationCore sink registry, persistence, bootstrap"
```

---

## Task 8: locationCoreService — GPS task + foreground notification reconciler (port from rideLocationService)

**Files:**
- Modify: `Carpool/src/services/location/location-core.service.ts`
- Modify: `Carpool/src/__tests__/services/location/location-core.test.ts` (extend)

- [ ] **Step 1: Write failing test for GPS startup + notification reconciler**

Append to `Carpool/src/__tests__/services/location/location-core.test.ts`:

```ts
import * as ExpoLocation from 'expo-location';
import * as TaskManager from 'expo-task-manager';
import notifee, { AndroidImportance } from '@notifee/react-native';

describe('locationCoreService — GPS lifecycle', () => {
  beforeEach(async () => {
    await locationCoreService.__resetForTests();
    jest.clearAllMocks();
    (TaskManager.isTaskRegisteredAsync as jest.Mock).mockResolvedValue(false);
  });

  it('starts expo-location task on first sink register', async () => {
    await locationCoreService.registerSink(makeSink('s1'));
    expect(ExpoLocation.startLocationUpdatesAsync).toHaveBeenCalledTimes(1);
  });

  it('does not start task again on second sink register (idempotent)', async () => {
    (TaskManager.isTaskRegisteredAsync as jest.Mock).mockResolvedValue(true);
    await locationCoreService.registerSink(makeSink('s1'));
    await locationCoreService.registerSink(makeSink('s2'));
    expect(ExpoLocation.startLocationUpdatesAsync).toHaveBeenCalledTimes(1);
  });

  it('stops expo-location task when last sink unregisters', async () => {
    const s1 = makeSink('s1');
    await locationCoreService.registerSink(s1);
    (TaskManager.isTaskRegisteredAsync as jest.Mock).mockResolvedValue(true);
    await locationCoreService.unregisterSink('s1');
    expect(ExpoLocation.stopLocationUpdatesAsync).toHaveBeenCalled();
    expect(notifee.cancelNotification).toHaveBeenCalled();
  });

  it('displays SOS notification with HIGH importance when SOS sink is active', async () => {
    const sos = makeSink('live-tok', 2);
    sos.notificationCopy = () => ({
      title: '🚨 ZygoRide — Emergency tracking active',
      body: 'Trusted contacts are receiving your live location.',
      channelId: 'ride-tracking-sos',
      importance: 'high',
      colorHex: '#D93025',
    });
    await locationCoreService.registerSink(sos);
    expect(notifee.createChannel).toHaveBeenCalledWith(expect.objectContaining({
      id: 'ride-tracking-sos',
      importance: AndroidImportance.HIGH,
    }));
    expect(notifee.displayNotification).toHaveBeenCalledWith(expect.objectContaining({
      title: expect.stringContaining('Emergency'),
    }));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/location-core.test.ts 2>&1 | tail -30
```
Expected: 4 new tests fail.

- [ ] **Step 3: Extend location-core with GPS lifecycle + notification reconciler**

Edit `Carpool/src/services/location/location-core.service.ts`. At the top add imports:

```ts
import notifee, { AndroidImportance } from '@notifee/react-native';
import * as ExpoLocation from 'expo-location';
import * as TaskManager from 'expo-task-manager';
import { Platform } from 'react-native';
import { locationPermission } from '../tracking/locationPermission';
```

Add the task definition module-side (wrapped in try/catch — mirrors existing `ride-location.service.ts` pattern):

```ts
const TASK_NAME = 'LOCATION_CORE_TASK';
let _taskManagerAvailable = false;

try {
  TaskManager.defineTask(TASK_NAME, async ({ data, error }) => {
    if (error) {
      logger.error('locationCore task error:', error);
      return;
    }
    if (!data) return;
    const { locations } = data as { locations: ExpoLocation.LocationObject[] };
    if (!locations?.length) return;
    const loc = locations[locations.length - 1];
    await locationCoreService.__handleSample({
      latitude: loc.coords.latitude,
      longitude: loc.coords.longitude,
      accuracy: loc.coords.accuracy ?? null,
      speed: loc.coords.speed ?? null,
      bearing: loc.coords.heading ?? null,
      timestamp: loc.timestamp,
    });
  });
  _taskManagerAvailable = true;
} catch (err) {
  logger.warn('locationCore: TaskManager native module unavailable', err);
}
```

Add inside the class:

```ts
  async __handleSample(sample: LocationSample): Promise<void> {
    const throttle = batteryService.getThrottleFactor();
    for (const sink of this.sinks.values()) {
      try {
        await sink.onSample(sample, throttle);
      } catch (err) {
        logger.warn(`locationCore: sink ${sink.id} onSample threw`, err);
      }
    }
  }

  private async ensureGpsRunning(): Promise<void> {
    if (!_taskManagerAvailable) {
      logger.warn('locationCore: TaskManager not available — GPS will not start');
      return;
    }
    const isRegistered = await TaskManager.isTaskRegisteredAsync(TASK_NAME);
    if (isRegistered) return;

    await locationPermission.ensure();

    await ExpoLocation.startLocationUpdatesAsync(TASK_NAME, {
      accuracy: ExpoLocation.Accuracy.BestForNavigation,
      timeInterval: 5_000,
      distanceInterval: 0,
      activityType: ExpoLocation.ActivityType.AutomotiveNavigation,
      showsBackgroundLocationIndicator: true,
      pausesUpdatesAutomatically: false,
      deferredUpdatesInterval: 0,
      deferredUpdatesDistance: 0,
    });
  }

  private async stopGpsIfIdle(): Promise<void> {
    if (this.sinks.size > 0) return;
    try {
      const isRegistered = await TaskManager.isTaskRegisteredAsync(TASK_NAME);
      if (isRegistered) await ExpoLocation.stopLocationUpdatesAsync(TASK_NAME);
    } catch (err) {
      logger.warn('locationCore: stopLocationUpdatesAsync error', err);
    }
    try {
      if (Platform.OS === 'android') await notifee.stopForegroundService();
      await notifee.cancelNotification('location-core-fg');
    } catch {
      // best-effort
    }
  }

  private async reconcileNotification(): Promise<void> {
    const top = this.getTopPrioritySink();
    if (!top) return;
    const copy = top.notificationCopy();
    if (Platform.OS === 'android') {
      await notifee.createChannel({
        id: copy.channelId,
        name: copy.channelId === 'ride-tracking-sos' ? 'Emergency Tracking' : 'Ride Tracking',
        importance: copy.importance === 'high' ? AndroidImportance.HIGH : AndroidImportance.LOW,
      });
    }
    await notifee.displayNotification({
      id: 'location-core-fg',
      title: copy.title,
      body: copy.body,
      android: {
        channelId: copy.channelId,
        asForegroundService: true,
        ongoing: true,
        pressAction: { id: 'default' },
        smallIcon: 'ic_notification',
        color: copy.colorHex,
      },
      ios: {
        foregroundPresentationOptions: { alert: false, badge: false, sound: false },
      },
    });
  }
```

Update `registerSink` to call `ensureGpsRunning` + `reconcileNotification` after the sink is added:

```ts
  async registerSink(sink: LocationSink, recoveryPayload?: Record<string, unknown>): Promise<void> {
    if (this.sinks.has(sink.id)) {
      logger.debug(`locationCore: sink ${sink.id} already registered, skipping`);
      return;
    }
    await sink.attach();
    this.sinks.set(sink.id, sink);
    await this.persist(recoveryPayload);
    await batteryService.start();
    await this.ensureGpsRunning();
    await this.reconcileNotification();
  }
```

Update `unregisterSink` to call `reconcileNotification` and `stopGpsIfIdle` after removing:

```ts
  async unregisterSink(id: string): Promise<void> {
    const sink = this.sinks.get(id);
    if (!sink) return;
    try {
      await sink.detach();
    } catch (err) {
      logger.warn(`locationCore: detach failed for ${id}`, err);
    }
    this.sinks.delete(id);
    await this.persist();
    if (this.sinks.size === 0) {
      batteryService.shutdown();
      await this.stopGpsIfIdle();
    } else {
      await this.reconcileNotification();
    }
  }
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/location-core.test.ts 2>&1 | tail -20
```
Expected: all tests in the file pass (the registry suite plus the new GPS lifecycle suite).

- [ ] **Step 5: Verify TypeScript**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```
Expected: exit 0.

- [ ] **Step 6: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/location/location-core.service.ts src/__tests__/services/location/location-core.test.ts && git commit -m "feat(safety): locationCore GPS task lifecycle + notification reconciler"
```

---

## Task 9: driver-ride sink (regression guard on existing 25m/30min throttle)

**Files:**
- Create: `Carpool/src/services/location/sinks/driver-ride.sink.ts`
- Create: `Carpool/src/__tests__/services/location/sinks/driver-ride.sink.test.ts`

- [ ] **Step 1: Write failing test**

Create `Carpool/src/__tests__/services/location/sinks/driver-ride.sink.test.ts`:

```ts
jest.mock('../../../../services/firebase/realtime.service', () => ({
  realtimeService: {
    setDriverLocation: jest.fn().mockResolvedValue(undefined),
    removeDriverLocation: jest.fn().mockResolvedValue(undefined),
  },
}));

jest.mock('geolib', () => ({
  getDistance: jest.fn(),
}));

import { realtimeService } from '../../../../services/firebase/realtime.service';
import { getDistance } from 'geolib';
import { createDriverRideSink } from '../../../../services/location/sinks/driver-ride.sink';

const setDriverLocation = realtimeService.setDriverLocation as jest.Mock;
const mockedGetDistance = getDistance as jest.Mock;

describe('driverRideSink', () => {
  beforeEach(() => jest.clearAllMocks());

  const baseSample = {
    latitude: 12.34, longitude: 56.78, accuracy: 5, speed: 0, bearing: 90, timestamp: 1_000_000,
  };

  it('writes on the very first sample', async () => {
    const sink = createDriverRideSink('ride_1');
    await sink.attach();
    await sink.onSample(baseSample, 1);
    expect(setDriverLocation).toHaveBeenCalledWith('ride_1', expect.objectContaining({
      latitude: 12.34, longitude: 56.78,
    }));
  });

  it('skips when distance < 25m AND time < 30min since last write', async () => {
    const sink = createDriverRideSink('ride_1');
    await sink.attach();
    await sink.onSample(baseSample, 1);
    mockedGetDistance.mockReturnValue(10);
    await sink.onSample({ ...baseSample, timestamp: baseSample.timestamp + 5_000 }, 1);
    expect(setDriverLocation).toHaveBeenCalledTimes(1);
  });

  it('writes when distance ≥ 25m even if time < 30min', async () => {
    const sink = createDriverRideSink('ride_1');
    await sink.attach();
    await sink.onSample(baseSample, 1);
    mockedGetDistance.mockReturnValue(30);
    await sink.onSample({ ...baseSample, timestamp: baseSample.timestamp + 5_000 }, 1);
    expect(setDriverLocation).toHaveBeenCalledTimes(2);
  });

  it('writes when time ≥ 30min even if distance < 25m', async () => {
    const sink = createDriverRideSink('ride_1');
    await sink.attach();
    await sink.onSample(baseSample, 1);
    mockedGetDistance.mockReturnValue(5);
    await sink.onSample({ ...baseSample, timestamp: baseSample.timestamp + 31 * 60 * 1000 }, 1);
    expect(setDriverLocation).toHaveBeenCalledTimes(2);
  });

  it('detach calls realtimeService.removeDriverLocation', async () => {
    const sink = createDriverRideSink('ride_1');
    await sink.attach();
    await sink.detach();
    expect(realtimeService.removeDriverLocation).toHaveBeenCalledWith('ride_1');
  });

  it('is not affected by battery throttle factor (driver-ride uses distance-based throttle only)', async () => {
    const sink = createDriverRideSink('ride_1');
    await sink.attach();
    await sink.onSample(baseSample, 4); // factor=4
    mockedGetDistance.mockReturnValue(30);
    await sink.onSample({ ...baseSample, timestamp: baseSample.timestamp + 5_000 }, 4);
    expect(setDriverLocation).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/sinks/driver-ride.sink.test.ts 2>&1 | tail -20
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement driver-ride sink**

Create `Carpool/src/services/location/sinks/driver-ride.sink.ts`:

```ts
import { getDistance } from 'geolib';
import { realtimeService } from '../../firebase/realtime.service';
import { logger } from '../../../utils/logger';
import type { LocationSample, LocationSink, NotificationCopy } from './types';

const THROTTLE_DISTANCE_M = 25;
const MAX_STATIONARY_TIME_MS = 30 * 60 * 1000;

export function createDriverRideSink(rideId: string, meta?: { fromAddress?: string; toAddress?: string }): LocationSink {
  let lastWritten: { latitude: number; longitude: number } | null = null;
  let lastWrittenTime = 0;

  return {
    id: 'driver-ride',
    mode: 'driver-ride',
    notificationPriority: 0,

    async attach() {
      lastWritten = null;
      lastWrittenTime = 0;
    },

    async detach() {
      try {
        await realtimeService.removeDriverLocation(rideId);
      } catch (err) {
        logger.warn('driverRideSink: removeDriverLocation failed', err);
      }
    },

    async onSample(sample: LocationSample, _throttleFactor: 1 | 2 | 4) {
      if (lastWritten) {
        const distance = getDistance(lastWritten, { latitude: sample.latitude, longitude: sample.longitude });
        const timeSinceLastWrite = sample.timestamp - lastWrittenTime;
        if (distance < THROTTLE_DISTANCE_M && timeSinceLastWrite < MAX_STATIONARY_TIME_MS) {
          return;
        }
      }
      try {
        await realtimeService.setDriverLocation(rideId, {
          latitude: sample.latitude,
          longitude: sample.longitude,
          heading: sample.bearing ?? 0,
          speed: sample.speed != null ? Math.max(0, sample.speed) : 0,
        });
        lastWritten = { latitude: sample.latitude, longitude: sample.longitude };
        lastWrittenTime = sample.timestamp;
      } catch (err) {
        logger.warn('driverRideSink: setDriverLocation failed', err);
      }
    },

    notificationCopy(): NotificationCopy {
      const cityLabel = (s?: string) => (s ? s.split(',')[0].trim() : '');
      const body = meta?.fromAddress && meta?.toAddress
        ? `${cityLabel(meta.fromAddress)} → ${cityLabel(meta.toAddress)}`
        : 'Sharing your location with passengers';
      return {
        title: 'ZygoRide — Trip Active',
        body,
        channelId: 'ride-tracking',
        importance: 'low',
        colorHex: '#4F46E5',
      };
    },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/sinks/driver-ride.sink.test.ts 2>&1 | tail -20
```
Expected: all 6 tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/location/sinks/driver-ride.sink.ts src/__tests__/services/location/sinks/driver-ride.sink.test.ts && git commit -m "feat(safety): driverRideSink with 25m/30min throttle regression guard"
```

---

## Task 10: live-tracking sink (stop-flag listener + degraded callback + battery throttle)

**Files:**
- Create: `Carpool/src/services/location/sinks/live-tracking.sink.ts`
- Create: `Carpool/src/__tests__/services/location/sinks/live-tracking.sink.test.ts`

- [ ] **Step 1: Write failing test**

Create `Carpool/src/__tests__/services/location/sinks/live-tracking.sink.test.ts`:

```ts
jest.mock('../../../../services/firebase/realtime.service', () => ({
  realtimeService: {
    setLiveTracking: jest.fn().mockResolvedValue(undefined),
    subscribeLiveTrackingStatus: jest.fn(),
  },
}));

jest.mock('../../../../services/tracking/battery.service', () => ({
  batteryService: { getBatteryPct: jest.fn(() => 80) },
}));

import { realtimeService } from '../../../../services/firebase/realtime.service';
import { createLiveTrackingSink } from '../../../../services/location/sinks/live-tracking.sink';

const setLiveTracking = realtimeService.setLiveTracking as jest.Mock;
const subscribeStatus = realtimeService.subscribeLiveTrackingStatus as jest.Mock;

const sample = (ts: number) => ({
  latitude: 1, longitude: 2, accuracy: 5, speed: 0, bearing: 0, timestamp: ts,
});

describe('liveTrackingSink — SOS mode', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    subscribeStatus.mockReturnValue(() => {});
  });

  it('writes every sample at throttle factor 1', async () => {
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop: jest.fn(), onDegraded: jest.fn() });
    await sink.attach();
    await sink.onSample(sample(1), 1);
    await sink.onSample(sample(2), 1);
    await sink.onSample(sample(3), 1);
    expect(setLiveTracking).toHaveBeenCalledTimes(3);
  });

  it('writes every 2nd sample at throttle factor 2', async () => {
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop: jest.fn(), onDegraded: jest.fn() });
    await sink.attach();
    await sink.onSample(sample(1), 2);
    await sink.onSample(sample(2), 2);
    await sink.onSample(sample(3), 2);
    await sink.onSample(sample(4), 2);
    expect(setLiveTracking).toHaveBeenCalledTimes(2);
  });

  it('writes every 4th sample at throttle factor 4', async () => {
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop: jest.fn(), onDegraded: jest.fn() });
    await sink.attach();
    for (let i = 1; i <= 8; i++) await sink.onSample(sample(i), 4);
    expect(setLiveTracking).toHaveBeenCalledTimes(2);
  });

  it('attaches a stop-flag listener and calls onStop when status becomes "STOPPED"', async () => {
    const onStop = jest.fn();
    let captured: ((s: string | null) => void) | null = null;
    subscribeStatus.mockImplementation((_token: string, cb: (s: string | null) => void) => {
      captured = cb;
      return () => {};
    });
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop, onDegraded: jest.fn() });
    await sink.attach();
    captured?.('STOPPED');
    expect(onStop).toHaveBeenCalledWith('tok_a');
  });

  it('calls onDegraded after 3 consecutive write failures', async () => {
    setLiveTracking.mockRejectedValue(new Error('rtdb error'));
    const onDegraded = jest.fn();
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop: jest.fn(), onDegraded });
    await sink.attach();
    await sink.onSample(sample(1), 1);
    await sink.onSample(sample(2), 1);
    expect(onDegraded).not.toHaveBeenCalled();
    await sink.onSample(sample(3), 1);
    expect(onDegraded).toHaveBeenCalledWith('tok_a');
  });

  it('resets degraded counter after a successful write', async () => {
    setLiveTracking.mockRejectedValueOnce(new Error('1')).mockRejectedValueOnce(new Error('2')).mockResolvedValueOnce(undefined).mockRejectedValue(new Error('3'));
    const onDegraded = jest.fn();
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop: jest.fn(), onDegraded });
    await sink.attach();
    await sink.onSample(sample(1), 1);
    await sink.onSample(sample(2), 1);
    await sink.onSample(sample(3), 1); // success
    await sink.onSample(sample(4), 1); // fail #1 again
    expect(onDegraded).not.toHaveBeenCalled();
  });

  it('detach unsubscribes the stop-flag listener', async () => {
    const unsub = jest.fn();
    subscribeStatus.mockReturnValue(unsub);
    const sink = createLiveTrackingSink('tok_a', 'sos', { onStop: jest.fn(), onDegraded: jest.fn() });
    await sink.attach();
    await sink.detach();
    expect(unsub).toHaveBeenCalled();
  });
});

describe('liveTrackingSink — share mode', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    subscribeStatus.mockReturnValue(() => {});
  });

  it('uses effective factor 2 × battery factor (share runs half cadence of SOS)', async () => {
    const sink = createLiveTrackingSink('tok_s', 'share', { onStop: jest.fn(), onDegraded: jest.fn() });
    await sink.attach();
    // throttle factor 1 from battery, but share base is 2 → effective 2
    await sink.onSample(sample(1), 1);
    await sink.onSample(sample(2), 1);
    await sink.onSample(sample(3), 1);
    await sink.onSample(sample(4), 1);
    expect(setLiveTracking).toHaveBeenCalledTimes(2);
  });

  it('has lower notification priority than SOS', async () => {
    const sosSink = createLiveTrackingSink('tok_sos', 'sos', { onStop: jest.fn(), onDegraded: jest.fn() });
    const shareSink = createLiveTrackingSink('tok_share', 'share', { onStop: jest.fn(), onDegraded: jest.fn() });
    expect(sosSink.notificationPriority).toBe(2);
    expect(shareSink.notificationPriority).toBe(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/sinks/live-tracking.sink.test.ts 2>&1 | tail -20
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement live-tracking sink**

Create `Carpool/src/services/location/sinks/live-tracking.sink.ts`:

```ts
import { realtimeService } from '../../firebase/realtime.service';
import { batteryService } from '../../tracking/battery.service';
import { logger } from '../../../utils/logger';
import type { LocationSample, LocationSink, NotificationCopy } from './types';

interface SinkCallbacks {
  onStop: (token: string) => void;
  onDegraded: (token: string) => void;
}

const DEGRADED_THRESHOLD = 3;

export function createLiveTrackingSink(
  token: string,
  mode: 'sos' | 'share',
  callbacks: SinkCallbacks,
): LocationSink {
  let sampleIndex = 0;
  let consecutiveFailures = 0;
  let unsubscribeStatus: (() => void) | null = null;
  const baseFactor = mode === 'sos' ? 1 : 2;

  return {
    id: `live-${token}`,
    mode,
    notificationPriority: mode === 'sos' ? 2 : 1,

    async attach() {
      sampleIndex = 0;
      consecutiveFailures = 0;
      unsubscribeStatus = realtimeService.subscribeLiveTrackingStatus(token, (status) => {
        if (status === 'STOPPED') callbacks.onStop(token);
      });
    },

    async detach() {
      if (unsubscribeStatus) {
        try { unsubscribeStatus(); } catch { /* best-effort */ }
        unsubscribeStatus = null;
      }
    },

    async onSample(sample: LocationSample, throttleFactor: 1 | 2 | 4) {
      sampleIndex += 1;
      const effective = baseFactor * throttleFactor;
      if (sampleIndex % effective !== 0) return;

      try {
        await realtimeService.setLiveTracking(token, {
          lat: sample.latitude,
          lng: sample.longitude,
          accuracy: sample.accuracy,
          speed: sample.speed,
          bearing: sample.bearing,
          batteryPct: batteryService.getBatteryPct(),
          updatedAt: sample.timestamp,
        });
        consecutiveFailures = 0;
      } catch (err) {
        consecutiveFailures += 1;
        logger.warn(`liveTrackingSink(${token}): write failed (${consecutiveFailures}/${DEGRADED_THRESHOLD})`, err);
        if (consecutiveFailures >= DEGRADED_THRESHOLD) {
          callbacks.onDegraded(token);
        }
      }
    },

    notificationCopy(): NotificationCopy {
      if (mode === 'sos') {
        return {
          title: '🚨 ZygoRide — Emergency tracking active',
          body: 'Trusted contacts are receiving your live location.',
          channelId: 'ride-tracking-sos',
          importance: 'high',
          colorHex: '#D93025',
        };
      }
      return {
        title: 'ZygoRide — Sharing live ride',
        body: 'Anyone with your share link can see your current location.',
        channelId: 'ride-tracking',
        importance: 'low',
        colorHex: '#4F46E5',
      };
    },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/sinks/live-tracking.sink.test.ts 2>&1 | tail -20
```
Expected: all 9 tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/location/sinks/live-tracking.sink.ts src/__tests__/services/location/sinks/live-tracking.sink.test.ts && git commit -m "feat(safety): liveTrackingSink with battery throttle, stop-flag listener, degraded callback"
```

---

## Task 11: liveTracking.service.ts public API

**Files:**
- Create: `Carpool/src/services/tracking/liveTracking.service.ts`
- Create: `Carpool/src/__tests__/services/tracking/liveTracking.test.ts`

- [ ] **Step 1: Write failing test**

Create `Carpool/src/__tests__/services/tracking/liveTracking.test.ts`:

```ts
jest.mock('../../../services/location/location-core.service', () => ({
  locationCoreService: {
    registerSink: jest.fn().mockResolvedValue(undefined),
    unregisterSink: jest.fn().mockResolvedValue(undefined),
    getActiveSinks: jest.fn(() => []),
  },
}));

jest.mock('../../../services/location/sinks/live-tracking.sink', () => ({
  createLiveTrackingSink: jest.fn((token, mode, cbs) => ({
    id: `live-${token}`, mode, notificationPriority: mode === 'sos' ? 2 : 1,
    attach: jest.fn(), detach: jest.fn(), onSample: jest.fn(),
    notificationCopy: () => ({ title: '', body: '', channelId: 'ride-tracking', importance: 'low', colorHex: '#000' }),
    __cbs: cbs,
  })),
}));

import { locationCoreService } from '../../../services/location/location-core.service';
import { createLiveTrackingSink } from '../../../services/location/sinks/live-tracking.sink';
import { liveTrackingService } from '../../../services/tracking/liveTracking.service';

const register = locationCoreService.registerSink as jest.Mock;
const unregister = locationCoreService.unregisterSink as jest.Mock;

describe('liveTrackingService', () => {
  beforeEach(() => jest.clearAllMocks());

  it('startTracking(token, "sos") registers a sos sink', async () => {
    await liveTrackingService.startTracking('tok_a', 'sos');
    expect(createLiveTrackingSink).toHaveBeenCalledWith('tok_a', 'sos', expect.any(Object));
    expect(register).toHaveBeenCalled();
  });

  it('startTracking(token, "share") registers a share sink', async () => {
    await liveTrackingService.startTracking('tok_b', 'share');
    expect(createLiveTrackingSink).toHaveBeenCalledWith('tok_b', 'share', expect.any(Object));
    expect(register).toHaveBeenCalled();
  });

  it('startTracking is idempotent — second call with same token is a no-op', async () => {
    (locationCoreService.getActiveSinks as jest.Mock).mockReturnValueOnce([]).mockReturnValueOnce([{ id: 'live-tok_a' }]);
    await liveTrackingService.startTracking('tok_a', 'sos');
    await liveTrackingService.startTracking('tok_a', 'sos');
    expect(register).toHaveBeenCalledTimes(1);
  });

  it('stopTracking(token) unregisters the sink', async () => {
    await liveTrackingService.stopTracking('tok_a');
    expect(unregister).toHaveBeenCalledWith('live-tok_a');
  });

  it('onStop callback from sink triggers stopTracking automatically', async () => {
    let capturedCbs: any = null;
    (createLiveTrackingSink as jest.Mock).mockImplementationOnce((_t, _m, cbs) => {
      capturedCbs = cbs;
      return { id: 'live-tok_a', mode: 'sos', notificationPriority: 2, attach: jest.fn(), detach: jest.fn(), onSample: jest.fn(), notificationCopy: () => ({}) };
    });
    await liveTrackingService.startTracking('tok_a', 'sos');
    capturedCbs.onStop('tok_a');
    await new Promise((r) => setImmediate(r));
    expect(unregister).toHaveBeenCalledWith('live-tok_a');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/tracking/liveTracking.test.ts 2>&1 | tail -20
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement liveTracking.service.ts**

Create `Carpool/src/services/tracking/liveTracking.service.ts`:

```ts
import { locationCoreService } from '../location/location-core.service';
import { createLiveTrackingSink } from '../location/sinks/live-tracking.sink';
import { logger } from '../../utils/logger';

type TrackingMode = 'sos' | 'share';

class LiveTrackingService {
  private degradedHandlers = new Map<string, (token: string) => void>();

  async startTracking(
    token: string,
    mode: TrackingMode,
    options?: { onDegraded?: (token: string) => void },
  ): Promise<void> {
    const sinkId = `live-${token}`;
    const active = locationCoreService.getActiveSinks();
    if (active.some((s) => s.id === sinkId)) {
      logger.debug(`liveTrackingService: ${sinkId} already active`);
      return;
    }
    if (options?.onDegraded) this.degradedHandlers.set(token, options.onDegraded);

    const sink = createLiveTrackingSink(token, mode, {
      onStop: (t) => {
        this.stopTracking(t).catch((err) =>
          logger.warn(`liveTrackingService: auto-stop failed for ${t}`, err),
        );
      },
      onDegraded: (t) => {
        const handler = this.degradedHandlers.get(t);
        if (handler) handler(t);
      },
    });
    await locationCoreService.registerSink(sink);
  }

  async stopTracking(token: string): Promise<void> {
    const sinkId = `live-${token}`;
    await locationCoreService.unregisterSink(sinkId);
    this.degradedHandlers.delete(token);
  }
}

export const liveTrackingService = new LiveTrackingService();
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/tracking/liveTracking.test.ts 2>&1 | tail -20
```
Expected: all 5 tests pass.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/tracking/liveTracking.service.ts src/__tests__/services/tracking/liveTracking.test.ts && git commit -m "feat(safety): liveTrackingService public API for Phase F/G"
```

---

## Task 12: Refactor rideLocationService into a facade over locationCoreService

**Files:**
- Modify: `Carpool/src/services/location/ride-location.service.ts`
- Modify: `Carpool/src/services/location/index.ts`
- Create: `Carpool/src/__tests__/services/location/ride-location.facade.test.ts`

- [ ] **Step 1: Write failing facade compat test**

Create `Carpool/src/__tests__/services/location/ride-location.facade.test.ts`:

```ts
jest.mock('../../../services/location/location-core.service', () => ({
  locationCoreService: {
    registerSink: jest.fn().mockResolvedValue(undefined),
    unregisterSink: jest.fn().mockResolvedValue(undefined),
    getActiveSinks: jest.fn(() => []),
  },
}));

jest.mock('../../../services/location/sinks/driver-ride.sink', () => ({
  createDriverRideSink: jest.fn((rideId, meta) => ({
    id: 'driver-ride', mode: 'driver-ride', notificationPriority: 0,
    attach: jest.fn(), detach: jest.fn(), onSample: jest.fn(),
    notificationCopy: () => ({ title: 'ZygoRide — Trip Active', body: '', channelId: 'ride-tracking', importance: 'low', colorHex: '#4F46E5' }),
    __rideId: rideId, __meta: meta,
  })),
}));

import { locationCoreService } from '../../../services/location/location-core.service';
import { createDriverRideSink } from '../../../services/location/sinks/driver-ride.sink';
import { rideLocationService } from '../../../services/location/ride-location.service';

const register = locationCoreService.registerSink as jest.Mock;
const unregister = locationCoreService.unregisterSink as jest.Mock;

describe('rideLocationService facade', () => {
  beforeEach(() => jest.clearAllMocks());

  it('startTracking(rideId, meta) registers a driver-ride sink via the core', async () => {
    await rideLocationService.startTracking('ride_123', { fromAddress: 'A', toAddress: 'B', totalDistanceM: 1000 });
    expect(createDriverRideSink).toHaveBeenCalledWith('ride_123', expect.objectContaining({ fromAddress: 'A', toAddress: 'B' }));
    expect(register).toHaveBeenCalled();
  });

  it('startTracking is idempotent for the same rideId', async () => {
    await rideLocationService.startTracking('ride_123');
    (locationCoreService.getActiveSinks as jest.Mock).mockReturnValueOnce([{ id: 'driver-ride' }]);
    await rideLocationService.startTracking('ride_123');
    expect(register).toHaveBeenCalledTimes(1);
  });

  it('stopTracking unregisters driver-ride from the core', async () => {
    await rideLocationService.startTracking('ride_123');
    await rideLocationService.stopTracking();
    expect(unregister).toHaveBeenCalledWith('driver-ride');
  });

  it('currentRideId returns the active rideId after startTracking', async () => {
    await rideLocationService.startTracking('ride_123');
    expect(rideLocationService.currentRideId).toBe('ride_123');
  });

  it('isTracking reflects driver-ride sink presence', async () => {
    (locationCoreService.getActiveSinks as jest.Mock).mockReturnValueOnce([{ id: 'driver-ride' }]);
    expect(rideLocationService.isTracking).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/ride-location.facade.test.ts 2>&1 | tail -20
```
Expected: FAIL — `rideLocationService` exists today but its API doesn't match the facade contract yet.

- [ ] **Step 3: Replace internals of ride-location.service.ts with a facade**

Replace the entire contents of `Carpool/src/services/location/ride-location.service.ts` with:

```ts
/**
 * RideLocationService — facade over locationCoreService
 *
 * Preserves the legacy public API (startTracking, stopTracking, updateNotification,
 * resumeTracking, isTracking, currentRideId) so existing callers in app/live-ride.tsx
 * continue to work unchanged. Internally delegates to locationCoreService, registering
 * a single 'driver-ride' sink. The 25m/30min write throttle now lives in the sink.
 *
 * Dependencies:
 *  - locationCoreService (sink registry + GPS task + foreground notification)
 *  - createDriverRideSink (sink implementation with throttle preserved)
 */

import { locationCoreService } from './location-core.service';
import { createDriverRideSink } from './sinks/driver-ride.sink';

export interface TrackingMeta {
  fromAddress?: string;
  toAddress?: string;
  totalDistanceM?: number;
}

class RideLocationService {
  private _activeRideId: string | null = null;

  async requestPermissions(): Promise<boolean> {
    // Delegated to locationPermission.ensure() via locationCoreService.registerSink.
    // Kept as a no-op for legacy callers; throws are surfaced via startTracking.
    return true;
  }

  async hasPermissions(): Promise<boolean> {
    // Kept for compat; not used in the new flow. Always true — actual check happens
    // inside locationCoreService.registerSink, which throws if permission denied.
    return true;
  }

  async startTracking(rideId: string, meta?: TrackingMeta): Promise<void> {
    const active = locationCoreService.getActiveSinks();
    if (active.some((s) => s.id === 'driver-ride') && this._activeRideId === rideId) {
      return;
    }
    if (this._activeRideId && this._activeRideId !== rideId) {
      await this.stopTracking();
    }
    const sink = createDriverRideSink(rideId, meta);
    await locationCoreService.registerSink(sink);
    this._activeRideId = rideId;
  }

  async stopTracking(): Promise<void> {
    await locationCoreService.unregisterSink('driver-ride');
    this._activeRideId = null;
  }

  async resumeTracking(rideId: string, meta?: TrackingMeta): Promise<void> {
    return this.startTracking(rideId, meta);
  }

  /**
   * Legacy method retained for live-ride.tsx ETA updates. Notification copy with
   * ETA is currently handled inside the core's reconciler; the body/progress
   * variants from the old service are out of scope for Phase B and tracked
   * separately (todo.md Phase H wiring).
   */
  async updateNotification(_distanceRemainingM: number | null, _etaSeconds: number | null): Promise<void> {
    // No-op in Phase B. The core's reconciler picks the top-priority sink's
    // notificationCopy on every register/unregister; ETA-driven body text will
    // be re-introduced in Phase H by extending NotificationCopy with optional eta fields.
  }

  get isTracking(): boolean {
    return locationCoreService.getActiveSinks().some((s) => s.id === 'driver-ride');
  }

  get currentRideId(): string | null {
    return this._activeRideId;
  }
}

export const rideLocationService = new RideLocationService();
```

- [ ] **Step 4: Update barrel export to re-export the core too**

Edit `Carpool/src/services/location/index.ts`. Append:

```ts
export { locationCoreService } from './location-core.service';
export { liveTrackingService } from '../tracking/liveTracking.service';
```

(Leave the existing `rideLocationService` re-export untouched.)

- [ ] **Step 5: Run facade tests + existing tests to verify pass**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location 2>&1 | tail -30
```
Expected: facade tests pass; pre-existing location tests (if any) also pass.

- [ ] **Step 6: Verify TypeScript across the whole project**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```
Expected: exit 0. If errors mention `updateNotification` callers expecting different behavior, that's expected scope deferral — the no-op body is documented; live-ride.tsx call sites still compile.

- [ ] **Step 7: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git add src/services/location/ride-location.service.ts src/services/location/index.ts src/__tests__/services/location/ride-location.facade.test.ts && git commit -m "refactor(safety): rideLocationService becomes a facade over locationCore"
```

---

## Task 13: Full-suite verification + deploy RTDB rules

**Files:**
- No code changes — verification + deploy only.

- [ ] **Step 1: Run the full frontend test suite**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest 2>&1 | tail -40
```
Expected: all suites pass. Pay particular attention to any test file that previously imported `rideLocationService` — those should still work via the facade.

- [ ] **Step 2: Run the full backend test suite**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npx jest 2>&1 | tail -40
```
Expected: all tests pass including the new `liveTracking` RTDB rule tests.

- [ ] **Step 3: Lint + type-check both sides**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit && cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions && npx tsc --noEmit && npm run lint 2>&1 | tail -10
```
Expected: exit 0 on both `tsc` runs and lint clean.

- [ ] **Step 4: Smoke-test in the emulator (optional but recommended)**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend && firebase emulators:start --only database
```
In another shell, hit the emulator RTDB at `127.0.0.1:9000` and verify `liveTracking/{token}` allows anonymous read, denies unauthenticated write, and allows owner write once the index node is seeded. (The rules tests already automate this; manual is for peace of mind before deploy.)

Stop emulators with Ctrl-C when done.

- [ ] **Step 5: Deploy RTDB rules**

Run:
```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend && firebase deploy --only database
```
Expected: `+  Deploy complete!` Firebase Console → Realtime Database → Rules tab shows the new `liveTracking`, `sosAlertsByToken`, `rideSharesByToken` blocks.

- [ ] **Step 6: Final commit**

If `npm install` has changed `package-lock.json` files that are not yet committed, commit them now. Otherwise nothing to commit.

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && git status
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend && git status
```
Verify both working trees clean.

- [ ] **Step 7: Update todo.md Phase B status**

Edit `/Users/sairajaggani/Desktop/Space/Project/todo.md`. In §0 status table, flip Phase B's status to `✅ Implemented; awaiting Phase F/G consumption for end-to-end verification.` In §4 acceptance criteria, check the boxes for items implemented in this plan; leave the items that need real-device manual QA (foreground 10-min lock test, battery saver real-device) unchecked and defer them to §J.4.

---

## Self-review

**Spec coverage check:**
- §4 architecture diagram (unified core + sinks + reconciler + AsyncStorage) → Tasks 5, 7, 8.
- §4.1 sink contract → Task 5.
- §4.2 notification priority → Tasks 8 (reconciler), 9 (driver-ride copy), 10 (sos + share copy).
- §5.1 facade refactor → Task 12.
- §5.2 new files: `location-core.service.ts` (7, 8), `driver-ride.sink.ts` (9), `live-tracking.sink.ts` (10), `liveTracking.service.ts` (11), `locationPermission.ts` (4), `battery.service.ts` (3).
- §5.3 RTDB rules → Task 2.
- §5.4 config (expo-battery + FOREGROUND_SERVICE_LOCATION) → Task 1.
- §6 data flow (fan-out + per-sink modulo throttle) → Tasks 8, 9, 10.
- §7 lifecycle and recovery → Task 7 (bootstrap test), Task 8 (GPS lifecycle).
- §8 error handling → Tasks 3 (battery fallback), 4 (canAskAgain), 9 (rtdb fail tolerated), 10 (degraded callback).
- §9 testing — every test file in §9.1 has a corresponding task.
- §10 acceptance criteria → Task 13.

**Placeholder scan:** Task 12's `updateNotification` no-op is documented as a known scope deferral — not a placeholder. No `TBD`/`TODO` markers in steps.

**Type consistency check:**
- `LocationSample` shape: defined in Task 5 (`latitude`, `longitude`, `accuracy`, `speed`, `bearing`, `timestamp`). Consumed in Tasks 8 (handler builds it), 9 (drives `setDriverLocation`), 10 (drives `setLiveTracking`). All match.
- `setLiveTracking` payload (`lat`, `lng`, `accuracy`, `speed`, `bearing`, `batteryPct`, `updatedAt`): defined in Task 6, consumed in Task 10. Matches.
- Sink id conventions: `driver-ride` (Task 9, used in Task 12 facade) and `live-${token}` (Task 10, used in Task 11 public API). Consistent.
- `subscribeLiveTrackingStatus` signature: returns an unsubscribe fn (Task 6); consumed as unsubscribe fn (Task 10). Matches.

Plan ready.

---

**Plan complete and saved to `docs/superpowers/plans/2026-06-09-phase-b-live-tracking.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

**Which approach?**
