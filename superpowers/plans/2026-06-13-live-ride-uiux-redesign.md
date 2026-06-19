# Live-Ride UI/UX Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign `live-ride.tsx` and its satellite components to reduce overlay clutter, put the OTP card front-and-center for both roles, make SOS emergency-first, and clean up the remaining performance inefficiencies.

**Architecture:** Single-CTA-card pattern — one contextual pill over the map, one primary card in the bottom sheet (changes per ride state), secondary actions row for Nav/Share/SOS. SosCountdownModal becomes a screen overlay that keeps the map visible. All floating FABs and competing overlays are removed.

**Tech Stack:** React Native 0.76 + Expo SDK 54, `@gorhom/bottom-sheet`, `react-native-gesture-handler`, TypeScript strict, `theme.colors.*` (no inline hex), `StyleSheet.create()` at bottom of each file.

**Spec:** `docs/superpowers/specs/2026-06-13-live-ride-uiux-redesign-design.md`

---

## File Map

| Action | Path | What changes |
|--------|------|-------------|
| Create | `Carpool/components/DriverOtpCard.tsx` | Per-passenger OTP card with state A (input) and B (boarded transition) |
| Modify | `Carpool/app/live-ride.tsx` | Remove floating FABs; add contextual pill + secondary row; restructure driver and passenger sheets |
| Modify | `Carpool/components/SosCountdownModal.tsx` | Full-screen Modal → absolute overlay with map peeking, emergency call CTA, Reduce Motion fix |
| Modify | `Carpool/components/SosActiveBanner.tsx` | Add Emergency/Driver/Link quick-action buttons; hold-to-confirm on I'm Safe |
| Modify | `Carpool/components/SosButton.tsx` | Ring-around-dot visual; 56→48dp; floating position stays |

---

## Task 1: Performance quick wins — UX-D-08 + UX-D-09

**Files:**
- Modify: `Carpool/app/live-ride.tsx:268` (`LastUpdatedText` interval)
- Modify: `Carpool/app/live-ride.tsx:642-648` (inline derived state → `useMemo`)

---

- [ ] **Step 1: Fix the 5-second `LastUpdatedText` re-render storm**

In `Carpool/app/live-ride.tsx`, find `LastUpdatedText` at line 265 and change the interval from 5 000ms to 30 000ms:

```tsx
function LastUpdatedText({ timestamp }: { timestamp: number }) {
  const [, forceUpdate] = useState(0);
  useEffect(() => {
    const interval = setInterval(() => forceUpdate((n) => n + 1), 30_000);
    return () => clearInterval(interval);
  }, []);
  const secsAgo = Math.round((Date.now() - timestamp) / 1000);
  if (secsAgo < 5) return null;
  const label = secsAgo < 60 ? `${secsAgo}s ago` : `${Math.floor(secsAgo / 60)}m ago`;
  return (
    <Text style={styles.lastUpdatedText}>updated {label}</Text>
  );
}
```

- [ ] **Step 2: Wrap `pendingPickups` and `needsPassengerTracking` in `useMemo`**

In `Carpool/app/live-ride.tsx`, replace lines 642–648 (the four inline derived computations after `orderedStops`) with:

```tsx
const pendingPickups = useMemo(
  () => orderedStops.filter((s) => s.status === 'PENDING'),
  [orderedStops],
);

const shouldNavigateToPickup = pendingPickups.length > 0;
const allPickedUp = orderedStops.length > 0 && pendingPickups.length === 0;

const needsPassengerTracking = useMemo(
  () => orderedStops.some((s) => s.status !== 'NO_SHOW'),
  [orderedStops],
);
```

- [ ] **Step 3: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 4: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add app/live-ride.tsx
git commit -m "perf: memoize pendingPickups/needsTracking, fix LastUpdatedText interval (UX-D-08/09)"
```

---

## Task 2: Create `DriverOtpCard` component

**Files:**
- Create: `Carpool/components/DriverOtpCard.tsx`

---

- [ ] **Step 1: Create the file**

Create `Carpool/components/DriverOtpCard.tsx`:

```tsx
import { Ionicons } from '@expo/vector-icons';
import React, { useCallback, useEffect, useRef, useState } from 'react';
import {
  ActivityIndicator,
  StyleSheet,
  Text,
  TextInput,
  TouchableOpacity,
  View,
} from 'react-native';

import { theme } from '../constants/theme';
import { PickupStop } from '../src/services/firebase/realtime.service';

export interface DriverOtpCardProps {
  /** The passenger currently being verified (pendingPickups[0]). */
  passenger: PickupStop;
  /** 1-based display position in the full orderedStops list. */
  stopNumber: number;
  /** Total number of stops (including already-boarded). */
  totalStops: number;
  /** Next passenger in queue for transition preview (pendingPickups[1]). */
  nextPassenger?: PickupStop;
  /** Called with the OTP string. Throws on failure (caller shows error). */
  onVerify: (otp: string) => Promise<void>;
  /** Called when driver taps "Mark no-show". Caller shows confirm dialog. */
  onNoShow: () => void;
}

export function DriverOtpCard({
  passenger,
  stopNumber,
  totalStops,
  nextPassenger,
  onVerify,
  onNoShow,
}: DriverOtpCardProps) {
  const [otpInput, setOtpInput] = useState('');
  const [verifying, setVerifying] = useState(false);
  const [transitioning, setTransitioning] = useState(false);
  const [verifiedFirstName, setVerifiedFirstName] = useState('');
  const prevPassengerIdRef = useRef(passenger.passengerId);

  // Reset local state whenever RTDB advances to a new passenger
  useEffect(() => {
    if (passenger.passengerId !== prevPassengerIdRef.current) {
      prevPassengerIdRef.current = passenger.passengerId;
      setOtpInput('');
      setVerifying(false);
      setTransitioning(false);
      setVerifiedFirstName('');
    }
  }, [passenger.passengerId]);

  const handleVerify = useCallback(async () => {
    const trimmed = otpInput.trim();
    if (trimmed.length < 4) return;
    setVerifying(true);
    try {
      await onVerify(trimmed);
      setVerifiedFirstName(passenger.passengerName.split(' ')[0]);
      setOtpInput('');
      setTransitioning(true);
    } catch {
      // onVerify caller (live-ride.tsx) handles error display
    } finally {
      setVerifying(false);
    }
  }, [otpInput, onVerify, passenger.passengerName]);

  const firstName = passenger.passengerName.split(' ')[0];

  // ── Transitioning: OTP verified, RTDB catching up ─────────────────────────
  if (transitioning) {
    return (
      <View style={styles.card}>
        <View style={styles.boardedRow}>
          <Ionicons name="checkmark-circle" size={20} color={theme.colors.success} />
          <Text style={styles.boardedText}>{verifiedFirstName} boarded</Text>
        </View>
        {nextPassenger && (
          <Text style={styles.queueLine}>
            Next: {nextPassenger.passengerName.split(' ')[0]}
          </Text>
        )}
      </View>
    );
  }

  // ── State A: OTP input ──────────────────────────────────────────────────────
  return (
    <View style={styles.card}>
      <Text style={styles.cardTitle}>
        {firstName} wants to board · {stopNumber} of {totalStops}
      </Text>
      <TextInput
        style={styles.otpInput}
        placeholder="Enter OTP"
        placeholderTextColor={theme.colors.text.tertiary}
        keyboardType="number-pad"
        maxLength={6}
        value={otpInput}
        onChangeText={setOtpInput}
        editable={!verifying}
        autoFocus
        returnKeyType="done"
        onSubmitEditing={handleVerify}
        accessibilityLabel={`Enter OTP for ${firstName}`}
      />
      <TouchableOpacity
        style={[styles.verifyBtn, (verifying || otpInput.trim().length < 4) && styles.verifyBtnDisabled]}
        onPress={handleVerify}
        disabled={verifying || otpInput.trim().length < 4}
        activeOpacity={0.8}
        accessibilityRole="button"
        accessibilityLabel="Verify OTP"
      >
        {verifying ? (
          <ActivityIndicator size="small" color={theme.colors.text.inverse} />
        ) : (
          <Text style={styles.verifyBtnText}>Verify</Text>
        )}
      </TouchableOpacity>
      <TouchableOpacity
        style={styles.noShowBtn}
        onPress={onNoShow}
        activeOpacity={0.8}
        accessibilityRole="button"
        accessibilityLabel={`Mark ${firstName} as no show`}
      >
        <Ionicons name="warning-outline" size={14} color={theme.colors.warning} />
        <Text style={styles.noShowText}>Mark no-show</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: theme.colors.surface,
    borderRadius: theme.borderRadius.lg,
    padding: theme.spacing.lg,
    marginBottom: theme.spacing.sm,
    borderLeftWidth: 4,
    borderLeftColor: theme.colors.primary,
    shadowColor: theme.colors.primaryDark,
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 2,
  },
  cardTitle: {
    fontSize: theme.fontSize.sm,
    fontWeight: theme.fontWeight.bold as '700',
    color: theme.colors.primary,
    marginBottom: theme.spacing.sm,
  },
  otpInput: {
    fontSize: theme.fontSize.xl,
    fontWeight: theme.fontWeight.bold as '700',
    color: theme.colors.text.primary,
    borderWidth: 1.5,
    borderColor: theme.colors.primary,
    borderRadius: theme.borderRadius.sm,
    paddingHorizontal: theme.spacing.md,
    paddingVertical: theme.spacing.sm,
    marginBottom: theme.spacing.sm,
    letterSpacing: 8,
    textAlign: 'center',
  },
  verifyBtn: {
    backgroundColor: theme.colors.primary,
    borderRadius: theme.borderRadius.md,
    paddingVertical: theme.spacing.md,
    alignItems: 'center',
    marginBottom: theme.spacing.sm,
  },
  verifyBtnDisabled: {
    opacity: 0.45,
  },
  verifyBtnText: {
    color: theme.colors.text.inverse,
    fontWeight: theme.fontWeight.bold as '700',
    fontSize: theme.fontSize.base,
  },
  noShowBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: theme.spacing.xs,
    paddingVertical: theme.spacing.xs,
    alignSelf: 'flex-start',
  },
  noShowText: {
    fontSize: theme.fontSize.sm,
    color: theme.colors.warning,
    fontWeight: theme.fontWeight.medium as '500',
  },
  boardedRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: theme.spacing.sm,
    marginBottom: theme.spacing.xs,
  },
  boardedText: {
    fontSize: theme.fontSize.base,
    fontWeight: theme.fontWeight.bold as '700',
    color: theme.colors.success,
  },
  queueLine: {
    fontSize: theme.fontSize.xs,
    color: theme.colors.text.secondary,
    marginTop: theme.spacing.xs,
  },
});
```

- [ ] **Step 2: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/DriverOtpCard.tsx
git commit -m "feat: add DriverOtpCard component — per-passenger OTP state machine"
```

---

## Task 3: Driver layout restructure in `live-ride.tsx`

**Files:**
- Modify: `Carpool/app/live-ride.tsx` (multiple sections)

All changes are in the driver view. Work through the sub-steps in order — each is a small, isolated edit.

---

- [ ] **Step 1: Update snap points + add import for `DriverOtpCard`**

At the top of `live-ride.tsx` (around line 54–63, after existing component imports), add:

```tsx
import { DriverOtpCard } from '../components/DriverOtpCard';
```

Find `driverSnapPoints` (line ~303):

```tsx
const driverSnapPoints = useMemo(() => ['30%', '55%', '85%'], []);
```

Change to:

```tsx
const driverSnapPoints = useMemo(() => ['10%', '55%', '85%'], []);
```

- [ ] **Step 2: Add auto-collapse when all passengers board**

After the existing `useEffect` that stops tracking on no-show (around line 651), add:

```tsx
// Collapse driver sheet to 10% when all passengers are on board
useEffect(() => {
  if (!isDriver || !allPickedUp) return;
  bottomSheetRef.current?.snapToIndex(0);
}, [isDriver, allPickedUp]);
```

- [ ] **Step 3: Add OTP verify handler that DriverOtpCard can call**

The existing `handleVerifyOtp` reads `otpValue` from state and calls `functionsService.verifyPassengerOtp`. Add a new handler that takes the OTP as a parameter:

```tsx
const handleVerifyOtpForPassenger = useCallback(async (otp: string) => {
  const result = await functionsService.verifyPassengerOtp(rideId!, otp);
  Toast.show({ type: 'success', text1: `${result.passengerName} verified!`, position: 'bottom' });
}, [rideId]);
```

Place this directly after `handleVerifyOtp` (around line 588).

- [ ] **Step 4: Remove the five floating overlay elements from the JSX**

In the JSX return (around lines 861–921), remove these five blocks entirely:

1. The floating ETA chip (`{etaSeconds !== null && (<View style={styles.floatingEtaChip}>...</View>)}`) — lines 861–879
2. The re-center FAB (`{!followDriver && (<TouchableOpacity style={styles.recenterFab}>...</TouchableOpacity>)}`) — lines 881–892
3. `ShareLiveRideButton` with `style={styles.shareLiveRideBtn}` — lines 894–903
4. The standalone `SosButton` with `style={styles.sosFab}` — lines 905–912
5. The "My Location" FAB (`<TouchableOpacity style={styles.myLocationFab}>`) — lines 913–921

- [ ] **Step 5: Add contextual pill above the driver sheet**

After the MapView closing tag and before the Header `SafeAreaView` (around line 831), add the contextual pill. It must be inside `GestureHandlerRootView` and positioned above the default sheet height:

```tsx
{/* ── Contextual pill ───────────────────────────────────────────── */}
{isDriver && ride?.status === 'STARTED' && (
  <View style={styles.contextualPill} pointerEvents="none">
    <Text style={styles.contextualPillText} numberOfLines={1}>
      {allPickedUp
        ? `Driving to ${ride.to.address.split(',')[0]} · ${formatEta(etaSeconds)}`
        : pendingPickups.length > 0
          ? `Pick up ${pendingPickups[0].passengerName.split(' ')[0]} · ${
              orderedStops.findIndex((s) => s.passengerId === pendingPickups[0].passengerId) + 1
            }/${orderedStops.length} · ${formatEta(etaSeconds)}`
          : `Head to destination · ${formatEta(etaSeconds)}`}
    </Text>
  </View>
)}
```

Add the corresponding styles to `StyleSheet.create` at the bottom of the file:

```tsx
contextualPill: {
  position: 'absolute',
  bottom: '58%', // sits ~16dp above the default 55% snap point
  alignSelf: 'center',
  backgroundColor: 'rgba(0,0,0,0.75)',
  paddingHorizontal: theme.spacing.lg,
  paddingVertical: theme.spacing.sm,
  borderRadius: theme.borderRadius.full,
  maxWidth: '80%',
  zIndex: 50,
},
contextualPillText: {
  color: theme.colors.text.inverse,
  fontSize: theme.fontSize.sm,
  fontWeight: theme.fontWeight.semibold as '600',
},
```

- [ ] **Step 6: Add `compact` prop to `SosButton` — do this before Step 7 which uses it**

Inside the driver `BottomSheet` → `BottomSheetScrollView` (around line 1092), replace the entire content block with the new layout. The new content order is:

1. Rerouting banner (amber, dismissible, shown only when `isRerouting`)
2. Primary card (one of: `DriverOtpCard`, "All aboard" state card, "No passengers" card)
3. Muted queue line (next passengers, shown during active pickup)
4. Passenger list (only in expanded sheet — index 2 — for the all-aboard state)
5. Secondary actions row (Nav, Share, SOS)
6. Complete Ride (only visible in expanded sheet when `allPickedUp`)

Replace from `<BottomSheetScrollView contentContainerStyle={styles.driverSheetContent}>` to its closing `</BottomSheetScrollView>` with:

```tsx
<BottomSheetScrollView contentContainerStyle={styles.driverSheetContent}>

  {/* ── Rerouting amber banner ──────────────────────────────────── */}
  {isRerouting && (
    <View style={styles.reroutingBanner}>
      <Ionicons name="warning" size={16} color={theme.colors.warning} />
      <Text style={styles.reroutingBannerText}>Route updated</Text>
    </View>
  )}

  {/* ── Primary card ───────────────────────────────────────────── */}
  {!needsPassengerTracking ? (
    <View style={styles.primaryCard}>
      <Text style={styles.primaryCardTitle}>
        {orderedStops.length === 0 ? 'No passengers booked' : 'All passengers no-show'}
      </Text>
      <Text style={styles.primaryCardSub}>Navigate directly to destination</Text>
    </View>
  ) : allPickedUp ? (
    <View style={[styles.primaryCard, styles.primaryCardGreen]}>
      <Text style={styles.primaryCardTitle}>All aboard — drive safely</Text>
      <Text style={styles.primaryCardSub}>
        {ride.to.address.split(',')[0]} · {formatEta(etaSeconds)}
      </Text>
    </View>
  ) : pendingPickups.length > 0 ? (
    <DriverOtpCard
      passenger={pendingPickups[0]}
      stopNumber={
        orderedStops.findIndex((s) => s.passengerId === pendingPickups[0].passengerId) + 1
      }
      totalStops={orderedStops.length}
      nextPassenger={pendingPickups[1]}
      onVerify={handleVerifyOtpForPassenger}
      onNoShow={() => handleMarkNoShow(pendingPickups[0].passengerId, pendingPickups[0].passengerName)}
    />
  ) : null}

  {/* ── Muted queue line ────────────────────────────────────────── */}
  {pendingPickups.length > 1 && (
    <Text style={styles.queuePreview}>
      {'Next: '}
      {pendingPickups
        .slice(1, 3)
        .map((s) => s.passengerName.split(' ')[0])
        .join(' → ')}
    </Text>
  )}

  {/* ── Secondary actions row ───────────────────────────────────── */}
  {ride?.status === 'STARTED' && (
    <View style={styles.secondaryActionsRow}>
      <TouchableOpacity
        style={styles.secAction}
        onPress={allPickedUp ? handleNavigateToDestination : handleNavigateToPickup}
        activeOpacity={0.8}
      >
        <Ionicons name="navigate-outline" size={20} color={theme.colors.text.secondary} />
        <Text style={styles.secActionText}>Nav</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={styles.secAction}
        onPress={rideShare ? revokeShare : createShare}
        disabled={shareLoading}
        activeOpacity={0.8}
      >
        <Ionicons
          name={rideShare ? 'stop-circle-outline' : 'share-social-outline'}
          size={20}
          color={theme.colors.text.secondary}
        />
        <Text style={styles.secActionText}>{rideShare ? 'Sharing' : 'Share'}</Text>
      </TouchableOpacity>

      {sosEligible && !sosActive && (
        <SosButton
          onLongPressComplete={() => setSosCountdownVisible(true)}
          disabled={sosFiring}
          compact
        />
      )}
    </View>
  )}

  {/* ── Expanded-sheet content: passenger list + Complete Ride ────── */}
  {allPickedUp && (
    <>
      <View style={styles.card}>
        <Text style={styles.cardTitle}>Passengers on board</Text>
        {orderedStops
          .filter((s) => s.status === 'PICKED_UP')
          .map((stop) => (
            <View key={stop.passengerId} style={styles.stopRow}>
              <Ionicons name="checkmark-circle" size={16} color={theme.colors.success} />
              <Text style={styles.stopName}>{stop.passengerName}</Text>
            </View>
          ))}
      </View>

      <TouchableOpacity
        style={[styles.completeBtn, actionLoading && styles.completeBtnDisabled]}
        onPress={handleCompleteRide}
        disabled={actionLoading}
        activeOpacity={0.8}
        accessibilityRole="button"
        accessibilityLabel="Complete ride"
      >
        {actionLoading ? (
          <ActivityIndicator color={theme.colors.text.inverse} />
        ) : (
          <>
            <Ionicons name="checkmark-circle" size={22} color={theme.colors.text.inverse} />
            <Text style={styles.completeBtnText}>
              Complete Ride — {orderedStops.filter((s) => s.status === 'PICKED_UP').length} passengers
            </Text>
          </>
        )}
      </TouchableOpacity>
    </>
  )}

  {/* Show Complete Ride in no-passengers mode too */}
  {!needsPassengerTracking && (
    <TouchableOpacity
      style={[styles.completeBtn, actionLoading && styles.completeBtnDisabled]}
      onPress={handleCompleteRide}
      disabled={actionLoading}
      activeOpacity={0.8}
    >
      {actionLoading ? (
        <ActivityIndicator color={theme.colors.text.inverse} />
      ) : (
        <>
          <Ionicons name="checkmark-circle" size={22} color={theme.colors.text.inverse} />
          <Text style={styles.completeBtnText}>Complete Ride</Text>
        </>
      )}
    </TouchableOpacity>
  )}

  <View style={styles.bottomSpacing} />
</BottomSheetScrollView>
```

Add new styles at the bottom of `StyleSheet.create`:

```tsx
primaryCard: {
  backgroundColor: theme.colors.surface,
  borderRadius: theme.borderRadius.lg,
  padding: theme.spacing.lg,
  marginBottom: theme.spacing.sm,
  borderLeftWidth: 4,
  borderLeftColor: theme.colors.primary,
  ...theme.shadows.sm,
},
primaryCardGreen: {
  borderLeftColor: theme.colors.success,
},
primaryCardTitle: {
  fontSize: theme.fontSize.base,
  fontWeight: theme.fontWeight.bold as '700',
  color: theme.colors.text.primary,
  marginBottom: theme.spacing.xs,
},
primaryCardSub: {
  fontSize: theme.fontSize.sm,
  color: theme.colors.text.secondary,
},
queuePreview: {
  fontSize: theme.fontSize.xs,
  color: theme.colors.text.tertiary,
  marginBottom: theme.spacing.sm,
  paddingHorizontal: theme.spacing.xs,
},
secondaryActionsRow: {
  flexDirection: 'row',
  alignItems: 'center',
  justifyContent: 'space-around',
  borderTopWidth: 1,
  borderTopColor: theme.colors.border,
  paddingTop: theme.spacing.sm,
  marginBottom: theme.spacing.sm,
},
secAction: {
  alignItems: 'center',
  gap: theme.spacing.xs,
  paddingHorizontal: theme.spacing.lg,
  paddingVertical: theme.spacing.xs,
},
secActionText: {
  fontSize: theme.fontSize.xs,
  color: theme.colors.text.secondary,
},
reroutingBanner: {
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
reroutingBannerText: {
  fontSize: theme.fontSize.sm,
  color: theme.colors.warning,
  fontWeight: theme.fontWeight.semibold as '600',
},
```

- [ ] **Step 7: Restructure the driver sheet content**

**⚠️ Prerequisite:** Step 6 must be complete (compact prop added to `SosButton.tsx`) before this step, since the sheet uses `<SosButton compact />`.

In `Carpool/app/live-ride.tsx`:

First, remove the now-unused state declarations (the old OTP section is replaced by `DriverOtpCard`). Find and delete these two lines near the top of `LiveRide()`:

```tsx
const [otpValue, setOtpValue] = useState('');
const [verifyingOtp, setVerifyingOtp] = useState(false);
```

Then, inside the driver `BottomSheet` → `BottomSheetScrollView` (around line 1092), replace the entire content block with:

```tsx
export interface SosButtonProps {
  onLongPressComplete: () => void;
  disabled?: boolean;
  style?: ViewStyle;
  compact?: boolean;  // renders as a small icon + label for secondary rows
}
```

In the return, add before `<GestureDetector>`:

```tsx
if (compact) {
  return (
    <GestureDetector gesture={gesture}>
      <Animated.View
        style={[styles.compactContainer, { opacity: disabled ? 0.4 : 1 }]}
        accessibilityRole="button"
        accessibilityLabel="Emergency SOS. Press and hold for three seconds to trigger."
        accessibilityState={{ disabled }}
      >
        <Ionicons name="alert-circle" size={20} color={theme.colors.error} />
        <Text style={styles.compactLabel}>SOS</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

Add to `StyleSheet.create` in `SosButton.tsx`:

```tsx
compactContainer: {
  alignItems: 'center',
  gap: 2,
  paddingHorizontal: theme.spacing.lg,
  paddingVertical: theme.spacing.xs,
},
compactLabel: {
  fontSize: theme.fontSize.xs,
  color: theme.colors.error,
  fontWeight: theme.fontWeight.bold as '700',
},
```

- [ ] **Step 8: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 9: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add app/live-ride.tsx components/SosButton.tsx
git commit -m "feat: driver layout — contextual pill, secondary row, DriverOtpCard, gated complete ride"
```

---

## Task 4: Passenger sheet redesign

**Files:**
- Modify: `Carpool/app/live-ride.tsx` (passenger `BottomSheet` section only)

---

- [ ] **Step 1: Add walk-time helper function**

Near `formatEta` (around line 86), add:

```tsx
function formatWalkTime(distanceM: number): string {
  // 83 m/min ≈ 5 km/h walking
  const mins = Math.max(1, Math.round(distanceM / 83));
  return `~${mins} min walk`;
}
```

- [ ] **Step 2: Rewrite the passenger BottomSheet content**

Find the passenger `<BottomSheet>` block (starting around line 924) and replace everything inside `<BottomSheetScrollView>` up to the closing tag with:

```tsx
<BottomSheetScrollView contentContainerStyle={styles.passengerBottomSheetContent}>

  {/* ── Driver info — single compressed line ────────────────── */}
  <Text style={styles.driverInfoLine} numberOfLines={1}>
    {formatEta(etaSeconds)}
    {' · '}
    {ride.driver?.firstName} {ride.driver?.lastName}
    {ride.vehicleDetails ? ` · ${ride.vehicleDetails.plate}` : ''}
    {(ride.driver?.rating ?? 0) > 0 ? ` ⭐ ${ride.driver!.rating.toFixed(1)}` : ''}
  </Text>

  {/* ── OTP hero or on-board hero ───────────────────────────── */}
  {!isPassengerPickedUp && booking?.pickupOtp ? (
    <View style={styles.otpHeroCard}>
      <Text style={styles.otpHeroLabel}>Show this to your driver</Text>
      <Text style={styles.otpHeroDigits}>{booking.pickupOtp}</Text>
      <Text style={styles.otpHeroHint}>Your pickup code</Text>
    </View>
  ) : isPassengerPickedUp ? (
    <View style={styles.onBoardHeroCard}>
      <Ionicons name="checkmark-circle" size={32} color={theme.colors.success} />
      <View style={{ flex: 1 }}>
        <Text style={styles.onBoardHeroTitle}>You're on board!</Text>
        <Text style={styles.onBoardHeroSub}>
          Drop-off in {formatEta(etaSeconds)} · {ride.to.address.split(',')[0]}
        </Text>
      </View>
    </View>
  ) : null}

  {/* ── Call + Message inline row ───────────────────────────── */}
  {!isPassengerPickedUp && (
    <View style={styles.actionRow}>
      <TouchableOpacity
        style={styles.actionBtn}
        onPress={handleCallDriver}
        activeOpacity={0.8}
        accessibilityRole="button"
        accessibilityLabel="Call driver"
      >
        <Ionicons name="call-outline" size={18} color={theme.colors.text.secondary} />
        <Text style={styles.actionBtnText}>Call</Text>
      </TouchableOpacity>
      <TouchableOpacity
        style={styles.actionBtn}
        onPress={() => {
          const driverUid = ride.driverId;
          const chatId = `${rideId}_${driverUid}`;
          router.push(`/chat?chatId=${chatId}&rideId=${rideId}`);
        }}
        activeOpacity={0.8}
        accessibilityRole="button"
        accessibilityLabel="Message driver"
      >
        <Ionicons name="chatbubble-outline" size={18} color={theme.colors.text.secondary} />
        <Text style={styles.actionBtnText}>Message</Text>
      </TouchableOpacity>
    </View>
  )}

  {/* ── Distance / sharing secondary row ────────────────────── */}
  <View style={styles.passengerSecRow}>
    <Text style={styles.passengerSecLeft}>
      {isSharing
        ? `📤 Sharing location${distanceToPickupM !== null && !isPassengerPickedUp ? ` · ${formatWalkTime(distanceToPickupM)}` : ''}`
        : distanceToPickupM !== null && !isPassengerPickedUp
          ? `${formatWalkTime(distanceToPickupM)} from pickup`
          : null}
    </Text>

    {sosEligible && !sosActive && (
      <SosButton
        onLongPressComplete={() => setSosCountdownVisible(true)}
        disabled={sosFiring}
        compact
      />
    )}
  </View>

  {/* ── Pickup / Drop-off address card ──────────────────────── */}
  {!isPassengerPickedUp && ride.from && (
    <View style={styles.card}>
      <Text style={styles.cardTitle}>Pickup Point</Text>
      <Text style={styles.cardAddress}>{ride.from.address}</Text>
      {ride.from.plusCode && (
        <View style={styles.plusCodeRow}>
          <Ionicons name="grid-outline" size={14} color={theme.colors.text.secondary} />
          <Text style={styles.plusCodeText}>{ride.from.plusCodeCompound || ride.from.plusCode}</Text>
        </View>
      )}
      <TouchableOpacity style={styles.navigateBtn} onPress={handleNavigateToPickup}>
        <Ionicons name="navigate" size={16} color={theme.colors.text.inverse} />
        <Text style={styles.navigateBtnText}>Navigate to Pickup</Text>
      </TouchableOpacity>
    </View>
  )}

  {isPassengerPickedUp && ride.to && (
    <View style={styles.card}>
      <View style={styles.dropoffTitleRow}>
        <Ionicons name="flag" size={16} color={theme.colors.error} />
        <Text style={styles.cardTitle}>Drop-off Point</Text>
      </View>
      <Text style={styles.cardAddress}>{ride.to.address}</Text>
      {ride.to.plusCode && (
        <View style={styles.plusCodeRow}>
          <Ionicons name="grid-outline" size={14} color={theme.colors.text.secondary} />
          <Text style={styles.plusCodeText}>{ride.to.plusCodeCompound || ride.to.plusCode}</Text>
        </View>
      )}
      <TouchableOpacity
        style={styles.dropoffNavigateBtn}
        onPress={() => openGoogleMapsNavigation(ride.to.latitude, ride.to.longitude, ride.to.plusCodeCompound)}
      >
        <Ionicons name="navigate" size={16} color={theme.colors.text.inverse} />
        <Text style={styles.navigateBtnText}>Navigate to Drop-off</Text>
      </TouchableOpacity>
    </View>
  )}

  <View style={styles.bottomSpacing} />
</BottomSheetScrollView>
```

- [ ] **Step 3: Add passenger sheet styles to `StyleSheet.create`**

```tsx
driverInfoLine: {
  fontSize: theme.fontSize.sm,
  color: theme.colors.text.secondary,
  marginBottom: theme.spacing.md,
  paddingHorizontal: theme.spacing.xs,
},
otpHeroCard: {
  backgroundColor: theme.colors.surface,
  borderRadius: theme.borderRadius.lg,
  padding: theme.spacing.xl,
  alignItems: 'center',
  borderWidth: 2,
  borderColor: theme.colors.warning,
  marginBottom: theme.spacing.md,
  ...theme.shadows.sm,
},
otpHeroLabel: {
  fontSize: theme.fontSize.xs,
  fontWeight: theme.fontWeight.bold as '700',
  color: theme.colors.warning,
  textTransform: 'uppercase',
  letterSpacing: 1,
  marginBottom: theme.spacing.sm,
},
otpHeroDigits: {
  fontSize: 40,
  fontWeight: theme.fontWeight.extrabold as '800',
  color: theme.colors.text.primary,
  letterSpacing: 12,
  marginBottom: theme.spacing.xs,
},
otpHeroHint: {
  fontSize: theme.fontSize.xs,
  color: theme.colors.text.tertiary,
},
onBoardHeroCard: {
  flexDirection: 'row',
  alignItems: 'center',
  gap: theme.spacing.md,
  backgroundColor: theme.colors.successLight,
  borderRadius: theme.borderRadius.lg,
  padding: theme.spacing.lg,
  marginBottom: theme.spacing.md,
  borderLeftWidth: 4,
  borderLeftColor: theme.colors.success,
},
onBoardHeroTitle: {
  fontSize: theme.fontSize.base,
  fontWeight: theme.fontWeight.bold as '700',
  color: theme.colors.success,
},
onBoardHeroSub: {
  fontSize: theme.fontSize.sm,
  color: theme.colors.text.secondary,
  marginTop: 2,
},
actionRow: {
  flexDirection: 'row',
  gap: theme.spacing.sm,
  marginBottom: theme.spacing.md,
},
actionBtn: {
  flex: 1,
  flexDirection: 'row',
  alignItems: 'center',
  justifyContent: 'center',
  gap: theme.spacing.xs,
  backgroundColor: theme.colors.background,
  borderRadius: theme.borderRadius.md,
  paddingVertical: theme.spacing.md,
  borderWidth: 1,
  borderColor: theme.colors.border,
},
actionBtnText: {
  fontSize: theme.fontSize.sm,
  color: theme.colors.text.secondary,
  fontWeight: theme.fontWeight.medium as '500',
},
passengerSecRow: {
  flexDirection: 'row',
  alignItems: 'center',
  justifyContent: 'space-between',
  marginBottom: theme.spacing.md,
  paddingHorizontal: theme.spacing.xs,
},
passengerSecLeft: {
  fontSize: theme.fontSize.xs,
  color: theme.colors.text.secondary,
  flex: 1,
},
```

- [ ] **Step 4: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors. Fix any type errors before continuing.

- [ ] **Step 5: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add app/live-ride.tsx
git commit -m "feat: passenger sheet — OTP hero, compressed driver info, walk time, persistent on-board state"
```

---

## Task 5: SOS countdown — full-screen Modal → overlay sheet

**Files:**
- Modify: `Carpool/components/SosCountdownModal.tsx`
- Modify: `Carpool/app/live-ride.tsx` (add `emergencyNumber` prop to call site)

---

- [ ] **Step 1: Rewrite `SosCountdownModal.tsx`**

Replace the entire file contents:

```tsx
import * as Haptics from 'expo-haptics';
import React, { useEffect, useRef, useState } from 'react';
import {
  AccessibilityInfo,
  Linking,
  Platform,
  StyleSheet,
  Text,
  TouchableOpacity,
  View,
} from 'react-native';

import { theme } from '../constants/theme';

export interface SosCountdownModalProps {
  visible: boolean;
  onConfirm: () => void;
  onCancel: () => void;
  /** Emergency dial number for the "Call Emergency Now" shortcut. */
  emergencyNumber: string;
}

const COUNTDOWN_SECONDS = 5;

export function SosCountdownModal({
  visible,
  onConfirm,
  onCancel,
  emergencyNumber,
}: SosCountdownModalProps) {
  const [seconds, setSeconds] = useState(COUNTDOWN_SECONDS);
  const [reduceMotion, setReduceMotion] = useState(false);
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const firedRef = useRef(false);

  useEffect(() => {
    AccessibilityInfo.isReduceMotionEnabled().then(setReduceMotion).catch(() => {});
  }, []);

  useEffect(() => {
    if (!visible) {
      setSeconds(COUNTDOWN_SECONDS);
      firedRef.current = false;
      if (intervalRef.current) clearInterval(intervalRef.current);
      return;
    }

    // Reduce Motion: do NOT auto-fire — show a manual "Send SOS Now" button
    if (reduceMotion) {
      setSeconds(0);
      return;
    }

    firedRef.current = false;
    setSeconds(COUNTDOWN_SECONDS);

    intervalRef.current = setInterval(() => {
      setSeconds((prev) => {
        const next = prev - 1;
        Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium).catch(() => {});
        if (next <= 0) {
          clearInterval(intervalRef.current!);
          if (!firedRef.current) {
            firedRef.current = true;
            setTimeout(onConfirm, 0);
          }
        }
        return next;
      });
    }, 1000);

    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, [visible, reduceMotion, onConfirm]);

  const handleCancel = () => {
    if (intervalRef.current) clearInterval(intervalRef.current);
    firedRef.current = true;
    onCancel();
  };

  const handleSendNow = () => {
    if (intervalRef.current) clearInterval(intervalRef.current);
    if (!firedRef.current) {
      firedRef.current = true;
      onConfirm();
    }
  };

  const handleCallEmergency = () => {
    const url = Platform.OS === 'ios'
      ? `telprompt:${emergencyNumber}`
      : `tel:${emergencyNumber}`;
    Linking.openURL(url).catch(() => {});
  };

  if (!visible) return null;

  return (
    <View style={styles.overlay}>
      {/* Top 25%: transparent — shows the live map underneath */}
      <View style={styles.mapPeek} pointerEvents="none" />

      {/* Bottom 75%: red sheet */}
      <View style={styles.sheet}>
        <View style={styles.sheetHandle} />

        {/* Emergency call — first and largest action */}
        <TouchableOpacity
          style={styles.emergencyCallBtn}
          onPress={handleCallEmergency}
          activeOpacity={0.8}
          accessibilityRole="button"
          accessibilityLabel={`Call emergency services — ${emergencyNumber}`}
        >
          <Text style={styles.emergencyCallBtnText}>📞 Call Emergency Now</Text>
        </TouchableOpacity>

        {/* Countdown or Reduce Motion prompt */}
        {reduceMotion ? (
          <View style={styles.reduceMotionBlock}>
            <Text style={styles.countdownLabel}>SOS READY</Text>
            <TouchableOpacity
              style={styles.sendNowBtnPrimary}
              onPress={handleSendNow}
              activeOpacity={0.8}
              accessibilityRole="button"
              accessibilityLabel="Send SOS now"
            >
              <Text style={styles.sendNowBtnPrimaryText}>Send SOS Now</Text>
            </TouchableOpacity>
          </View>
        ) : (
          <>
            <Text style={styles.countdown}>{Math.max(seconds, 0)}</Text>
            <Text style={styles.countdownLabel}>
              SOS IN {Math.max(seconds, 0)} SECOND{seconds !== 1 ? 'S' : ''}
            </Text>
          </>
        )}

        <Text style={styles.contactsNotice}>
          Your trusted contacts will be notified
        </Text>

        <TouchableOpacity
          style={styles.cancelBtn}
          onPress={handleCancel}
          activeOpacity={0.8}
          accessibilityRole="button"
          accessibilityLabel="Cancel SOS"
        >
          <Text style={styles.cancelBtnText}>CANCEL</Text>
        </TouchableOpacity>

        {!reduceMotion && (
          <TouchableOpacity
            style={styles.sendNowBtn}
            onPress={handleSendNow}
            activeOpacity={0.8}
            accessibilityRole="button"
            accessibilityLabel="Send SOS immediately"
          >
            <Text style={styles.sendNowBtnText}>Send SOS Now</Text>
          </TouchableOpacity>
        )}
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  overlay: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    bottom: 0,
    zIndex: 300,
    elevation: 30,
    flexDirection: 'column',
  },
  mapPeek: {
    flex: 0,
    height: '25%',
    backgroundColor: 'transparent',
  },
  sheet: {
    flex: 1,
    backgroundColor: theme.colors.error,
    borderRadius: 16,
    paddingHorizontal: theme.spacing.xl,
    paddingBottom: theme.spacing.xxxl,
    paddingTop: theme.spacing.sm,
    alignItems: 'center',
  },
  sheetHandle: {
    width: 36,
    height: 4,
    backgroundColor: 'rgba(255,255,255,0.4)',
    borderRadius: 2,
    marginBottom: theme.spacing.lg,
    alignSelf: 'center',
  },
  emergencyCallBtn: {
    width: '100%',
    backgroundColor: '#fff',
    borderRadius: theme.borderRadius.lg,
    paddingVertical: theme.spacing.md,
    alignItems: 'center',
    marginBottom: theme.spacing.lg,
  },
  emergencyCallBtnText: {
    color: theme.colors.error,
    fontWeight: theme.fontWeight.extrabold as '800',
    fontSize: theme.fontSize.base,
  },
  countdown: {
    fontSize: 80,
    fontWeight: theme.fontWeight.extrabold as '800',
    color: '#fff',
    lineHeight: 88,
  },
  countdownLabel: {
    fontSize: theme.fontSize.sm,
    fontWeight: theme.fontWeight.bold as '700',
    color: 'rgba(255,255,255,0.9)',
    letterSpacing: 1.5,
    marginBottom: theme.spacing.xl,
  },
  reduceMotionBlock: {
    alignItems: 'center',
    marginBottom: theme.spacing.xl,
    gap: theme.spacing.lg,
  },
  contactsNotice: {
    fontSize: theme.fontSize.sm,
    color: 'rgba(255,255,255,0.75)',
    textAlign: 'center',
    marginBottom: theme.spacing.xl,
  },
  cancelBtn: {
    width: '100%',
    paddingVertical: theme.spacing.xl,
    borderRadius: theme.borderRadius.lg,
    backgroundColor: '#fff',
    alignItems: 'center',
    marginBottom: theme.spacing.md,
  },
  cancelBtnText: {
    fontSize: theme.fontSize.lg,
    fontWeight: theme.fontWeight.extrabold as '800',
    color: theme.colors.error,
    letterSpacing: 2,
  },
  sendNowBtn: {
    width: '100%',
    paddingVertical: theme.spacing.lg,
    borderRadius: theme.borderRadius.lg,
    backgroundColor: 'rgba(255,255,255,0.2)',
    alignItems: 'center',
    borderWidth: 2,
    borderColor: 'rgba(255,255,255,0.5)',
  },
  sendNowBtnText: {
    fontSize: theme.fontSize.base,
    fontWeight: theme.fontWeight.semibold as '600',
    color: '#fff',
  },
  sendNowBtnPrimary: {
    backgroundColor: '#fff',
    borderRadius: theme.borderRadius.lg,
    paddingHorizontal: theme.spacing.xxl,
    paddingVertical: theme.spacing.lg,
  },
  sendNowBtnPrimaryText: {
    color: theme.colors.error,
    fontWeight: theme.fontWeight.extrabold as '800',
    fontSize: theme.fontSize.base,
  },
});
```

- [ ] **Step 2: Update the `SosCountdownModal` call site in `live-ride.tsx`**

Find the `<SosCountdownModal>` usage (around line 1302):

```tsx
<SosCountdownModal
  visible={sosCountdownVisible}
  onConfirm={handleSosConfirmed}
  onCancel={() => setSosCountdownVisible(false)}
/>
```

Replace with:

```tsx
<SosCountdownModal
  visible={sosCountdownVisible}
  onConfirm={handleSosConfirmed}
  onCancel={() => setSosCountdownVisible(false)}
  emergencyNumber={emergencyNumber}
/>
```

- [ ] **Step 3: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 4: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/SosCountdownModal.tsx app/live-ride.tsx
git commit -m "feat: SOS countdown — map-visible overlay sheet, emergency call CTA, Reduce Motion fix"
```

---

## Task 6: SOS active banner — quick actions + hold-to-confirm

**Files:**
- Modify: `Carpool/components/SosActiveBanner.tsx`
- Modify: `Carpool/app/live-ride.tsx` (add new props to call site)

---

- [ ] **Step 1: Rewrite `SosActiveBanner.tsx`**

Replace the entire file:

```tsx
import React, { useRef, useState } from 'react';
import {
  ActivityIndicator,
  Animated,
  Linking,
  Platform,
  ScrollView,
  Share,
  StyleSheet,
  Text,
  TouchableOpacity,
  View,
} from 'react-native';
import Toast from 'react-native-toast-message';

import { theme } from '../constants/theme';
import type { SosAlert } from '../src/types/models/sosAlert.types';

export interface SosActiveBannerProps {
  alert: SosAlert;
  onResolve: (reason: 'SAFE' | 'FALSE_ALARM') => Promise<void>;
  /** Phone number for the driver (undefined in driver view). */
  driverPhone?: string;
}

const HOLD_DURATION_MS = 1000;

export function SosActiveBanner({ alert, onResolve, driverPhone }: SosActiveBannerProps) {
  const [resolving, setResolving] = useState(false);
  const [safeBtnHolding, setSafeBtnHolding] = useState(false);
  const holdProgress = useRef(new Animated.Value(0)).current;
  const holdTimer = useRef<ReturnType<typeof setTimeout> | null>(null);
  const holdAnim = useRef<Animated.CompositeAnimation | null>(null);

  const handleResolve = async (reason: 'SAFE' | 'FALSE_ALARM') => {
    setResolving(true);
    try {
      await onResolve(reason);
    } catch {
      Toast.show({ type: 'error', text1: 'Could not resolve SOS', text2: 'Please try again.' });
      setResolving(false);
    }
  };

  const startSafeHold = () => {
    setSafeBtnHolding(true);
    holdAnim.current = Animated.timing(holdProgress, {
      toValue: 1,
      duration: HOLD_DURATION_MS,
      useNativeDriver: false,
    });
    holdAnim.current.start(({ finished }) => {
      if (finished) handleResolve('SAFE');
    });
  };

  const cancelSafeHold = () => {
    setSafeBtnHolding(false);
    if (holdAnim.current) holdAnim.current.stop();
    holdProgress.setValue(0);
    if (holdTimer.current) clearTimeout(holdTimer.current);
  };

  const handleCallEmergency = () => {
    const url = Platform.OS === 'ios'
      ? `telprompt:${alert.emergencyNumber}`
      : `tel:${alert.emergencyNumber}`;
    Linking.openURL(url).catch(() => {});
  };

  const handleCallDriver = () => {
    if (!driverPhone) return;
    const url = Platform.OS === 'ios' ? `telprompt:${driverPhone}` : `tel:${driverPhone}`;
    Linking.openURL(url).catch(() => {});
  };

  const handleShareTracking = async () => {
    try {
      await Share.share({ url: alert.trackingUrl, message: alert.trackingUrl });
    } catch {
      // User dismissed share sheet — not an error
    }
  };

  const safeBtnWidth = holdProgress.interpolate({
    inputRange: [0, 1],
    outputRange: ['0%', '100%'],
  });

  return (
    <View style={styles.banner}>
      <View style={styles.topRow}>
        <Text style={styles.title}>🚨 SOS Active</Text>
        {resolving ? (
          <ActivityIndicator color="#fff" size="small" />
        ) : (
          <View style={styles.resolveRow}>
            {/* I'm Safe — hold 1s to confirm */}
            <TouchableOpacity
              style={styles.safeBtn}
              onPressIn={startSafeHold}
              onPressOut={cancelSafeHold}
              activeOpacity={0.9}
              accessibilityRole="button"
              accessibilityLabel="I'm safe — hold for 1 second to dismiss SOS"
            >
              <Animated.View style={[styles.safeBtnFill, { width: safeBtnWidth }]} />
              <Text style={styles.safeBtnText}>
                {safeBtnHolding ? 'Hold…' : "I'm Safe"}
              </Text>
            </TouchableOpacity>

            <TouchableOpacity
              style={styles.falseBtn}
              onPress={() => handleResolve('FALSE_ALARM')}
              activeOpacity={0.8}
              accessibilityRole="button"
              accessibilityLabel="False alarm"
            >
              <Text style={styles.falseBtnText}>False Alarm</Text>
            </TouchableOpacity>
          </View>
        )}
      </View>

      {/* Quick action row */}
      <ScrollView
        horizontal
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={styles.quickActionsRow}
      >
        <TouchableOpacity
          style={styles.quickAction}
          onPress={handleCallEmergency}
          activeOpacity={0.8}
          accessibilityRole="button"
          accessibilityLabel={`Call emergency — ${alert.emergencyNumber}`}
        >
          <Text style={styles.quickActionText}>📞 {alert.emergencyNumber}</Text>
        </TouchableOpacity>

        {driverPhone && (
          <TouchableOpacity
            style={styles.quickAction}
            onPress={handleCallDriver}
            activeOpacity={0.8}
            accessibilityRole="button"
            accessibilityLabel="Call driver"
          >
            <Text style={styles.quickActionText}>📞 Driver</Text>
          </TouchableOpacity>
        )}

        <TouchableOpacity
          style={styles.quickAction}
          onPress={handleShareTracking}
          activeOpacity={0.8}
          accessibilityRole="button"
          accessibilityLabel="Share tracking link"
        >
          <Text style={styles.quickActionText}>🔗 Tracking Link</Text>
        </TouchableOpacity>
      </ScrollView>
    </View>
  );
}

const styles = StyleSheet.create({
  banner: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    backgroundColor: theme.colors.error,
    paddingHorizontal: theme.spacing.lg,
    paddingTop: 48,
    paddingBottom: theme.spacing.sm,
    zIndex: 200,
    elevation: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.3,
    shadowRadius: 4,
  },
  topRow: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    marginBottom: theme.spacing.sm,
  },
  title: {
    color: '#fff',
    fontWeight: theme.fontWeight.bold as '700',
    fontSize: theme.fontSize.base,
  },
  resolveRow: {
    flexDirection: 'row',
    gap: theme.spacing.sm,
  },
  safeBtn: {
    backgroundColor: 'rgba(255,255,255,0.25)',
    borderRadius: theme.borderRadius.sm,
    paddingHorizontal: theme.spacing.md,
    paddingVertical: theme.spacing.sm,
    minWidth: 80,
    alignItems: 'center',
    overflow: 'hidden',
    position: 'relative',
    borderWidth: 1.5,
    borderColor: '#fff',
  },
  safeBtnFill: {
    position: 'absolute',
    top: 0,
    left: 0,
    bottom: 0,
    backgroundColor: 'rgba(255,255,255,0.35)',
  },
  safeBtnText: {
    color: '#fff',
    fontWeight: theme.fontWeight.bold as '700',
    fontSize: theme.fontSize.sm,
  },
  falseBtn: {
    backgroundColor: 'rgba(255,255,255,0.15)',
    borderRadius: theme.borderRadius.sm,
    paddingHorizontal: theme.spacing.md,
    paddingVertical: theme.spacing.sm,
    borderWidth: 1,
    borderColor: 'rgba(255,255,255,0.5)',
    minWidth: 80,
    alignItems: 'center',
  },
  falseBtnText: {
    color: '#fff',
    fontWeight: theme.fontWeight.semibold as '600',
    fontSize: theme.fontSize.sm,
  },
  quickActionsRow: {
    flexDirection: 'row',
    gap: theme.spacing.sm,
    paddingBottom: theme.spacing.xs,
  },
  quickAction: {
    backgroundColor: 'rgba(255,255,255,0.15)',
    borderRadius: theme.borderRadius.full,
    paddingHorizontal: theme.spacing.md,
    paddingVertical: theme.spacing.xs,
    borderWidth: 1,
    borderColor: 'rgba(255,255,255,0.3)',
  },
  quickActionText: {
    color: '#fff',
    fontSize: theme.fontSize.xs,
    fontWeight: theme.fontWeight.semibold as '600',
  },
});
```

- [ ] **Step 2: Update the `SosActiveBanner` call site in `live-ride.tsx`**

Find `{sosActive && sosAlert && (<SosActiveBanner alert={sosAlert} onResolve={handleSosResolve} />)}` (around line 733).

Replace with:

```tsx
{sosActive && sosAlert && (
  <SosActiveBanner
    alert={sosAlert}
    onResolve={handleSosResolve}
    driverPhone={!isDriver ? ride?.driver?.phone : undefined}
  />
)}
```

- [ ] **Step 3: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 4: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/SosActiveBanner.tsx app/live-ride.tsx
git commit -m "feat: SOS active banner — emergency/driver/tracking quick actions, hold-to-confirm safe"
```

---

## Task 7: SOS button ring visual

**Files:**
- Modify: `Carpool/components/SosButton.tsx`

---

- [ ] **Step 1: Change the full-size button to ring-around-dot style**

In `Carpool/components/SosButton.tsx`, update `styles.container` and add a `dot` style. The existing container is a 56×56 filled red circle. Change it to a transparent container with only a red border (ring), and add a smaller filled dot inside:

Update `styles`:

```tsx
const styles = StyleSheet.create({
  container: {
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: 'transparent',
    borderWidth: 3,
    borderColor: theme.colors.error,
    alignItems: 'center',
    justifyContent: 'center',
    overflow: 'visible',
    elevation: 6,
    shadowColor: theme.colors.error,
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.4,
    shadowRadius: 6,
  },
  dot: {
    width: 18,
    height: 18,
    borderRadius: 9,
    backgroundColor: theme.colors.error,
  },
  fill: {
    position: 'absolute',
    width: '100%',
    height: '100%',
    borderRadius: 24,
    backgroundColor: theme.colors.error,
    opacity: 0,
  },
  label: {
    // kept for compact variant; hidden in full variant
    color: '#fff',
    fontWeight: '800',
    fontSize: 10,
    letterSpacing: 0.5,
    position: 'absolute',
  },
  compactContainer: {
    alignItems: 'center',
    gap: 2,
    paddingHorizontal: theme.spacing.lg,
    paddingVertical: theme.spacing.xs,
  },
  compactLabel: {
    fontSize: theme.fontSize.xs,
    color: theme.colors.error,
    fontWeight: theme.fontWeight.bold as '700',
  },
});
```

In the full (non-compact) JSX, replace `<Text style={styles.label}>SOS</Text>` with `<View style={styles.dot} />`. The animated `fillStyle` ring scales around the container during hold:

```tsx
return (
  <GestureDetector gesture={gesture}>
    <Animated.View
      style={[styles.container, style, ringStyle]}
      accessibilityRole="button"
      accessibilityLabel="Emergency SOS. Press and hold for three seconds to trigger."
      accessibilityState={{ disabled }}
    >
      <Animated.View style={[styles.fill, fillStyle]} />
      <View style={styles.dot} />
    </Animated.View>
  </GestureDetector>
);
```

Update `ringStyle` to not override `borderColor` (it's now a fixed property, not animated):

```tsx
const ringStyle = useAnimatedStyle(() => ({
  transform: [{ scale: 1 + progress.value * 0.15 }],
  opacity: disabled ? 0.4 : 1,
}));
```

- [ ] **Step 2: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add components/SosButton.tsx
git commit -m "feat: SOS button — ring-around-dot visual, 48dp, compact variant for secondary row"
```

---

## Self-Review Checklist

After all tasks are complete:

- [ ] Run `npx tsc --noEmit` one final time — 0 errors
- [ ] Driver can see contextual pill with correct passenger name + N/M + ETA
- [ ] OTP card auto-focuses input, "Mark no-show" is visible without scrolling
- [ ] Sheet collapses to 10% after all passengers board; Complete Ride only accessible when expanded
- [ ] Passenger OTP digits are large and visible without scrolling
- [ ] Walk time displays for passenger distance, not raw metres
- [ ] SOS countdown shows map in top 25%, emergency call button is first action
- [ ] Reduce Motion: countdown shows 0 immediately, user must tap "Send SOS Now" — no auto-fire
- [ ] SOS Active Banner shows Emergency / Driver / Link quick actions; I'm Safe requires 1s hold
- [ ] SOS button visually distinct from Share: ring+dot vs text row entry
- [ ] No floating ETA chip, re-center FAB, share pill, or my-location FAB over the map
