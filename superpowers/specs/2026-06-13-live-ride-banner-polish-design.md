# Live Ride Banner — Bug Fixes & UX Polish

**Date:** 2026-06-13
**Scope:** `LiveRideBanner.tsx`, `useActiveRide.ts`, `tabs/_layout.tsx`, `booked-ride-details.tsx`
**Out of scope:** Full live-ride screen redesign (see `2026-06-13-live-ride-uiux-redesign-design.md`), backend changes

---

## 1. Problems Being Solved

### Bugs

| ID | File | Description |
|----|------|-------------|
| B1 | `LiveRideBanner.tsx` | Pulse animation is broken — `Animated.View` rings have no `Animated.loop`; they are static scaled circles |
| B2 | `LiveRideBanner.tsx` | Nested `TouchableOpacity` inside a `TouchableOpacity` — gesture conflicts; `e.stopPropagation()` is a web API, does nothing in React Native |
| B3 | `LiveRideBanner.tsx` | `style?: any` — violates strict TypeScript; should be `StyleProp<ViewStyle>` |
| B4 | `tabs/_layout.tsx` | Passes `paddingBottom: theme.spacing.md` via style prop, silently overriding the banner's own `Math.max(insets.bottom, theme.spacing.md)` safe-area logic — can clip content on notched phones |
| B5 | `useActiveRide.ts` | Stale closure — `role` state value is captured in snapshot callbacks at effect setup time (always `null`); the clearance path when driver's ride ends may never fire |
| B6 | `booked-ride-details.tsx` | Two `console.error` calls (lines 139, 155) — violates code standards (`no console.log` in committed code) |

### UX Issues

| ID | Description |
|----|-------------|
| U1 | Copy mismatch: banner label says `"ACTIVE"`, live-ride screen header says `"LIVE"` — inconsistent brand moment |
| U2 | Two taps to navigate: passenger must expand the banner, then tap "Tap to view live tracking" inside it — pure friction for the most common action |
| U3 | Expanded state persists across tab switches — stale open banner on unrelated screens |
| U4 | `onEndRide` prop never passed from `_layout.tsx` — the "Complete Ride" button in the driver expanded view is dead code in practice |

---

## 2. Design Decisions

### 2.1 Remove expand/collapse — single-tap navigation

**Old behaviour:** tap banner → expand (shows full route + action button) → tap action to navigate or complete.

**New behaviour:** tap anywhere on the banner → navigate to `/live-ride?rideId=...&role=...` directly. No expansion state. No chevron icon.

For the driver, a small dedicated "Complete" chip (right side of banner, inside a `Pressable` with its own hit area) allows completing the ride without opening the full screen. This replaces the dead `onEndRide` expanded button.

This eliminates B2 (no nested touchables needed), U2 (one tap), U3 (no expanded state to persist).

### 2.2 Real pulse animation

Use a single animated ring that loops: scale from `1.0 → 2.0`, opacity from `0.6 → 0`, over 1500ms, repeating. Use `Animated.loop(Animated.parallel([...]))`. Start the loop in `useEffect` on mount, stop it on unmount.

Position the ring using absolute positioning centered behind the live dot, sized to the dot dimensions so the scale animation expands outward cleanly.

### 2.3 Fix stale closure in useActiveRide

Replace `role` state reads inside snapshot callbacks with a `useRef` that mirrors the state:

```typescript
const roleRef = useRef<'driver' | 'passenger' | null>(null);
// whenever setRole(x) is called, also set roleRef.current = x
// callbacks read roleRef.current instead of role
```

This ensures clearance logic fires correctly when the driver's ride ends.

### 2.4 Fix paddingBottom override

Remove `paddingBottom: theme.spacing.md` from the `style` prop passed in `_layout.tsx`. The banner already handles its own bottom padding via `Math.max(insets.bottom, theme.spacing.md)`.

### 2.5 Standardise copy to "LIVE"

Change `liveText` from `"ACTIVE"` to `"LIVE"` to match the live-ride screen header.

### 2.6 Remove console.error in booked-ride-details

The two `console.error` calls are in Firestore listener error callbacks. `firestoreService` already handles logging internally. Remove both calls (pass a no-op `() => {}` for the error handler, matching the pattern used elsewhere in the file's `loadData` function).

---

## 3. Component Changes

### `LiveRideBanner.tsx`

- Add pulse animation: `useRef(new Animated.Value(0))`, loop `scale 1→2` + `opacity 0.6→0` in `useEffect`
- Replace outer `TouchableOpacity` with `View` (non-pressable container)
- Add a `Pressable` wrapping the compact content (left ~85%) → `handleNavigateToLiveRide`
- Driver only: add `Pressable` "Complete" chip on the right side of compact row → calls `onEndRide`
- Remove expand/collapse state entirely (`expanded`, `setExpanded`, chevron icon, `expandedContent` JSX, `expandedContent` style)
- Remove `handleBannerPress`, the nested route-info `TouchableOpacity`, and `passengerInfo` `TouchableOpacity`
- Change `"ACTIVE"` → `"LIVE"`
- Change `style?: any` → `style?: StyleProp<ViewStyle>`
- Import `StyleProp`, `ViewStyle`, `Pressable` from react-native

### `useActiveRide.ts`

- Add `roleRef = useRef<'driver' | 'passenger' | null>(null)`
- Wrap `setRole(x)` calls with a helper `updateRole(x)` that sets both state and ref
- Replace `role` reads inside snapshot callbacks with `roleRef.current`

### `tabs/_layout.tsx`

- Remove `paddingBottom: theme.spacing.md` from the `style` prop on `LiveRideBanner`
- Add `onEndRide` prop: `() => router.push(`/live-ride?rideId=${activeRide.rideId}&role=driver`)` — navigates to live-ride screen where the actual completeRide CF call lives

### `booked-ride-details.tsx`

- Remove `console.error` on line 139 (ride listener error): replace with `() => {}`
- Remove `console.error` on line 155 (booking listener error): replace with `() => {}`

---

## 4. What Does NOT Change

- Visual identity (primary colour, pill shape, route text format)
- The `SosActiveBanner` component
- The live-ride.tsx screen itself
- Backend / Cloud Functions
- `booked-ride-details.tsx` live tracking footer layout (the 3-button stack is functional, not in scope)

---

## 5. Implementation Order

1. Fix `useActiveRide.ts` stale closure (B5) — no visual change, safe first
2. Fix `booked-ride-details.tsx` console.error calls (B6) — trivial
3. Refactor `LiveRideBanner.tsx` — pulse animation (B1) + remove expand/nested touchables (B2) + copy fix (U1) + TypeScript fix (B3) + driver Complete chip (U4)
4. Fix `tabs/_layout.tsx` — paddingBottom removal (B4) + wire `onEndRide` (U4)
