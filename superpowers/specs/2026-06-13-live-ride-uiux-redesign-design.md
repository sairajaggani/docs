# Live-Ride UI/UX Redesign — Design Spec

**Date:** 2026-06-13
**Scope:** `Carpool/app/live-ride.tsx` + satellite components (`SosButton`, `SosCountdownModal`, `SosActiveBanner`, `ShareLiveRideButton`, `LiveRideBanner`) + hooks (`useRideTracking`, `useRideShare`, `useSosState`, `usePassengerLocation`)
**Out of scope:** QR-code OTP, backend security fixes (BUG-01/SEC-01/SEC-02), other app screens

---

## 1. Design Principles

1. **The map is the product.** Every overlay element must justify its presence. Default is nothing on the map.
2. **One primary action visible at all times.** No competing CTAs at the same visual weight.
3. **SOS is not a feature — it's emergency infrastructure.** Never let it share visual rank with sharing affordances.

---

## 2. Driver Screen

### 2.1 Layout

**Header (40dp):** thin chip — back arrow, `● LIVE` chip, `⋯` menu, profile avatar. No status text in the header; that moves to the contextual pill.

**Map:** full-screen. No floating FABs. No ETA chip. No share pill. No re-center FAB. The only map overlay is:

- **Contextual pill** — a single rounded card with `position: absolute` fixed to the screen, centered horizontally, sitting just above the BottomSheet handle (~16dp gap). Its *content* updates based on ride state (`Pick up [Name] · N/M · X min` during pickup, `Driving to [Destination] · X min` during transit) — the position does not move. Replaces UX-D-01 clutter (ETA chip + re-center FAB + share pill removed from map).

**BottomSheet snap points:** 10% (collapsed) / 45% (default) / 85% (expanded). `mapPadding.bottom` dynamically mirrors the current snap height so the camera never hides the active stop behind the sheet (fixes UX-D-10).

**Primary card (inside sheet at 45%):** one card, changes per state — see §2.2.

**Secondary actions row (bottom of sheet at 45%):**
```
🗺 Open Nav   📤 Share   🆘 SOS
```
SOS is right-aligned, red, with at least 80dp horizontal clearance from Share. No other floating buttons exist.

**Rerouting banner:** when `isRerouting === true`, show a dismissible amber banner at the top of the sheet: `"Route updated — +2 min"`. Replaces the invisible 12px header text (fixes UX-D-06).

### 2.2 Primary Card — State Machine

**State A — Pending pickup (`pendingPickups.length > 0`):**
```
╔══════════════════════════════════════════╗
║  Riya wants to board  ·  1 of 3         ║
║  [ _ ] [ _ ] [ _ ] [ _ ]  (OTP input)  ║
║  ⚠️ Mark no-show                         ║
╚══════════════════════════════════════════╝
```
- Card title shows passenger name + N of M counter.
- 4-digit OTP input, auto-focused.
- On successful verify: card briefly shows `✓ Riya boarded`, then auto-advances to next passenger after 1.5 sec (no driver tap needed).
- `⚠️ Mark no-show` is a first-class text button in the card — no sheet scrolling required (fixes UX-D-05).
- No-show confirm: a brief modal `"Mark Riya as no-show? This cannot be undone."` with Confirm / Cancel.

**State B — Between passengers (transitioning):**
```
╔══════════════════════════════════════════╗
║  ✓ Riya boarded                         ║
║  Next: Arjun (1.4 km)                   ║
║  [ Pick up Arjun ]  ← shown after 1.5s  ║
╚══════════════════════════════════════════╝
```
CTA button appears after 1.5 sec to prevent mis-taps right after a verify.

**State C — All passengers picked up (`allPickedUp === true`):**
Sheet collapses to 10% automatically:
```
3 onboard · Tap to view  ↑
```
Map shows full-screen. Contextual pill switches to `Driving to HSR Layout · 28 min`.

Expanded sheet (85%) shows passenger list + one action:
```
[ Complete Ride — 3 passengers, 22 km logged ]
```
Complete Ride requires an explicit confirm step: `"End ride for 3 passengers?"` with passenger names listed. No accidental taps possible (fixes UX-D-03).

**State D — No pending pickups, no passengers (edge case):** show "Ride in progress — no passengers boarded" with Complete Ride available.

### 2.3 Derived state memoization (fixes UX-D-09)

Move these out of inline render into `useMemo`:
- `pendingPickups` — filter of `pickupStops` where not verified
- `allPickedUp` — `pendingPickups.length === 0 && pickupStops.length > 0`
- `needsPassengerTracking` — any passenger within 500m

---

## 3. Passenger Screen

### 3.1 Layout

**Header:** identical pattern to driver — back arrow, `● LIVE`, no status text.

**Map:** full-screen. Shows driver car (animated), passenger pickup pin, and passenger's own location pin. No floating elements.

**BottomSheet (default 45%):**

```
Driver info line (one line):
  4 min · Riya M. · KA-01-AB-1234 ⭐ 4.8

╔══════════════════════════════════════╗
║  Show this to your driver           ║
║                                     ║
║     8    4    2    1                ║  ← large digits, hero
║                                     ║
║  Your pickup code                   ║
╚══════════════════════════════════════╝

[ 📞 Call driver ]   [ 💬 Message ]

📤 Sharing location · 4 min walk from pickup
🆘 SOS                              (right-aligned)
```

- OTP digits are always the first and largest element in the sheet (fixes UX-P-01).
- Driver info is a single compressed line — no separate driver card (fixes UX-P-01 secondary clutter).
- Distance shows estimated walk time `~X min walk` not raw metres (fixes UX-P-05).
- One sharing indicator: the secondary row line. Remove `sharingInfoBar` and `sharingBadge` (fixes UX-P-03).
- Call button shown only when `booking.status === 'CONFIRMED'` (fixes UX-P-04).

### 3.2 On-board state

When `isPassengerPickedUp === true`, the OTP hero card is replaced permanently by:
```
╔══════════════════════════════════════╗
║  You're on board 🎉                 ║
║  Drop-off in 28 min · HSR Layout    ║
╚══════════════════════════════════════╝
```
No disappearing 3-second banner. The card stays for the duration of the ride (fixes UX-P-06).

Secondary row becomes:
```
📤 Share live ride                🆘 SOS
```

---

## 4. SOS Redesign

### 4.1 SOS Button

- Visual: red ring (48dp diameter) containing a smaller white dot. Not a pill. Not matching the Share button shape.
- Position: `bottom: 16dp, right: 16dp` — always pinned. 48dp clear area around it (no other control within that radius).
- Trigger: 3-second long-press with three-stage haptic feedback (unchanged).
- Hidden during countdown and while SOS is active. The SOS Active Banner is the sole control surface during an active alert.

### 4.2 SOS Countdown (replaces full-screen modal)

A `BottomSheet` covering 75% of screen height. Top 25% remains as the live map (user retains location context — fixes UX-S-01).

Sheet layout (top to bottom):
1. **"📞 Call Emergency Now"** — full-width white button in the red sheet header. Dials immediately, bypasses countdown.
2. **Countdown number** — large centred digit in the red area.
3. **"SOS IN X SECONDS"** label.
4. **Contact notice** — `"3 contacts will be notified · tracking link shared with viewers"`.
5. **CANCEL** — large, white-bordered red button. Easy to tap.

**Reduce Motion fix (fixes UX-S-03):** when `reduceMotionEnabled === true`:
- Do NOT auto-fire.
- Show the sheet with countdown already at 0.
- The "Call Emergency Now" button is pre-active (pulsing outline).
- A "Send SOS Now" button appears where the countdown was.
- User taps Send or Cancel. Safety window preserved.

### 4.3 SOS Active Banner (replaces current top banner)

Persistent bar that floats above the BottomSheet (z-index above sheet, does not scroll with it — fixes UX-S-04):

```
⚠ SOS ACTIVE   📞 Emergency   📞 Driver   🔗 Link   [I'm Safe]
```

- All 5 elements in one row (horizontal scroll if narrow).
- `📞 Emergency` and `📞 Driver` are tap-to-call.
- `🔗 Link` copies the tracking URL to clipboard + opens share sheet.
- `I'm Safe` requires a 1-second hold (with a progress indicator) to confirm — prevents accidental cancels (fixes UX-S-02 by adding emergency shortcuts, prevents mis-cancel).

---

## 5. Performance Fixes

| ID | Fix | Impact |
|----|-----|--------|
| EFF-01 | Remove second `watchPositionAsync` in `live-ride.tsx:393-406`. Register a `marker` sink on the existing `locationCoreService` firehose so the car marker updates from the same GPS stream as the RTDB writer. | ~30–40% battery reduction on Android |
| EFF-02 | Move `decodePolyline(reroutedPolyline)` call into `useMemo([reroutedPolyline])`. Current: runs in render path on every RTDB tick. | Removes O(n) decode per frame |
| EFF-03 | In `DriverMarker`, only start `Animated.timing` on heading change when `Math.abs(newHeading - prevHeading) > 3`. Suppresses animation on micro-jitter on straight roads. | Removes JS thread spin every 2 sec |
| EFF-04 | Add 1-second debounce on `setPassengerLocations` callback. Three passengers walking = three RTDB streams; without debounce = three map re-renders per second. | Reduces map re-renders ~3× |
| EFF-05 | Detect swipe origin to distinguish sheet drag vs map drag. Only clear `followDriver` when the pan gesture originated on the map, not on the sheet. | Fixes follow-mode being killed by sheet swipe (UX-D-10 related) |
| UX-D-08 | Replace `LastUpdatedText` 5-sec `setInterval` with a `useMemo` on `lastLocationUpdateTime` + a single `useEffect` that forces a re-render every 30 sec. | Removes unnecessary re-renders |
| UX-D-09 | Move `pendingPickups`, `allPickedUp`, `needsPassengerTracking` into `useMemo`. | Reduces GC pressure on high-FPS map screen |

---

## 6. Component Changes Summary

| Component / File | Change |
|---|---|
| `live-ride.tsx` | Major refactor: remove FABs, add contextual pill, drive sheet state from pickup state machine, add secondary row, memoize derived state, remove second watchPositionAsync |
| `SosButton.tsx` | New visual (ring not pill), 48dp clear area |
| `SosCountdownModal.tsx` | Replace full-screen View with BottomSheet at 75%, add emergency-call CTA, fix Reduce Motion |
| `SosActiveBanner.tsx` | Add 3 quick-action buttons, hold-to-confirm on "I'm Safe", float above sheet |
| `ShareLiveRideButton.tsx` | Remove as standalone FAB; integrate Share action into secondary actions row in sheet |
| `LiveRideBanner.tsx` | Remove or repurpose; rerouting state moves to amber sheet banner |
| New: `DriverOtpCard.tsx` | Encapsulates State A/B/C primary card logic for driver |
| New: `PassengerOtpHero.tsx` | Hero OTP display + on-board state for passenger |
| `useRideTracking.ts` | Add memoized derived values, debounced passenger locations |
| `usePassengerLocation.ts` | No change to public API; EFF-01 fix is inside this service |

---

## 7. Implementation Order

1. **Memoization + EFF fixes** — no visual change, safe to ship first. (`live-ride.tsx`, `useRideTracking.ts`)
2. **Driver OTP card state machine** — `DriverOtpCard.tsx` + wiring into sheet. Most impactful driver change.
3. **Driver layout cleanup** — remove FABs, add contextual pill, secondary row, `mapPadding` dynamic binding.
4. **Passenger OTP hero + on-board state** — `PassengerOtpHero.tsx` + live-ride passenger branch.
5. **SOS countdown redesign** — `SosCountdownModal.tsx` refactor.
6. **SOS active banner** — `SosActiveBanner.tsx` enhancements.
7. **SOS button visual** — `SosButton.tsx` ring style + spacing.

---

## 8. Out of Scope

- QR-code OTP (passenger JWT QR + `verifyPassengerOtpByQr` CF) — future enhancement
- Backend fixes: BUG-01 (6-digit OTP uniqueness), SEC-01 (no plaintext OTP in push), SEC-02 (per-OTP attempt cap) — separate session(Done)
- Other app screens (tabs, details, search, profile) — next pass
