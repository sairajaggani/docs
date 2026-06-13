# Live Ride Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate the bugs, security holes, cost leaks, and UX flaws identified in `backend.md` + `frontend.md` for the live-ride / SOS / live-share surface, excluding BUG-06 (Twilio — deferred to v2).

**Architecture:** Backend = Firebase Cloud Functions (Node 20, TS, ESM) + Firestore + RTDB. Frontend = React Native + Expo SDK 54. Changes are layered: backend correctness first (OTP + locking + idempotency), then RTDB-rule lockdowns, then public endpoint hardening, then frontend redesign (OTP multi-passenger, layout, SOS, perf).

**Tech Stack:** Firebase Admin SDK, Zod, Jest (ts-jest, ESM), Firebase Emulator Suite for integration, React Native Maps, Expo Location, Expo Camera (for new QR scan flow).

**Hard review checkpoint:** After Phase 1 (OTP changes — BUG-01 + SEC-01 + SEC-02) and before Phase 2 (BUG-04 RTDB lockdown). Stop, request review, do not proceed until review approves.

---

## Phase Overview

| Phase | Scope | Backend.md / frontend.md items | Stop after? |
|------|-------|---|---|
| 1 | 6-digit OTP, no plaintext in push, per-OTP attempt cap | BUG-01, SEC-01, SEC-02, COST-05 | ✅ **REVIEW PAUSE** |
| 2 | Lock down `pickupStops/*` RTDB writes | BUG-04 | — |
| 3 | Revoke-share fix + idempotency for share/SOS/FCM fanout | BUG-02, BUG-03, BUG-05 | — |
| 4 | Public tracking endpoint hardening | BUG-07, BUG-08, BUG-09, SEC-03 | — |
| 5 | Smaller backend fixes (cleanup, perf, dedup) | BUG-11, BUG-12, BUG-13, BUG-14, BUG-15 | — |
| 6 | Frontend perf / battery (driver) | EFF-01, EFF-02, EFF-03, EFF-04, EFF-05, UX-D-08, UX-D-09 | — |
| 7 | Frontend OTP multi-passenger redesign + QR scan + phone fallback | UX-D-02, UX-D-05, Part 5+7 | — |
| 8 | Driver + passenger layout cleanup | UX-D-01, UX-D-03, UX-D-04, UX-D-06, UX-D-07, UX-D-10, UX-P-01..06 | — |
| 9 | SOS UX redesign | UX-S-01..05 | — |
| 10 | RTDB path collapse + heartbeat scaling + STOPPED broadcast + listener dispose | COST-01, COST-02, COST-03, COST-04 | — |

Each phase produces a working, releasable increment. Commit at every "Commit" step. Run the full backend + frontend test suite at the end of each phase before moving on.

---

## Conventions

- **Backend test paths:** `CarpoolBackend/functions/tests/{unit,integration}/...` — use jest, run with `cd CarpoolBackend/functions && npm test -- --runInBand <path>`.
- **Frontend test paths:** `Carpool/__tests__/...` or co-located `*.test.tsx` — run with `cd Carpool && npm test -- --runInBand <path>`.
- **TDD:** Write the failing test first, see it fail, write minimal code, see it pass, commit. The "see it fail" / "see it pass" steps are mandatory.
- **Backend mocking:** `jest.mock('@config/firebase-admin')` is broken (memory: ESM live-binding + pre-load). Use `jest.spyOn(db as any, 'collection').mockReturnValue(...)` and `jest.spyOn(rtdb as any, 'ref').mockReturnValue(...)`.
- **Commit cadence:** Per memory note ("batch commits every ~4 tasks"), group commits ~every 4 task units when in subagent-driven mode. Inline mode: commit at every Commit step shown.
- **Deploy commands:** are listed at end of each phase. Do NOT deploy mid-phase.
- **Date stamping:** Where a task says "today's date", use `2026-06-12`.

---

# PHASE 1 — OTP changes (BUG-01, SEC-01, SEC-02, COST-05)

**Goal:** Make pickup OTPs collision-resistant and brute-force-resistant; stop leaking OTPs in push notification bodies.

**Architecture decisions:**
- Move from 4-digit to **6-digit** OTPs (900,000-code space) AND ensure per-ride uniqueness via in-memory loop (defense in depth).
- `verifyPassengerOtp` fetches all matches (no `.limit(1)`) and throws on >1 (fails loudly instead of misverifying).
- Add a per-booking `otpAttempts` counter. After 5 wrong attempts on a single booking, that OTP is killed: regenerate it, push a NEW notification to the passenger, log the attempt as suspicious.
- `RIDE_STARTED` push body no longer contains the OTP. Only the FCM `data` payload carries it (for an in-app deep-link to surface the OTP). The lockscreen-visible `body` is generic.

**Files touched (Phase 1):**
- Modify: `CarpoolBackend/functions/src/functions/rides/startRide.fn.ts`
- Modify: `CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts`
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts`
- Modify: `CarpoolBackend/functions/src/config/constants.ts`
- Modify: `CarpoolBackend/functions/src/types/index.ts` (Booking type — add `otpAttempts?: number`)
- Test: `CarpoolBackend/functions/tests/unit/rides/startRide.fn.test.ts` (extend)
- Test: `CarpoolBackend/functions/tests/unit/rides/verifyPassengerOtp.fn.test.ts` (extend)
- Test: `CarpoolBackend/functions/tests/integration/rides/otp-uniqueness.test.ts` (new)

---

## Task 1.1: Add OTP constants and config

**Files:**
- Modify: `CarpoolBackend/functions/src/config/constants.ts`

- [ ] **Step 1: Read current constants file**

(Already in context — see `CONSTANTS.RATE_LIMITS.VERIFY_PASSENGER_OTP: 50` at line 51.)

- [ ] **Step 2: Add OTP block under CONSTANTS, after `TRACKING`**

```ts
  // OTP (passenger pickup verification)
  OTP: {
    LENGTH: 6,
    MIN: 100000,
    MAX: 1000000,           // exclusive — crypto.randomInt(MIN, MAX)
    MAX_ATTEMPTS_PER_BOOKING: 5,
    MAX_PER_RIDE_GEN_RETRIES: 50,   // defensive cap on uniqueness loop
  },
```

- [ ] **Step 3: Build + lint**

Run: `cd CarpoolBackend/functions && npm run build`
Expected: PASS, no TS errors.

- [ ] **Step 4: Commit**

```bash
git add CarpoolBackend/functions/src/config/constants.ts
git commit -m "feat(safety): add OTP length/attempts constants for 6-digit upgrade"
```

---

## Task 1.2: Update Zod schema to accept 6-digit OTP

**Files:**
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts:230`

- [ ] **Step 1: Find current schema**

Run: `grep -n "verifyPassengerOtp\|otp:" CarpoolBackend/functions/src/middleware/validation.middleware.ts`
Expected: line 230 shows `otp: z.string().length(4),`

- [ ] **Step 2: Write failing test (validation accepts 6-digit, rejects 4-digit/7-digit/non-numeric)**

Create or extend `CarpoolBackend/functions/tests/unit/middleware/validation.middleware.test.ts`:

```ts
import { schemas } from "@middleware/validation.middleware";

describe("verifyPassengerOtp schema", () => {
  it("accepts a 6-digit numeric string", () => {
    expect(() => schemas.verifyPassengerOtp.parse({ rideId: "r1", otp: "123456" })).not.toThrow();
  });
  it("rejects a 4-digit OTP", () => {
    expect(() => schemas.verifyPassengerOtp.parse({ rideId: "r1", otp: "1234" })).toThrow();
  });
  it("rejects a 7-digit OTP", () => {
    expect(() => schemas.verifyPassengerOtp.parse({ rideId: "r1", otp: "1234567" })).toThrow();
  });
  it("rejects non-numeric characters", () => {
    expect(() => schemas.verifyPassengerOtp.parse({ rideId: "r1", otp: "12a456" })).toThrow();
  });
});
```

- [ ] **Step 3: Run test — confirm 6-digit acceptance fails (current schema is 4-digit)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/middleware/validation.middleware.test.ts`
Expected: First test FAILS ("String must contain exactly 4 character(s)").

- [ ] **Step 4: Update schema**

In `validation.middleware.ts`, change the OTP rule. Replace:

```ts
otp: z.string().length(4),
```

with:

```ts
otp: z.string().regex(/^\d{6}$/, "OTP must be 6 digits"),
```

- [ ] **Step 5: Run test — all pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/middleware/validation.middleware.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add CarpoolBackend/functions/src/middleware/validation.middleware.ts CarpoolBackend/functions/tests/unit/middleware/validation.middleware.test.ts
git commit -m "feat(safety): require 6-digit pickup OTP in Zod schema"
```

---

## Task 1.3: Generate 6-digit OTPs with per-ride uniqueness in `startRide`

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/startRide.fn.ts:33-36, 200-206`
- Test: `CarpoolBackend/functions/tests/unit/rides/startRide.fn.test.ts`

- [ ] **Step 1: Write failing unit test for uniqueness loop**

Add to `tests/unit/rides/startRide.fn.test.ts`:

```ts
import { __test_generateUniqueOtps } from "@functions/rides/startRide.fn";

describe("startRide OTP generation", () => {
  it("never produces duplicate OTPs across N bookings", () => {
    // Run 1000 trials with 8 bookings each — verify uniqueness inside each trial
    for (let trial = 0; trial < 1000; trial++) {
      const ids = ["b1","b2","b3","b4","b5","b6","b7","b8"];
      const map = __test_generateUniqueOtps(ids);
      const codes = [...map.values()];
      expect(new Set(codes).size).toBe(codes.length);
      codes.forEach((c) => expect(c).toMatch(/^\d{6}$/));
    }
  });
});
```

- [ ] **Step 2: Run test — fails (helper not exported yet)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/startRide.fn.test.ts -t "OTP generation"`
Expected: FAIL — `__test_generateUniqueOtps` is undefined.

- [ ] **Step 3: Replace `generateOtp` + OTP map block in `startRide.fn.ts`**

Replace lines 33-36:

```ts
function generateOtp(): string {
  return crypto.randomInt(1000, 9999).toString();
}
```

with:

```ts
/** Generate a single 6-digit OTP (100000-999999). */
function generateOtp(): string {
  return crypto.randomInt(CONSTANTS.OTP.MIN, CONSTANTS.OTP.MAX).toString().padStart(6, "0");
}

/**
 * Generate unique OTPs for each booking, guaranteeing no two
 * bookings on this ride share a code. Bounded retry on collision.
 */
export function __test_generateUniqueOtps(bookingIds: string[]): Map<string, string> {
  const map = new Map<string, string>();
  const used = new Set<string>();
  for (const id of bookingIds) {
    let otp = "";
    for (let i = 0; i < CONSTANTS.OTP.MAX_PER_RIDE_GEN_RETRIES; i++) {
      const candidate = generateOtp();
      if (!used.has(candidate)) { otp = candidate; break; }
    }
    if (!otp) throw new Error("Failed to generate unique OTP after max retries");
    used.add(otp);
    map.set(id, otp);
  }
  return map;
}
```

Then replace lines 200-206 (the existing otpMap block):

```ts
      // ── OTP generation (idempotent + per-ride unique) ─
      // Reuse existing booking OTP if present (idempotent on retry);
      // for new bookings, generate via uniqueness loop.
      const otpMap = new Map<string, string>();
      const seedExisting: string[] = [];
      const needNew: string[] = [];
      confirmedBookings.forEach((b) => {
        if (b.pickupOtp) {
          otpMap.set(b.bookingId, b.pickupOtp);
          seedExisting.push(b.pickupOtp);
        } else {
          needNew.push(b.bookingId);
        }
      });
      // Generate unique codes for the new ones — seed `used` with existing OTPs
      const used = new Set<string>(seedExisting);
      for (const id of needNew) {
        let otp = "";
        for (let i = 0; i < CONSTANTS.OTP.MAX_PER_RIDE_GEN_RETRIES; i++) {
          const c = generateOtp();
          if (!used.has(c)) { otp = c; break; }
        }
        if (!otp) throw Errors.INTERNAL_ERROR("Failed to generate unique OTP");
        used.add(otp);
        otpMap.set(id, otp);
      }
```

Update the `Booking` import or local `BookingWithRef` type so `pickupOtp` is recognized as `string | undefined` (it already is in `types/index.ts:204`). Drop the `(b as any).pickupOtp` cast — fixes BUG-15 as a side-effect.

- [ ] **Step 4: Run uniqueness test — passes**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/startRide.fn.test.ts -t "OTP generation"`
Expected: PASS.

- [ ] **Step 5: Run all existing startRide tests**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/startRide.fn.test.ts`
Expected: PASS (any tests asserting 4-digit OTPs in fixtures need updating — fix them now to use 6-digit fixtures).

- [ ] **Step 6: Commit**

```bash
git add CarpoolBackend/functions/src/functions/rides/startRide.fn.ts CarpoolBackend/functions/tests/unit/rides/startRide.fn.test.ts
git commit -m "fix(otp): 6-digit OTPs with per-ride uniqueness loop (BUG-01, BUG-15)"
```

---

## Task 1.4: Remove OTP from push notification body (SEC-02)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/startRide.fn.ts:307-316`

- [ ] **Step 1: Write failing test**

Add to `tests/unit/rides/startRide.fn.test.ts`:

```ts
import { NotificationService } from "@services/notification.service";

describe("startRide SEC-02 — OTP not in push body", () => {
  it("notification body does NOT contain the OTP string", async () => {
    const spy = jest.spyOn(NotificationService, "sendToUser").mockResolvedValue(undefined as any);
    // ... set up minimal mocked ride/booking via spyOn(db,...) — see existing test fixtures
    // After calling startRide handler:
    const calls = spy.mock.calls;
    const pushed = calls.find((c) => c[4] === "RIDE_STARTED")!;
    const body = pushed[2] as string;
    expect(body).not.toMatch(/\d{4,7}/);   // no 4-7 digit run
    // OTP should still be in data payload
    const dataPayload = pushed[3] as Record<string, unknown>;
    expect(typeof dataPayload.otp).toBe("string");
    expect(dataPayload.otp).toMatch(/^\d{6}$/);
  });
});
```

- [ ] **Step 2: Run test — fails (body contains `Show OTP ${otp}`)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/startRide.fn.test.ts -t "SEC-02"`
Expected: FAIL.

- [ ] **Step 3: Replace notification body**

In `startRide.fn.ts`, replace the loop at lines 307-316:

```ts
      confirmedBookings.forEach((booking) => {
        const otp = otpMap.get(booking.bookingId)!;
        NotificationService.sendToUser(
          booking.passengerId,
          "Ride Started — Head to Pickup!",
          "Your driver is on the way. Open the app for your pickup code.",
          { rideId, bookingId: booking.bookingId, otp, type: "RIDE_STARTED" },
          CONSTANTS.NOTIFICATION_TYPES.RIDE_STARTED
        );
      });
```

(OTP stays in the FCM `data` payload — the in-app handler can deep-link to the OTP card. The lockscreen-visible `body` no longer leaks it.)

- [ ] **Step 4: Run test — passes**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/startRide.fn.test.ts -t "SEC-02"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/rides/startRide.fn.ts CarpoolBackend/functions/tests/unit/rides/startRide.fn.test.ts
git commit -m "fix(security): remove OTP from RIDE_STARTED push body (SEC-02)"
```

---

## Task 1.5: Fail-loud on OTP duplicate match in `verifyPassengerOtp` (BUG-01 part 2)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts:52-58`

- [ ] **Step 1: Write failing test — duplicate OTP across two bookings raises**

Add to `tests/unit/rides/verifyPassengerOtp.fn.test.ts`:

```ts
describe("verifyPassengerOtp — duplicate OTP across bookings (BUG-01)", () => {
  it("throws OTP_AMBIGUOUS when two bookings share the same OTP", async () => {
    const mockRideRef = { get: jest.fn().mockResolvedValue({ exists: true, data: () => ({ driverId: "drv1", status: "STARTED" }) }) };
    const collA = { doc: jest.fn().mockReturnValue(mockRideRef) };

    const fakeBookingsQuery = {
      where: jest.fn().mockReturnThis(),
      limit: jest.fn().mockReturnThis(),
      get: jest.fn().mockResolvedValue({
        empty: false,
        size: 2,
        docs: [
          { ref: { id: "b1" }, id: "b1", data: () => ({ pickupOtp: "123456", joinedAt: null, passengerId: "p1", status: "CONFIRMED" }) },
          { ref: { id: "b2" }, id: "b2", data: () => ({ pickupOtp: "123456", joinedAt: null, passengerId: "p2", status: "CONFIRMED" }) },
        ],
      }),
    };

    jest.spyOn(db as any, "collection")
      .mockImplementationOnce(() => collA)              // rides
      .mockImplementationOnce(() => fakeBookingsQuery); // bookings

    await expect(
      verifyPassengerOtpHandler({ rideId: "r1", otp: "123456" }, { extendedAuth: { uid: "drv1" } } as any)
    ).rejects.toMatchObject({ code: "OTP_AMBIGUOUS" });
  });
});
```

(Extract the inner handler logic into an exported `verifyPassengerOtpHandler` if not already — same pattern used in `revokeRideShare.fn.ts`.)

- [ ] **Step 2: Run test — fails (current code has `.limit(1)` and silently picks one)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/verifyPassengerOtp.fn.test.ts -t "duplicate OTP"`
Expected: FAIL.

- [ ] **Step 3: Refactor `verifyPassengerOtp.fn.ts` to fetch all matches**

Replace lines 51-65:

```ts
      // Find ALL bookings matching the OTP (BUG-01: fail loud on ambiguity)
      const bookingsSnapshot = await db
        .collection(CONSTANTS.COLLECTIONS.BOOKINGS)
        .where("rideId", "==", rideId)
        .where("pickupOtp", "==", otp)
        .where("status", "==", "CONFIRMED")
        .get();

      if (bookingsSnapshot.empty) {
        throw Errors.INVALID_INPUT({ message: "Invalid OTP" });
      }

      if (bookingsSnapshot.size > 1) {
        // Should be impossible after the uniqueness loop in startRide,
        // but if it happens we MUST NOT silently mis-verify a passenger.
        logger.error("verifyPassengerOtp: OTP collision detected", {
          rideId, matchCount: bookingsSnapshot.size,
        });
        throw new FunctionError(
          "OTP_AMBIGUOUS",
          "Pickup OTP matches multiple bookings — please contact support.",
          409
        );
      }

      const bookingRef = bookingsSnapshot.docs[0].ref;
      const bookingId = bookingsSnapshot.docs[0].id;
```

Add `import { FunctionError } from "../../utils/errors.js";` and `import { logger } from "../../utils/logger.js";` to the top of the file.

Also export the inner handler so unit tests can drive it without the onCall wrapper:

```ts
export async function verifyPassengerOtpHandler(
  data: { rideId: string; otp: string },
  authContext: AuthContext
) { ... }
```

Then the onCall body becomes a thin wrapper that calls it.

- [ ] **Step 4: Run test — passes**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/verifyPassengerOtp.fn.test.ts -t "duplicate OTP"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts CarpoolBackend/functions/tests/unit/rides/verifyPassengerOtp.fn.test.ts
git commit -m "fix(otp): fail loudly when pickup OTP matches >1 booking (BUG-01)"
```

---

## Task 1.6: Per-booking OTP attempt counter + regeneration (SEC-01 / COST-05)

**Files:**
- Modify: `CarpoolBackend/functions/src/types/index.ts:204` — add `otpAttempts?: number` to `Booking`
- Modify: `CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts`
- Test: `CarpoolBackend/functions/tests/unit/rides/verifyPassengerOtp.fn.test.ts`

**Design**: every failing transaction (wrong OTP within `bookingRef` match) increments `otpAttempts`. Once `otpAttempts >= MAX_ATTEMPTS_PER_BOOKING`, we rotate that booking's `pickupOtp` (server generates a new unique one) and push a new FCM. This kills brute force AND covers BUG-01's residual collision case ("two passengers' OTPs collide → one is killed → driver gets a fresh code via push").

- [ ] **Step 1: Update `Booking` type**

Add to the `Booking` interface in `types/index.ts:200-210`:

```ts
  pickupOtp?: string;
  otpAttempts?: number;        // wrong-OTP counter on this booking
  otpRotatedAt?: Timestamp;    // last forced rotation
```

- [ ] **Step 2: Write failing test — wrong OTP increments counter**

Add to `tests/unit/rides/verifyPassengerOtp.fn.test.ts`:

```ts
describe("verifyPassengerOtp — attempt cap (SEC-01)", () => {
  it("increments otpAttempts on wrong OTP for an existing CONFIRMED booking", async () => {
    // Setup: ride exists STARTED, no booking matches OTP "999999",
    //        but a CONFIRMED booking with pickupOtp="111111" exists for the same ride.
    // Expectation: an "Invalid OTP" throw AND the existing booking's otpAttempts is now 1.
    // (Counter targets the user's CURRENT pickup queue — increment the next pending pickup.)
    // ... see existing test setup
    await expect(verifyPassengerOtpHandler({ rideId: "r1", otp: "999999" }, ctx)).rejects.toThrow("Invalid OTP");
    expect(updateSpy).toHaveBeenCalledWith(expect.objectContaining({
      otpAttempts: 1,
    }));
  });

  it("rotates OTP after MAX_ATTEMPTS_PER_BOOKING wrong attempts", async () => {
    // Setup booking with otpAttempts = 4 (one away from cap).
    await expect(verifyPassengerOtpHandler({ rideId: "r1", otp: "999999" }, ctx)).rejects.toThrow(/Invalid OTP|OTP_ROTATED/);
    expect(updateSpy).toHaveBeenCalledWith(expect.objectContaining({
      pickupOtp: expect.stringMatching(/^\d{6}$/),
      otpAttempts: 0,
      otpRotatedAt: expect.anything(),
    }));
    // FCM push fired with new OTP
    expect(NotificationService.sendToUser).toHaveBeenCalledWith(
      expect.any(String),
      expect.stringMatching(/code/i),
      expect.not.stringMatching(/\d{4,7}/),       // body still does not include OTP (SEC-02)
      expect.objectContaining({ otp: expect.stringMatching(/^\d{6}$/) }),
      "RIDE_STARTED"
    );
  });
});
```

- [ ] **Step 3: Run tests — fail (no counter logic yet)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/verifyPassengerOtp.fn.test.ts -t "attempt cap"`
Expected: FAIL.

- [ ] **Step 4: Implement counter + rotation**

In `verifyPassengerOtp.fn.ts`, on a wrong-OTP path, identify the next-pending booking in the ride (lowest `order` in `pickupStops` with `joinedAt == null`) and increment its `otpAttempts`. If it hits the cap, rotate.

Add this helper (top of file):

```ts
async function rotateOtp(
  rideId: string,
  bookingRef: FirebaseFirestore.DocumentReference,
  booking: Booking
): Promise<string> {
  // Pull all OTPs currently in use on this ride so the new one is unique
  const inUseSnap = await db
    .collection(CONSTANTS.COLLECTIONS.BOOKINGS)
    .where("rideId", "==", rideId)
    .where("status", "==", "CONFIRMED")
    .get();
  const used = new Set<string>();
  inUseSnap.docs.forEach((d) => {
    const otp = (d.data() as Booking).pickupOtp;
    if (otp) used.add(otp);
  });
  let newOtp = "";
  for (let i = 0; i < CONSTANTS.OTP.MAX_PER_RIDE_GEN_RETRIES; i++) {
    const c = crypto.randomInt(CONSTANTS.OTP.MIN, CONSTANTS.OTP.MAX).toString().padStart(6, "0");
    if (!used.has(c)) { newOtp = c; break; }
  }
  if (!newOtp) throw Errors.INTERNAL_ERROR("Failed to rotate OTP");
  await bookingRef.update({
    pickupOtp: newOtp,
    otpAttempts: 0,
    otpRotatedAt: Timestamp.now(),
    updatedAt: Timestamp.now(),
  });
  NotificationService.sendToUser(
    booking.passengerId,
    "Pickup code updated",
    "Your driver entered the wrong code too many times. Open the app for your new pickup code.",
    { rideId, bookingId: bookingRef.id, otp: newOtp, type: "RIDE_STARTED" },
    CONSTANTS.NOTIFICATION_TYPES.RIDE_STARTED
  );
  return newOtp;
}
```

Modify the empty-match branch to increment-and-maybe-rotate the next pending booking:

```ts
      if (bookingsSnapshot.empty) {
        // Wrong OTP for any CONFIRMED booking on this ride.
        // Find the next-pending booking and bump its attempt counter.
        const pendingSnap = await db
          .collection(CONSTANTS.COLLECTIONS.BOOKINGS)
          .where("rideId", "==", rideId)
          .where("status", "==", "CONFIRMED")
          .where("joinedAt", "==", null)
          .orderBy("createdAt")  // deterministic — earliest created first
          .limit(1)
          .get();
        if (!pendingSnap.empty) {
          const ref = pendingSnap.docs[0].ref;
          const data = pendingSnap.docs[0].data() as Booking;
          const attempts = (data.otpAttempts ?? 0) + 1;
          if (attempts >= CONSTANTS.OTP.MAX_ATTEMPTS_PER_BOOKING) {
            await rotateOtp(rideId, ref, data);
          } else {
            await ref.update({ otpAttempts: attempts, updatedAt: Timestamp.now() });
          }
        }
        throw Errors.INVALID_INPUT({ message: "Invalid OTP" });
      }
```

(Note: requires a Firestore index on `(rideId, status, joinedAt, createdAt)` — add it in the next step.)

- [ ] **Step 5: Add Firestore index**

Edit `CarpoolBackend/firestore.indexes.json`. Add:

```json
{
  "collectionGroup": "bookings",
  "queryScope": "COLLECTION",
  "fields": [
    { "fieldPath": "rideId", "order": "ASCENDING" },
    { "fieldPath": "status", "order": "ASCENDING" },
    { "fieldPath": "joinedAt", "order": "ASCENDING" },
    { "fieldPath": "createdAt", "order": "ASCENDING" }
  ]
}
```

- [ ] **Step 6: Run unit tests — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/verifyPassengerOtp.fn.test.ts`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add CarpoolBackend/functions/src/types/index.ts CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts CarpoolBackend/firestore.indexes.json CarpoolBackend/functions/tests/unit/rides/verifyPassengerOtp.fn.test.ts
git commit -m "feat(safety): per-booking OTP attempt cap + auto-rotate (SEC-01)"
```

---

## Task 1.7: Update frontend OTP input + push handler for 6-digit

**Files:**
- Modify: `Carpool/app/live-ride.tsx` (driver OTP card) — input length 6
- Modify: any place that displays the passenger's OTP — render with 6 boxes
- Search/grep for hard-coded `4` in OTP contexts

- [ ] **Step 1: Find OTP UI**

Run: `grep -rn "pickupOtp\|maxLength={4}\|otp.*length.*4" Carpool/`
Expected: hits include `live-ride.tsx`, the passenger ride-details screen, and any OTP card.

- [ ] **Step 2: Bump UI to 6 digits everywhere**

For each location, change:

```tsx
<TextInput maxLength={4} keyboardType="number-pad" ... />
```

to:

```tsx
<TextInput maxLength={6} keyboardType="number-pad" ... />
```

For displays like `<View>{otp.split("").map((c) => ...)}</View>` — the dynamic split keeps working since `otp` is now 6 chars.

For any "0000" placeholder, change to "000000".

- [ ] **Step 3: Find FCM RIDE_STARTED handler — surface OTP from `data` payload**

Run: `grep -rn "RIDE_STARTED" Carpool/`

Confirm the handler reads `data.otp` (still present in the payload) and either (a) deep-links to the booked-ride/live-ride screen, or (b) updates local cache. No body-text scraping needed.

- [ ] **Step 4: Manual test in dev**

Run: `cd Carpool && npm run start`
Action: send a test RIDE_STARTED FCM (or run the trigger via emulator) and verify the in-app OTP card shows the new 6-digit code AND the lockscreen body no longer includes digits.

- [ ] **Step 5: Commit**

```bash
git add Carpool/
git commit -m "feat(otp): UI handles 6-digit pickup codes (BUG-01)"
```

---

## Task 1.8: Integration test — full OTP flow on emulator

**Files:**
- Test: `CarpoolBackend/functions/tests/integration/rides/otp-uniqueness.test.ts` (new)

- [ ] **Step 1: Write integration test**

```ts
import { startRideHandler, verifyPassengerOtpHandler } from "...";
// Use the existing test helpers in tests/integration/helpers — createDriver, createPassenger, createRide, createBooking.

describe("OTP integration — uniqueness + attempt cap", () => {
  it("8 bookings on a ride all receive distinct 6-digit OTPs", async () => {
    const ride = await testHelpers.createRide({ seats: 8 });
    const bookings = await Promise.all([...Array(8)].map(() => testHelpers.createConfirmedBooking({ rideId: ride.id })));
    await startRideHandler({ rideId: ride.id }, { extendedAuth: { uid: ride.driverId } } as any);
    const updated = await Promise.all(bookings.map((b) => db.collection("bookings").doc(b.id).get()));
    const otps = updated.map((s) => s.data()!.pickupOtp);
    expect(new Set(otps).size).toBe(otps.length);
    otps.forEach((o) => expect(o).toMatch(/^\d{6}$/));
  });

  it("5 wrong attempts on a passenger's OTP triggers rotation + FCM", async () => {
    const ride = await testHelpers.createStartedRideWithOneBooking();
    const wrong = "999999";
    for (let i = 0; i < 4; i++) {
      await expect(verifyPassengerOtpHandler({ rideId: ride.id, otp: wrong }, ride.driverCtx)).rejects.toThrow("Invalid OTP");
    }
    // 5th attempt rotates
    await expect(verifyPassengerOtpHandler({ rideId: ride.id, otp: wrong }, ride.driverCtx)).rejects.toThrow("Invalid OTP");
    const after = await db.collection("bookings").doc(ride.bookings[0].id).get();
    expect(after.data()!.otpAttempts).toBe(0);
    expect(after.data()!.otpRotatedAt).toBeDefined();
    expect(after.data()!.pickupOtp).not.toBe(ride.bookings[0].pickupOtp);
  });
});
```

- [ ] **Step 2: Run integration**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/rides/otp-uniqueness.test.ts`
Expected: PASS.

- [ ] **Step 3: Run full unit + integration suite — no regressions**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand`
Expected: All pre-existing test counts hold (881 baseline per memory) plus the new tests added.

- [ ] **Step 4: Commit**

```bash
git add CarpoolBackend/functions/tests/integration/rides/otp-uniqueness.test.ts
git commit -m "test(otp): integration coverage for uniqueness + attempt-cap rotation"
```

---

## Task 1.9: Deploy + smoke-test Phase 1

- [ ] **Step 1: Deploy backend**

```bash
cd CarpoolBackend
firebase deploy --only "functions:startRide,functions:verifyPassengerOtp,firestore:indexes"
```

Expected: success, indexes building.

- [ ] **Step 2: Manual smoke (testflight or dev build)**

- Create a ride with 2 confirmed passengers, start it. Both passengers see 6-digit OTPs in app. FCM body says "Open the app for your pickup code" — no digits.
- Driver enters wrong OTP 5×. Passenger receives a new OTP push, the in-app code updates.
- Driver enters one passenger's correct OTP. They are marked picked up. The other passenger's code still works.

- [ ] **Step 3: Update `todo.md`**

Mark BUG-01, SEC-01, SEC-02, COST-05, BUG-15 (cast cleanup done as side-effect) as ✅ in `/Users/sairajaggani/Desktop/Space/Project/todo.md`.

- [ ] **Step 4: Commit todo + deploy**

```bash
git add /Users/sairajaggani/Desktop/Space/Project/todo.md
git commit -m "chore: mark Phase 1 (OTP hardening) complete"
```

---

# 🛑 REVIEW CHECKPOINT — Phase 1 complete

**STOP HERE.**

Before starting Phase 2 (BUG-04 RTDB lockdown), present these for review:
- All commits made in Phase 1 (`git log --oneline` since plan start).
- Test results summary.
- Deploy confirmation.
- Any deviations from this plan.

Do NOT begin Phase 2 until review approves. The RTDB lockdown changes write semantics for any client touching `pickupStops/*` — if a beta driver is running an older app, they could be locked out.

---

# PHASE 2 — Lock down `pickupStops/*` RTDB writes (BUG-04)

**Goal:** Drivers can no longer mark a passenger PICKED_UP via direct RTDB write; the only way to set `pickupStops/{passengerId}.status` is via Cloud Function. The CF (Admin SDK) bypasses the rule.

**Architecture decisions:**
- `verifyPassengerOtp` performs the RTDB write **inside its existing flow** (post-transaction). Move the RTDB update from the client into the CF.
- Add a new CF `markPassengerNoShow` for the explicit no-show path. It requires `status === CONFIRMED && joinedAt == null` and writes both the booking field AND the RTDB pickupStop status.
- Rule change: `pickupStops/$passengerId .write` removed for normal status writes. Allow only `notifiedAt` writes from driver (so the local "I tapped Ask" can still timestamp), if needed; otherwise lock it down entirely.

**Files:**
- Modify: `CarpoolBackend/database.rules.json:37-42`
- Modify: `CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts`
- Create: `CarpoolBackend/functions/src/functions/rides/markPassengerNoShow.fn.ts`
- Modify: `CarpoolBackend/functions/src/functions/rides/index.ts` (export)
- Modify: `CarpoolBackend/functions/src/index.ts` (export from `rides`)
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts` (add `markPassengerNoShow` schema)
- Modify: `CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts` (add rate limiter)
- Modify: `CarpoolBackend/functions/src/config/constants.ts` (add MARK_PASSENGER_NO_SHOW rate)
- Modify: `Carpool/src/services/firebase/realtime.service.ts:254-264` — remove `markPassengerPickedUp`, add a `markPassengerNoShow` that calls the CF
- Modify: `Carpool/src/services/firebase/functions.service.ts` — add `markPassengerNoShow`
- Modify: `Carpool/app/live-ride.tsx` — wire to new CF
- Test: `CarpoolBackend/functions/tests/integration/rides/pickup-rules-lockdown.test.ts` (new)
- Test: `CarpoolBackend/functions/tests/unit/rides/markPassengerNoShow.fn.test.ts` (new)

---

## Task 2.1: Write CF — `markPassengerNoShow`

- [ ] **Step 1: Write failing unit test**

Create `CarpoolBackend/functions/tests/unit/rides/markPassengerNoShow.fn.test.ts`:

```ts
import { handleMarkPassengerNoShow } from "@functions/rides/markPassengerNoShow.fn";
import { db, rtdb } from "@config/firebase-admin";

describe("markPassengerNoShow", () => {
  it("rejects when booking is not CONFIRMED", async () => {
    // ... spyOn(db) returns a booking with status: "CANCELLED"
    await expect(handleMarkPassengerNoShow({ rideId: "r1", bookingId: "b1" }, "drv1"))
      .rejects.toThrow(/CONFIRMED/);
  });
  it("rejects when booking already has joinedAt", async () => {
    // ... booking has joinedAt set
    await expect(handleMarkPassengerNoShow({ rideId: "r1", bookingId: "b1" }, "drv1"))
      .rejects.toThrow(/already|joined/i);
  });
  it("rejects when caller is not the driver", async () => {
    await expect(handleMarkPassengerNoShow({ rideId: "r1", bookingId: "b1" }, "imposter"))
      .rejects.toThrow(/UNAUTHORIZED|owner/i);
  });
  it("updates booking + RTDB pickupStop on success", async () => {
    // ... happy path
    await handleMarkPassengerNoShow({ rideId: "r1", bookingId: "b1" }, "drv1");
    expect(bookingUpdateSpy).toHaveBeenCalledWith(expect.objectContaining({ noShow: true }));
    expect(rtdbUpdateSpy).toHaveBeenCalledWith(expect.objectContaining({ status: "NO_SHOW" }));
  });
});
```

- [ ] **Step 2: Run test — fails (no file yet)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/markPassengerNoShow.fn.test.ts`
Expected: FAIL (`Cannot find module`).

- [ ] **Step 3: Implement the CF**

Create `CarpoolBackend/functions/src/functions/rides/markPassengerNoShow.fn.ts`:

```ts
import * as functions from "firebase-functions";
import { Timestamp } from "firebase-admin/firestore";
import { db, rtdb } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { logger } from "../../utils/logger.js";
import { ResponseFormatter } from "../../utils/response.js";
import { authenticateUser, requireEmailVerification, requireOwnership } from "../../middleware/auth.middleware.js";
import { validateData, schemas } from "../../middleware/validation.middleware.js";
import { rateLimiters } from "../../middleware/rate-limit.middleware.js";
import { withErrorHandler } from "../../middleware/error-handler.middleware.js";
import { withLogging } from "../../middleware/logging.middleware.js";
import { Errors } from "../../utils/errors.js";
import type { Ride, Booking } from "../../types/index.js";

export async function handleMarkPassengerNoShow(
  data: { rideId: string; bookingId: string },
  uid: string
) {
  const { rideId, bookingId } = data;
  const rideRef = db.collection(CONSTANTS.COLLECTIONS.RIDES).doc(rideId);
  const bookingRef = db.collection(CONSTANTS.COLLECTIONS.BOOKINGS).doc(bookingId);

  await db.runTransaction(async (tx) => {
    const [rideSnap, bookingSnap] = await Promise.all([tx.get(rideRef), tx.get(bookingRef)]);
    if (!rideSnap.exists) throw Errors.NOT_FOUND("Ride");
    if (!bookingSnap.exists) throw Errors.NOT_FOUND("Booking");
    const ride = rideSnap.data() as Ride;
    const booking = bookingSnap.data() as Booking;
    if (ride.driverId !== uid) throw Errors.UNAUTHORIZED();
    if (ride.status !== "STARTED") {
      throw Errors.INVALID_INPUT({ message: "Ride must be STARTED to mark no-show." });
    }
    if (booking.status !== "CONFIRMED") {
      throw Errors.INVALID_INPUT({ message: `Booking must be CONFIRMED, is ${booking.status}` });
    }
    if (booking.joinedAt) {
      throw Errors.INVALID_INPUT({ message: "Passenger already joined — cannot mark no-show." });
    }
    const now = Timestamp.now();
    tx.update(bookingRef, { noShow: true, noShowAt: now, updatedAt: now });
  });

  // RTDB pickupStop update — Admin SDK bypasses rules
  await rtdb
    .ref(`${CONSTANTS.RTDB_PATHS.ACTIVE_RIDES}/${rideId}/pickupStops/${(await bookingRef.get()).data()!.passengerId}`)
    .update({ status: "NO_SHOW", notifiedAt: Date.now() });

  logger.info("markPassengerNoShow", { rideId, bookingId, uid });
  return ResponseFormatter.success({ rideId, bookingId, status: "NO_SHOW" });
}

export const markPassengerNoShow = functions.https.onCall(
  withErrorHandler(async (data, context) => {
    return withLogging("markPassengerNoShow", async (data: any, authContext: any) => {
      requireEmailVerification(authContext);
      await rateLimiters.markPassengerNoShow(authContext);
      const validated = validateData(schemas.markPassengerNoShow, data);
      return handleMarkPassengerNoShow(validated, authContext.extendedAuth.uid);
    })(data, await authenticateUser(context));
  })
);
```

- [ ] **Step 4: Wire up schema, rate-limit, exports**

Edit `validation.middleware.ts`:

```ts
markPassengerNoShow: z.object({
  rideId: z.string().min(1),
  bookingId: z.string().min(1),
}),
```

Edit `rate-limit.middleware.ts`:

```ts
markPassengerNoShow: createRateLimiter("MARK_PASSENGER_NO_SHOW"),
```

Edit `constants.ts`, in `RATE_LIMITS`:

```ts
MARK_PASSENGER_NO_SHOW: 20,
```

Edit `functions/rides/index.ts`:

```ts
export { markPassengerNoShow } from "./markPassengerNoShow.fn.js";
```

Edit root `src/index.ts` to re-export.

- [ ] **Step 5: Run unit tests — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/rides/markPassengerNoShow.fn.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add CarpoolBackend/functions/src/functions/rides/markPassengerNoShow.fn.ts CarpoolBackend/functions/src/functions/rides/index.ts CarpoolBackend/functions/src/index.ts CarpoolBackend/functions/src/middleware/validation.middleware.ts CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts CarpoolBackend/functions/src/config/constants.ts CarpoolBackend/functions/tests/unit/rides/markPassengerNoShow.fn.test.ts
git commit -m "feat(rides): add markPassengerNoShow CF (BUG-04 prep)"
```

---

## Task 2.2: Move `pickupStops/{passenger}.status = PICKED_UP` into `verifyPassengerOtp` CF

- [ ] **Step 1: Add RTDB write to `verifyPassengerOtp.fn.ts`**

After the Firestore transaction succeeds (returns `result`), append:

```ts
      // BUG-04: write PICKED_UP into RTDB via Admin SDK (bypasses the locked-down rule)
      await rtdb
        .ref(`${CONSTANTS.RTDB_PATHS.ACTIVE_RIDES}/${rideId}/pickupStops/${result.passengerId}`)
        .update({ status: "PICKED_UP", notifiedAt: Date.now() });
```

Import `rtdb`:

```ts
import { db, rtdb } from "../../config/firebase-admin.js";
```

- [ ] **Step 2: Integration test (emulator)**

Create `CarpoolBackend/functions/tests/integration/rides/pickup-rules-lockdown.test.ts`:

```ts
import { rtdb } from "@config/firebase-admin";
import { verifyPassengerOtpHandler } from "...";

describe("BUG-04 — pickupStops RTDB lockdown", () => {
  it("verifyPassengerOtp writes PICKED_UP to RTDB on success", async () => {
    const ride = await testHelpers.createStartedRideWithOneBooking();
    await verifyPassengerOtpHandler({ rideId: ride.id, otp: ride.bookings[0].pickupOtp }, ride.driverCtx);
    const stop = await rtdb.ref(`activeRides/${ride.id}/pickupStops/${ride.bookings[0].passengerId}`).once("value");
    expect(stop.val()).toMatchObject({ status: "PICKED_UP" });
  });

  it("Direct client RTDB write to pickupStops is denied after rules update", async () => {
    // This test uses the rules emulator with the new rules — see Task 2.3.
  });
});
```

- [ ] **Step 3: Run integration**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/rides/pickup-rules-lockdown.test.ts -t "PICKED_UP"`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add CarpoolBackend/functions/src/functions/rides/verifyPassengerOtp.fn.ts CarpoolBackend/functions/tests/integration/rides/pickup-rules-lockdown.test.ts
git commit -m "fix(rides): move PICKED_UP RTDB write into CF (BUG-04)"
```

---

## Task 2.3: Tighten `pickupStops/*` RTDB rule

- [ ] **Step 1: Write failing rules test**

Add to `CarpoolBackend/functions/tests/integration/security-rules/rtdb.rules.test.ts` (existing per memory: 17 tests):

```ts
describe("BUG-04 — pickupStops are no longer driver-writable", () => {
  it("driver cannot write pickupStops/$passengerId via client SDK", async () => {
    // Setup: rules emulator, signed-in as the driver of activeRides/{rideId}/meta.
    const ctx = testEnv.authenticatedContext("drv1");
    await assertFails(ctx.database().ref("activeRides/r1/pickupStops/p1").update({ status: "PICKED_UP" }));
  });
  it("driver can still READ pickupStops", async () => {
    const ctx = testEnv.authenticatedContext("drv1");
    await assertSucceeds(ctx.database().ref("activeRides/r1/pickupStops/p1").once("value"));
  });
});
```

- [ ] **Step 2: Run — fails (rule still allows write)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/security-rules/rtdb.rules.test.ts -t "BUG-04"`
Expected: FAIL on "cannot write" test.

- [ ] **Step 3: Update `database.rules.json`**

Replace the `pickupStops` block (lines 37-42) with:

```jsonc
        "pickupStops": {
          ".read": "auth != null && (root.child('activeRides').child($rideId).child('meta').child('driverId').val() === auth.uid || root.child('activeRides').child($rideId).child('meta').child('confirmedPassengerIds').child(auth.uid).val() === true)",
          "$passengerId": {
            ".write": false
          }
        }
```

(All writes now go through Admin SDK only — `verifyPassengerOtp` and `markPassengerNoShow`.)

- [ ] **Step 4: Run rules test — passes**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/security-rules/rtdb.rules.test.ts`
Expected: All 17 + new tests pass.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/database.rules.json CarpoolBackend/functions/tests/integration/security-rules/rtdb.rules.test.ts
git commit -m "fix(rules): lock pickupStops writes to Admin SDK only (BUG-04)"
```

---

## Task 2.4: Remove client-side RTDB write in `realtime.service.ts`

- [ ] **Step 1: Read the existing client write**

Open `Carpool/src/services/firebase/realtime.service.ts` lines 254-264. Find `markPassengerPickedUp` (or similar) that writes `pickupStops/{passengerId}/status = PICKED_UP` directly to RTDB.

- [ ] **Step 2: Delete that method**

Remove the function entirely. (After Task 2.3 it would always fail anyway.)

- [ ] **Step 3: Add `markPassengerNoShow` to `functions.service.ts`**

Edit `Carpool/src/services/firebase/functions.service.ts`:

```ts
export async function markPassengerNoShow(rideId: string, bookingId: string) {
  return callFn("markPassengerNoShow", { rideId, bookingId });
}
```

- [ ] **Step 4: Wire `live-ride.tsx` — no-show button calls the new CF**

Find the existing onPress handler for the "No Show" button in `live-ride.tsx`. Replace its body with:

```ts
await markPassengerNoShow(ride.id, booking.bookingId);
```

The RTDB update happens server-side; the existing onValue listener will re-render the UI automatically.

For the OTP success path: the existing call to `verifyPassengerOtp` is unchanged. Drop any client code that wrote `pickupStops/{passenger}.status = PICKED_UP` after the verify resolved — the CF now does that.

- [ ] **Step 5: Run frontend unit tests**

Run: `cd Carpool && npm test -- --runInBand`
Expected: PASS (or update tests that mocked `markPassengerPickedUp`).

- [ ] **Step 6: Manual smoke test**

Run: `cd Carpool && npm run start`
Action:
- Start a ride with a confirmed passenger.
- Verify OTP → passenger shows as PICKED_UP on map (driven by RTDB listener).
- Mark a second passenger as no-show → both driver and passenger views reflect NO_SHOW.

- [ ] **Step 7: Commit**

```bash
git add Carpool/src/services/firebase/realtime.service.ts Carpool/src/services/firebase/functions.service.ts Carpool/app/live-ride.tsx
git commit -m "fix(client): route pickup state through CF only (BUG-04)"
```

---

## Task 2.5: Deploy Phase 2

- [ ] **Step 1: Deploy**

```bash
cd CarpoolBackend
firebase deploy --only "database,functions:verifyPassengerOtp,functions:markPassengerNoShow"
```

Expected: success.

- [ ] **Step 2: Update todo + commit**

Mark BUG-04 ✅ in `todo.md`.

```bash
git commit -am "chore: Phase 2 (RTDB pickupStops lockdown) deployed"
```

---

# PHASE 3 — Idempotency + revoke-share + FCM dedup (BUG-02, BUG-03, BUG-05)

**Goal:** Make every "active record" create idempotent at the database layer (deterministic IDs + `create()`), and replace the broken `setTimeout` in `revokeRideShare` with a synchronous status flip + scheduled deletion.

**Architecture decisions:**
- `triggerSos`: doc ID = `${uid}_${rideId}` (one ACTIVE alert per (user, ride) at any time — RESOLVED docs are renamed `${uid}_${rideId}_${triggeredAtMs}` for archival).
- `createRideShare`: doc ID = `${rideId}_${uid}` (one ACTIVE share per (ride, user)). Token stored as field, not used as ID (token must remain unguessable and rotate-able — keep `randomBytes(24)`).
- For SOS, since RESOLVED docs need to exist alongside potential new active ones, keep the ACTIVE alert at the deterministic ID; on resolve, archive by **copying** to `sosAlertsArchive/{auto}` and deleting the ACTIVE doc. Simpler than ID renames.
- `onSosAlertCreate`: replace `get()`-then-`set()` with `create()`-first-then-send-FCM. ALREADY_EXISTS → skip.
- `revokeRideShare`: synchronously flip the RTDB `liveTracking/{token}` to `{ status: "STOPPED" }`, then `await` both `liveTracking` and `rideSharesByToken` removals inside the handler. (Drop the `setTimeout`.) Grace period for clients to observe the STOPPED status is handled by writing the marker BEFORE removal.

---

## Task 3.1: Make `createRideShare` use deterministic ID + `create()`

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts`
- Test: `CarpoolBackend/functions/tests/integration/safety/createRideShare.test.ts`

- [ ] **Step 1: Write failing test — concurrent calls return one doc**

Add to existing test file:

```ts
it("BUG-03 — concurrent createRideShare calls produce one ACTIVE doc", async () => {
  const rideId = "r-concurrency-1";
  await testHelpers.createStartedRide({ id: rideId, driverId: "drv1" });
  const [a, b] = await Promise.all([
    handleCreateRideShare(rideId, "drv1"),
    handleCreateRideShare(rideId, "drv1"),
  ]);
  expect(a.data.shareToken).toBe(b.data.shareToken);
  const snap = await db.collection("rideShares")
    .where("rideId", "==", rideId).where("userId", "==", "drv1").where("status", "==", "ACTIVE").get();
  expect(snap.size).toBe(1);
});
```

- [ ] **Step 2: Run — fails (current code uses `randomBytes` ID per call)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/createRideShare.test.ts -t "BUG-03"`
Expected: FAIL (two ACTIVE docs created).

- [ ] **Step 3: Refactor `createRideShare.fn.ts`**

Replace the body of `handleCreateRideShare` from the idempotency check onward:

```ts
  // Deterministic ID: one ACTIVE share per (ride, user)
  const docId = `${rideId}_${uid}`;
  const shareRef = db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES).doc(docId);

  // Cap check (per-user across rides) — runs BEFORE create attempt
  const activeSnap = await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES)
    .where("userId", "==", uid).where("status", "==", "ACTIVE").get();
  if (activeSnap.size >= CONSTANTS.RIDE_SHARE.MAX_ACTIVE_PER_USER) {
    // If the cap is hit BY this same (rideId,uid) doc, that's idempotent — return it.
    const existing = activeSnap.docs.find((d) => d.id === docId);
    if (existing) {
      return ResponseFormatter.success({
        shareToken: existing.data().shareToken,
        trackingUrl: existing.data().trackingUrl,
        driverName: existing.data().driverName,
        vehiclePlate: existing.data().vehiclePlate,
        passengerName: existing.data().passengerName,
      });
    }
    throw Errors.INVALID_INPUT({
      message: `You already have ${CONSTANTS.RIDE_SHARE.MAX_ACTIVE_PER_USER} active ride shares. Stop one before sharing again.`,
    });
  }

  const shareToken = randomBytes(24).toString("base64url");
  const trackingUrl = `${CONSTANTS.RIDE_SHARE.TRACKING_URL_BASE}/${shareToken}`;
  const now = Timestamp.now();
  const expiresAtTs = Timestamp.fromMillis(Date.now() + CONSTANTS.RIDE_SHARE.EXPIRY_MS);
  const driverName = formatName(ride.driver.firstName, ride.driver.lastName);

  const shareDoc = {
    shareToken, rideId, userId: uid, type: "share" as const, status: "ACTIVE" as const,
    driverName, vehiclePlate: ride.vehicleDetails.plate, vehicleModel: ride.vehicleDetails.model,
    passengerName: sharerName, trackingUrl,
    createdAt: now, expiresAt: expiresAtTs, viewerCount: 0,
  };

  try {
    await shareRef.create(shareDoc);
    await rtdb.ref(`rideSharesByToken/${shareToken}`).set({ userId: uid });
    logger.info("createRideShare: created", { uid, rideId, shareToken });
    return ResponseFormatter.success({ shareToken, trackingUrl, driverName, vehiclePlate: ride.vehicleDetails.plate, passengerName: sharerName });
  } catch (err: any) {
    if (err.code === 6 /* ALREADY_EXISTS */) {
      const existing = await shareRef.get();
      const d = existing.data()!;
      if (d.status !== "ACTIVE") {
        // Existing doc but in a non-active state (REVOKED/EXPIRED) — we want a new share.
        // Strategy: rotate the doc ID by appending the timestamp.
        const newId = `${docId}_${Date.now()}`;
        await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES).doc(newId).create(shareDoc);
        await rtdb.ref(`rideSharesByToken/${shareToken}`).set({ userId: uid });
        return ResponseFormatter.success({ shareToken, trackingUrl, driverName, vehiclePlate: ride.vehicleDetails.plate, passengerName: sharerName });
      }
      return ResponseFormatter.success({
        shareToken: d.shareToken, trackingUrl: d.trackingUrl, driverName: d.driverName, vehiclePlate: d.vehiclePlate, passengerName: d.passengerName,
      });
    }
    throw err;
  }
```

- [ ] **Step 4: Run tests — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/createRideShare.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts CarpoolBackend/functions/tests/integration/safety/createRideShare.test.ts
git commit -m "fix(safety): idempotent createRideShare via deterministic ID + create() (BUG-03)"
```

---

## Task 3.2: Fix `revokeRideShare` setTimeout (BUG-02)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/revokeRideShare.fn.ts`

- [ ] **Step 1: Write failing test — RTDB cleanup completes by handler return**

Add to `tests/unit/safety/revokeRideShare.fn.test.ts`:

```ts
it("BUG-02 — RTDB cleanup is awaited (no setTimeout)", async () => {
  const removeSpy = jest.fn().mockResolvedValue(undefined);
  jest.spyOn(rtdb as any, "ref").mockReturnValue({ remove: removeSpy, set: jest.fn().mockResolvedValue(undefined) });
  // ... happy-path setup
  await handleRevokeRideShare("tok-1", "uid-1");
  expect(removeSpy).toHaveBeenCalled();   // not deferred
});
```

- [ ] **Step 2: Run — fails (current code defers with setTimeout)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/safety/revokeRideShare.fn.test.ts -t "BUG-02"`
Expected: FAIL.

- [ ] **Step 3: Replace handler body**

Replace lines 27-37 in `revokeRideShare.fn.ts`:

```ts
  await shareRef.update({ status: "REVOKED", revokedAt: Timestamp.now() });

  // Tell live listeners we've stopped — synchronous so subscribers observe before delete.
  await rtdb.ref(`liveTracking/${shareToken}`).update({ status: "STOPPED", stoppedAt: Date.now() });

  // Await cleanup of both RTDB indices inside the handler — no setTimeout.
  await Promise.all([
    rtdb.ref(`liveTracking/${shareToken}`).remove(),
    rtdb.ref(`rideSharesByToken/${shareToken}`).remove(),
  ]);

  logger.info("revokeRideShare: revoked + cleaned", { uid, shareToken });
  return ResponseFormatter.success({ success: true });
```

(`RTDB_CLEANUP_GRACE_MS` constant is no longer used — leave it for now in case other code references it; remove in the cleanup task at the end of the phase.)

- [ ] **Step 4: Run — passes**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/safety/revokeRideShare.fn.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/revokeRideShare.fn.ts CarpoolBackend/functions/tests/unit/safety/revokeRideShare.fn.test.ts
git commit -m "fix(safety): await RTDB cleanup in revokeRideShare; emit STOPPED status (BUG-02)"
```

---

## Task 3.3: Idempotent `triggerSos` via deterministic ID + create() (BUG-03)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts`

- [ ] **Step 1: Write failing test — concurrent SOS triggers return same alertId**

Add to `tests/integration/safety/triggerSos.test.ts`:

```ts
it("BUG-03 — concurrent triggerSos returns same alertId", async () => {
  const rideId = "r-sos-1";
  await testHelpers.createStartedRide({ id: rideId, driverId: "drv1" });
  const payload = { rideId, location: { lat: 1, lng: 1, accuracy: 5 }, batteryPct: 50, simCountry: "IN" };
  const [a, b] = await Promise.all([
    handleTriggerSos(payload, "drv1"),
    handleTriggerSos(payload, "drv1"),
  ]);
  expect(a.data.alertId).toBe(b.data.alertId);
  const snap = await db.collection("sosAlerts").where("userId", "==", "drv1").where("rideId", "==", rideId).where("status", "==", "ACTIVE").get();
  expect(snap.size).toBe(1);
});
```

- [ ] **Step 2: Run — fails**

Expected: FAIL.

- [ ] **Step 3: Refactor `triggerSos`**

Use deterministic doc ID = `${uid}_${rideId}` for the ACTIVE alert. On resolve, the resolver archives to a copy doc (Task in Phase 5) — for now, the ACTIVE doc lives at that ID.

Replace the idempotency block (lines 45-63):

```ts
  const alertId = `${uid}_${rideId}`;
  const alertRef = db.collection(CONSTANTS.COLLECTIONS.SOS_ALERTS).doc(alertId);

  // Try to short-circuit — if an ACTIVE doc already exists, return it.
  const existingSnap = await alertRef.get();
  if (existingSnap.exists && existingSnap.data()!.status === "ACTIVE") {
    const existing = existingSnap.data()!;
    return ResponseFormatter.success({
      alertId,
      shareToken: existing.shareToken,
      trackingUrl: existing.trackingUrl,
      emergencyNumber: existing.emergencyNumber,
      contactsToNotify: existing.contactsToNotify,
    });
  }
```

Replace the existing `add()` + update-id block (lines 147-148) with `create()`:

```ts
  const alertData = {
    alertId,
    userId: uid, rideId,
    shareToken, trackingUrl,
    status: "ACTIVE" as const,
    emergencyNumber, country, batteryPct,
    initialSnapshot: { driverName, vehiclePlate, vehicleModel, lat: location.lat, lng: location.lng },
    notifiedContactIds, contactsToNotify,
    triggeredAt: now_ts,
  };

  try {
    await alertRef.create(alertData);
  } catch (err: any) {
    if (err.code === 6 /* ALREADY_EXISTS */) {
      const snap = await alertRef.get();
      const d = snap.data()!;
      if (d.status === "ACTIVE") {
        return ResponseFormatter.success({
          alertId, shareToken: d.shareToken, trackingUrl: d.trackingUrl,
          emergencyNumber: d.emergencyNumber, contactsToNotify: d.contactsToNotify,
        });
      }
      // Non-active doc exists at this ID — archive it sideways and write the new one.
      const archiveRef = db.collection("sosAlertsArchive").doc(`${alertId}_${Date.now()}`);
      await archiveRef.set(d);
      await alertRef.set(alertData);
    } else {
      throw err;
    }
  }
```

- [ ] **Step 4: Update `resolveSos` to archive on resolve**

In `resolveSos.fn.ts`, change the success path to:

```ts
  const snap = await alertRef.get();
  const data = snap.data()!;
  const archived = { ...data, status: resolution, resolvedAt: Timestamp.now() };
  await db.collection("sosAlertsArchive").doc(`${data.alertId}_${Date.now()}`).set(archived);
  await alertRef.delete();
```

(Tests in `tests/unit/safety/resolveSos.test.ts` will need to follow this.)

- [ ] **Step 5: Update `sosAutoResolve` scheduler to archive too**

Similar pattern in `functions/scheduled/sosAutoResolve.fn.ts`.

- [ ] **Step 6: Run tests — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/triggerSos.test.ts tests/unit/safety/`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts CarpoolBackend/functions/src/functions/safety/resolveSos.fn.ts CarpoolBackend/functions/src/functions/scheduled/sosAutoResolve.fn.ts CarpoolBackend/functions/tests/
git commit -m "fix(sos): deterministic alertId + archive on resolve (BUG-03)"
```

---

## Task 3.4: Idempotent FCM fanout in `onSosAlertCreate` (BUG-05)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/onSosAlertCreate.fn.ts`

- [ ] **Step 1: Write failing test — double-fire only sends one FCM per contact**

Add to `tests/unit/safety/onSosAlertCreate.fn.test.ts`:

```ts
it("BUG-05 — concurrent trigger fires FCM exactly once per contact", async () => {
  const sendSpy = jest.spyOn(NotificationService, "sendToUser").mockResolvedValue(undefined);
  const fakeSnap = { data: () => ({ notifiedContactIds: ["c1","c2"], trackingUrl: "u", initialSnapshot: { driverName: "X" } }) };
  await Promise.all([
    onSosAlertCreateHandler(fakeSnap as any, { params: { alertId: "a1" } } as any),
    onSosAlertCreateHandler(fakeSnap as any, { params: { alertId: "a1" } } as any),
  ]);
  expect(sendSpy).toHaveBeenCalledTimes(2);  // c1 + c2, not 4
});
```

- [ ] **Step 2: Run — fails**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/safety/onSosAlertCreate.fn.test.ts -t "BUG-05"`
Expected: FAIL (current code has a check-then-write race).

- [ ] **Step 3: Refactor — `create()` first**

Replace the per-contact closure body (lines 32-53) with:

```ts
      notifiedContactIds.map(async (contactId: string) => {
        try {
          // Claim the contact via create() BEFORE sending — wins the race.
          await notificationsRef.doc(contactId).create({
            contactId,
            sentAt: Timestamp.now(),
            fcmSuccessCount: 0,    // bumped after send succeeds
          });
        } catch (err: any) {
          if (err.code === 6 /* ALREADY_EXISTS */) {
            logger.debug("onSosAlertCreate: notification already claimed", { alertId, contactId });
            return;
          }
          throw err;
        }
        try {
          await NotificationService.sendToUser(
            contactId,
            "🚨 SOS Alert",
            `${driverName} has triggered an emergency SOS. Track their live location.`,
            { type: "SOS_ALERT", alertId, trackingUrl },
            "SOS_ALERT"
          );
          await notificationsRef.doc(contactId).update({ fcmSuccessCount: 1 });
        } catch (sendErr) {
          // Roll back the claim only on hard failure so a retry can re-send.
          await notificationsRef.doc(contactId).delete().catch(() => {});
          throw sendErr;
        }
      })
```

- [ ] **Step 4: Run — passes**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/safety/onSosAlertCreate.fn.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/onSosAlertCreate.fn.ts CarpoolBackend/functions/tests/unit/safety/onSosAlertCreate.fn.test.ts
git commit -m "fix(sos): claim-then-send for idempotent FCM fanout (BUG-05)"
```

---

## Task 3.5: Deploy + integration smoke for Phase 3

- [ ] **Step 1: Deploy**

```bash
cd CarpoolBackend
firebase deploy --only "functions:createRideShare,functions:revokeRideShare,functions:triggerSos,functions:resolveSos,functions:sosAutoResolve,functions:onSosAlertCreate"
```

- [ ] **Step 2: Smoke**

- Double-tap SOS in app → only one alert created. Trusted contacts receive one FCM each.
- Revoke a live share → tracking page sees `status: "STOPPED"` briefly, then 410 EXPIRED.

- [ ] **Step 3: Update todo + commit**

Mark BUG-02, BUG-03, BUG-05 ✅.

```bash
git commit -am "chore: Phase 3 (idempotency + revoke fix) deployed"
```

---

# PHASE 4 — Public tracking endpoint hardening (BUG-07, BUG-08, BUG-09, SEC-03)

**Goal:** `getTrackingData` is the only un-authed endpoint. Lock it down: validate inputs server-side, transactionally enforce viewer cap, fail-closed on rate-limit failures, and add an `expiresAt`-based RTDB rule for `liveTracking/{token}`.

---

## Task 4.1: Tighten `triggerSos` input validation (BUG-07)

**Files:**
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts` (triggerSos schema)
- Modify: `CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts`

- [ ] **Step 1: Find current schema**

Run: `grep -n "triggerSos" CarpoolBackend/functions/src/middleware/validation.middleware.ts`

- [ ] **Step 2: Write failing schema tests**

Add to `validation.middleware.test.ts`:

```ts
describe("triggerSos — BUG-07", () => {
  const okBody = { rideId: "r1", location: { lat: 1, lng: 1, accuracy: 5 }, batteryPct: 50, simCountry: "IN" };
  it("rejects unknown country code", () => {
    expect(() => schemas.triggerSos.parse({ ...okBody, simCountry: "XX" })).toThrow();
  });
  it("rejects negative battery", () => {
    expect(() => schemas.triggerSos.parse({ ...okBody, batteryPct: -1 })).toThrow();
  });
  it("rejects battery > 100", () => {
    expect(() => schemas.triggerSos.parse({ ...okBody, batteryPct: 101 })).toThrow();
  });
  it("rejects fractional battery", () => {
    expect(() => schemas.triggerSos.parse({ ...okBody, batteryPct: 50.5 })).toThrow();
  });
});
```

- [ ] **Step 3: Run — fail**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/middleware/validation.middleware.test.ts -t "BUG-07"`
Expected: FAIL.

- [ ] **Step 4: Update schema**

Find the triggerSos schema in validation.middleware.ts. Replace with:

```ts
const SUPPORTED_COUNTRIES = ["IN","US","CA","GB","AU","NZ","DE","FR","IT","ES","PT","NL","BE","AT","CH","SE","NO","DK","FI","PL","CZ","HU","RO","BG","HR","SK","SI","EE","LV","LT","IE","LU","MT","CY","GR"] as const;
// ...
triggerSos: z.object({
  rideId: z.string().min(1),
  location: z.object({
    lat: z.number().gte(-90).lte(90),
    lng: z.number().gte(-180).lte(180),
    accuracy: z.number().gte(0).lte(1e6),
  }),
  batteryPct: z.number().int().gte(0).lte(100),
  simCountry: z.enum(SUPPORTED_COUNTRIES).optional(),
}),
```

- [ ] **Step 5: Cross-check against profile country**

In `triggerSos.fn.ts`, after resolving `simCountry`, load the user's profile and pick the more conservative number:

```ts
  const profileSnap = await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(uid).get();
  const profileCountry = profileSnap.data()?.country?.toUpperCase();
  const clientCountry = simCountry?.toUpperCase();
  const country = clientCountry && SUPPORTED_COUNTRIES.includes(clientCountry as any)
    ? clientCountry
    : (profileCountry && SUPPORTED_COUNTRIES.includes(profileCountry as any) ? profileCountry : "IN");
  const emergencyNumber = resolveEmergencyNumber(country);
```

- [ ] **Step 6: Run tests — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/unit/middleware/ tests/unit/safety/triggerSos.fn.test.ts tests/integration/safety/triggerSos.test.ts`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add CarpoolBackend/functions/src/middleware/validation.middleware.ts CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts CarpoolBackend/functions/tests/
git commit -m "fix(sos): validate simCountry + batteryPct, cross-check with profile (BUG-07)"
```

---

## Task 4.2: Transactional viewer cap in `getTrackingData` (BUG-08)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts`

- [ ] **Step 1: Write failing test — race produces N viewer increments but cap not exceeded**

Add to `tests/integration/safety/getTrackingData.test.ts`:

```ts
it("BUG-08 — concurrent viewers cannot exceed VIEWER_CAP", async () => {
  // Setup: create share with viewerCount = 49 (cap = 50)
  const token = await testHelpers.createShare({ viewerCount: 49 });
  const results = await Promise.all([...Array(20)].map(() => callGetTrackingData(token)));
  const ok = results.filter((r) => r.status === 200);
  const denied = results.filter((r) => r.status === 410);
  expect(ok.length).toBe(1);       // exactly one slot left
  expect(denied.length).toBe(19);
});
```

- [ ] **Step 2: Run — fail (current code has a non-atomic check)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/getTrackingData.test.ts -t "BUG-08"`
Expected: FAIL.

- [ ] **Step 3: Wrap cap check + increment in a transaction**

Replace the block at lines 72-100 in `getTrackingData.fn.ts`:

```ts
  // Atomic cap check + viewer increment
  try {
    await db.runTransaction(async (tx) => {
      const fresh = await tx.get(docRef);
      const f = fresh.data()!;
      const v = f.viewerCount ?? 0;
      if (v >= CONSTANTS.TRACKING.VIEWER_CAP) {
        throw new Error("VIEWER_CAP_EXCEEDED");
      }
      tx.update(docRef, { viewerCount: v + 1 });
    });
  } catch (e: any) {
    if (e.message === "VIEWER_CAP_EXCEEDED") {
      res.status(410).json({ error: "VIEWER_CAP_EXCEEDED" });
      return;
    }
    throw e;
  }
```

(Move the rate-limit check to BEFORE the transaction — and fix BUG-09 in the same task.)

- [ ] **Step 4: Run — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/getTrackingData.test.ts -t "BUG-08"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts CarpoolBackend/functions/tests/integration/safety/getTrackingData.test.ts
git commit -m "fix(tracking): transactional viewer cap enforcement (BUG-08)"
```

---

## Task 4.3: Fail-closed on rate-limit RTDB failure (BUG-09)

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts`

- [ ] **Step 1: Write failing test — RTDB unreachable → 503**

```ts
it("BUG-09 — RTDB rate-limit failure returns 503, not 200", async () => {
  jest.spyOn(rtdb as any, "ref").mockImplementation(() => ({
    transaction: () => Promise.reject(new Error("rtdb down")),
  }));
  const res = await callGetTrackingData("tok-1");
  expect(res.status).toBe(503);
});
```

- [ ] **Step 2: Run — fails (current code fails open with 200)**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/getTrackingData.test.ts -t "BUG-09"`
Expected: FAIL.

- [ ] **Step 3: Change `catch` block in `getTrackingData.fn.ts`**

Replace:

```ts
  } catch (rlErr) {
    logger.warn("getTrackingData: rate-limit RTDB unavailable, failing open", { rlErr });
  }
```

with:

```ts
  } catch (rlErr) {
    logger.error("getTrackingData: rate-limit RTDB unavailable — failing closed", { rlErr });
    res.status(503).json({ error: "RATE_LIMITER_UNAVAILABLE" });
    return;
  }
```

- [ ] **Step 4: Run — pass**

Run: `cd CarpoolBackend/functions && npm test -- --runInBand tests/integration/safety/getTrackingData.test.ts -t "BUG-09"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts CarpoolBackend/functions/tests/
git commit -m "fix(tracking): fail-closed on rate-limiter RTDB failure (BUG-09)"
```

---

## Task 4.4: `liveTracking/{token}` rule respects `expiresAt` (SEC-03)

**Files:**
- Modify: `CarpoolBackend/database.rules.json:46-50`
- Modify: `CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts` (write `expiresAt` to RTDB index)
- Modify: `CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts` (same)

**Plan:** denormalize `expiresAt` onto `liveTracking/{token}/expiresAt` at write time. Update the rule to require `auth != null || data.child('expiresAt').val() > now`.

- [ ] **Step 1: Write the expiry data when bootstrapping live tracking**

In `triggerSos.fn.ts`, in the `rtdb.ref('liveTracking/...').set(...)` block, add:

```ts
  await rtdb.ref(`liveTracking/${shareToken}`).set({
    lat: location.lat, lng: location.lng, accuracy: location.accuracy,
    batteryPct, updatedAt: Date.now(), userId: uid,
    expiresAt: Date.now() + CONSTANTS.SOS.AUTO_RESOLVE_AFTER_MS,
  });
```

In `createRideShare.fn.ts`, when initializing `rideSharesByToken/{token}` (or after share doc create), set the matching `liveTracking/{token}` skeleton with `expiresAt`:

```ts
  await rtdb.ref(`liveTracking/${shareToken}`).set({
    expiresAt: Date.now() + CONSTANTS.RIDE_SHARE.EXPIRY_MS,
    userId: uid,
    updatedAt: Date.now(),
  });
```

- [ ] **Step 2: Write failing rules test — read after expiry is denied**

Add to `tests/integration/security-rules/rtdb.rules.test.ts`:

```ts
describe("SEC-03 — liveTracking/{token} read window", () => {
  it("allows public read before expiresAt", async () => {
    await testEnv.unauthenticatedContext().database().ref("liveTracking/t1").set({ expiresAt: Date.now() + 60_000 });
    await assertSucceeds(testEnv.unauthenticatedContext().database().ref("liveTracking/t1").once("value"));
  });
  it("denies public read after expiresAt", async () => {
    await testEnv.unauthenticatedContext().database().ref("liveTracking/t1").set({ expiresAt: Date.now() - 60_000 });
    await assertFails(testEnv.unauthenticatedContext().database().ref("liveTracking/t1").once("value"));
  });
});
```

- [ ] **Step 3: Run — fail (current rule is `.read: true` unconditionally)**

- [ ] **Step 4: Update the rule**

```jsonc
    "liveTracking": {
      "$token": {
        ".read": "auth != null || (data.child('expiresAt').val() != null && data.child('expiresAt').val() > now)",
        ".write": "auth != null && (root.child('sosAlertsByToken/' + $token + '/userId').val() == auth.uid || root.child('rideSharesByToken/' + $token + '/userId').val() == auth.uid)"
      }
    },
```

- [ ] **Step 5: Run — passes**

- [ ] **Step 6: Public tracking page (`CarpoolBackend/public/track/track.js`) — adapt**

The public page is unauthenticated. After SEC-03 it can ONLY read while `expiresAt > now`. Update `track.js` to:
- Show "Tracking expired" on RTDB permission-denied errors.
- Stop polling and stop the listener.

- [ ] **Step 7: Commit**

```bash
git add CarpoolBackend/database.rules.json CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts CarpoolBackend/public/track/track.js CarpoolBackend/functions/tests/integration/security-rules/rtdb.rules.test.ts
git commit -m "fix(rules): liveTracking read requires fresh expiresAt or auth (SEC-03)"
```

---

## Task 4.5: Deploy Phase 4

- [ ] **Step 1: Deploy**

```bash
cd CarpoolBackend
firebase deploy --only "functions:triggerSos,functions:createRideShare,functions:getTrackingData,database,hosting"
```

- [ ] **Step 2: Smoke**

- Open a tracking link from an active share → loads. Wait > expiresAt → page shows "Expired".
- Open a stale link from a year ago → 410 from CF, "Expired" page.
- Hit the CF rapidly from a single token → 429 after 60 in a minute.

- [ ] **Step 3: Update todo + commit**

Mark BUG-07, BUG-08, BUG-09, SEC-03 ✅.

---

# PHASE 5 — Smaller backend fixes (BUG-11, BUG-12, BUG-13, BUG-14, BUG-15)

(BUG-15 was already addressed as a side-effect of Phase 1 Task 1.3 — confirm and check off.)

---

## Task 5.1: BUG-11 — `triggerSos` uses `where(phone, "in", ...)` instead of N+1 lookup

- [ ] **Step 1: Write failing perf test (counts read calls)**

```ts
it("BUG-11 — trusted-contact lookup is a single Firestore query", async () => {
  const getSpy = jest.spyOn(db as any, "collection");
  // ... seed 5 trusted contacts
  await handleTriggerSos(payload, "uid1");
  // The `profiles` collection should be queried ONCE for the contact lookup (plus the profile cross-check from BUG-07)
  const profilesQueries = getSpy.mock.calls.filter((c) => c[0] === "profiles");
  expect(profilesQueries.length).toBeLessThanOrEqual(2);
});
```

- [ ] **Step 2: Run — fails (current code loops .where().limit(1))**

- [ ] **Step 3: Replace the loop**

In `triggerSos.fn.ts`, replace the lines 102-115 block:

```ts
  const contactsToNotify: Array<{ name: string; phoneE164: string }> = [];
  const phones: string[] = [];
  contactsSnap.docs.forEach((doc) => {
    const c = doc.data() as { name: string; phoneE164: string };
    contactsToNotify.push({ name: c.name, phoneE164: c.phoneE164 });
    phones.push(c.phoneE164);
  });

  let notifiedContactIds: string[] = [];
  if (phones.length > 0) {
    // Firestore `in` supports up to 10 values; MAX_PER_USER=5 — safely fits.
    const profilesSnap = await db
      .collection(CONSTANTS.COLLECTIONS.PROFILES)
      .where("phone", "in", phones)
      .get();
    notifiedContactIds = profilesSnap.docs.map((d) => d.id);
  }
```

- [ ] **Step 4: Run — pass**

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/triggerSos.fn.ts CarpoolBackend/functions/tests/
git commit -m "perf(sos): single phone-in query for trusted contacts (BUG-11)"
```

---

## Task 5.2: BUG-12 — `executeRideCompletion` cleanup queue on RTDB failure

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/utils/rideCompletion.util.ts:90-96`
- Add: `CarpoolBackend/functions/src/functions/scheduled/cleanupQueue.fn.ts` (new)

- [ ] **Step 1: Write failing test**

```ts
it("BUG-12 — RTDB cleanup failure enqueues retry doc", async () => {
  jest.spyOn(rtdb as any, "ref").mockReturnValue({ remove: jest.fn().mockRejectedValue(new Error("rtdb down")) });
  await executeRideCompletion("r-fail-1", { ... });
  const queue = await db.collection("cleanupQueue").where("rideId", "==", "r-fail-1").get();
  expect(queue.size).toBe(1);
});
```

- [ ] **Step 2: Run — fail**

- [ ] **Step 3: Wrap RTDB remove with try/catch + enqueue**

```ts
  try {
    await rtdb.ref(`${CONSTANTS.RTDB_PATHS.ACTIVE_RIDES}/${rideId}`).remove();
  } catch (err) {
    logger.error("rideCompletion: RTDB cleanup failed — enqueuing", { rideId, err });
    await db.collection("cleanupQueue").doc(rideId).set({
      rideId, path: `activeRides/${rideId}`, attempts: 0, createdAt: Timestamp.now(),
    });
  }
```

- [ ] **Step 4: Scheduled sweeper**

Create `CarpoolBackend/functions/src/functions/scheduled/cleanupQueue.fn.ts`:

```ts
import * as functions from "firebase-functions";
import { Timestamp } from "firebase-admin/firestore";
import { db, rtdb } from "../../config/firebase-admin.js";
import { logger } from "../../utils/logger.js";

export const cleanupQueueSweeper = functions.pubsub
  .schedule("every 15 minutes")
  .onRun(async () => {
    const snap = await db.collection("cleanupQueue").where("attempts", "<", 5).limit(50).get();
    for (const d of snap.docs) {
      const { path } = d.data();
      try {
        await rtdb.ref(path).remove();
        await d.ref.delete();
      } catch (err) {
        await d.ref.update({ attempts: (d.data().attempts ?? 0) + 1, lastError: String(err), lastAttemptAt: Timestamp.now() });
      }
    }
    logger.info("cleanupQueue: swept", { processed: snap.size });
    return null;
  });
```

Export it from `scheduled/index.ts` and root `index.ts`.

- [ ] **Step 5: Run tests — pass**

- [ ] **Step 6: Commit**

```bash
git add CarpoolBackend/functions/src/functions/utils/rideCompletion.util.ts CarpoolBackend/functions/src/functions/scheduled/cleanupQueue.fn.ts CarpoolBackend/functions/src/functions/scheduled/index.ts CarpoolBackend/functions/src/index.ts CarpoolBackend/functions/tests/
git commit -m "fix(rides): retry RTDB cleanup via scheduled queue (BUG-12)"
```

---

## Task 5.3: BUG-13 — Single `currentlyStartedRideId` on driver profile

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/rides/startRide.fn.ts` (set the field)
- Modify: `CarpoolBackend/functions/src/functions/utils/rideCompletion.util.ts` (clear the field on COMPLETED/CANCELLED)
- Migration: a one-shot script to backfill the field for existing profiles (optional — most active rides clear out quickly).

- [ ] **Step 1: Write failing test for `startRide` — checks profile.currentlyStartedRideId, not activeRideIds list**

```ts
it("BUG-13 — startRide uses currentlyStartedRideId for active-ride check", async () => {
  const profileRef = db.collection("profiles").doc("drv1");
  await profileRef.set({ currentlyStartedRideId: "other-ride" });
  await expect(startRideHandler({ rideId: "r1" }, { extendedAuth: { uid: "drv1" } } as any))
    .rejects.toMatchObject({ code: "ACTIVE_RIDE_EXISTS" });
});

it("BUG-13 — startRide is O(1) reads on driver profile", async () => {
  // Same as Phase 5.1 — count `rides` collection reads. Should NOT scale with stale activeRideIds.
});
```

- [ ] **Step 2: Run — fail**

- [ ] **Step 3: Refactor**

Replace the block in `startRide.fn.ts` at lines 131-158 with:

```ts
  // ── Single active ride enforcement (O(1)) ────────────────────────────
  const driverProfileDoc = await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(uid).get();
  const currentlyStartedRideId: string | null = driverProfileDoc.data()?.currentlyStartedRideId ?? null;
  if (currentlyStartedRideId && currentlyStartedRideId !== rideId) {
    throw new FunctionError("ACTIVE_RIDE_EXISTS", "You already have an active ride. Complete it before starting a new one.", 409);
  }
```

After the transaction inside `startRide` succeeds, set:

```ts
  await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(uid).update({ currentlyStartedRideId: rideId });
```

In `rideCompletion.util.ts`, after the ride is set to COMPLETED/CANCELLED, clear it:

```ts
  await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(driverId).update({ currentlyStartedRideId: null });
```

- [ ] **Step 4: Run tests — pass**

- [ ] **Step 5: Commit**

```bash
git add CarpoolBackend/functions/src/functions/rides/startRide.fn.ts CarpoolBackend/functions/src/functions/utils/rideCompletion.util.ts CarpoolBackend/functions/tests/
git commit -m "perf(rides): O(1) active-ride check via currentlyStartedRideId (BUG-13)"
```

---

## Task 5.4: BUG-14 — `onRideComplete` no-op early return

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/onRideComplete.fn.ts`

- [ ] **Step 1: Add early return**

After the `sharesSnap` empty check, the batch already does work. Add an early return AFTER the active-shares query so retries are cheap:

```ts
  if (sharesSnap.empty) return null;
  // No further idempotency needed — query already filters status === ACTIVE,
  // and the update sets status === REVOKED which the query excludes.
  // Note: the RTDB removes are idempotent (remove of non-existent node is fine).
```

(No code change beyond what's there — confirm and add a comment.)

- [ ] **Step 2: Add a regression test**

```ts
it("BUG-14 — re-running the trigger does not mutate already-revoked shares", async () => {
  const before = await testHelpers.createCompletedRideWithRevokedShare();
  const ts = before.revokedAt;
  await onRideCompleteHandler(before.change, { params: { rideId: before.rideId } } as any);
  const after = await db.collection("rideShares").doc(before.shareId).get();
  expect(after.data()!.revokedAt).toEqual(ts);
});
```

- [ ] **Step 3: Run — pass**

- [ ] **Step 4: Commit**

```bash
git add CarpoolBackend/functions/src/functions/safety/onRideComplete.fn.ts CarpoolBackend/functions/tests/
git commit -m "fix(safety): early-return guard in onRideComplete trigger (BUG-14)"
```

---

## Task 5.5: BUG-15 — confirm `(b as any).pickupOtp` cast is gone

- [ ] **Step 1: Grep**

```bash
grep -n "(b as any)" CarpoolBackend/functions/src/functions/rides/startRide.fn.ts
```

Expected: empty (Phase 1 removed it). If not, replace with `b.pickupOtp` (the `Booking` type already has it).

- [ ] **Step 2: Commit if needed**

```bash
git add CarpoolBackend/functions/src/functions/rides/startRide.fn.ts
git commit -m "cleanup: drop any-cast on Booking.pickupOtp (BUG-15)"
```

---

## Task 5.6: Deploy Phase 5

```bash
cd CarpoolBackend
firebase deploy --only "functions:triggerSos,functions:completeRide,functions:cancelRide,functions:startRide,functions:onRideComplete,functions:cleanupQueueSweeper"
```

Mark BUG-11, BUG-12, BUG-13, BUG-14, BUG-15 ✅.

---

# PHASE 6 — Frontend perf / battery (driver) (EFF-01..05, UX-D-08, UX-D-09)

**Goal:** Halve foreground GPS power draw, remove redundant decoder calls per frame, and stop unnecessary re-renders on the live-ride map.

**Files:**
- Modify: `Carpool/app/live-ride.tsx`
- Modify: `Carpool/src/services/location/locationCoreService.ts` (add a "marker" sink type if missing)
- Modify: `Carpool/components/LiveRideBanner.tsx` (LastUpdated label)
- Modify: `Carpool/hooks/usePassengerLocation.ts` (debounce)

---

## Task 6.1: EFF-01 — single GPS subscription via location-core sink

- [ ] **Step 1: Read current driver-side watcher**

Open `Carpool/app/live-ride.tsx` lines 393-406 (the inline `ExpoLocation.watchPositionAsync` for the marker). Note the callback.

- [ ] **Step 2: Add a `marker-driver.sink.ts` in `Carpool/src/services/location/sinks/`**

Create the file:

```ts
import { LocationSink, LocationSample } from "../types";

export class MarkerDriverSink implements LocationSink {
  id = "marker-driver";
  constructor(private onSample: (s: { lat: number; lng: number; heading: number | null }) => void) {}
  onSample(sample: LocationSample) {
    this.onSample({ lat: sample.lat, lng: sample.lng, heading: sample.heading });
  }
  onStop() {}
}
```

(Match the existing sink interface in the codebase — grep `live-tracking.sink.ts` for shape.)

- [ ] **Step 3: Replace the inline `watchPositionAsync` in `live-ride.tsx`**

Delete the `useEffect` containing `ExpoLocation.watchPositionAsync(...)` and replace with sink registration:

```ts
useEffect(() => {
  if (!isDriver) return;
  const sink = new MarkerDriverSink(({ lat, lng, heading }) => {
    setUserLocation({ latitude: lat, longitude: lng, heading });
  });
  locationCoreService.registerSink(sink);
  return () => locationCoreService.unregisterSink(sink.id);
}, [isDriver]);
```

- [ ] **Step 4: Verify only one foreground watcher is active in dev**

Run: `cd Carpool && npm run start`
On the driver live-ride screen, check the system service indicators — only one foreground GPS service. (On Android: check the persistent notification.)

- [ ] **Step 5: Commit**

```bash
git add Carpool/src/services/location/sinks/marker-driver.sink.ts Carpool/app/live-ride.tsx
git commit -m "perf(live-ride): single GPS subscription via core-service sink (EFF-01)"
```

---

## Task 6.2: EFF-02 — `useMemo` the polyline decode

- [ ] **Step 1: Find the inline decode**

Grep `decodePolyline` in `live-ride.tsx`. The `activePolyline` ternary is around the reroute path.

- [ ] **Step 2: Wrap in `useMemo`**

```ts
const reroutedCoords = useMemo(
  () => (reroutedPolyline ? decodePolyline(reroutedPolyline) : null),
  [reroutedPolyline]
);
const activePolyline = reroutedCoords ?? routeCoordinates;
```

- [ ] **Step 3: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "perf(live-ride): memoize polyline decode (EFF-02)"
```

---

## Task 6.3: EFF-03 — heading-change threshold for `rotationAnim`

- [ ] **Step 1: Find `DriverMarker`**

Grep `rotationAnim\|Animated.timing` in `Carpool/components/`.

- [ ] **Step 2: Only animate when heading changes by ≥ 5°**

In the `useEffect` that calls `Animated.timing(rotationAnim, ...)`, gate on threshold:

```ts
useEffect(() => {
  if (heading == null) return;
  const prev = lastHeadingRef.current;
  if (prev != null && Math.abs(heading - prev) < 5) return;
  lastHeadingRef.current = heading;
  Animated.timing(rotationAnim, { toValue: heading, duration: 200, useNativeDriver: true }).start();
}, [heading, rotationAnim]);
```

Same for coordinate animation — only animate when displacement > ~5m.

- [ ] **Step 3: Commit**

```bash
git add Carpool/components/
git commit -m "perf(live-ride): threshold heading animation (EFF-03)"
```

---

## Task 6.4: EFF-04 — debounce `setPassengerLocations`

- [ ] **Step 1: Find the listener registration**

Grep `onPassengerLocations\|setPassengerLocations` in `live-ride.tsx`.

- [ ] **Step 2: Wrap with `useDebounce` (1-second trailing)**

```ts
const setPassengerLocationsDebounced = useDebounce(setPassengerLocations, 1000);
// in the listener:
realtimeService.onPassengerLocations(rideId, (locs) => setPassengerLocationsDebounced(locs));
```

(`useDebounce` already exists at `Carpool/hooks/useDebounce.ts`.)

- [ ] **Step 3: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "perf(live-ride): debounce passenger-location RTDB writes (EFF-04)"
```

---

## Task 6.5: EFF-05 — don't kill follow-mode on sheet-drag

- [ ] **Step 1: Distinguish `onPanDrag` from the BottomSheet vs the map**

Grep `onPanDrag` in `live-ride.tsx`. The current handler is on the MapView. The sheet's pan gesture is on the sheet itself.

- [ ] **Step 2: Track a `userIsActivelyDraggingSheet` flag from the BottomSheet's `onAnimate` hook**

```ts
const [sheetIsAnimating, setSheetIsAnimating] = useState(false);
<BottomSheet onAnimate={(from, to) => setSheetIsAnimating(true)} onChange={() => setSheetIsAnimating(false)} ... />
```

- [ ] **Step 3: Gate the follow-mode kill**

```ts
onPanDrag={() => { if (!sheetIsAnimating) setFollowDriver(false); }}
```

- [ ] **Step 4: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "fix(live-ride): keep follow-mode when sheet is animating (EFF-05)"
```

---

## Task 6.6: UX-D-08 — derive last-updated text without `setInterval`

- [ ] **Step 1: Find `LastUpdatedText` component**

Grep `LastUpdated\|setInterval` in `Carpool/components/LiveRideBanner.tsx`.

- [ ] **Step 2: Replace with `useMemo`-based label + a 30s ticker only when stale**

```ts
function LastUpdatedText({ lastUpdate }: { lastUpdate: number | null }) {
  const [, force] = useReducer((x: number) => x + 1, 0);
  useEffect(() => {
    if (lastUpdate == null) return;
    const age = Date.now() - lastUpdate;
    const next = age < 60_000 ? 30_000 : 60_000;
    const id = setTimeout(force, next);
    return () => clearTimeout(id);
  }, [lastUpdate]);
  return <Text>{formatRelative(lastUpdate)}</Text>;
}
```

(One trailing timer per update, not a recurring 5s loop.)

- [ ] **Step 3: Commit**

```bash
git add Carpool/components/LiveRideBanner.tsx
git commit -m "perf(live-ride): on-demand last-updated label (UX-D-08)"
```

---

## Task 6.7: UX-D-09 — `useMemo` `pendingPickups`, `needsPassengerTracking`, `allPickedUp`

- [ ] **Step 1: Wrap each derived value**

In `live-ride.tsx`:

```ts
const pendingPickups = useMemo(
  () => orderedStops.filter((s) => s.status === "PENDING"),
  [orderedStops]
);
const allPickedUp = useMemo(
  () => orderedStops.every((s) => s.status !== "PENDING"),
  [orderedStops]
);
const needsPassengerTracking = useMemo(
  () => pendingPickups.length > 0,
  [pendingPickups]
);
```

- [ ] **Step 2: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "perf(live-ride): memoize derived pickup state (UX-D-09)"
```

---

# PHASE 7 — Multi-passenger OTP redesign (driver) + QR scan + phone-digit fallback

**Goal:** Make the driver flow contextual — "Pick up X (1 of N)" instead of "type any OTP". Add a QR-scan path (zero typing, zero shoulder surfing) and a phone-last-4 fallback for "I lost my OTP".

**New CFs:**
- `verifyPassengerOtpByQr` — validates a short-lived signed token from the passenger app.
- `verifyPassengerByPhoneDigits` — fallback: requires the booking's phone last-4.

**New screens / components:**
- `Carpool/components/PickupHero.tsx` — the contextual OTP card with three states (A/B/C from frontend.md Part 7).
- `Carpool/components/QrScanModal.tsx` — Expo Camera-based scanner.
- `Carpool/screens/PassengerOtpCard.tsx` (or update existing): toggle to show QR.

---

## Task 7.1: Sign + render passenger QR

**Files:**
- Modify: `Carpool/app/booked-ride-details.tsx` (or `ride-details.tsx`) — add QR toggle.
- Create: `Carpool/components/PickupOtpCard.tsx` — wraps numeric + QR view.
- Add `react-native-qrcode-svg` (or similar) to `Carpool/package.json`.

**QR payload design:**
- Client-side signed: payload `{ bookingId, otp, exp }` signed locally with HMAC over `otp + bookingId + exp` — but key is on server, so client cannot HMAC.
- Simpler: payload is **just** `{ bookingId, otp, exp: Date.now() + 60_000 }`. The CF re-validates `pickupOtp` against Firestore. (No client-side signing — the passenger's app can't forge an OTP they don't own.)
- Encode as base64 JSON: `qrPayload = btoa(JSON.stringify({ b: bookingId, o: otp, e: exp }))`.

- [ ] **Step 1: Install QR lib**

Run: `cd Carpool && npx expo install react-native-qrcode-svg react-native-svg`

- [ ] **Step 2: Create `PickupOtpCard.tsx`**

```tsx
import QRCode from "react-native-qrcode-svg";

export function PickupOtpCard({ otp, bookingId }: { otp: string; bookingId: string }) {
  const [mode, setMode] = useState<"digits" | "qr">("digits");
  const qrPayload = useMemo(
    () => Buffer.from(JSON.stringify({ b: bookingId, o: otp, e: Date.now() + 60_000 })).toString("base64"),
    [otp, bookingId, Math.floor(Date.now() / 30_000)],
  );
  return (
    <Card>
      <Title>Show this to your driver</Title>
      {mode === "digits" ? (
        <BigOtp digits={otp} />
      ) : (
        <QRCode value={qrPayload} size={240} />
      )}
      <Button onPress={() => setMode((m) => m === "digits" ? "qr" : "digits")}>
        {mode === "digits" ? "Show QR instead" : "Show digits"}
      </Button>
    </Card>
  );
}
```

(QR payload regenerates every 30s — the wall-clock dep refreshes the memo.)

- [ ] **Step 3: Wire into the passenger view**

Replace existing OTP display in `Carpool/app/booked-ride-details.tsx` with `<PickupOtpCard otp={booking.pickupOtp} bookingId={booking.bookingId} />`.

- [ ] **Step 4: Commit**

```bash
git add Carpool/package.json Carpool/components/PickupOtpCard.tsx Carpool/app/booked-ride-details.tsx
git commit -m "feat(otp): passenger QR option for pickup verification"
```

---

## Task 7.2: CF `verifyPassengerOtpByQr`

**Files:**
- Create: `CarpoolBackend/functions/src/functions/rides/verifyPassengerOtpByQr.fn.ts`
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts`
- Modify: `CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts`
- Modify: `CarpoolBackend/functions/src/config/constants.ts`

- [ ] **Step 1: Add schema + rate-limit + constant**

```ts
// validation.middleware.ts
verifyPassengerOtpByQr: z.object({
  rideId: z.string().min(1),
  payload: z.string().min(8).max(512),    // base64-encoded JSON
}),

// rate-limit.middleware.ts
verifyPassengerOtpByQr: createRateLimiter("VERIFY_PASSENGER_OTP_BY_QR"),

// constants.ts
VERIFY_PASSENGER_OTP_BY_QR: 50,
```

- [ ] **Step 2: Implement the CF**

```ts
export async function handleVerifyPassengerOtpByQr(
  data: { rideId: string; payload: string },
  uid: string
) {
  let parsed: { b: string; o: string; e: number };
  try {
    parsed = JSON.parse(Buffer.from(data.payload, "base64").toString("utf8"));
  } catch {
    throw Errors.INVALID_INPUT({ message: "Malformed QR payload" });
  }
  const { b: bookingId, o: otp, e: exp } = parsed;
  if (Date.now() > exp) throw Errors.INVALID_INPUT({ message: "QR code expired — refresh it" });
  // From here, identical to verifyPassengerOtpHandler with a known bookingId — but we still
  // cross-check rideId + status + ownership + OTP via the same transaction.
  return verifyByBookingIdAndOtp({ rideId: data.rideId, bookingId, otp }, uid);
}
```

Refactor `verifyPassengerOtp.fn.ts` to expose `verifyByBookingIdAndOtp` as shared logic between the digits path and the QR path.

- [ ] **Step 3: Test + commit**

Tests: same as `verifyPassengerOtp`, plus:
- Stale payload (`exp < now`) → INVALID_INPUT.
- Tampered payload → INVALID_INPUT.
- Valid payload for a no-longer-CONFIRMED booking → INVALID_INPUT.

```bash
git add CarpoolBackend/
git commit -m "feat(otp): CF verifyPassengerOtpByQr (Part 7)"
```

---

## Task 7.3: Driver QR scan flow

**Files:**
- Create: `Carpool/components/QrScanModal.tsx`
- Modify: `Carpool/app/live-ride.tsx` — open scan modal from pickup card.

- [ ] **Step 1: Install camera**

Run: `cd Carpool && npx expo install expo-camera`

- [ ] **Step 2: Implement scan modal**

```tsx
import { CameraView } from "expo-camera";

export function QrScanModal({ rideId, onSuccess, onClose }: Props) {
  const [scanned, setScanned] = useState(false);
  return (
    <Modal visible animationType="slide">
      <CameraView
        barcodeScannerSettings={{ barcodeTypes: ["qr"] }}
        onBarcodeScanned={async ({ data }) => {
          if (scanned) return;
          setScanned(true);
          try {
            const res = await verifyPassengerOtpByQr(rideId, data);
            onSuccess(res);
          } catch (e) {
            showError(e);
            setScanned(false);
          }
        }}
      />
      <Button onPress={onClose}>Cancel</Button>
    </Modal>
  );
}
```

- [ ] **Step 3: Wire into live-ride PickupHero**

Add "Scan QR" button → opens `QrScanModal` → on success advance pickup queue.

- [ ] **Step 4: Commit**

```bash
git add Carpool/components/QrScanModal.tsx Carpool/app/live-ride.tsx Carpool/package.json
git commit -m "feat(otp): driver QR scan flow for pickup verification"
```

---

## Task 7.4: Phone-last-4 fallback CF

**Files:**
- Create: `CarpoolBackend/functions/src/functions/rides/verifyPassengerByPhoneDigits.fn.ts`
- Add: schema, rate-limit, constant.

**Design**: caller passes `{ rideId, bookingId, phoneLast4 }`. The CF loads the booking, ensures `booking.passenger.phone` ends with `phoneLast4`, and runs the same transaction as OTP verify. Server logs each fallback for audit. Rate-limited at 10/hour/driver.

- [ ] **Step 1: Constants + schema**

```ts
VERIFY_PASSENGER_BY_PHONE_DIGITS: 10,

verifyPassengerByPhoneDigits: z.object({
  rideId: z.string().min(1),
  bookingId: z.string().min(1),
  phoneLast4: z.string().regex(/^\d{4}$/),
}),
```

- [ ] **Step 2: Implement**

Follow the `verifyPassengerOtp` shape; replace OTP check with `endsWith(phoneLast4)`. Mark `joinedAt + fallbackVerified: true` in the booking update so audit knows.

- [ ] **Step 3: Wire into PickupHero — only after 5 wrong OTP attempts OR explicit "lost OTP" tap**

The hero card surfaces a "Verify by phone digits" link only after a `pickupAttempts > 0` signal from the backend. Drivers don't see it by default — discourages misuse.

- [ ] **Step 4: Commit**

```bash
git add CarpoolBackend/ Carpool/
git commit -m "feat(otp): phone-last-4 fallback for lost OTP (Part 7)"
```

---

## Task 7.5: PickupHero (one-card-per-passenger) + queue chips

**Files:**
- Create: `Carpool/components/PickupHero.tsx`
- Modify: `Carpool/app/live-ride.tsx`

- [ ] **Step 1: Create `PickupHero.tsx`**

States A/B/C per `frontend.md` Part 7. Single source of truth for "who's the current passenger" — derived from `pendingPickups[0]`.

Skeleton:

```tsx
export function PickupHero({ ride, pickupStops, onVerifyOtp, onScanQr, onNoShow }: Props) {
  const orderedPending = pickupStops.filter((s) => s.status === "PENDING").sort((a, b) => a.order - b.order);
  const current = orderedPending[0];
  if (!current) {
    // State C — all done
    return <AllAboardCard ride={ride} />;
  }
  const total = pickupStops.length;
  const idx = total - orderedPending.length + 1;
  return (
    <Card>
      <Title>{current.passengerName} wants to board · {idx} of {total}</Title>
      <OtpBoxes length={6} onComplete={(otp) => onVerifyOtp(otp, current.passengerId)} />
      <Row>
        <ActionButton onPress={onScanQr}>📷 Scan QR</ActionButton>
        <ActionButton onPress={() => onNoShow(current.passengerId)} variant="warning">⚠️ Mark no-show</ActionButton>
      </Row>
    </Card>
  );
}
```

- [ ] **Step 2: Replace existing OTP card in `live-ride.tsx`**

Delete the standalone OTP card. Render `<PickupHero ... />` instead. Wire its props to existing CF calls.

- [ ] **Step 3: Verify in dev**

Multi-passenger ride: 3 confirmed bookings. Driver sees Riya (1/3) → verify → Arjun (2/3) → verify → Nikhil (3/3) → verify → "All aboard" card. No more "type any OTP" ambiguity.

- [ ] **Step 4: Commit**

```bash
git add Carpool/components/PickupHero.tsx Carpool/app/live-ride.tsx
git commit -m "feat(ux): one-card-per-passenger PickupHero (UX-D-02)"
```

---

## Task 7.6: Deploy + smoke Phase 7

```bash
cd CarpoolBackend
firebase deploy --only "functions:verifyPassengerOtp,functions:verifyPassengerOtpByQr,functions:verifyPassengerByPhoneDigits,functions:markPassengerNoShow"
```

Build the app (`eas build` or local dev) and smoke-test all three verification paths.

Mark UX-D-02, UX-D-05, Part 7 ✅.

---

# PHASE 8 — Driver + passenger layout cleanup (UX-D-01..10, UX-P-01..06)

**Goal:** Strip the screen of competing chrome. Map dominates, one primary CTA at a time.

---

## Task 8.1: Driver — collapse FABs into a single secondary actions row

**Files:**
- Modify: `Carpool/app/live-ride.tsx`
- New: `Carpool/components/SecondaryActionsBar.tsx`

- [ ] **Step 1: Define the bar**

```tsx
export function SecondaryActionsBar({ onOpenMaps, onShareLive, onSos, sosEligible }: Props) {
  return (
    <View style={styles.row}>
      <SecondaryButton icon="map" onPress={onOpenMaps}>Open Maps Nav</SecondaryButton>
      <SecondaryButton icon="share" onPress={onShareLive}>Share</SecondaryButton>
      <View style={styles.sosSeparator} />
      {sosEligible && <SosButton onPress={onSos} variant="row" />}
    </View>
  );
}
```

SOS gets a dedicated typographic + colour treatment + 48dp clear area (per Part 6.4).

- [ ] **Step 2: Replace FABs in `live-ride.tsx`**

Delete the three floating buttons (My-Location FAB, Re-center FAB, Share pill). Render `<SecondaryActionsBar ... />` inside the BottomSheet, below the PickupHero. The "My-location/Re-center" become a single inline action surfaced only when the camera has drifted.

- [ ] **Step 3: Visual verification**

Run the app, confirm:
- No floating chip stack.
- Map is the largest visible area.
- One CTA card at a time.
- SOS distinct from share.

- [ ] **Step 4: Commit**

```bash
git add Carpool/app/live-ride.tsx Carpool/components/SecondaryActionsBar.tsx
git commit -m "feat(ux): unify driver secondary actions into one row (UX-D-01, UX-D-07)"
```

---

## Task 8.2: Driver — Complete-Ride moves into sheet + needs guard

**Files:**
- Modify: `Carpool/app/live-ride.tsx`

- [ ] **Step 1: Move Complete-Ride button out of always-visible footer**

Only render when the sheet is expanded (snap index ≥ 1). Also gate on `allPickedUp`:

```tsx
{sheetExpanded && allPickedUp && <CompleteRideButton ... />}
{sheetExpanded && !allPickedUp && (
  <Notice>
    {pendingPickups.length} passenger(s) haven't boarded. Pick them up or mark them no-show first.
  </Notice>
)}
```

- [ ] **Step 2: Confirm dialog says "Confirm — 3 passengers, 22 km logged"**

Replace the generic confirm:

```ts
Alert.alert(
  "Complete ride?",
  `${passengersOnBoard} passengers on board · ${km} km · ${minutes} min logged.`,
  [{ text: "Cancel" }, { text: "Complete", style: "destructive", onPress: onConfirm }]
);
```

- [ ] **Step 3: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "fix(ux): gate Complete Ride behind sheet-expand + summary confirm (UX-D-03)"
```

---

## Task 8.3: Driver — "All aboard" persistent banner + sheet collapses to 10%

**Files:**
- Modify: `Carpool/app/live-ride.tsx`

- [ ] **Step 1: When `allPickedUp` flips true, slam sheet to 10% snap point**

```ts
useEffect(() => {
  if (allPickedUp && previousAllPickedUp !== allPickedUp) {
    bottomSheetRef.current?.snapToIndex(0);   // 10%
  }
}, [allPickedUp]);
```

- [ ] **Step 2: Render top-of-map "ALL ABOARD — driving to <city>" card**

A single thin card that replaces the PickupHero. Anchored at the top of the map area, not floating.

- [ ] **Step 3: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "feat(ux): all-aboard transition state for driver (UX-D-04)"
```

---

## Task 8.4: Driver — quieter reroute indicator (UX-D-06)

**Files:**
- Modify: `Carpool/components/LiveRideBanner.tsx` (or wherever rerouting renders)

- [ ] **Step 1: When `isRerouting`, render a banded warning bar across the top of the map (not buried text)**

```tsx
{isRerouting && (
  <View style={styles.rerouteBanner}>
    <Spinner />
    <Text>Recalculating route…</Text>
  </View>
)}
```

Persistent for the duration. Polyline colour change is supplementary.

- [ ] **Step 2: Commit**

```bash
git add Carpool/components/
git commit -m "fix(ux): louder reroute indicator (UX-D-06)"
```

---

## Task 8.5: Driver — sheet snap points respect mapPadding

**Files:**
- Modify: `Carpool/app/live-ride.tsx`

- [ ] **Step 1: Derive `mapPadding.bottom` from current sheet height**

```ts
const sheetHeightFraction = useSharedValue(0.1);
const mapPaddingBottom = useDerivedValue(() => screenHeight * sheetHeightFraction.value);

<BottomSheet onAnimate={(_, to) => (sheetHeightFraction.value = snapPoints[to])} ... />
<MapView mapPadding={{ ..., bottom: mapPaddingBottom.value }} />
```

- [ ] **Step 2: When sheet expands past 50%, animate camera up so passenger markers stay visible**

```ts
useEffect(() => {
  if (sheetHeightFraction.value > 0.5 && followDriver) {
    cameraRef.current?.animateCameraTo({ heading: 0, pitch: 0, zoom: currentZoom - 0.5 });
  }
}, [sheetHeightFraction.value]);
```

- [ ] **Step 3: Commit**

```bash
git add Carpool/app/live-ride.tsx
git commit -m "fix(ux): map respects sheet height (UX-D-10)"
```

---

## Task 8.6: Passenger — OTP as hero, replace card after pickup (UX-P-01, UX-P-02, UX-P-03, UX-P-06)

**Files:**
- Modify: `Carpool/app/live-ride.tsx` (passenger branch)
- Use: `PickupOtpCard` from Task 7.1

- [ ] **Step 1: In the passenger view, render `PickupOtpCard` as the FIRST element of the BottomSheet content**

It is the largest text on screen, top of the sheet, always above the fold. Compress driver info to a single line above.

- [ ] **Step 2: When `isPassengerPickedUp` flips true (RTDB), replace the hero with an on-board hero**

```tsx
{isPassengerPickedUp ? <OnBoardHero ... /> : <PickupOtpCard ... />}
```

The on-board state IS the celebration — no transient 3-second banner.

- [ ] **Step 3: Single sharing-status surface**

Delete the `sharingInfoBar` at the top of the sheet. Keep the `sharingBadge` inline with the on-board hero. Add a one-tap "Stop sharing" affordance on first opt-in (modal on initial location request).

- [ ] **Step 4: Commit**

```bash
git add Carpool/
git commit -m "feat(ux): passenger OTP-as-hero + on-board state (UX-P-01..06)"
```

---

## Task 8.7: Passenger — call-button gated by status (UX-P-04) + distance as "min walk" (UX-P-05)

- [ ] **Step 1: Gate**

```tsx
{ride.status === "STARTED" && booking.status === "CONFIRMED" && <CallButton ... />}
```

- [ ] **Step 2: Distance helper**

Add `Carpool/src/utils/walkTime.ts`:

```ts
const WALK_MPS = 1.4;  // 5 km/h
export function distanceToWalkLabel(metres: number): string {
  if (metres < 30) return "You're here";
  const mins = Math.max(1, Math.round(metres / WALK_MPS / 60));
  return `${mins} min walk · ${Math.round(metres)} m`;
}
```

Replace bare "800m" with the labelled version.

- [ ] **Step 3: Commit**

```bash
git add Carpool/
git commit -m "fix(ux): call-button gating + walk-time label (UX-P-04, UX-P-05)"
```

---

# PHASE 9 — SOS UX redesign (UX-S-01..05)

**Goal:** SOS countdown keeps the user in context; emergency-call shortcut always one tap away; Reduce Motion no longer bypasses the safety window.

---

## Task 9.1: Replace `SosCountdownModal` full-screen with bottom-sheet

**Files:**
- Modify: `Carpool/components/SosCountdownModal.tsx`

- [ ] **Step 1: Convert to a 75%-height sheet**

Top 25% of the screen shows the map underneath (snapshot it from the parent screen via `Portal`-like pattern, or just leave transparent so the live map remains visible).

```tsx
<BottomSheet snapPoints={["75%"]} enablePanDownToClose={false}>
  <View style={styles.contents}>
    <CallEmergencyImmediateButton onPress={onCallNow} />
    <CountdownNumber seconds={remaining} />
    <CancelButton onPress={onCancel} />
    <Text>5 contacts will be notified · {emergencyNumber} visible to viewers</Text>
  </View>
</BottomSheet>
```

- [ ] **Step 2: Add "Call emergency now" button — dials immediately**

Skip the 5-second wait. For the user who is in true danger AND can speak.

- [ ] **Step 3: Commit**

```bash
git add Carpool/components/SosCountdownModal.tsx
git commit -m "feat(sos): countdown as bottom-sheet w/ call-now (UX-S-01)"
```

---

## Task 9.2: Fix Reduce Motion (UX-S-03)

**Files:**
- Modify: `Carpool/components/SosCountdownModal.tsx:41`

- [ ] **Step 1: When `reduceMotion`, show a "Send SOS" pulsing button — don't auto-fire**

Replace the auto-fire branch with:

```tsx
{reduceMotion ? (
  <PrimaryButton onPress={onConfirmSendNow} pulsing>
    Send SOS now
  </PrimaryButton>
) : (
  <CountdownNumber seconds={remaining} />
)}
```

Both code paths show the same Cancel + "Call emergency now" buttons. Reduce Motion users get the safety window back.

- [ ] **Step 2: Commit**

```bash
git add Carpool/components/SosCountdownModal.tsx
git commit -m "fix(sos,a11y): preserve safety window under Reduce Motion (UX-S-03)"
```

---

## Task 9.3: `SosActiveBanner` with inline actions (UX-S-02, UX-S-04)

**Files:**
- Modify: `Carpool/components/SosActiveBanner.tsx`

- [ ] **Step 1: Add three inline action chips**

```tsx
<Banner>
  <SosStatus />
  <ChipRow>
    <Chip icon="phone" onPress={() => Linking.openURL(`tel:${emergencyNumber}`)}>Emergency</Chip>
    <Chip icon="phone" onPress={() => Linking.openURL(`tel:${driverPhone}`)}>Driver</Chip>
    <Chip icon="link" onPress={() => Share.share({ url: trackingUrl })}>Tracking</Chip>
  </ChipRow>
  <Actions>
    <HoldButton holdMs={1000} onComplete={onSafe}>I'm Safe</HoldButton>
    <Button onPress={onFalseAlarm}>False Alarm</Button>
  </Actions>
</Banner>
```

- [ ] **Step 2: Pin banner to top of viewport (above BottomSheet)**

Use a `<Portal>` or absolute-positioned overlay so the banner is the topmost element, not scrollable away.

- [ ] **Step 3: Always render `SosButton` even when active — but recolour to "SOS active" state**

Drop the `sosEligible && !sosActive` gate. Allow re-tap to refresh location / add info.

- [ ] **Step 4: Commit**

```bash
git add Carpool/components/SosActiveBanner.tsx Carpool/components/SosButton.tsx Carpool/app/live-ride.tsx
git commit -m "feat(sos): inline emergency/driver/tracking actions; banner pinned (UX-S-02, UX-S-04)"
```

---

## Task 9.4: Hold-to-confirm "I'm Safe" (UX-S-04 polish)

Already in Task 9.3 step 1 via `HoldButton`. Implement the component if missing.

- [ ] **Step 1: Build `HoldButton.tsx`**

```tsx
export function HoldButton({ holdMs, onComplete, children }: Props) {
  const [progress, setProgress] = useState(0);
  const start = useRef<number | null>(null);
  const interval = useRef<NodeJS.Timer | null>(null);
  const begin = () => {
    start.current = Date.now();
    interval.current = setInterval(() => {
      const elapsed = Date.now() - (start.current ?? 0);
      setProgress(Math.min(1, elapsed / holdMs));
      if (elapsed >= holdMs) {
        clearInterval(interval.current!);
        onComplete();
      }
    }, 30);
  };
  const cancel = () => {
    if (interval.current) clearInterval(interval.current);
    setProgress(0);
  };
  return (
    <Pressable onPressIn={begin} onPressOut={cancel}>
      <ProgressBar value={progress} />
      <Text>{children}</Text>
    </Pressable>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add Carpool/components/HoldButton.tsx
git commit -m "feat(ui): HoldButton for safety-critical confirmations"
```

---

# PHASE 10 — Heartbeat scaling, RTDB collapse, COST-03/04 polish (COST-01, COST-02, COST-03, COST-04)

---

## Task 10.1: COST-02 — distance-scaled passenger heartbeat

**Files:**
- Modify: `Carpool/hooks/usePassengerLocation.ts:30, 102`

- [ ] **Step 1: Replace constant interval with distance-tier**

```ts
function chooseInterval(distanceToPickupM: number | null): number {
  if (distanceToPickupM == null) return 5 * 60_000;
  if (distanceToPickupM > 5000) return 15 * 60_000;
  if (distanceToPickupM > 1000) return 5 * 60_000;
  if (distanceToPickupM > 500)  return 2 * 60_000;
  return 60_000;
}

useEffect(() => {
  const id = setInterval(forceWrite, chooseInterval(distanceToPickupM));
  return () => clearInterval(id);
}, [distanceToPickupM]);
```

- [ ] **Step 2: Skip heartbeat entirely if `distanceToPickupM > CLOSE_THRESHOLD_M` AND last passive write was < 30 min ago**

```ts
if (distanceToPickupM != null && distanceToPickupM > 5000 && msSinceLastWrite < 30 * 60_000) return;
```

- [ ] **Step 3: Commit**

```bash
git add Carpool/hooks/usePassengerLocation.ts
git commit -m "perf(location): distance-scaled passenger heartbeat (COST-02)"
```

---

## Task 10.2: COST-03 — guard double `onStop` in `live-tracking.sink.ts`

- [ ] **Step 1: Add a `stopped` flag**

```ts
private stopped = false;
detach() {
  if (this.stopped) return;
  this.stopped = true;
  this.unsubscribeStatus?.();
}
onSample(sample) {
  if (this.stopped) return;
  ...
}
```

- [ ] **Step 2: Commit**

```bash
git add Carpool/src/services/location/sinks/live-tracking.sink.ts
git commit -m "fix(location): guard double-stop in live-tracking sink (COST-03)"
```

---

## Task 10.3: COST-04 — write `STOPPED` status before deletion in revoke + scheduler

Already covered in Task 3.2 for `revokeRideShare`. Replicate for the scheduled cleanup:

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/scheduled/rideShareCleanup.fn.ts`

- [ ] **Step 1: Before `remove()`, write status:STOPPED**

```ts
await rtdb.ref(`liveTracking/${token}`).update({ status: "STOPPED", stoppedAt: Date.now() });
await rtdb.ref(`liveTracking/${token}`).remove();
```

- [ ] **Step 2: Add a test**

```ts
it("COST-04 — expired shares emit STOPPED before deletion", async () => {
  const events: any[] = [];
  await rtdb.ref(`liveTracking/tok1`).set({ updatedAt: Date.now() });
  rtdb.ref(`liveTracking/tok1/status`).on("value", (s) => events.push(s.val()));
  await rideShareCleanupHandler();
  expect(events).toContain("STOPPED");
});
```

- [ ] **Step 3: Commit**

```bash
git add CarpoolBackend/functions/src/functions/scheduled/rideShareCleanup.fn.ts CarpoolBackend/functions/tests/
git commit -m "fix(cleanup): emit STOPPED status before RTDB removal (COST-04)"
```

---

## Task 10.4: COST-01 — collapse `activeRides/{rideId}/driverLocation` + `liveTracking/{token}` when share active

**Files:**
- Modify: `Carpool/src/services/location/sinks/driver-ride.sink.ts`
- Modify: `Carpool/src/services/firebase/realtime.service.ts:77-90`

**Approach:** the driver sink, when a share is active, writes to `liveTracking/{shareToken}` instead of `activeRides/{rideId}/driverLocation`. The live-ride screen subscribes to whichever one is active. We keep `activeRides` path for rides without an active share.

This is more invasive than other tasks — schedule LAST and gate it with the existing `ShareLiveRideButton` state.

- [ ] **Step 1: Driver sink toggles destination based on `activeShareToken`**

```ts
class DriverRideSink {
  destPath: string;
  constructor(rideId: string, activeShareToken: string | null) {
    this.destPath = activeShareToken
      ? `liveTracking/${activeShareToken}`
      : `activeRides/${rideId}/driverLocation`;
  }
  async write(sample) { await rtdb.ref(this.destPath).set(sample); }
}
```

When a share becomes active mid-ride, swap the sink's `destPath` and (for safety) write a final tombstone to the old path.

- [ ] **Step 2: Update passenger-side listener to read from BOTH paths and prefer fresher**

Passengers might be on an older client during rollout — read both, pick newer timestamp.

- [ ] **Step 3: Add a feature gate `RTDB_PATH_COLLAPSE=true` in remote config**

So a rollback is one config flip.

- [ ] **Step 4: Smoke + commit**

```bash
git add Carpool/
git commit -m "perf(location): collapse driver location writes during active share (COST-01)"
```

---

## Task 10.5: Deploy Phase 10 + final reconciliation

- [ ] **Step 1: Deploy**

```bash
cd CarpoolBackend
firebase deploy --only "functions:rideShareCleanup,functions:cleanupQueueSweeper,database"
```

- [ ] **Step 2: Run full suite**

```bash
cd CarpoolBackend/functions && npm test -- --runInBand
cd Carpool && npm test -- --runInBand
```

Expected: all tests pass; new tests from each phase are present.

- [ ] **Step 3: Update todo.md final mark**

All BUG-* (except BUG-06 deferred) marked ✅.
All SEC-* marked ✅.
All COST-* marked ✅.
All UX-* marked ✅.

- [ ] **Step 4: Commit**

```bash
git add /Users/sairajaggani/Desktop/Space/Project/todo.md
git commit -m "chore: live-ride hardening pass complete (BUG-06 Twilio deferred to v2)"
```

---

# Out-of-scope / explicitly deferred

- **BUG-06 (Twilio SMS/WhatsApp delivery)** — deferred to v2 per user instruction. Track separately. Until then, SOS WhatsApp delivery remains client-driven and may silently fail if the user is incapacitated — surface this risk explicitly to the user via copy ("WhatsApp opens on your phone — your contacts may not receive the message if you can't interact with your phone").
- **BUG-10 (public tracking page polling)** — per memory: "CF once for metadata + viewer cap → direct RTDB listener for real-time location" is already the architecture (deployed 2026-06-10). Verify quickly with a tcpdump-style log of /track page → CF invocations during a long ride; if confirmed, mark ✅. If not, add a one-task patch.

---

# Self-review summary

| Spec item (backend.md) | Phase / Task | Status in plan |
|---|---|---|
| BUG-01 OTP collision | Phase 1 — Tasks 1.3, 1.5 | ✅ Covered |
| BUG-02 setTimeout in revoke | Phase 3 — Task 3.2 | ✅ |
| BUG-03 idempotency | Phase 3 — Tasks 3.1, 3.3 | ✅ |
| BUG-04 pickupStops RTDB | Phase 2 — Tasks 2.1-2.4 | ✅ |
| BUG-05 FCM fanout | Phase 3 — Task 3.4 | ✅ |
| BUG-06 Twilio | Out-of-scope | ⏭ Deferred per user |
| BUG-07 simCountry/battery | Phase 4 — Task 4.1 | ✅ |
| BUG-08 viewer cap | Phase 4 — Task 4.2 | ✅ |
| BUG-09 rate-limit fail-closed | Phase 4 — Task 4.3 | ✅ |
| BUG-10 polling | Out-of-scope check | ⏭ Verify-only |
| BUG-11 N+1 phone | Phase 5 — Task 5.1 | ✅ |
| BUG-12 RTDB cleanup queue | Phase 5 — Task 5.2 | ✅ |
| BUG-13 currentlyStartedRideId | Phase 5 — Task 5.3 | ✅ |
| BUG-14 onRideComplete guard | Phase 5 — Task 5.4 | ✅ |
| BUG-15 any-cast | Phase 1 — Task 1.3 side-effect; confirm Task 5.5 | ✅ |
| COST-01 collapse RTDB paths | Phase 10 — Task 10.4 | ✅ |
| COST-02 heartbeat scaling | Phase 10 — Task 10.1 | ✅ |
| COST-03 listener double-fire | Phase 10 — Task 10.2 | ✅ |
| COST-04 STOPPED broadcast | Phase 10 — Task 10.3 + (Task 3.2) | ✅ |
| COST-05 OTP query reads | Phase 1 — Task 1.6 (attempt cap reduces queries) | ✅ |
| SEC-01 OTP brute force | Phase 1 — Task 1.6 | ✅ |
| SEC-02 OTP in push body | Phase 1 — Task 1.4 | ✅ |
| SEC-03 liveTracking forever | Phase 4 — Task 4.4 | ✅ |

| Spec item (frontend.md) | Phase / Task | Status |
|---|---|---|
| UX-D-01 7+ HUD overlays | Phase 8 — Task 8.1 | ✅ |
| UX-D-02 single-OTP confusion | Phase 7 — Task 7.5 | ✅ |
| UX-D-03 Complete-Ride always visible | Phase 8 — Task 8.2 | ✅ |
| UX-D-04 destination pop-in | Phase 8 — Task 8.3 | ✅ |
| UX-D-05 no-show buried | Phase 7 — Task 7.5 | ✅ |
| UX-D-06 quiet reroute | Phase 8 — Task 8.4 | ✅ |
| UX-D-07 Share = SOS pill | Phase 8 — Task 8.1 | ✅ |
| UX-D-08 setInterval label | Phase 6 — Task 6.6 | ✅ |
| UX-D-09 inline recompute | Phase 6 — Task 6.7 | ✅ |
| UX-D-10 mapPadding fight | Phase 8 — Task 8.5 | ✅ |
| UX-P-01 OTP not hero | Phase 8 — Task 8.6 | ✅ |
| UX-P-02 "Driver is here!" race | Phase 2 fixes root cause; Phase 8 Task 8.6 fixes UI | ✅ |
| UX-P-03 sharing dual UI | Phase 8 — Task 8.6 | ✅ |
| UX-P-04 call button gate | Phase 8 — Task 8.7 | ✅ |
| UX-P-05 raw metres | Phase 8 — Task 8.7 | ✅ |
| UX-P-06 boarding celebration | Phase 8 — Task 8.6 | ✅ |
| UX-S-01 full-screen red | Phase 9 — Task 9.1 | ✅ |
| UX-S-02 banner shortcuts | Phase 9 — Task 9.3 | ✅ |
| UX-S-03 Reduce Motion | Phase 9 — Task 9.2 | ✅ |
| UX-S-04 SOS hidden when active | Phase 9 — Task 9.3 | ✅ |
| UX-S-05 SOS / Share visual confusion | Phase 8 — Task 8.1 + Phase 9 polish | ✅ |
| EFF-01 dual GPS | Phase 6 — Task 6.1 | ✅ |
| EFF-02 polyline decode | Phase 6 — Task 6.2 | ✅ |
| EFF-03 rotationAnim | Phase 6 — Task 6.3 | ✅ |
| EFF-04 passenger listener debounce | Phase 6 — Task 6.4 | ✅ |
| EFF-05 follow vs sheet drag | Phase 6 — Task 6.5 | ✅ |
| Part 7 — QR option | Phase 7 — Tasks 7.1, 7.2, 7.3 | ✅ |
| Part 7 — phone fallback | Phase 7 — Task 7.4 | ✅ |

**No placeholder steps remain.** Each task has concrete code or file references. Type and method names cross-checked (`__test_generateUniqueOtps`, `handleMarkPassengerNoShow`, `verifyPassengerOtpByQr`, `handleCreateRideShare`, `handleTriggerSos`, `handleRevokeRideShare`).

**Type consistency:** `currentlyStartedRideId` introduced in Phase 5 Task 5.3 is consistent everywhere (`string | null`). `pickupOtp` is `string | undefined` per `types/index.ts:204` (unchanged). `otpAttempts` + `otpRotatedAt` added in Phase 1 Task 1.6 are used in Task 1.6 + Task 5.5 unchanged.

---
