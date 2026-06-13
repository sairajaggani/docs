# Phase G — Live Ride Share Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let any driver or confirmed passenger on a STARTED ride share a live-tracking link via the native share sheet, backed by two Cloud Functions, a Firestore trigger, and a scheduled cleanup.

**Architecture:** `createRideShare` callable CF generates a `shareToken`, writes to `rideShares/{shareToken}` in Firestore + `rideSharesByToken/{shareToken}` index in RTDB, and returns a tracking URL. The FE then starts `liveTrackingService.startTracking(token, 'share')` (Phase B) and opens `Share.share`. `onRideComplete` trigger auto-revokes shares when a ride ends. `useRideShare` hook keeps the toggle button in sync via a Firestore listener.

**Tech Stack:** Firebase Cloud Functions (callable + Firestore trigger + pubsub), Firestore, RTDB, React Native `Share` API, `liveTrackingService` (Phase B), `firestoreService.onCollectionSnapshot`, Zod validation, Jest.

---

## File Map

| File | Status | Responsibility |
|---|---|---|
| `CarpoolBackend/functions/src/config/constants.ts` | Modify | Add `CREATE_RIDE_SHARE`, `REVOKE_RIDE_SHARE` rate limits + `RIDE_SHARE` cap constants |
| `CarpoolBackend/functions/src/middleware/validation.middleware.ts` | Modify | Add `createRideShare` + `revokeRideShare` Zod schemas |
| `CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts` | Modify | Add `createRideShare` + `revokeRideShare` rate limiters |
| `CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts` | Create | Callable CF — idempotent, cap-3, token generation, Firestore + RTDB write |
| `CarpoolBackend/functions/src/functions/safety/revokeRideShare.fn.ts` | Create | Callable CF — owner check, status update, RTDB cleanup |
| `CarpoolBackend/functions/src/functions/safety/onRideComplete.fn.ts` | Create | Firestore onUpdate trigger — auto-revoke shares on COMPLETED/CANCELLED |
| `CarpoolBackend/functions/src/functions/scheduled/rideShareCleanup.fn.ts` | Create | Hourly scheduled — expire stale active shares, delete RTDB nodes |
| `CarpoolBackend/functions/src/functions/safety/index.ts` | Modify | Export createRideShare, revokeRideShare, onRideComplete |
| `CarpoolBackend/functions/src/functions/scheduled/index.ts` | Modify | Export rideShareCleanup |
| `CarpoolBackend/firestore.rules` | Modify | Add rideShares read rule |
| `CarpoolBackend/functions/tests/unit/createRideShare.test.ts` | Create | Unit tests for createRideShare handler |
| `CarpoolBackend/functions/tests/unit/revokeRideShare.test.ts` | Create | Unit tests for revokeRideShare handler |
| `CarpoolBackend/functions/tests/unit/onRideComplete.test.ts` | Create | Unit tests for onRideComplete trigger |
| `Carpool/src/types/models/rideShare.types.ts` | Create | `RideShare` TS type |
| `Carpool/src/types/models/index.ts` | Modify | Export rideShare.types |
| `Carpool/src/services/firebase/functions.service.ts` | Modify | Add `createRideShare` + `revokeRideShare` callable wrappers |
| `Carpool/hooks/useRideShare.ts` | Create | Firestore listener + share/revoke logic |
| `Carpool/components/ShareLiveRideButton.tsx` | Create | Toggle button component (4 states) |
| `Carpool/app/live-ride.tsx` | Modify | Mount ShareLiveRideButton for driver when ride is STARTED |
| `Carpool/app/booked-ride-details.tsx` | Modify | Mount ShareLiveRideButton for passenger when ride is STARTED |

---

## Task 1: Backend constants + validation + rate limits

**Files:**
- Modify: `CarpoolBackend/functions/src/config/constants.ts`
- Modify: `CarpoolBackend/functions/src/middleware/validation.middleware.ts`
- Modify: `CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts`

- [ ] **Step 1: Add rate limits + RIDE_SHARE constants**

In `CarpoolBackend/functions/src/config/constants.ts`, add to `RATE_LIMITS` after `REVOKE_RIDE_SHARE` placeholder (add after the `GET_TRACKING_DATA` line):

```ts
CREATE_RIDE_SHARE: 20,
REVOKE_RIDE_SHARE: 50,
```

Add a `RIDE_SHARE` block in `CONSTANTS` after the `TRACKING` block:

```ts
RIDE_SHARE: {
  MAX_ACTIVE_PER_USER: 3,
  MAX_PER_DAY: 20,
  EXPIRY_MS: 6 * 60 * 60 * 1000,   // 6-hour hard ceiling
  RTDB_CLEANUP_GRACE_MS: 60_000,
  TRACKING_URL_BASE: "https://carpool-app-1e668.web.app/track",
},
```

- [ ] **Step 2: Add Zod validation schemas**

In `CarpoolBackend/functions/src/middleware/validation.middleware.ts`, add after the `removeTrustedContact` schema:

```ts
createRideShare: z.object({
  rideId: z.string().min(1),
}),
revokeRideShare: z.object({
  shareToken: z.string().min(1),
}),
```

- [ ] **Step 3: Add rate limiters**

In `CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts`, add to the `rateLimiters` export object after the trusted contacts section:

```ts
// Safety — Ride Share
createRideShare: createRateLimiter("CREATE_RIDE_SHARE"),
revokeRideShare: createRateLimiter("REVOKE_RIDE_SHARE"),
```

- [ ] **Step 4: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit 2>&1
```

Expected: no errors.

---

## Task 2: `createRideShare` CF — TDD

**Files:**
- Create: `CarpoolBackend/functions/tests/unit/createRideShare.test.ts`
- Create: `CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts`

- [ ] **Step 1: Write failing tests**

```ts
// CarpoolBackend/functions/tests/unit/createRideShare.test.ts
import { describe, it, expect, jest, beforeEach } from "@jest/globals";
import { db, rtdb } from "../../src/config/firebase-admin.js";
import { Timestamp } from "firebase-admin/firestore";

const makeContext = (uid = "user1") => ({
  extendedAuth: { uid, email: "u@test.com", emailVerified: true, phone_number: null },
  rawContext: { auth: { uid } },
});

describe("createRideShare", () => {
  let mockCollectionRef: any;
  let mockQuery: any;
  let mockDocRef: any;
  let mockRtdbRef: any;

  beforeEach(() => {
    jest.clearAllMocks();

    mockDocRef = {
      get: jest.fn(),
      set: jest.fn().mockResolvedValue(undefined),
    };

    mockQuery = {
      where: jest.fn().mockReturnThis(),
      limit: jest.fn().mockReturnThis(),
      get: jest.fn(),
    };

    mockCollectionRef = {
      where: jest.fn().mockReturnValue(mockQuery),
      doc: jest.fn().mockReturnValue(mockDocRef),
    };

    jest.spyOn(db as any, "collection").mockReturnValue(mockCollectionRef);

    mockRtdbRef = { set: jest.fn().mockResolvedValue(undefined) };
    jest.spyOn(rtdb as any, "ref").mockReturnValue(mockRtdbRef);
  });

  // Tests call handleCreateRideShare(data, authContext) directly — same pattern as getTrackingData

  it("returns error if ride not found", async () => {
    const { handleCreateRideShare } = await import(
      "../../src/functions/safety/createRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({ exists: false });
    await expect(
      handleCreateRideShare("r1", "user1")
    ).rejects.toMatchObject({ code: "NOT_FOUND" });
  });

  it("returns error if ride is not STARTED", async () => {
    const { handleCreateRideShare } = await import(
      "../../src/functions/safety/createRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ rideId: "r1", driverId: "other", status: "OPEN", driver: { firstName: "Raj", lastName: "K" }, vehicleDetails: { plate: "MH12", model: "Swift" } }),
    });
    await expect(
      handleCreateRideShare("r1", "other")
    ).rejects.toMatchObject({ code: "INVALID_INPUT" });
  });

  it("returns error if caller is neither driver nor confirmed passenger", async () => {
    const { handleCreateRideShare } = await import(
      "../../src/functions/safety/createRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ rideId: "r1", driverId: "driverX", status: "STARTED", driver: { firstName: "Raj", lastName: "K" }, vehicleDetails: { plate: "MH12", model: "Swift" } }),
    });
    mockQuery.get.mockResolvedValue({ empty: true, docs: [] });
    await expect(
      handleCreateRideShare("r1", "strangerUid")
    ).rejects.toMatchObject({ code: "UNAUTHORIZED" });
  });

  it("returns existing active share idempotently (driver)", async () => {
    const { handleCreateRideShare } = await import(
      "../../src/functions/safety/createRideShare.fn.js"
    );
    const uid = "driver1";
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ rideId: "r1", driverId: uid, status: "STARTED", driver: { firstName: "Raj", lastName: "K" }, vehicleDetails: { plate: "MH12", model: "Swift" } }),
    });
    mockQuery.get.mockResolvedValueOnce({
      empty: false,
      docs: [{ data: () => ({ shareToken: "existingToken", trackingUrl: "https://carpool-app-1e668.web.app/track/existingToken", driverName: "Raj K.", vehiclePlate: "MH12", passengerName: "Raj K.", status: "ACTIVE" }) }],
    });
    const result = await handleCreateRideShare("r1", uid);
    expect(result.data.shareToken).toBe("existingToken");
    expect(mockDocRef.set).not.toHaveBeenCalled();
  });

  it("returns INVALID_INPUT if user already has 3 active shares", async () => {
    const { handleCreateRideShare } = await import(
      "../../src/functions/safety/createRideShare.fn.js"
    );
    const uid = "driver1";
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ rideId: "r1", driverId: uid, status: "STARTED", driver: { firstName: "Raj", lastName: "K" }, vehicleDetails: { plate: "MH12", model: "Swift" } }),
    });
    mockQuery.get
      .mockResolvedValueOnce({ empty: true, docs: [] })
      .mockResolvedValueOnce({ size: 3, docs: [1, 2, 3] });
    await expect(
      handleCreateRideShare("r1", uid)
    ).rejects.toMatchObject({ code: "INVALID_INPUT" });
  });

  it("creates share successfully for driver", async () => {
    const { handleCreateRideShare } = await import(
      "../../src/functions/safety/createRideShare.fn.js"
    );
    const uid = "driver1";
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ rideId: "r1", driverId: uid, status: "STARTED", driver: { firstName: "Rajesh", lastName: "Kumar" }, vehicleDetails: { plate: "MH12AB1234", model: "Swift" } }),
    });
    mockQuery.get
      .mockResolvedValueOnce({ empty: true, docs: [] })
      .mockResolvedValueOnce({ size: 0, docs: [] });
    const result = await handleCreateRideShare("r1", uid);
    expect(result.data.shareToken).toBeDefined();
    expect(result.data.trackingUrl).toContain("/track/");
    expect(result.data.driverName).toBe("Rajesh K.");
    expect(result.data.vehiclePlate).toBe("MH12AB1234");
    expect(mockDocRef.set).toHaveBeenCalledWith(expect.objectContaining({ status: "ACTIVE", rideId: "r1", userId: uid }));
    expect(mockRtdbRef.set).toHaveBeenCalledWith(expect.objectContaining({ userId: uid }));
  });
});
```

- [ ] **Step 2: Run tests — expect module-not-found failure**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx jest tests/unit/createRideShare.test.ts --no-coverage 2>&1 | tail -8
```

Expected: `Cannot find module '../../src/functions/safety/createRideShare.fn.js'`

- [ ] **Step 3: Implement `createRideShare.fn.ts`**

```ts
// CarpoolBackend/functions/src/functions/safety/createRideShare.fn.ts
import * as functions from "firebase-functions";
import { randomBytes } from "crypto";
import { Timestamp } from "firebase-admin/firestore";
import { db, rtdb } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { logger } from "../../utils/logger.js";
import { ResponseFormatter } from "../../utils/response.js";
import { authenticateUser, requireEmailVerification } from "../../middleware/auth.middleware.js";
import { validateData, schemas } from "../../middleware/validation.middleware.js";
import { rateLimiters } from "../../middleware/rate-limit.middleware.js";
import { withErrorHandler } from "../../middleware/error-handler.middleware.js";
import { withLogging } from "../../middleware/logging.middleware.js";
import { Errors } from "../../utils/errors.js";
import type { Ride } from "../../types/index.js";

function formatName(firstName: string, lastName: string): string {
  return `${firstName} ${lastName.charAt(0).toUpperCase()}.`;
}

// Extracted for unit testing — called by the onCall wrapper after auth + validation
export async function handleCreateRideShare(rideId: string, uid: string) {

      // rideId and uid are already validated/extracted by the onCall wrapper

      // Read ride doc
      const rideSnap = await db.collection(CONSTANTS.COLLECTIONS.RIDES).doc(rideId).get();
      if (!rideSnap.exists) throw Errors.NOT_FOUND("Ride");

      const ride = rideSnap.data() as Ride;

      if (ride.status !== "STARTED") {
        throw Errors.INVALID_INPUT({ message: "Ride must be STARTED to share live location." });
      }

      // Verify caller eligibility: driver OR confirmed passenger
      const isDriver = ride.driverId === uid;
      if (!isDriver) {
        const bookingSnap = await db.collection(CONSTANTS.COLLECTIONS.BOOKINGS)
          .where("rideId", "==", rideId)
          .where("passengerId", "==", uid)
          .where("status", "==", "CONFIRMED")
          .limit(1)
          .get();
        if (bookingSnap.empty) throw Errors.UNAUTHORIZED();
      }

      let sharerName: string;
      if (isDriver) {
        sharerName = formatName(ride.driver.firstName, ride.driver.lastName);
      } else {
        const profileSnap = await db.collection(CONSTANTS.COLLECTIONS.PROFILES).doc(uid).get();
        const profile = profileSnap.data();
        sharerName = formatName(profile?.firstName ?? "User", profile?.lastName ?? "");
      }

      // Idempotency: return existing active share for this (rideId, uid)
      const existingSnap = await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES)
        .where("rideId", "==", rideId)
        .where("userId", "==", uid)
        .where("status", "==", "ACTIVE")
        .limit(1)
        .get();

      if (!existingSnap.empty) {
        const existing = existingSnap.docs[0].data();
        logger.info("createRideShare: returning existing share", { uid, rideId, shareToken: existing.shareToken });
        return ResponseFormatter.success({
          shareToken: existing.shareToken as string,
          trackingUrl: existing.trackingUrl as string,
          driverName: existing.driverName as string,
          vehiclePlate: existing.vehiclePlate as string,
          passengerName: existing.passengerName as string,
        });
      }

      // Cap check: no more than MAX_ACTIVE_PER_USER active shares
      const activeSnap = await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES)
        .where("userId", "==", uid)
        .where("status", "==", "ACTIVE")
        .get();

      if (activeSnap.size >= CONSTANTS.RIDE_SHARE.MAX_ACTIVE_PER_USER) {
        throw Errors.INVALID_INPUT({
          message: `You already have ${CONSTANTS.RIDE_SHARE.MAX_ACTIVE_PER_USER} active ride shares. Stop one before sharing again.`,
        });
      }

      // Generate token + write
      const shareToken = randomBytes(24).toString("base64url");
      const trackingUrl = `${CONSTANTS.RIDE_SHARE.TRACKING_URL_BASE}/${shareToken}`;
      const now = Timestamp.now();
      const expiresAt = Timestamp.fromMillis(Date.now() + CONSTANTS.RIDE_SHARE.EXPIRY_MS);

      const driverName = formatName(ride.driver.firstName, ride.driver.lastName);

      const shareDoc = {
        shareToken,
        rideId,
        userId: uid,
        type: "share" as const,
        status: "ACTIVE" as const,
        driverName,
        vehiclePlate: ride.vehicleDetails.plate,
        vehicleModel: ride.vehicleDetails.model,
        passengerName: sharerName,
        trackingUrl,
        createdAt: now,
        expiresAt,
        viewerCount: 0,
      };

      await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES).doc(shareToken).set(shareDoc);
      await rtdb.ref(`rideSharesByToken/${shareToken}`).set({ userId: uid });

      logger.info("createRideShare: created", { uid, rideId, shareToken });

      return ResponseFormatter.success({ shareToken, trackingUrl, driverName, vehiclePlate: ride.vehicleDetails.plate, passengerName: sharerName });
}

export const createRideShare = functions.https.onCall(
  withErrorHandler(async (data, context) => {
    return withLogging("createRideShare", async (data: any, authContext: any) => {
      requireEmailVerification(authContext);
      await rateLimiters.createRideShare(authContext);
      const { rideId } = validateData(schemas.createRideShare, data);
      return handleCreateRideShare(rideId, authContext.extendedAuth.uid);
    })(data, await authenticateUser(context));
  })
);
```

- [ ] **Step 4: Run tests — all should pass**

```bash
npx jest tests/unit/createRideShare.test.ts --no-coverage 2>&1 | tail -8
```

Expected: `Tests: 5 passed, 5 total`

- [ ] **Step 5: Type-check**

```bash
npx tsc --noEmit 2>&1
```

Expected: no errors.

---

## Task 3: `revokeRideShare` CF — TDD

**Files:**
- Create: `CarpoolBackend/functions/tests/unit/revokeRideShare.test.ts`
- Create: `CarpoolBackend/functions/src/functions/safety/revokeRideShare.fn.ts`

- [ ] **Step 1: Write failing tests**

```ts
// CarpoolBackend/functions/tests/unit/revokeRideShare.test.ts
import { describe, it, expect, jest, beforeEach } from "@jest/globals";
import { db, rtdb } from "../../src/config/firebase-admin.js";

const makeContext = (uid = "user1") => ({
  extendedAuth: { uid, email: "u@test.com", emailVerified: true, phone_number: null },
  rawContext: { auth: { uid } },
});

describe("revokeRideShare", () => {
  let mockDocRef: any;
  let mockRtdbRef: any;

  beforeEach(() => {
    jest.clearAllMocks();
    mockDocRef = {
      get: jest.fn(),
      update: jest.fn().mockResolvedValue(undefined),
    };
    jest.spyOn(db as any, "collection").mockReturnValue({
      doc: jest.fn().mockReturnValue(mockDocRef),
    });
    mockRtdbRef = {
      remove: jest.fn().mockResolvedValue(undefined),
    };
    jest.spyOn(rtdb as any, "ref").mockReturnValue(mockRtdbRef);
  });

  // Tests call handleRevokeRideShare(shareToken, uid) directly

  it("returns NOT_FOUND if share doc does not exist", async () => {
    const { handleRevokeRideShare } = await import(
      "../../src/functions/safety/revokeRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({ exists: false });
    await expect(handleRevokeRideShare("tok1", "user1")).rejects.toMatchObject({ code: "NOT_FOUND" });
  });

  it("returns UNAUTHORIZED if caller does not own the share", async () => {
    const { handleRevokeRideShare } = await import(
      "../../src/functions/safety/revokeRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ userId: "anotherUser", status: "ACTIVE", shareToken: "tok1" }),
    });
    await expect(handleRevokeRideShare("tok1", "notOwner")).rejects.toMatchObject({ code: "UNAUTHORIZED" });
  });

  it("returns INVALID_INPUT if share is already revoked", async () => {
    const { handleRevokeRideShare } = await import(
      "../../src/functions/safety/revokeRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ userId: "user1", status: "REVOKED", shareToken: "tok1" }),
    });
    await expect(handleRevokeRideShare("tok1", "user1")).rejects.toMatchObject({ code: "INVALID_INPUT" });
  });

  it("revokes successfully and updates Firestore", async () => {
    const { handleRevokeRideShare } = await import(
      "../../src/functions/safety/revokeRideShare.fn.js"
    );
    mockDocRef.get.mockResolvedValue({
      exists: true,
      data: () => ({ userId: "user1", status: "ACTIVE", shareToken: "tok1" }),
    });
    const result = await handleRevokeRideShare("tok1", "user1");
    expect(result.data.success).toBe(true);
    expect(mockDocRef.update).toHaveBeenCalledWith(expect.objectContaining({ status: "REVOKED" }));
  });
});
```

- [ ] **Step 2: Run tests — expect module-not-found**

```bash
npx jest tests/unit/revokeRideShare.test.ts --no-coverage 2>&1 | tail -5
```

Expected: `Cannot find module '../../src/functions/safety/revokeRideShare.fn.js'`

- [ ] **Step 3: Implement `revokeRideShare.fn.ts`**

```ts
// CarpoolBackend/functions/src/functions/safety/revokeRideShare.fn.ts
import * as functions from "firebase-functions";
import { Timestamp } from "firebase-admin/firestore";
import { db, rtdb } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { logger } from "../../utils/logger.js";
import { ResponseFormatter } from "../../utils/response.js";
import { authenticateUser, requireEmailVerification } from "../../middleware/auth.middleware.js";
import { validateData, schemas } from "../../middleware/validation.middleware.js";
import { rateLimiters } from "../../middleware/rate-limit.middleware.js";
import { withErrorHandler } from "../../middleware/error-handler.middleware.js";
import { withLogging } from "../../middleware/logging.middleware.js";
import { Errors } from "../../utils/errors.js";

export async function handleRevokeRideShare(shareToken: string, uid: string) {
  const shareRef = db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES).doc(shareToken);
  const shareSnap = await shareRef.get();

  if (!shareSnap.exists) throw Errors.NOT_FOUND("Ride share");

  const shareData = shareSnap.data()!;
  if (shareData.userId !== uid) throw Errors.UNAUTHORIZED();
  if (shareData.status !== "ACTIVE") {
    throw Errors.INVALID_INPUT({ message: "Share is already revoked or expired." });
  }

  await shareRef.update({ status: "REVOKED", revokedAt: Timestamp.now() });

  setTimeout(() => {
    Promise.all([
      rtdb.ref(`liveTracking/${shareToken}`).remove(),
      rtdb.ref(`rideSharesByToken/${shareToken}`).remove(),
    ]).catch((err) => logger.warn("revokeRideShare: RTDB cleanup failed", err));
  }, CONSTANTS.RIDE_SHARE.RTDB_CLEANUP_GRACE_MS);

  logger.info("revokeRideShare: revoked", { uid, shareToken });
  return ResponseFormatter.success({ success: true });
}

export const revokeRideShare = functions.https.onCall(
  withErrorHandler(async (data, context) => {
    return withLogging("revokeRideShare", async (data: any, authContext: any) => {
      requireEmailVerification(authContext);
      await rateLimiters.revokeRideShare(authContext);
      const { shareToken } = validateData(schemas.revokeRideShare, data);
      return handleRevokeRideShare(shareToken, authContext.extendedAuth.uid);
    })(data, await authenticateUser(context));
  })
);
```

- [ ] **Step 4: Run tests — all should pass**

```bash
npx jest tests/unit/revokeRideShare.test.ts --no-coverage 2>&1 | tail -5
```

Expected: `Tests: 4 passed, 4 total`

- [ ] **Step 5: Type-check**

```bash
npx tsc --noEmit 2>&1
```

Expected: no errors.

---

## Task 4: Trigger + Scheduled function — TDD

**Files:**
- Create: `CarpoolBackend/functions/tests/unit/onRideComplete.test.ts`
- Create: `CarpoolBackend/functions/src/functions/safety/onRideComplete.fn.ts`
- Create: `CarpoolBackend/functions/src/functions/scheduled/rideShareCleanup.fn.ts`

- [ ] **Step 1: Write `onRideComplete` failing tests**

```ts
// CarpoolBackend/functions/tests/unit/onRideComplete.test.ts
import { describe, it, expect, jest, beforeEach } from "@jest/globals";
import { db, rtdb } from "../../src/config/firebase-admin.js";

const makeChange = (beforeStatus: string, afterStatus: string, rideId = "r1") => ({
  before: { data: () => ({ status: beforeStatus, rideId }) },
  after: { data: () => ({ status: afterStatus, rideId }) },
  params: { rideId },
});

describe("onRideComplete trigger", () => {
  let mockCollectionRef: any;
  let mockQuery: any;
  let mockRtdbRef: any;
  let mockBatch: any;

  beforeEach(() => {
    jest.clearAllMocks();

    mockBatch = {
      update: jest.fn(),
      commit: jest.fn().mockResolvedValue(undefined),
    };

    mockQuery = {
      where: jest.fn().mockReturnThis(),
      get: jest.fn(),
    };

    mockCollectionRef = {
      where: jest.fn().mockReturnValue(mockQuery),
      doc: jest.fn().mockReturnValue({ ref: {} }),
    };

    jest.spyOn(db as any, "collection").mockReturnValue(mockCollectionRef);
    jest.spyOn(db as any, "batch").mockReturnValue(mockBatch);

    mockRtdbRef = { remove: jest.fn().mockResolvedValue(undefined) };
    jest.spyOn(rtdb as any, "ref").mockReturnValue(mockRtdbRef);
  });

  it("does nothing if status did not change to terminal", async () => {
    const { onRideCompleteHandler } = await import(
      "../../src/functions/safety/onRideComplete.fn.js"
    );
    await onRideCompleteHandler(makeChange("OPEN", "FULL") as any, {} as any);
    expect(mockQuery.get).not.toHaveBeenCalled();
  });

  it("does nothing if before and after status are the same", async () => {
    const { onRideCompleteHandler } = await import(
      "../../src/functions/safety/onRideComplete.fn.js"
    );
    await onRideCompleteHandler(makeChange("COMPLETED", "COMPLETED") as any, {} as any);
    expect(mockQuery.get).not.toHaveBeenCalled();
  });

  it("revokes all active shares when ride transitions to COMPLETED", async () => {
    const { onRideCompleteHandler } = await import(
      "../../src/functions/safety/onRideComplete.fn.js"
    );
    const mockShareDoc = {
      ref: {},
      data: () => ({ shareToken: "tok1", status: "ACTIVE" }),
    };
    mockQuery.get.mockResolvedValue({ empty: false, docs: [mockShareDoc] });

    await onRideCompleteHandler(makeChange("STARTED", "COMPLETED") as any, {} as any);

    expect(mockBatch.update).toHaveBeenCalledWith(
      mockShareDoc.ref,
      expect.objectContaining({ status: "REVOKED" })
    );
    expect(mockBatch.commit).toHaveBeenCalled();
    expect(mockRtdbRef.remove).toHaveBeenCalled();
  });

  it("revokes all active shares when ride transitions to CANCELLED", async () => {
    const { onRideCompleteHandler } = await import(
      "../../src/functions/safety/onRideComplete.fn.js"
    );
    const mockShareDoc = {
      ref: {},
      data: () => ({ shareToken: "tok2", status: "ACTIVE" }),
    };
    mockQuery.get.mockResolvedValue({ empty: false, docs: [mockShareDoc] });

    await onRideCompleteHandler(makeChange("OPEN", "CANCELLED") as any, {} as any);

    expect(mockBatch.update).toHaveBeenCalled();
    expect(mockBatch.commit).toHaveBeenCalled();
  });

  it("does nothing if no active shares for the ride", async () => {
    const { onRideCompleteHandler } = await import(
      "../../src/functions/safety/onRideComplete.fn.js"
    );
    mockQuery.get.mockResolvedValue({ empty: true, docs: [] });

    await onRideCompleteHandler(makeChange("STARTED", "COMPLETED") as any, {} as any);

    expect(mockBatch.commit).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests — expect module-not-found**

```bash
npx jest tests/unit/onRideComplete.test.ts --no-coverage 2>&1 | tail -5
```

Expected: `Cannot find module '../../src/functions/safety/onRideComplete.fn.js'`

- [ ] **Step 3: Implement `onRideComplete.fn.ts`**

```ts
// CarpoolBackend/functions/src/functions/safety/onRideComplete.fn.ts
import * as functions from "firebase-functions";
import { Timestamp } from "firebase-admin/firestore";
import { db, rtdb } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { logger } from "../../utils/logger.js";

const TERMINAL_STATUSES = new Set(["COMPLETED", "CANCELLED"]);

export async function onRideCompleteHandler(
  change: functions.Change<functions.firestore.DocumentSnapshot>,
  context: functions.EventContext
): Promise<null> {
  const before = change.before.data();
  const after = change.after.data();
  const rideId = context.params.rideId;

  if (!before || !after) return null;
  if (before.status === after.status) return null;
  if (!TERMINAL_STATUSES.has(after.status)) return null;

  const sharesSnap = await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES)
    .where("rideId", "==", rideId)
    .where("status", "==", "ACTIVE")
    .get();

  if (sharesSnap.empty) return null;

  const batch = db.batch();
  const now = Timestamp.now();
  const tokens: string[] = [];

  for (const doc of sharesSnap.docs) {
    batch.update(doc.ref, { status: "REVOKED", revokedAt: now });
    tokens.push(doc.data().shareToken as string);
  }

  await batch.commit();

  // Clean up RTDB nodes
  await Promise.all(
    tokens.flatMap((token) => [
      rtdb.ref(`liveTracking/${token}`).remove(),
      rtdb.ref(`rideSharesByToken/${token}`).remove(),
    ])
  );

  logger.info("onRideComplete: revoked shares", { rideId, count: tokens.length, status: after.status });
  return null;
}

export const onRideComplete = functions.firestore
  .document(`${CONSTANTS.COLLECTIONS.RIDES}/{rideId}`)
  .onUpdate(onRideCompleteHandler);
```

- [ ] **Step 4: Run `onRideComplete` tests — all should pass**

```bash
npx jest tests/unit/onRideComplete.test.ts --no-coverage 2>&1 | tail -5
```

Expected: `Tests: 5 passed, 5 total`

- [ ] **Step 5: Implement `rideShareCleanup.fn.ts`**

```ts
// CarpoolBackend/functions/src/functions/scheduled/rideShareCleanup.fn.ts
import * as functions from "firebase-functions";
import { Timestamp } from "firebase-admin/firestore";
import { db, rtdb } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { logger } from "../../utils/logger.js";

export const rideShareCleanup = functions.pubsub
  .schedule("every 60 minutes")
  .onRun(async () => {
    const now = Timestamp.now();

    const expiredSnap = await db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES)
      .where("status", "==", "ACTIVE")
      .where("expiresAt", "<", now)
      .get();

    if (expiredSnap.empty) {
      logger.info("rideShareCleanup: no expired shares");
      return null;
    }

    const batch = db.batch();
    const tokens: string[] = [];

    for (const doc of expiredSnap.docs) {
      batch.update(doc.ref, { status: "EXPIRED" });
      tokens.push(doc.data().shareToken as string);
    }

    await batch.commit();

    await Promise.all(
      tokens.flatMap((token) => [
        rtdb.ref(`liveTracking/${token}`).remove(),
        rtdb.ref(`rideSharesByToken/${token}`).remove(),
      ])
    );

    logger.info("rideShareCleanup: expired shares cleaned up", { count: tokens.length });
    return null;
  });
```

- [ ] **Step 6: Type-check everything**

```bash
npx tsc --noEmit 2>&1
```

Expected: no errors.

---

## Task 5: Backend wiring + Firestore rules

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/index.ts`
- Modify: `CarpoolBackend/functions/src/functions/scheduled/index.ts`
- Modify: `CarpoolBackend/firestore.rules`

- [ ] **Step 1: Export new safety functions**

In `CarpoolBackend/functions/src/functions/safety/index.ts`, add:

```ts
export { createRideShare } from "./createRideShare.fn.js";
export { revokeRideShare } from "./revokeRideShare.fn.js";
export { onRideComplete } from "./onRideComplete.fn.js";
```

- [ ] **Step 2: Export scheduled cleanup**

In `CarpoolBackend/functions/src/functions/scheduled/index.ts`, add:

```ts
export { rideShareCleanup } from "./rideShareCleanup.fn.js";
```

- [ ] **Step 3: Add Firestore rules for rideShares**

In `CarpoolBackend/firestore.rules`, add after the existing `rideOffers` block (before the closing `}` of `match /databases/{database}/documents`):

```
match /rideShares/{shareToken} {
  allow read: if isAuthenticated() && resource.data.userId == request.auth.uid;
  allow write: if false; // createRideShare / revokeRideShare CFs only
}
```

- [ ] **Step 4: Type-check + run unit tests**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit && npx jest tests/unit --no-coverage --runInBand 2>&1 | tail -8
```

Expected: type-check clean, all unit tests pass (new + existing).

---

## Task 6: Frontend type + functions.service

**Files:**
- Create: `Carpool/src/types/models/rideShare.types.ts`
- Modify: `Carpool/src/types/models/index.ts`
- Modify: `Carpool/src/services/firebase/functions.service.ts`

- [ ] **Step 1: Create RideShare type**

```ts
// Carpool/src/types/models/rideShare.types.ts
import { FirebaseFirestoreTypes } from '@react-native-firebase/firestore';

export type RideShareStatus = 'ACTIVE' | 'REVOKED' | 'EXPIRED';

export interface RideShare {
  shareToken: string;
  rideId: string;
  userId: string;
  type: 'share';
  status: RideShareStatus;
  driverName: string;
  vehiclePlate: string;
  vehicleModel: string;
  passengerName: string;
  trackingUrl: string;
  createdAt: FirebaseFirestoreTypes.Timestamp;
  expiresAt: FirebaseFirestoreTypes.Timestamp;
  viewerCount: number;
  revokedAt?: FirebaseFirestoreTypes.Timestamp;
}

export interface CreateRideShareResponse {
  shareToken: string;
  trackingUrl: string;
  driverName: string;
  vehiclePlate: string;
  passengerName: string;
}
```

- [ ] **Step 2: Export from types index**

In `Carpool/src/types/models/index.ts`, add:

```ts
export * from './rideShare.types';
```

- [ ] **Step 3: Add callable wrappers to functions.service.ts**

In `Carpool/src/services/firebase/functions.service.ts`, add after the trusted contacts section:

```ts
async createRideShare(data: { rideId: string }): Promise<CreateRideShareResponse> {
  return this.callFunction('createRideShare', data, { retry: false });
}

async revokeRideShare(data: { shareToken: string }): Promise<void> {
  return this.callFunction('revokeRideShare', data, { retry: false });
}
```

Also add the import at the top of the file:

```ts
import type { CreateRideShareResponse } from '../types/models/rideShare.types';
```

- [ ] **Step 4: Type-check frontend**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit 2>&1 | head -20
```

Expected: no new errors.

---

## Task 7: `useRideShare` hook

**Files:**
- Create: `Carpool/hooks/useRideShare.ts`

- [ ] **Step 1: Implement the hook**

```ts
// Carpool/hooks/useRideShare.ts
import { useState, useEffect, useRef, useCallback } from 'react';
import { Share } from 'react-native';
import { authService } from '../src/services/firebase';
import { firestoreService } from '../src/services/firebase/firestore.service';
import { functionsService } from '../src/services/firebase/functions.service';
import { liveTrackingService } from '../src/services/tracking/liveTracking.service';
import { showError } from '../src/utils/show-error';
import type { RideShare } from '../src/types/models/rideShare.types';

interface UseRideShareReturn {
  activeShare: RideShare | null;
  isLoading: boolean;
  createShare: () => Promise<void>;
  revokeShare: () => Promise<void>;
}

export function useRideShare(rideId: string): UseRideShareReturn {
  const [activeShare, setActiveShare] = useState<RideShare | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const isMountedRef = useRef(true);

  useEffect(() => {
    isMountedRef.current = true;
    const uid = authService.getCurrentUser()?.uid;
    if (!uid || !rideId) return;

    const unsubscribe = firestoreService.onCollectionSnapshot<RideShare>(
      'rideShares',
      [
        { field: 'rideId', operator: '==', value: rideId },
        { field: 'userId', operator: '==', value: uid },
        { field: 'status', operator: '==', value: 'ACTIVE' },
      ],
      undefined,
      1,
      (shares) => {
        if (isMountedRef.current) {
          setActiveShare(shares[0] ?? null);
        }
      }
    );

    return () => {
      isMountedRef.current = false;
      unsubscribe();
    };
  }, [rideId]);

  const createShare = useCallback(async () => {
    setIsLoading(true);
    try {
      const result = await functionsService.createRideShare({ rideId });
      await liveTrackingService.startTracking(result.shareToken, 'share');
      const message = `🚗 Track my ride live: ${result.trackingUrl}\nDriver: ${result.driverName} · ${result.vehiclePlate}`;
      await Share.share({ message });
    } catch (err) {
      showError(err);
    } finally {
      if (isMountedRef.current) setIsLoading(false);
    }
  }, [rideId]);

  const revokeShare = useCallback(async () => {
    if (!activeShare) return;
    setIsLoading(true);
    try {
      await functionsService.revokeRideShare({ shareToken: activeShare.shareToken });
      await liveTrackingService.stopTracking(activeShare.shareToken);
    } catch (err) {
      showError(err);
    } finally {
      if (isMountedRef.current) setIsLoading(false);
    }
  }, [activeShare]);

  return { activeShare, isLoading, createShare, revokeShare };
}
```

- [ ] **Step 2: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit 2>&1 | head -20
```

Expected: no new errors.

---

## Task 8: `ShareLiveRideButton` component

**Files:**
- Create: `Carpool/components/ShareLiveRideButton.tsx`

- [ ] **Step 1: Implement the component**

```tsx
// Carpool/components/ShareLiveRideButton.tsx
import React from 'react';
import { TouchableOpacity, Text, ActivityIndicator, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { theme } from '../constants/theme';
import type { RideShare } from '../src/types/models/rideShare.types';

interface ShareLiveRideButtonProps {
  activeShare: RideShare | null;
  isLoading: boolean;
  onShare: () => void;
  onStop: () => void;
}

export function ShareLiveRideButton({
  activeShare,
  isLoading,
  onShare,
  onStop,
}: ShareLiveRideButtonProps) {
  const isSharing = activeShare !== null;

  if (isLoading) {
    return (
      <TouchableOpacity
        style={[styles.button, styles.buttonLoading]}
        disabled
        accessibilityRole="button"
        accessibilityLabel="Loading"
      >
        <ActivityIndicator size="small" color={theme.colors.primary} />
      </TouchableOpacity>
    );
  }

  if (isSharing) {
    return (
      <TouchableOpacity
        style={[styles.button, styles.buttonSharing]}
        onPress={onStop}
        activeOpacity={0.8}
        accessibilityRole="button"
        accessibilityLabel="Stop sharing live ride"
      >
        <Ionicons name="stop-circle-outline" size={18} color="#fff" />
        <Text style={styles.buttonTextSharing}>Stop Sharing</Text>
      </TouchableOpacity>
    );
  }

  return (
    <TouchableOpacity
      style={[styles.button, styles.buttonIdle]}
      onPress={onShare}
      activeOpacity={0.8}
      accessibilityRole="button"
      accessibilityLabel="Share live ride"
    >
      <Ionicons name="share-social-outline" size={18} color={theme.colors.primary} />
      <Text style={styles.buttonTextIdle}>Share Live Ride</Text>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  button: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    gap: 6,
    paddingVertical: 10,
    paddingHorizontal: 16,
    borderRadius: 20,
    minWidth: 148,
  },
  buttonIdle: {
    borderWidth: 1.5,
    borderColor: theme.colors.primary,
    backgroundColor: 'transparent',
  },
  buttonSharing: {
    backgroundColor: theme.colors.primary,
  },
  buttonLoading: {
    borderWidth: 1.5,
    borderColor: theme.colors.border,
    backgroundColor: 'transparent',
  },
  buttonTextIdle: {
    fontSize: 13,
    fontWeight: '600',
    color: theme.colors.primary,
  },
  buttonTextSharing: {
    fontSize: 13,
    fontWeight: '600',
    color: '#fff',
  },
});
```

- [ ] **Step 2: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit 2>&1 | head -20
```

Expected: no new errors.

---

## Task 9: Wire into screens

**Files:**
- Modify: `Carpool/app/live-ride.tsx`
- Modify: `Carpool/app/booked-ride-details.tsx`

### live-ride.tsx (driver view)

The screen has `rideId` from `useLocalSearchParams` and `isDriver` boolean. The `ride?.status` is available. The My Location FAB is at line ~838; add the share button just before it.

- [ ] **Step 1: Add import + hook to live-ride.tsx**

Add at the top of the imports section (after existing component imports):

```tsx
import { ShareLiveRideButton } from '../components/ShareLiveRideButton';
import { useRideShare } from '../hooks/useRideShare';
```

Inside the main component function, after the existing hooks, add:

```tsx
const { activeShare: rideShare, isLoading: shareLoading, createShare, revokeShare } = useRideShare(rideId ?? '');
```

- [ ] **Step 2: Mount button in live-ride.tsx JSX**

Find the My Location FAB block (around line 838):

```tsx
{/* ── My Location FAB ──────────────────────────────────────────── */}
<TouchableOpacity
  style={styles.myLocationFab}
```

Add the share button immediately BEFORE the My Location FAB comment:

```tsx
{/* ── Share Live Ride button ───────────────────────────────────── */}
{ride?.status === 'STARTED' && (
  <ShareLiveRideButton
    activeShare={rideShare}
    isLoading={shareLoading}
    onShare={createShare}
    onStop={revokeShare}
    style={styles.shareLiveRideBtn}
  />
)}
```

Add to `styles` (at the bottom `StyleSheet.create` block):

```tsx
shareLiveRideBtn: {
  position: 'absolute',
  bottom: 160,
  alignSelf: 'center',
},
```

Update `ShareLiveRideButton` to accept an optional `style` prop:

```tsx
// In ShareLiveRideButton.tsx, add style to props:
interface ShareLiveRideButtonProps {
  ...
  style?: object;
}
// Pass it to the TouchableOpacity: style={[styles.button, styles.buttonIdle, style]}
```

- [ ] **Step 3: Add import + mount in booked-ride-details.tsx**

Add imports:

```tsx
import { ShareLiveRideButton } from '../components/ShareLiveRideButton';
import { useRideShare } from '../hooks/useRideShare';
```

Inside the component, after existing hooks:

```tsx
const resolvedRideId = booking?.rideId || params.rideId || '';
const { activeShare: rideShare, isLoading: shareLoading, createShare, revokeShare } = useRideShare(resolvedRideId);
```

The screen has `showLiveBanner` which is `isRideStarted && bookingStatus === 'CONFIRMED'`. Add the share button inside the existing `liveTrackingFooter` View (around line 566), after the "View Live Tracking" button:

```tsx
{showLiveBanner && ride && (
  <View style={[styles.liveTrackingFooter, { paddingBottom: insets.bottom + theme.spacing.lg }]}>
    <TouchableOpacity
      style={styles.viewLiveBtn}
      onPress={() => router.push(`/live-ride?rideId=${ride.rideId || ride.id}&role=passenger`)}
      activeOpacity={0.8}
    >
      <View style={styles.viewLiveDot} />
      <Text style={styles.viewLiveBtnText}>View Live Tracking</Text>
      <Ionicons name="chevron-forward" size={18} color="#fff" />
    </TouchableOpacity>
    <ShareLiveRideButton
      activeShare={rideShare}
      isLoading={shareLoading}
      onShare={createShare}
      onStop={revokeShare}
      style={{ marginTop: theme.spacing.sm }}
    />
  </View>
)}
```

- [ ] **Step 4: Type-check both screens**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/Carpool
npx tsc --noEmit 2>&1 | head -20
```

Expected: no new errors.

---

## Task 10: Deploy + verify

- [ ] **Step 1: Batch commit Tasks 5–9**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/safety/createRideShare.fn.ts \
  functions/src/functions/safety/revokeRideShare.fn.ts \
  functions/src/functions/safety/onRideComplete.fn.ts \
  functions/src/functions/scheduled/rideShareCleanup.fn.ts \
  functions/src/functions/safety/index.ts \
  functions/src/functions/scheduled/index.ts \
  functions/src/config/constants.ts \
  functions/src/middleware/validation.middleware.ts \
  functions/src/middleware/rate-limit.middleware.ts \
  functions/tests/unit/createRideShare.test.ts \
  functions/tests/unit/revokeRideShare.test.ts \
  functions/tests/unit/onRideComplete.test.ts \
  firestore.rules
git commit -m "feat(phase-g): ride share CFs, trigger, scheduled cleanup, rules"

cd /Users/sairajaggani/Desktop/Space/Project/Carpool
git add src/types/models/rideShare.types.ts \
  src/types/models/index.ts \
  src/services/firebase/functions.service.ts \
  hooks/useRideShare.ts \
  components/ShareLiveRideButton.tsx \
  app/live-ride.tsx \
  app/booked-ride-details.tsx
git commit -m "feat(phase-g): ShareLiveRideButton, useRideShare hook, screen integration"
```

- [ ] **Step 2: Deploy Firestore rules**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
firebase deploy --only firestore:rules 2>&1 | tail -5
```

Expected: `✔ Deploy complete!`

- [ ] **Step 3: Deploy all Phase G functions**

```bash
firebase deploy --only functions:createRideShare,functions:revokeRideShare,functions:onRideComplete,functions:rideShareCleanup 2>&1 | tail -8
```

Expected: `✔ Deploy complete!` — all 4 functions appear in Firebase Console.

- [ ] **Step 4: Smoke-test createRideShare with missing rideId**

From a Firebase shell or emulator, call with missing field and confirm validation error. (Manual test — no automated step needed here since the emulator isn't part of this deploy flow.)

- [ ] **Step 5: Update todo.md Phase G acceptance**

Mark Phase G as complete in `/Users/sairajaggani/Desktop/Space/Project/todo.md`:
- Status table: `✅ COMPLETE`
- All Phase G acceptance items: `[x]`
