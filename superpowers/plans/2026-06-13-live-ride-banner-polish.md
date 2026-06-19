# Live Ride Banner — Bug Fixes & UX Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 6 bugs and 4 UX issues in the global live-ride banner and related files without changing the visual identity.

**Architecture:** Four targeted file edits, no new files. (1) `useActiveRide` stale closure fixed with a `roleRef` that mirrors role state for callbacks. (2) Two dead `console.error` calls removed from `booked-ride-details`. (3) `LiveRideBanner` rewritten: real `Animated.loop` pulse, single-tap navigation, sibling `Pressable` for driver Done chip — no nested touchables, no expand state. (4) `_layout.tsx` cleaned up: paddingBottom override removed, `onEndRide` wired.

**Tech Stack:** React Native, Expo SDK 54, TypeScript strict, Expo Router v6, `Animated` API (native driver), `Pressable`

---

## File Map

| File | Change |
|------|--------|
| `Carpool/hooks/useActiveRide.ts` | Add `roleRef`, pair all `setRole` calls with ref updates, replace `role` reads in callbacks with `roleRef.current` |
| `Carpool/app/booked-ride-details.tsx` | Remove `console.error` on lines 139 and 155 |
| `Carpool/components/LiveRideBanner.tsx` | Full rewrite — pulse animation, single-tap nav, LIVE label, driver Done chip, `StyleProp<ViewStyle>` |
| `Carpool/app/tabs/_layout.tsx` | Remove `paddingBottom` from style prop, add `onEndRide` |

---

## Task 1: Fix stale closure in useActiveRide.ts

**Files:**
- Modify: `Carpool/hooks/useActiveRide.ts`

The `role` state is read inside three snapshot callbacks that are set up once in a `useEffect([], [])`. They always see the initial `null` because they captured it at mount. Adding a `useRef` that is mutated alongside every `setRole` call gives callbacks a current value to read without re-running the effect.

- [ ] **Step 1: Add `useRef` to the import and declare `roleRef`**

Line 12 — change:
```typescript
import { useEffect, useState } from 'react';
```
to:
```typescript
import { useEffect, useRef, useState } from 'react';
```

After line 33 (`const [role, setRole] = useState...`), insert:
```typescript
  const roleRef = useRef<'driver' | 'passenger' | null>(null);
```

- [ ] **Step 2: Fix the driver snapshot callback (lines 61–85)**

Replace the entire `if (rides.length > 0) { ... } else { ... }` block with:

```typescript
        if (rides.length > 0) {
          const ride = rides[0];
          driverRideFound = true;
          setActiveRide({
            rideId: ride.rideId || ride.id,
            fromAddress: ride.from?.address || '',
            toAddress: ride.to?.address || '',
            duration: ride.route?.duration,
            durationValue: ride.route?.durationValue,
          });
          roleRef.current = 'driver';
          setRole('driver');
        } else {
          driverRideFound = false;
          setActiveRide((prev) => {
            if (prev && roleRef.current === 'driver') return null;
            return prev;
          });
          if (roleRef.current === 'driver') {
            roleRef.current = null;
            setRole(null);
          }
        }
```

- [ ] **Step 3: Fix the passenger bookings callback (lines 108–113)**

Replace:
```typescript
        if (bookings.length === 0) {
          if (!driverRideFound && role === 'passenger') {
            setActiveRide(null);
            setRole(null);
          }
          return;
        }
```
with:
```typescript
        if (bookings.length === 0) {
          if (!driverRideFound && roleRef.current === 'passenger') {
            setActiveRide(null);
            roleRef.current = null;
            setRole(null);
          }
          return;
        }
```

- [ ] **Step 4: Fix the passenger ride snapshot callback (lines 131–153)**

Replace the inner `if/else if` block:
```typescript
              if (rideDoc && rideDoc.status === 'STARTED' && !found) {
                found = true;
                const matchingBooking = bookings.find((b: any) => b.rideId === rId);
                const driverInfo = rideDoc.driver;
                setActiveRide({
                  rideId: rideDoc.rideId || rideDoc.id || rId,
                  fromAddress: rideDoc.from?.address || '',
                  toAddress: rideDoc.to?.address || '',
                  duration: rideDoc.route?.duration,
                  durationValue: rideDoc.route?.durationValue,
                  driverName: driverInfo
                    ? `${driverInfo.firstName || ''} ${driverInfo.lastName || ''}`.trim()
                    : undefined,
                });
                setRole('passenger');
              } else if (rideDoc && rideDoc.status !== 'STARTED' && found) {
                found = false;
                if (!driverRideFound) {
                  setActiveRide(null);
                  setRole(null);
                }
              }
```
with:
```typescript
              if (rideDoc && rideDoc.status === 'STARTED' && !found) {
                found = true;
                const driverInfo = rideDoc.driver;
                setActiveRide({
                  rideId: rideDoc.rideId || rideDoc.id || rId,
                  fromAddress: rideDoc.from?.address || '',
                  toAddress: rideDoc.to?.address || '',
                  duration: rideDoc.route?.duration,
                  durationValue: rideDoc.route?.durationValue,
                  driverName: driverInfo
                    ? `${driverInfo.firstName || ''} ${driverInfo.lastName || ''}`.trim()
                    : undefined,
                });
                roleRef.current = 'passenger';
                setRole('passenger');
              } else if (rideDoc && rideDoc.status !== 'STARTED' && found) {
                found = false;
                if (!driverRideFound) {
                  setActiveRide(null);
                  roleRef.current = null;
                  setRole(null);
                }
              }
```

- [ ] **Step 5: TypeScript check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit 2>&1 | grep "useActiveRide" | head -10
```

Expected: no output (no errors in that file).

---

## Task 2: Remove console.error from booked-ride-details.tsx

**Files:**
- Modify: `Carpool/app/booked-ride-details.tsx`

`firestoreService` already logs listener errors internally. The two `console.error` calls violate code standards and add no value.

- [ ] **Step 1: Fix line 139 — ride listener error handler**

Replace:
```typescript
            (err) => console.error('Ride listener error:', err),
```
with:
```typescript
            () => {},
```

- [ ] **Step 2: Fix line 155 — booking listener error handler**

Replace:
```typescript
      (err) => console.error('Booking listener error:', err),
```
with:
```typescript
      () => {},
```

- [ ] **Step 3: Commit Tasks 1 + 2**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add hooks/useActiveRide.ts app/booked-ride-details.tsx
git commit -m "$(cat <<'EOF'
fix: stale closure in useActiveRide + remove console.error in booked-ride-details

- Add roleRef to mirror role state so snapshot callbacks always read the
  current value rather than the null captured at mount
- Remove console.error calls from Firestore listener error handlers in
  booked-ride-details (firestoreService already logs internally)

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Rewrite LiveRideBanner.tsx

**Files:**
- Modify: `Carpool/components/LiveRideBanner.tsx`

Complete rewrite. Key changes vs the old file:
- Real `Animated.loop` pulse (scale 1→2.4, opacity 0.7→0, 1500ms repeat)
- Outer container is now a plain `View` — no `TouchableOpacity` wrapping everything
- Single `Pressable` covers the route area and navigates to live-ride on tap
- Driver Done chip is a **sibling** `Pressable` (not nested) — no gesture conflict
- Passenger sees a forward chevron hint instead
- Label changed from `"ACTIVE"` → `"LIVE"`
- `style` prop typed as `StyleProp<ViewStyle>` — no `any`
- Removed: `expanded` state, chevron toggle, `expandedContent`, `handleBannerPress`, `onPress` prop, nested route-info touchable, `passengerInfo` touchable, old static pulse containers

- [ ] **Step 1: Replace the full file content**

Write the following to `Carpool/components/LiveRideBanner.tsx`:

```tsx
import { Ionicons } from '@expo/vector-icons';
import { useRouter } from 'expo-router';
import React, { useEffect, useRef } from 'react';
import {
  Animated,
  Pressable,
  StyleProp,
  StyleSheet,
  Text,
  View,
  ViewStyle,
} from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { theme } from '../constants/theme';

interface LiveRideBannerProps {
  ride: {
    rideId?: string;
    id?: string;
    fromAddress?: string;
    toAddress?: string;
    duration?: string;
    durationValue?: number;
    from?: { address?: string };
    to?: { address?: string };
  };
  role?: 'driver' | 'passenger';
  driverName?: string;
  position?: 'top' | 'bottom';
  onEndRide?: () => void;
  style?: StyleProp<ViewStyle>;
}

export default function LiveRideBanner({
  ride,
  role = 'driver',
  driverName,
  position = 'bottom',
  onEndRide,
  style,
}: LiveRideBannerProps) {
  const router = useRouter();
  const insets = useSafeAreaInsets();
  const pulseAnim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    const loop = Animated.loop(
      Animated.sequence([
        Animated.timing(pulseAnim, {
          toValue: 1,
          duration: 1500,
          useNativeDriver: true,
        }),
        Animated.timing(pulseAnim, {
          toValue: 0,
          duration: 0,
          useNativeDriver: true,
        }),
      ]),
    );
    loop.start();
    return () => loop.stop();
  }, []);

  const rideId = ride.rideId || ride.id || '';

  const getFromAddress = (): string =>
    ride.fromAddress || ride.from?.address || 'Pickup';

  const getToAddress = (): string =>
    ride.toAddress || ride.to?.address || 'Destination';

  const getShortRoute = (): string => {
    const from = getFromAddress().split(',')[0];
    const to = getToAddress().split(',')[0];
    return `${from} → ${to}`;
  };

  const getETA = (): string => {
    if (ride.duration) return ride.duration;
    if (ride.durationValue) {
      return `${Math.round(ride.durationValue / 60)} mins`;
    }
    return 'Calculating...';
  };

  const handleNavigate = () => {
    if (rideId) {
      router.push(`/live-ride?rideId=${rideId}&role=${role}`);
    }
  };

  const pulseScale = pulseAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [1, 2.4],
  });

  const pulseOpacity = pulseAnim.interpolate({
    inputRange: [0, 0.4, 1],
    outputRange: [0.7, 0.3, 0],
  });

  const isBottom = position === 'bottom';

  return (
    <View
      style={[
        styles.container,
        isBottom
          ? [
              styles.bottomPosition,
              { paddingBottom: Math.max(insets.bottom, theme.spacing.md) },
            ]
          : styles.topPosition,
        style,
      ]}
    >
      <View style={styles.compactRow}>
        {/* Route area — full-width pressable, navigates to live-ride */}
        <Pressable
          style={styles.routeArea}
          onPress={handleNavigate}
          android_ripple={{ color: 'rgba(255,255,255,0.1)' }}
        >
          <View style={styles.liveIndicatorRow}>
            <View style={styles.liveDotContainer}>
              <Animated.View
                style={[
                  styles.pulseRing,
                  { transform: [{ scale: pulseScale }], opacity: pulseOpacity },
                ]}
              />
              <View style={styles.liveDot} />
            </View>
            <Text style={styles.liveText}>LIVE</Text>
          </View>

          <Text style={styles.routeText} numberOfLines={1}>
            {getShortRoute()}
          </Text>

          <View style={styles.metaRow}>
            <Ionicons name="time-outline" size={12} color="#fff" />
            <Text style={styles.metaText}>ETA: {getETA()}</Text>
            {role === 'passenger' && driverName && (
              <>
                <Text style={styles.separator}>{'•'}</Text>
                <Ionicons name="person-outline" size={12} color="#fff" />
                <Text style={styles.metaText} numberOfLines={1}>
                  {driverName}
                </Text>
              </>
            )}
          </View>
        </Pressable>

        {/* Driver: sibling Done chip — not nested inside routeArea Pressable */}
        {role === 'driver' && onEndRide ? (
          <Pressable
            style={styles.completeChip}
            onPress={onEndRide}
            hitSlop={8}
            android_ripple={{ color: 'rgba(255,255,255,0.2)' }}
          >
            <Text style={styles.completeChipText}>Done</Text>
          </Pressable>
        ) : (
          <Ionicons
            name="chevron-forward"
            size={18}
            color="rgba(255,255,255,0.7)"
          />
        )}
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    backgroundColor: theme.colors.primary,
    paddingHorizontal: theme.spacing.lg,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    elevation: 8,
    overflow: 'hidden',
  },
  topPosition: {
    paddingTop: 48,
    paddingBottom: theme.spacing.md,
  },
  bottomPosition: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    paddingTop: theme.spacing.md,
    shadowOffset: { width: 0, height: -4 },
  },
  compactRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: theme.spacing.md,
  },
  routeArea: {
    flex: 1,
  },
  liveIndicatorRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: theme.spacing.xs,
    marginBottom: 2,
  },
  liveDotContainer: {
    width: 20,
    height: 20,
    justifyContent: 'center',
    alignItems: 'center',
  },
  pulseRing: {
    position: 'absolute',
    top: 6,
    left: 6,
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#80ef44',
  },
  liveDot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#80ef44',
  },
  liveText: {
    fontSize: 11,
    fontWeight: theme.fontWeight.bold as '700',
    color: '#fff',
    letterSpacing: 0.5,
  },
  routeText: {
    fontSize: theme.fontSize.base,
    fontWeight: theme.fontWeight.semibold as '600',
    color: '#fff',
    marginBottom: 2,
  },
  metaRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 4,
  },
  metaText: {
    fontSize: 11,
    color: 'rgba(255, 255, 255, 0.9)',
  },
  separator: {
    fontSize: 11,
    color: 'rgba(255, 255, 255, 0.6)',
    marginHorizontal: 4,
  },
  completeChip: {
    backgroundColor: 'rgba(255,255,255,0.2)',
    borderRadius: theme.borderRadius.full,
    paddingHorizontal: theme.spacing.md,
    paddingVertical: theme.spacing.xs,
    borderWidth: 1,
    borderColor: 'rgba(255,255,255,0.4)',
  },
  completeChipText: {
    fontSize: theme.fontSize.sm,
    fontWeight: theme.fontWeight.semibold as '600',
    color: '#fff',
  },
});
```

- [ ] **Step 2: TypeScript check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit 2>&1 | grep "LiveRideBanner" | head -10
```

Expected: no output.

---

## Task 4: Fix tabs/_layout.tsx

**Files:**
- Modify: `Carpool/app/tabs/_layout.tsx`

Two changes: (1) remove `paddingBottom: theme.spacing.md` from the style override — the banner's own `Math.max(insets.bottom, theme.spacing.md)` handles this correctly and the override was silently winning. (2) pass `onEndRide` so the driver Done chip is wired to navigate to the live-ride screen.

- [ ] **Step 1: Update the LiveRideBanner usage**

Find this block (~lines 101–115):
```tsx
            {activeRide && role && (
                <LiveRideBanner
                    ride={{
                        rideId: activeRide.rideId,
                        fromAddress: activeRide.fromAddress,
                        toAddress: activeRide.toAddress,
                        duration: activeRide.duration,
                        durationValue: activeRide.durationValue,
                    }}
                    role={role}
                    driverName={activeRide.driverName}
                    position="bottom"
                    style={{ bottom: TAB_BAR_HEIGHT, paddingBottom: theme.spacing.md }}
                />
            )}
```

Replace with:
```tsx
            {activeRide && role && (
                <LiveRideBanner
                    ride={{
                        rideId: activeRide.rideId,
                        fromAddress: activeRide.fromAddress,
                        toAddress: activeRide.toAddress,
                        duration: activeRide.duration,
                        durationValue: activeRide.durationValue,
                    }}
                    role={role}
                    driverName={activeRide.driverName}
                    position="bottom"
                    style={{ bottom: TAB_BAR_HEIGHT }}
                    onEndRide={
                        role === 'driver'
                            ? () => router.push(`/live-ride?rideId=${activeRide.rideId}&role=driver`)
                            : undefined
                    }
                />
            )}
```

- [ ] **Step 2: TypeScript check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit 2>&1 | head -20
```

Expected: no errors.

- [ ] **Step 3: Commit Tasks 3 + 4**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/LiveRideBanner.tsx app/tabs/_layout.tsx
git commit -m "$(cat <<'EOF'
fix: live ride banner — real pulse animation, single-tap nav, LIVE label, driver Done chip

- Replace static Animated.View rings with a real Animated.loop
  (scale 1→2.4, opacity 0.7→0, 1500ms) — pulse was broken since launch
- Remove expand/collapse: banner now navigates to live-ride on single tap
- Remove nested TouchableOpacity components (e.stopPropagation is a no-op in RN)
- Add sibling Pressable Done chip for drivers (no gesture conflict)
- Passenger view shows forward chevron affordance
- Change label ACTIVE → LIVE to match live-ride screen header
- Fix style prop type: any → StyleProp<ViewStyle>
- Remove paddingBottom override in _layout.tsx (banner handles its own insets)
- Wire onEndRide in _layout.tsx so Done chip navigates to live-ride screen

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```
