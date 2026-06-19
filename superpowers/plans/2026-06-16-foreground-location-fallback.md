# Foreground Location Fallback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow the app to function when the driver grants only "While Using App" location permission, falling back to `watchPositionAsync` instead of hard-blocking, and show a driver-friendly banner (no word "tracking") that lets them upgrade to background access.

**Architecture:** `locationCoreService.ensureGpsRunning` switches from requesting permissions itself to just checking them via `getStatus()`, then branches — background granted → `startLocationUpdatesAsync` (unchanged), background denied → `watchPositionAsync` foreground fallback. A new `isForegroundOnly` getter and `upgradeToBackground()` method let `live-ride.tsx` detect and resolve the degraded state. `locationPermissionFlow.ts` gets driver-friendly copy (no "tracking") and a new `driver-ride-upgrade` reason for the banner tap path.

**Tech Stack:** React Native, Expo SDK 54, expo-location, TypeScript, Jest

---

## File Map

| File | Change |
|---|---|
| `Carpool/src/services/location/location-core.service.ts` | Add `_fgWatchSub`, `isForegroundOnly`, `upgradeToBackground`, update `ensureGpsRunning` + `stopGpsIfIdle` + `__resetForTests` |
| `Carpool/src/services/tracking/locationPermissionFlow.ts` | Update driver copy (no "tracking"), add `driver-ride-upgrade` reason |
| `Carpool/app/live-ride.tsx` | Accept `BACKGROUND_DENIED`, add `foregroundOnlyMode` state, banner, upgrade handler |
| `Carpool/src/__tests__/services/location/location-core.test.ts` | Add tests for foreground-only mode and `upgradeToBackground` |

---

## Task 1: Update `locationPermissionFlow.ts` — driver-friendly copy + upgrade reason

**Files:**
- Modify: `Carpool/src/services/tracking/locationPermissionFlow.ts`

- [ ] **Step 1: Add `driver-ride-upgrade` to the reason union type**

In `locationPermissionFlow.ts`, the `FlowOptions` interface currently allows:
```ts
reason?: 'share' | 'sos' | 'driver-ride' | 'passenger-ride';
```
Change it to:
```ts
reason?: 'share' | 'sos' | 'driver-ride' | 'passenger-ride' | 'driver-ride-upgrade';
```

- [ ] **Step 2: Replace the `driver-ride` copy and add `driver-ride-upgrade`**

Replace the `PRE_PROMPT_COPY` object (lines 14–35 in the current file) with:

```ts
const PRE_PROMPT_COPY: Record<NonNullable<FlowOptions['reason']>, { title: string; body: string }> = {
  share: {
    title: 'Live sharing needs your location',
    body:
      'To share your live position with the people you choose, ZygoRide needs access to your location even when the app is in the background. We only use it during active shares — never silently.',
  },
  sos: {
    title: 'Emergency location needs your location',
    body:
      'Triggering SOS sends your live position to your trusted contacts so they can find you. ZygoRide needs background location access for this to work even if the app is locked.',
  },
  'driver-ride': {
    title: 'Let passengers see where you are',
    body:
      'Your passengers need to see your position during pickup and the ride. Allow background access so they can find you even when your screen is off — we only use it during active rides.',
  },
  'driver-ride-upgrade': {
    title: 'Stay visible to your passengers',
    body:
      "Your passengers can't see where you are when your screen locks. Allow background access and they'll always know where to meet you — only used during active rides.",
  },
  'passenger-ride': {
    title: 'Live ride needs your location',
    body:
      'Your driver and ZygoRide need to know where you are during the pickup and ride. We use background location only during active rides.',
  },
};
```

- [ ] **Step 3: Update `ALWAYS_EXPLAINER` to remove "tracking" language**

Replace lines 37–43 (the `ALWAYS_EXPLAINER` const) with:

```ts
const ALWAYS_EXPLAINER = {
  title: 'One more step',
  body:
    Platform.OS === 'ios'
      ? 'Tap "Always Allow" on the next screen — so your passengers can find you even when your phone is locked. Only used during active rides.'
      : 'Tap "Allow all the time" on the next screen — so your passengers can find you even when the app is in the background. Only used during active rides.',
};
```

- [ ] **Step 4: Update `showSettingsRecovery` to remove "tracking" language**

Replace the `message` variable inside `showSettingsRecovery` (lines 91–93) with:

```ts
const message =
  reason === 'sos'
    ? 'Emergency location needs "Always" access. Open Settings and choose "Always" under Location.'
    : 'Passengers need to see your position even with the screen locked. Open Settings and choose "Always" under Location.';
```

- [ ] **Step 5: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit 2>&1 | head -30
```

Expected: no errors in `locationPermissionFlow.ts`.

---

## Task 2: Update `location-core.service.ts` — foreground fallback + upgrade

**Files:**
- Modify: `Carpool/src/services/location/location-core.service.ts`

- [ ] **Step 1: Add `_fgWatchSub` field to the class**

Inside `class LocationCoreService`, after the `sampleListeners` field (line 55), add:

```ts
private _fgWatchSub: ExpoLocation.LocationSubscription | null = null;
```

- [ ] **Step 2: Add `isForegroundOnly` getter**

After the `subscribeToSamples` method (after line 159), add:

```ts
get isForegroundOnly(): boolean {
  return this._fgWatchSub !== null && !this._gpsStarted;
}
```

- [ ] **Step 3: Replace `ensureGpsRunning`**

Replace the entire `ensureGpsRunning` method (lines 161–187) with:

```ts
private async ensureGpsRunning(): Promise<void> {
  if (this._gpsStarted || this._fgWatchSub) return;

  const status = await locationPermission.getStatus();
  if (!status.foreground) {
    throw new Error('LOCATION_PERMISSION_FOREGROUND_DENIED');
  }

  if (!_taskManagerAvailable || !status.background) {
    this._fgWatchSub = await ExpoLocation.watchPositionAsync(
      {
        accuracy: ExpoLocation.Accuracy.BestForNavigation,
        timeInterval: 5_000,
        distanceInterval: 0,
      },
      (loc) => {
        void this.__handleSample({
          latitude: loc.coords.latitude,
          longitude: loc.coords.longitude,
          accuracy: loc.coords.accuracy ?? null,
          speed: loc.coords.speed ?? null,
          bearing: loc.coords.heading ?? null,
          timestamp: loc.timestamp,
        });
      },
    );
    if (!_taskManagerAvailable) {
      logger.warn('locationCore: TaskManager unavailable — running in foreground-only mode');
    }
    return;
  }

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
  this._gpsStarted = true;
}
```

- [ ] **Step 4: Add `upgradeToBackground` public method**

After `ensureGpsRunning` (before `stopGpsIfIdle`), add:

```ts
async upgradeToBackground(): Promise<void> {
  if (!_taskManagerAvailable || !this._fgWatchSub || this._gpsStarted) return;
  this._fgWatchSub.remove();
  this._fgWatchSub = null;
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
  this._gpsStarted = true;
}
```

- [ ] **Step 5: Update `stopGpsIfIdle` to clean up `_fgWatchSub`**

Replace `stopGpsIfIdle` (lines 189–204) with:

```ts
private async stopGpsIfIdle(): Promise<void> {
  if (this.sinks.size > 0) return;
  if (this._fgWatchSub) {
    this._fgWatchSub.remove();
    this._fgWatchSub = null;
  }
  this._gpsStarted = false;
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
```

- [ ] **Step 6: Update `__resetForTests` to clean up `_fgWatchSub`**

Replace `__resetForTests` (lines 247–252) with:

```ts
async __resetForTests(): Promise<void> {
  for (const id of Array.from(this.sinks.keys())) {
    await this.unregisterSink(id);
  }
  this.sinks.clear();
  this._gpsStarted = false;
  if (this._fgWatchSub) {
    this._fgWatchSub.remove();
    this._fgWatchSub = null;
  }
}
```

- [ ] **Step 7: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit 2>&1 | head -30
```

Expected: no errors in `location-core.service.ts`.

---

## Task 3: Add tests for foreground-only mode

**Files:**
- Modify: `Carpool/src/__tests__/services/location/location-core.test.ts`

- [ ] **Step 1: Add `watchPositionAsync` mock to the `expo-location` mock at the top of the file**

The existing mock (lines 12–17) only has `startLocationUpdatesAsync` and `stopLocationUpdatesAsync`. Replace it with:

```ts
jest.mock('expo-location', () => ({
  startLocationUpdatesAsync: jest.fn().mockResolvedValue(undefined),
  stopLocationUpdatesAsync: jest.fn().mockResolvedValue(undefined),
  watchPositionAsync: jest.fn().mockResolvedValue({ remove: jest.fn() }),
  Accuracy: { BestForNavigation: 6, Balanced: 3 },
  ActivityType: { AutomotiveNavigation: 3 },
}));
```

- [ ] **Step 2: Add foreground-only test suite**

Add the following test suite after the existing `locationCoreService — GPS lifecycle` describe block:

```ts
describe('locationCoreService — foreground-only fallback', () => {
  beforeEach(async () => {
    await locationCoreService.__resetForTests();
    jest.clearAllMocks();
    (TaskManager.isTaskRegisteredAsync as jest.Mock).mockResolvedValue(false);
  });

  it('uses watchPositionAsync when background permission is denied', async () => {
    const { locationPermission } = require('../../../services/tracking/locationPermission');
    (locationPermission.getStatus as jest.Mock).mockResolvedValueOnce({
      foreground: true,
      background: false,
      canAskForeground: false,
      canAskBackground: true,
    });

    await locationCoreService.registerSink(makeSink('s1'));

    expect(ExpoLocation.watchPositionAsync).toHaveBeenCalledTimes(1);
    expect(ExpoLocation.startLocationUpdatesAsync).not.toHaveBeenCalled();
    expect(locationCoreService.isForegroundOnly).toBe(true);
  });

  it('isForegroundOnly is false after full background start', async () => {
    const { locationPermission } = require('../../../services/tracking/locationPermission');
    (locationPermission.getStatus as jest.Mock).mockResolvedValueOnce({
      foreground: true,
      background: true,
      canAskForeground: false,
      canAskBackground: false,
    });

    await locationCoreService.registerSink(makeSink('s1'));

    expect(ExpoLocation.startLocationUpdatesAsync).toHaveBeenCalledTimes(1);
    expect(locationCoreService.isForegroundOnly).toBe(false);
  });

  it('upgradeToBackground stops watchPositionAsync and starts background task', async () => {
    const removeMock = jest.fn();
    (ExpoLocation.watchPositionAsync as jest.Mock).mockResolvedValueOnce({ remove: removeMock });

    const { locationPermission } = require('../../../services/tracking/locationPermission');
    (locationPermission.getStatus as jest.Mock).mockResolvedValueOnce({
      foreground: true,
      background: false,
      canAskForeground: false,
      canAskBackground: true,
    });

    await locationCoreService.registerSink(makeSink('s1'));
    expect(locationCoreService.isForegroundOnly).toBe(true);

    await locationCoreService.upgradeToBackground();

    expect(removeMock).toHaveBeenCalledTimes(1);
    expect(ExpoLocation.startLocationUpdatesAsync).toHaveBeenCalledTimes(1);
    expect(locationCoreService.isForegroundOnly).toBe(false);
  });

  it('upgradeToBackground is a no-op when already in background mode', async () => {
    await locationCoreService.registerSink(makeSink('s1'));
    expect(locationCoreService.isForegroundOnly).toBe(false);

    await locationCoreService.upgradeToBackground();

    expect(ExpoLocation.startLocationUpdatesAsync).toHaveBeenCalledTimes(1); // only from registerSink
  });

  it('stopGpsIfIdle cleans up watchPositionAsync subscription', async () => {
    const removeMock = jest.fn();
    (ExpoLocation.watchPositionAsync as jest.Mock).mockResolvedValueOnce({ remove: removeMock });

    const { locationPermission } = require('../../../services/tracking/locationPermission');
    (locationPermission.getStatus as jest.Mock).mockResolvedValueOnce({
      foreground: true,
      background: false,
      canAskForeground: false,
      canAskBackground: true,
    });

    await locationCoreService.registerSink(makeSink('s1'));
    await locationCoreService.unregisterSink('s1');

    expect(removeMock).toHaveBeenCalledTimes(1);
    expect(locationCoreService.isForegroundOnly).toBe(false);
  });

  it('throws FOREGROUND_DENIED when foreground permission is denied', async () => {
    const { locationPermission } = require('../../../services/tracking/locationPermission');
    (locationPermission.getStatus as jest.Mock).mockResolvedValueOnce({
      foreground: false,
      background: false,
      canAskForeground: false,
      canAskBackground: false,
    });

    await expect(locationCoreService.registerSink(makeSink('s1'))).rejects.toThrow(
      'LOCATION_PERMISSION_FOREGROUND_DENIED',
    );
  });
});
```

- [ ] **Step 3: Run the new tests**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/location-core.test.ts --no-coverage 2>&1 | tail -20
```

Expected: all tests pass (existing + 6 new).

---

## Task 4: Update `live-ride.tsx` — accept foreground-only, show upgrade banner

**Files:**
- Modify: `Carpool/app/live-ride.tsx`

- [ ] **Step 1: Add `foregroundOnlyMode` state**

After the `sosResponse` state declaration (line 332), add:

```ts
const [foregroundOnlyMode, setForegroundOnlyMode] = useState(false);
```

- [ ] **Step 2: Update the driver permission + tracking block to proceed on `BACKGROUND_DENIED`**

Replace lines 368–380 (the `if (isDriver && rideDoc)` block) with:

```ts
if (isDriver && rideDoc) {
  (async () => {
    const outcome = await requestLocationPermissionFlow({ reason: 'driver-ride' });
    if (outcome === 'FOREGROUND_DENIED' || outcome === 'CANCELED') return;
    try {
      await rideLocationService.startTracking(rideId, {
        fromAddress: rideDoc.from?.address,
        toAddress: rideDoc.to?.address,
        totalDistanceM: rideDoc.route?.distanceValue,
      });
      setForegroundOnlyMode(locationCoreService.isForegroundOnly);
    } catch (err) {
      logger.warn('live-ride: rideLocationService.startTracking failed', err);
    }
  })();
}
```

- [ ] **Step 3: Add `handleUpgradeLocationAccess` callback**

After the `handleSheetChanges` callback (after line 318), add:

```ts
const handleUpgradeLocationAccess = useCallback(async () => {
  const outcome = await requestLocationPermissionFlow({ reason: 'driver-ride-upgrade' });
  if (outcome !== 'GRANTED') return;
  try {
    await locationCoreService.upgradeToBackground();
    setForegroundOnlyMode(false);
  } catch (err) {
    logger.warn('live-ride: upgradeToBackground failed', err);
  }
}, []);
```

- [ ] **Step 4: Add foreground-only banner in the driver BottomSheet**

Inside the driver `BottomSheetScrollView` content (after the rerouting banner block that ends around line 1042), add:

```tsx
{/* ── Foreground-only visibility banner ──────────────────────── */}
{foregroundOnlyMode && (
  <TouchableOpacity
    style={styles.fgOnlyBanner}
    onPress={handleUpgradeLocationAccess}
    activeOpacity={0.8}
    accessibilityRole="button"
    accessibilityLabel="Passengers cannot see you when the screen locks. Tap to fix."
  >
    <Ionicons name="eye-off-outline" size={16} color={theme.colors.warning} />
    <Text style={styles.fgOnlyBannerText}>
      Passengers can't see you when screen locks
    </Text>
    <Text style={styles.fgOnlyBannerAction}>Fix this</Text>
  </TouchableOpacity>
)}
```

The rerouting banner block to insert after:
```tsx
{isRerouting && (
  <View style={styles.reroutingBanner}>
    <Ionicons name="warning" size={16} color={theme.colors.warning} />
    <Text style={styles.reroutingBannerText}>Route updated</Text>
  </View>
)}
```

- [ ] **Step 5: Add banner styles to `StyleSheet.create()`**

After the `reroutingBannerText` style entry (around line 1680), add:

```ts
fgOnlyBanner: {
  flexDirection: 'row',
  alignItems: 'center',
  gap: theme.spacing.sm,
  backgroundColor: theme.colors.warning + '20',
  borderRadius: theme.borderRadius.sm,
  paddingHorizontal: theme.spacing.md,
  paddingVertical: theme.spacing.sm,
  marginBottom: theme.spacing.sm,
  borderLeftWidth: 3,
  borderLeftColor: theme.colors.warning,
},
fgOnlyBannerText: {
  flex: 1,
  fontSize: theme.fontSize.xs,
  color: theme.colors.warning,
  fontWeight: theme.fontWeight.semibold as '600',
},
fgOnlyBannerAction: {
  fontSize: theme.fontSize.xs,
  color: theme.colors.primary,
  fontWeight: theme.fontWeight.semibold as '600',
},
```

- [ ] **Step 6: Final type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit 2>&1 | head -30
```

Expected: no errors.

- [ ] **Step 7: Run full location test suite**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx jest src/__tests__/services/location/ src/__tests__/services/tracking/ --no-coverage 2>&1 | tail -20
```

Expected: all pass.
