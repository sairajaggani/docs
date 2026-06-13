# Phase B — Live Tracking Service (Design Spec)

**Phase of:** Carpool Safety / Emergency SOS + Live Ride Share (`todo.md` §4)
**Date:** 2026-06-09
**Status:** Brainstorming complete; awaiting user review before plan write-up
**Source of truth for higher-level decisions:** `/Users/sairajaggani/Desktop/Space/Project/todo.md`

---

## 1. Goal

Build a reusable client-side service that streams device location + battery to Firebase Realtime Database under a server-issued `shareToken`. The service is consumed by two upcoming phases:

- **Phase F (SOS active flow):** publishes location to `liveTracking/{shareToken}` so trusted contacts viewing the public tracking page (Phase C) see live updates.
- **Phase G (Live Ride Share):** same publish pattern for proactive share sessions.

Phase B ships the client service, RTDB rules, and tests. No backend Cloud Functions for SOS/share are created here — those are Phases E and G.

## 2. Constraints and Pre-existing State (audited)

- `expo-location` and `expo-task-manager` are installed; native build wires them in.
- `expo-battery` is **not** installed.
- A driver-side tracking service already exists at `Carpool/src/services/location/ride-location.service.ts`. It owns the only `expo-task-manager` task, the only `expo-location.startLocationUpdatesAsync` subscription in the app, and an `@notifee/react-native` foreground notification. Imported from 5 call sites in `Carpool/app/live-ride.tsx` and re-exported from `Carpool/src/services/location/index.ts`.
- `expo-location` allows **only one** registered location task per app process. A second `startLocationUpdatesAsync` call replaces the first.
- Android can only own **one** foreground-service notification at a time (via `@notifee`'s `asForegroundService: true`).
- `Carpool/app.config.js` already declares iOS `UIBackgroundModes: ["location", "remote-notification"]`. Android permissions include `ACCESS_BACKGROUND_LOCATION` and `FOREGROUND_SERVICE` — **but is missing `FOREGROUND_SERVICE_LOCATION`** (Android 14+ requirement for foreground location service).
- `CarpoolBackend/database.rules.json` already scopes `activeRides/{rideId}` reads/writes to driver + confirmed passengers. There is no `liveTracking/{shareToken}` block yet.

## 3. Architectural Decisions (locked during brainstorming)

| Concern | Decision | Why |
|---|---|---|
| Coexistence with `rideLocationService` | **Unified location core.** One GPS source fans samples to multiple sinks (driver-ride, sos, share). | Parallel cores are impossible: `expo-location` and Android's foreground-service slot each enforce one owner. Sibling services would silently kill driver tracking when SOS fires. |
| Public API stability | **Facade pattern.** `ride-location.service.ts` keeps its file path, exported symbol (`rideLocationService`), and method signatures. Internals delegate to a new `location-core.service.ts`. | Zero call-site churn in `live-ride.tsx`. New code uses the core directly. |
| Battery-saver mechanic | **Sample-then-throttle in callback.** GPS runs at the fastest cadence any sink needs (5s for SOS). Each sink writes every Nth sample, where N depends on a battery-derived throttle factor `1 \| 2 \| 4`. | `expo-location.timeInterval` can't be mutated without stopping + restarting the task — that causes gaps. GPS radio is the dominant battery cost; callback frequency is cheap. |
| Stop signal | **Server-driven flag.** Each sink listens to `rtdb.ref('liveTracking/{token}/status').on('value')`. Backend (Phase E/G CFs) flips it to `'STOPPED'`; client unregisters the sink in response. | Cross-device safe ("I'm Safe" tapped on another phone), OS-kill recoverable, and matches the 2hr auto-resolve and ride-complete revoke paths cleanly. |

## 4. Architecture

```
┌─ Phase F (SOS) ──────────────┐  ┌─ Phase G (Share) ─┐  ┌─ live-ride.tsx ──────┐
│  sosService.start(token)     │  │  shareService     │  │  rideLocationService │
└──────────────┬───────────────┘  └────────┬──────────┘  └──────────┬──────────┘
               │                           │                        │
               ▼                           ▼                        │  (facade)
        ┌─────────────────────────── liveTracking.service.ts ──────┴──────────┐
        │  startTracking(shareToken, mode: 'sos'|'share')                     │
        │  stopTracking(shareToken)                                           │
        └─────────────────────────────────┬───────────────────────────────────┘
                                          ▼
        ┌─────────────────────────── locationCoreService ─────────────────────┐
        │  • Single expo-task-manager task                                    │
        │  • Single expo-location.startLocationUpdatesAsync subscription      │
        │  • Sink registry: registerSink(mode, payload), unregisterSink(id)   │
        │  • AsyncStorage: activeSinks[], lastWriteByToken                    │
        │  • Foreground notification reconciler (priority: SOS > share > ride)│
        └────────┬──────────────────┬──────────────────┬─────────────────────┘
                 ▼                  ▼                  ▼
         driverRideSink         sosSink           rideShareSink
         (activeRides/         (liveTracking/    (liveTracking/
          {rideId}/             {shareToken})     {shareToken})
          driverLocation)
                                  + RTDB stop-flag listener (one per token)
```

### 4.1 Sink contract

```ts
interface LocationSink {
  id: string;                          // 'driver-ride' or `live-${shareToken}`
  mode: 'driver-ride' | 'sos' | 'share';
  onSample(sample: LocationSample, throttleFactor: 1 | 2 | 4): Promise<void>;
  attach(): Promise<void>;             // subscribe to stop-flag, init notification state
  detach(): Promise<void>;             // unsubscribe listeners, flush, no RTDB cleanup (server-owned)
  notificationPriority: 0 | 1 | 2;     // 0=ride, 1=share, 2=sos
  notificationCopy(): NotificationCopy;
}
```

### 4.2 Notification priority

One foreground notification slot. Reconciler runs whenever sink set changes:

- Highest-priority sink active = SOS → `"🚨 ZygoRide — Emergency tracking active"` on a new `ride-tracking-sos` channel with `AndroidImportance.HIGH` and `theme.colors.danger` color.
- Else share active → `"ZygoRide — Sharing live ride"` on existing `ride-tracking` channel.
- Else driver-ride active → current copy `"ZygoRide — Trip Active"` (unchanged behavior).
- No sinks → notification removed, foreground service stopped.

## 5. Files

### 5.1 Refactor

- `Carpool/src/services/location/ride-location.service.ts` — becomes a thin facade. Keeps exports `rideLocationService`, methods `startTracking(rideId, meta)`, `stopTracking()`, `updateNotification(distM, etaSeconds)`, `resumeTracking(rideId, meta)`, getters `isTracking`, `currentRideId`. Internally each call registers / unregisters / updates a single `driver-ride` sink on the core. 25m/30min throttle preserved in the sink, not the facade.

### 5.2 New

- `Carpool/src/services/location/location-core.service.ts` — GPS owner. Sink registry, fan-out, notification reconciler, AsyncStorage persistence, kill-recovery, `expo-task-manager.defineTask`, `expo-location.startLocationUpdatesAsync` / `stopLocationUpdatesAsync`.
- `Carpool/src/services/location/sinks/driver-ride.sink.ts` — wraps `realtimeService.setDriverLocation`. 25m + 30min throttle.
- `Carpool/src/services/location/sinks/live-tracking.sink.ts` — writes `liveTracking/{shareToken}/{lat,lng,accuracy,speed,bearing,batteryPct,updatedAt}`. Listens to `liveTracking/{shareToken}/status`; on `'STOPPED'` unregisters itself. On 3 consecutive write failures fires an `onTrackingDegraded(token)` callback so Phase F can surface "tracking unreliable" UX.
- `Carpool/src/services/tracking/liveTracking.service.ts` — public API. `startTracking(shareToken, mode: 'sos'|'share')` registers a `live-tracking` sink; `stopTracking(shareToken)` unregisters. Double-start is idempotent.
- `Carpool/src/services/tracking/locationPermission.ts` — foreground + background permission orchestration. Re-prompt logic respects `canAskAgain`; falls through to `Linking.openSettings()` when false.
- `Carpool/src/services/tracking/battery.service.ts` — wraps `expo-battery`. Single subscription per app lifecycle. Exposes synchronous `getThrottleFactor(): 1 | 2 | 4` (defaults to `1` on error or before first reading). Thresholds: `≥20% → 1`, `10–20% → 2`, `<10% → 4`.

### 5.3 Backend

- `CarpoolBackend/database.rules.json` — add:
  ```json
  "liveTracking": {
    "$token": {
      ".read": true,
      ".write": "auth != null && (root.child('sosAlertsByToken/' + $token + '/userId').val() == auth.uid || root.child('rideSharesByToken/' + $token + '/userId').val() == auth.uid)"
    }
  },
  "sosAlertsByToken": { "$token": { ".read": false, ".write": false } },
  "rideSharesByToken": { "$token": { ".read": false, ".write": false } }
  ```
  Index nodes (`sosAlertsByToken`, `rideSharesByToken`) are written by backend CFs in Phase E and G; not in scope here. Rules still ship in Phase B so RTDB doesn't reject the very first write when Phase E ships.

### 5.4 Config

- `Carpool/app.config.js` — add `FOREGROUND_SERVICE_LOCATION` to the `android.permissions` array (Android 14+ requirement). Existing iOS background modes are sufficient.
- `package.json` — add `expo-battery` via `npx expo install expo-battery` (from `Carpool/`).

## 6. Data Flow (one GPS sample)

1. `expo-location` callback fires (BestForNavigation, ~5s nominal).
2. Task handler rehydrates `activeSinks` + `lastWriteByToken` from AsyncStorage if the JS context was recreated (kill-recovery).
3. `batteryService.getThrottleFactor()` → `1 | 2 | 4`.
4. Fan out the sample to every registered sink:
   - `driverRideSink.onSample`: skip if `distance < 25m` AND `now − lastWrittenTime < 30min` (the sink's own `lastWrittenLocation` / `lastWrittenTime` — same throttle as today's `rideLocationService`); else `realtimeService.setDriverLocation`. Driver-ride is **not** affected by battery throttle factor.
   - `sosSink.onSample`: increment sample index; skip unless `index % throttleFactor === 0`; else write to `liveTracking/{token}`. Concrete cadences at GPS 5s nominal: factor `1` → 5s writes; factor `2` → 10s writes; factor `4` → 20s writes.
   - `shareSink.onSample`: effective factor = `2 * throttleFactor` (so 5s GPS becomes 10s nominal writes). Concrete cadences: factor `1` → 10s writes; factor `2` → 20s; factor `4` → 40s.
5. Persist updated per-sink write state to AsyncStorage (driver-ride: `lastWrittenLocation` + `lastWrittenTime`; live-tracking sinks: `lastWriteByToken` map).

## 7. Lifecycle and Recovery

- **Register sink:** core ensures location task + foreground notification are running, calls `sink.attach()` (which subscribes to the stop-flag), appends to `activeSinks`, persists, reconciles notification.
- **Unregister sink:** calls `sink.detach()`, removes from `activeSinks`, persists. If `activeSinks` is empty, stops the location task and the foreground notification.
- **App killed by OS:** background location task is preserved by the OS until the user reboots or denies permission. When the task next fires, the JS context is recreated empty. Task handler rehydrates `activeSinks` from AsyncStorage; each sink's `attach()` is re-invoked to re-establish stop-flag listeners.
- **App foregrounded:** `AppState` listener re-attaches any stop-flag listeners that errored out while in background (auth refresh, network).
- **Server-driven stop:** sink's stop-flag listener fires `'STOPPED'`; sink calls `unregisterSink(this.id)`.

## 8. Error Handling

| Failure | Behavior |
|---|---|
| Foreground permission denied | `liveTracking.service.startTracking` rejects with typed error; caller (Phase F/G) surfaces via `showError`. No partial state. |
| Background permission denied (foreground granted) | Sink still registers; one-shot rationale toast with "Open Settings" link. Phase H wires onboarding. |
| RTDB write fails | `logger.warn`; SDK queues offline replay. SOS sink only: after 3 consecutive failures, fires `onTrackingDegraded(token)`; degraded state recovers on next successful write. |
| `expo-battery` throws / returns invalid | `getThrottleFactor` returns `1`. Tracking continues at full cadence. |
| Stop-flag listener errors (auth lost, network) | Logged; sink stays active; listener re-attaches on next `AppState` foreground transition. |
| Forged shareToken (UID mismatch) | RTDB rules deny; sink receives permission-denied; fires `onTrackingDegraded` and unregisters itself. No silent retry loop. |
| `expo-task-manager` native module not available (Expo Go) | `_taskManagerAvailable` flag stays false (existing pattern); `startTracking` logs a warning and no-ops. Same behavior as today's driver-ride flow. |

## 9. Testing

### 9.1 Unit tests (Jest, frontend, mocked deps)

| File | Covers |
|---|---|
| `Carpool/src/__tests__/services/location/location-core.test.ts` | Sink registry: register / unregister / idempotency; fan-out invokes each sink once per sample; notification reconciler picks highest priority; kill-recovery rehydrates `activeSinks` from AsyncStorage; permission failure aborts cleanly |
| `Carpool/src/__tests__/services/location/sinks/driver-ride.sink.test.ts` | 25m + 30min throttle preserved (regression guard); writes route through `realtimeService.setDriverLocation` with correct payload shape |
| `Carpool/src/__tests__/services/location/sinks/live-tracking.sink.test.ts` | Throttle factor 1/2/4 writes every 1st/2nd/4th sample; stop-flag `'STOPPED'` unregisters the sink; 3 consecutive write failures fire `onTrackingDegraded`; battery service throw defaults to factor 1 |
| `Carpool/src/__tests__/services/tracking/battery.service.test.ts` | `getThrottleFactor` returns 1 / 2 / 4 at >20% / 10–20% / <10%; subscription unsubscribes on shutdown |
| `Carpool/src/__tests__/services/tracking/liveTracking.test.ts` | Public API: `startTracking(token, 'sos')` registers sink with mode=sos; `stopTracking(token)` unregisters and detaches stop-flag listener; double-start is idempotent |
| `Carpool/src/__tests__/services/location/ride-location.facade.test.ts` | Facade compat: `rideLocationService.startTracking(rideId, meta)` still routes through the new core; existing notification copy preserved when no SOS/share active; `updateNotification(distM, etaSeconds)` still updates ETA |

### 9.2 Backend integration tests (Jest + Firebase emulator)

| File | Covers |
|---|---|
| `CarpoolBackend/functions/tests/security/rules.test.ts` (extend with a new `describe('Realtime DB — liveTracking rules')` block) | `liveTracking/{token}` read is public; write requires UID matching index node; forged token writes denied; `sosAlertsByToken` / `rideSharesByToken` deny FE read/write entirely |

### 9.3 TDD order

Rules tests → core sink-registry test → driver-ride sink regression test → live-tracking sink → battery service → liveTracking public API → facade compat. Each fails first, then implementation lands.

### 9.4 Coverage target

≥90% statements on `Carpool/src/services/location/` and `Carpool/src/services/tracking/` per `todo.md` §J.1.

### 9.5 Out of scope (defer to `todo.md` §J.4 manual QA on real devices)

- Real GPS with phone locked ≥10 min in foreground-service mode.
- Real Android foreground-service notification rendering and persistence.
- Real battery threshold cross during a live SOS.
- Real iOS background location continuation across screen-lock.

## 10. Phase B Acceptance Criteria

(Restated from `todo.md` §4, refined here.)

- [ ] `expo-battery` installed via `npx expo install`.
- [ ] `Carpool/app.config.js` includes `FOREGROUND_SERVICE_LOCATION` in `android.permissions`.
- [ ] `locationCoreService` owns the only `expo-location` task and foreground notification.
- [ ] `rideLocationService` facade preserves API and behavior; `live-ride.tsx` unchanged.
- [ ] `liveTracking.service.ts` `startTracking` / `stopTracking` register and unregister sinks idempotently.
- [ ] Adaptive throttle factor `1 / 2 / 4` applied at sink level based on `batteryService.getThrottleFactor()`.
- [ ] Server-driven stop: writing `'STOPPED'` to `liveTracking/{token}/status` causes client to unregister the sink within one event loop tick.
- [ ] RTDB rules deny writes from non-owner UID; deny FE read/write of index nodes.
- [ ] All unit and rules tests in §9 pass. Coverage ≥90% on the two service directories.

## 11. Out of Scope (Phase B)

- Backend CFs `triggerSos`, `resolveSos`, `createRideShare`, `revokeRideShare`, `sosAutoResolve`, `rideShareCleanup` (Phases E, G).
- Public tracking web page (Phase C).
- SOS UI components (Phase D).
- SOS active orchestration in `sos.service.ts` (Phase F).
- Settings toggles and permission onboarding screens (Phase H).
- Driver consent modal (Phase I).

## 12. Open Questions (will resolve during plan write-up)

- Should `liveTracking.service.ts` ship with a temporary mock of `sosAlertsByToken` / `rideSharesByToken` index nodes for local dev testing before Phase E ships those CFs? Likely yes — write a dev-only helper behind a guard.
- Notification channel migration: existing `ride-tracking` channel stays; the new `ride-tracking-sos` channel is created lazily on first SOS use. No migration needed.

## 13. Deploy

Phase B is client-only except for the RTDB rules update:

```bash
cd CarpoolBackend
firebase deploy --only database
```

The client-side service ships in the next EAS build.

---

**Next step:** `superpowers:writing-plans` consumes this spec and produces a per-step implementation plan with review checkpoints. TDD runs from that plan; no code lands before the plan is approved.
