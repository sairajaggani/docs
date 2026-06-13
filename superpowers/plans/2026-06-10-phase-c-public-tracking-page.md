# Phase C — Public Tracking Web Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a public web page at `carpool-app-1e668.web.app/track/{token}` that shows live location via a Leaflet map, backed by a `getTrackingData` HTTPS Cloud Function and a direct RTDB listener.

**Architecture:** Firebase Hosting serves static HTML+CSS+JS. On page load the browser calls `getTrackingData` CF once (validates token, enforces viewer cap, returns metadata). Live location updates stream via Firebase JS SDK direct RTDB listener on `/liveTracking/{token}`. A 60-second poll re-calls the CF to detect expiry/resolution.

**Tech Stack:** Firebase Hosting, Firebase RTDB (Firebase JS SDK v10 compat CDN), Leaflet + OpenStreetMap, Firebase Cloud Functions (onRequest, TypeScript ESM), Firestore, cors npm package.

---

## File Map

| File | Status | Responsibility |
|---|---|---|
| `CarpoolBackend/firebase.json` | Modify | Add hosting block + security headers |
| `CarpoolBackend/firestore.indexes.json` | Modify | Add shareToken indexes on sosAlerts + rideShares |
| `CarpoolBackend/functions/src/config/constants.ts` | Modify | Add GET_TRACKING_DATA rate limit + VIEWER_CAP |
| `CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts` | Create | HTTPS CF handler: token lookup, viewer cap, rate limit, metadata response |
| `CarpoolBackend/functions/src/functions/safety/index.ts` | Modify | Export getTrackingData |
| `CarpoolBackend/functions/tests/unit/getTrackingData.test.ts` | Create | Unit tests for CF handler logic |
| `CarpoolBackend/public/track/index.html` | Create | Page shell: Leaflet CDN, Firebase SDK CDN, meta tags |
| `CarpoolBackend/public/track/track.css` | Create | Mobile-first styles |
| `CarpoolBackend/public/track/track.js` | Create | State machine: CF fetch, RTDB listener, map render, stale indicator |

---

## Task 1: Hosting config + constants

**Files:**
- Modify: `CarpoolBackend/firebase.json`
- Modify: `CarpoolBackend/functions/src/config/constants.ts`

- [ ] **Step 1: Add hosting config to firebase.json**

Open `CarpoolBackend/firebase.json`. The file currently has `functions`, `emulators`, `database`, `firestore`, `storage`, `extensions` keys. Add a `hosting` key at the top level:

```json
"hosting": {
  "public": "public",
  "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
  "rewrites": [{ "source": "/track/**", "destination": "/track/index.html" }],
  "headers": [
    {
      "source": "/track/**",
      "headers": [
        { "key": "X-Robots-Tag", "value": "noindex, nofollow" },
        { "key": "Cache-Control", "value": "no-store, max-age=0" },
        { "key": "Referrer-Policy", "value": "no-referrer" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" }
      ]
    }
  ]
}
```

- [ ] **Step 2: Add rate limit + viewer cap constants**

In `CarpoolBackend/functions/src/config/constants.ts`, add to the `RATE_LIMITS` object (after the `LIST_TRUSTED_CONTACTS` line):

```ts
GET_TRACKING_DATA: 60,  // per minute per token
```

Add a new top-level key in the `CONSTANTS` object (after the `TRUSTED_CONTACT` block):

```ts
TRACKING: {
  VIEWER_CAP: 50,
  STALE_THRESHOLD_MS: 30_000,
  STATUS_POLL_INTERVAL_MS: 60_000,
},
```

- [ ] **Step 3: Create the public directory**

```bash
mkdir -p /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/public/track
```

- [ ] **Step 4: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add firebase.json functions/src/config/constants.ts
git commit -m "feat(phase-c): hosting config + tracking constants"
```

---

## Task 2: Firestore indexes for shareToken lookup

**Files:**
- Modify: `CarpoolBackend/firestore.indexes.json`

The `getTrackingData` CF does `where('shareToken', '==', token)` on both `sosAlerts` and `rideShares`. Firestore auto-indexes single fields, but adding them explicitly to `firestore.indexes.json` ensures they are deployed and prevents query failures after rules redeploy.

- [ ] **Step 1: Add shareToken field override entries**

In `CarpoolBackend/firestore.indexes.json`, inside the `"fieldOverrides"` array (add the array if it doesn't exist at the top level alongside `"indexes"`):

```json
"fieldOverrides": [
  {
    "collectionGroup": "sosAlerts",
    "fieldPath": "shareToken",
    "indexes": [
      { "order": "ASCENDING", "queryScope": "COLLECTION" }
    ]
  },
  {
    "collectionGroup": "rideShares",
    "fieldPath": "shareToken",
    "indexes": [
      { "order": "ASCENDING", "queryScope": "COLLECTION" }
    ]
  }
]
```

- [ ] **Step 2: Verify JSON is valid**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
node -e "JSON.parse(require('fs').readFileSync('firestore.indexes.json','utf8')); console.log('valid')"
```

Expected: `valid`

- [ ] **Step 3: Commit**

```bash
git add firestore.indexes.json
git commit -m "feat(phase-c): shareToken field indexes on sosAlerts + rideShares"
```

---

## Task 3: `getTrackingData` CF — write failing tests first

**Files:**
- Create: `CarpoolBackend/functions/tests/unit/getTrackingData.test.ts`

The CF handler is extracted as a pure async function for testability. Tests use mock express req/res objects and spy on `db` and `rtdb`.

- [ ] **Step 1: Create the test file**

```ts
// CarpoolBackend/functions/tests/unit/getTrackingData.test.ts
import { describe, it, expect, jest, beforeEach } from "@jest/globals";
import { db, rtdb } from "../../src/config/firebase-admin.js";

// Handler imported after CF file is created in Task 4
// For now import the shape we expect
let handleGetTrackingData: (req: any, res: any) => Promise<void>;

const makeReq = (query: Record<string, string> = {}) => ({
  method: "GET",
  query,
  headers: {},
  ip: "127.0.0.1",
});

const makeRes = () => {
  const res: any = {};
  res.status = jest.fn().mockReturnValue(res);
  res.json = jest.fn().mockReturnValue(res);
  res.setHeader = jest.fn().mockReturnValue(res);
  return res;
};

const makeDocSnap = (exists: boolean, data?: object) => ({
  exists,
  data: () => data,
  id: "doc123",
});

describe("getTrackingData handler", () => {
  let mockCollectionRef: any;
  let mockQuery: any;
  let mockDocRef: any;

  beforeEach(() => {
    jest.clearAllMocks();

    mockDocRef = {
      get: jest.fn(),
      update: jest.fn().mockResolvedValue(undefined),
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
  });

  it("returns 400 when token is missing", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    const req = makeReq({});
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(400);
    expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ error: "MISSING_TOKEN" }));
  });

  it("returns 404 when token not found in sosAlerts or rideShares", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    mockQuery.get.mockResolvedValue({ empty: true, docs: [] });
    const req = makeReq({ token: "nonexistent" });
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(404);
    expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ error: "NOT_FOUND" }));
  });

  it("returns 410 EXPIRED when status is RESOLVED", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    const sosData = {
      shareToken: "tok1", status: "RESOLVED", userId: "u1",
      viewerCount: 0, resolvedAt: Date.now(),
    };
    mockQuery.get
      .mockResolvedValueOnce({ empty: false, docs: [{ id: "alert1", data: () => sosData }] })
      .mockResolvedValueOnce({ empty: true, docs: [] });
    const req = makeReq({ token: "tok1" });
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(410);
    expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ error: "EXPIRED" }));
  });

  it("returns 410 VIEWER_CAP_EXCEEDED when viewerCount >= 50", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    const sosData = {
      shareToken: "tok2", status: "ACTIVE", userId: "u1",
      viewerCount: 50,
      driverName: "Rajesh K.", vehiclePlate: "MH12AB1234",
      vehicleModel: "Swift", passengerName: "Saira R.",
      country: "IN", triggeredAt: Date.now(), expiresAt: Date.now() + 3_600_000,
    };
    mockQuery.get
      .mockResolvedValueOnce({ empty: false, docs: [{ id: "alert2", data: () => sosData }] })
      .mockResolvedValueOnce({ empty: true, docs: [] });
    mockDocRef.update.mockResolvedValue(undefined);
    const req = makeReq({ token: "tok2" });
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(410);
    expect(res.json).toHaveBeenCalledWith(
      expect.objectContaining({ error: "VIEWER_CAP_EXCEEDED" })
    );
  });

  it("returns 200 with SOS metadata for active SOS token", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    const now = Date.now();
    const sosData = {
      shareToken: "tok3", status: "ACTIVE", userId: "u1",
      viewerCount: 2,
      driverName: "Rajesh K.", vehiclePlate: "MH12AB1234",
      vehicleModel: "Swift Dzire", passengerName: "Saira R.",
      country: "IN", triggeredAt: now, expiresAt: now + 3_600_000,
      initialLocation: { lat: 19.076, lng: 72.877, accuracy: 10, speed: null, bearing: null },
    };
    mockQuery.get
      .mockResolvedValueOnce({ empty: false, docs: [{ id: "alert3", data: () => sosData }] })
      .mockResolvedValueOnce({ empty: true, docs: [] });
    mockDocRef.update.mockResolvedValue(undefined);

    // RTDB mock returns last known location
    const rtdbSnap = { val: () => ({ lat: 19.076, lng: 72.877, accuracy: 10, speed: 20, bearing: 90, batteryPct: 80, updatedAt: now }) };
    const mockRtdbRef = { once: jest.fn().mockResolvedValue(rtdbSnap) };
    jest.spyOn(rtdb as any, "ref").mockReturnValue(mockRtdbRef);

    const req = makeReq({ token: "tok3" });
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(200);
    const body = (res.json as jest.Mock).mock.calls[0][0];
    expect(body.type).toBe("sos");
    expect(body.driverName).toBe("Rajesh K.");
    expect(body.emergencyNumber).toBe("112");
    expect(body.location.lat).toBe(19.076);
    // viewerCount incremented
    expect(mockDocRef.update).toHaveBeenCalledWith({ viewerCount: 3 });
  });

  it("returns 200 with share metadata for active rideShare token", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    const now = Date.now();
    const shareData = {
      shareToken: "tok4", status: "ACTIVE", userId: "u2",
      viewerCount: 0,
      driverName: "Ravi S.", vehiclePlate: "KA01AB5678",
      vehicleModel: "Innova", passengerName: "Meera J.",
      expiresAt: now + 1_800_000,
    };
    mockQuery.get
      .mockResolvedValueOnce({ empty: true, docs: [] })
      .mockResolvedValueOnce({ empty: false, docs: [{ id: "share4", data: () => shareData }] });
    mockDocRef.update.mockResolvedValue(undefined);

    const rtdbSnap = { val: () => ({ lat: 12.97, lng: 77.59, accuracy: 5, speed: 30, bearing: 180, batteryPct: 60, updatedAt: now }) };
    const mockRtdbRef = { once: jest.fn().mockResolvedValue(rtdbSnap) };
    jest.spyOn(rtdb as any, "ref").mockReturnValue(mockRtdbRef);

    const req = makeReq({ token: "tok4" });
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(200);
    const body = (res.json as jest.Mock).mock.calls[0][0];
    expect(body.type).toBe("share");
    expect(body.emergencyNumber).toBeUndefined();
  });

  it("returns only status + expiresAt when metadataOnly=true", async () => {
    const { handleGetTrackingData } = await import(
      "../../src/functions/safety/getTrackingData.fn.js"
    );
    const now = Date.now();
    const sosData = {
      shareToken: "tok5", status: "ACTIVE", userId: "u1",
      viewerCount: 5, expiresAt: now + 3_600_000,
      driverName: "X", vehiclePlate: "Y", vehicleModel: "Z", passengerName: "W",
      country: "IN", triggeredAt: now,
    };
    mockQuery.get
      .mockResolvedValueOnce({ empty: false, docs: [{ id: "a5", data: () => sosData }] })
      .mockResolvedValueOnce({ empty: true, docs: [] });

    const req = makeReq({ token: "tok5", metadataOnly: "true" });
    const res = makeRes();
    await handleGetTrackingData(req, res);
    expect(res.status).toHaveBeenCalledWith(200);
    const body = (res.json as jest.Mock).mock.calls[0][0];
    expect(body).toEqual({ status: "ACTIVE", expiresAt: sosData.expiresAt });
    // should NOT increment viewerCount on metadataOnly
    expect(mockDocRef.update).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests — expect them to fail (module not found)**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx jest tests/unit/getTrackingData.test.ts --no-coverage 2>&1 | tail -20
```

Expected: import error — `getTrackingData.fn.js` does not exist yet.

---

## Task 4: `getTrackingData` CF — implementation

**Files:**
- Create: `CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts`

- [ ] **Step 1: Create the CF file**

```ts
// CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts
import * as functions from "firebase-functions";
import cors from "cors";
import type { Request, Response } from "express";
import { db, rtdb } from "../../config/firebase-admin.js";
import { CONSTANTS } from "../../config/constants.js";
import { logger } from "../../utils/logger.js";

const corsHandler = cors({ origin: true });

const COUNTRY_EMERGENCY: Record<string, string> = {
  IN: "112", US: "911", CA: "911", GB: "999",
  AU: "000", DE: "112", FR: "112", NL: "112",
};

function resolveEmergencyNumber(country?: string): string {
  if (!country) return "112";
  return COUNTRY_EMERGENCY[country.toUpperCase()] ?? "112";
}

const RESOLVED_STATUSES = new Set(["RESOLVED", "RESOLVED_TIMEOUT", "EXPIRED"]);

export async function handleGetTrackingData(req: Request, res: Response): Promise<void> {
  const token = req.query.token as string | undefined;
  const metadataOnly = req.query.metadataOnly === "true";

  if (!token) {
    res.status(400).json({ error: "MISSING_TOKEN" });
    return;
  }

  // Parallel lookup in sosAlerts + rideShares
  const [sosSnap, shareSnap] = await Promise.all([
    db.collection(CONSTANTS.COLLECTIONS.SOS_ALERTS)
      .where("shareToken", "==", token)
      .limit(1)
      .get(),
    db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES)
      .where("shareToken", "==", token)
      .limit(1)
      .get(),
  ]);

  const sosDoc = sosSnap.docs[0] ?? null;
  const shareDoc = shareSnap.docs[0] ?? null;

  if (!sosDoc && !shareDoc) {
    res.status(404).json({ error: "NOT_FOUND" });
    return;
  }

  const docRef = sosDoc
    ? db.collection(CONSTANTS.COLLECTIONS.SOS_ALERTS).doc(sosDoc.id)
    : db.collection(CONSTANTS.COLLECTIONS.RIDE_SHARES).doc(shareDoc!.id);

  const docData = (sosDoc ?? shareDoc)!.data()!;
  const type = sosDoc ? "sos" : "share";

  // Check expired/resolved
  if (RESOLVED_STATUSES.has(docData.status) || (docData.expiresAt && Date.now() > docData.expiresAt)) {
    res.status(410).json({ error: "EXPIRED", resolvedAt: docData.resolvedAt ?? docData.expiresAt });
    return;
  }

  // metadataOnly — status poll, no viewer cap increment
  if (metadataOnly) {
    res.status(200).json({ status: docData.status, expiresAt: docData.expiresAt });
    return;
  }

  // Per-token rate limit: 60 req/min via RTDB bucket keyed on token hash
  const tokenHash = Buffer.from(token).toString("base64url").slice(0, 16);
  const rlKey = `${CONSTANTS.RTDB_PATHS.RATE_LIMITS}/getTrackingData/${tokenHash}`;
  const now = Date.now();
  let rateLimited = false;
  await rtdb.ref(rlKey).transaction((cur: { count: number; resetAt: number } | null) => {
    if (!cur || now > cur.resetAt) return { count: 1, resetAt: now + 60_000 };
    if (cur.count >= CONSTANTS.RATE_LIMITS.GET_TRACKING_DATA) { rateLimited = true; return; }
    return { count: cur.count + 1, resetAt: cur.resetAt };
  });
  if (rateLimited) {
    res.status(429).json({ error: "RATE_LIMITED" });
    return;
  }

  // Viewer cap: check + increment atomically
  const currentViewers: number = docData.viewerCount ?? 0;
  if (currentViewers >= CONSTANTS.TRACKING.VIEWER_CAP) {
    res.status(410).json({ error: "VIEWER_CAP_EXCEEDED" });
    return;
  }
  await docRef.update({ viewerCount: currentViewers + 1 });

  // Fetch latest RTDB location
  const locationSnap = await rtdb.ref(`liveTracking/${token}`).once("value");
  const liveLocation = locationSnap.val();

  const location = liveLocation ?? docData.initialLocation ?? null;

  const responseBody: Record<string, unknown> = {
    type,
    status: docData.status,
    passengerName: docData.passengerName,
    driverName: docData.driverName,
    vehiclePlate: docData.vehiclePlate,
    vehicleModel: docData.vehicleModel,
    location,
    updatedAt: liveLocation?.updatedAt ?? null,
    expiresAt: docData.expiresAt,
  };

  if (type === "sos") {
    responseBody.triggeredAt = docData.triggeredAt;
    responseBody.emergencyNumber = resolveEmergencyNumber(docData.country);
  }

  logger.info("getTrackingData: served", {
    type,
    tokenHash: Buffer.from(token).toString("base64").slice(0, 8),
    viewerCount: currentViewers + 1,
  });

  res.status(200).json(responseBody);
}

export const getTrackingData = functions.https.onRequest((req, res) => {
  corsHandler(req, res, () => handleGetTrackingData(req, res));
});
```

- [ ] **Step 2: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit 2>&1
```

Expected: no errors.

- [ ] **Step 3: Run tests — all should pass**

```bash
npx jest tests/unit/getTrackingData.test.ts --no-coverage 2>&1 | tail -20
```

Expected: `Tests: 6 passed, 6 total`

- [ ] **Step 4: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/safety/getTrackingData.fn.ts functions/tests/unit/getTrackingData.test.ts
git commit -m "feat(phase-c): getTrackingData HTTPS CF with tests"
```

---

## Task 5: Export CF + update safety index

**Files:**
- Modify: `CarpoolBackend/functions/src/functions/safety/index.ts`

- [ ] **Step 1: Add export to safety/index.ts**

Open `CarpoolBackend/functions/src/functions/safety/index.ts` and add:

```ts
export { getTrackingData } from "./getTrackingData.fn.js";
```

The main `CarpoolBackend/functions/src/index.ts` already has `export * from "./functions/safety/index.js"` so no change needed there.

- [ ] **Step 2: Type-check**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit 2>&1
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add functions/src/functions/safety/index.ts
git commit -m "feat(phase-c): export getTrackingData from safety index"
```

---

## Task 6: Static page — HTML + CSS

**Files:**
- Create: `CarpoolBackend/public/track/index.html`
- Create: `CarpoolBackend/public/track/track.css`

- [ ] **Step 1: Create index.html**

```html
<!-- CarpoolBackend/public/track/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="robots" content="noindex, nofollow" />
  <title>Live Tracking — ZygoRide</title>

  <!-- Leaflet -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <!-- Firebase SDK (compat) -->
  <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-database-compat.js"></script>

  <link rel="stylesheet" href="track.css" />
</head>
<body>
  <!-- Banner -->
  <div id="banner" class="banner banner--hidden"></div>

  <!-- Map -->
  <div id="map"></div>

  <!-- Info strip -->
  <div id="info-strip" class="info-strip info-strip--hidden">
    <div class="info-row">
      <div>
        <div class="info-label">DRIVER</div>
        <div id="driver-name" class="info-value">—</div>
        <div id="vehicle-info" class="info-sub">—</div>
      </div>
      <div class="info-right">
        <div class="info-label">UPDATED</div>
        <div id="updated-at" class="info-value">—</div>
      </div>
    </div>
    <a id="emergency-call" class="emergency-btn emergency-btn--hidden" href="#">📞 Call Emergency</a>
  </div>

  <!-- Full-screen overlay for loading / expired / error -->
  <div id="overlay" class="overlay">
    <div class="overlay-content">
      <div id="overlay-icon" class="overlay-icon">⏳</div>
      <div id="overlay-title" class="overlay-title">Loading…</div>
      <div id="overlay-sub" class="overlay-sub"></div>
    </div>
  </div>

  <script src="track.js"></script>
</body>
</html>
```

- [ ] **Step 2: Create track.css**

```css
/* CarpoolBackend/public/track/track.css */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --color-sos: #D93025;
  --color-sos-dark: #B71C1C;
  --color-share: #1A73E8;
  --color-share-dark: #0D47A1;
  --color-bg: #0F172A;
  --color-surface: #1E293B;
  --color-text: #F1F5F9;
  --color-sub: #94A3B8;
  --color-stale: #EF4444;
  --font: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

html, body { height: 100%; background: var(--color-bg); color: var(--color-text); font-family: var(--font); }

/* Banner */
.banner {
  position: fixed; top: 0; left: 0; right: 0; z-index: 1000;
  padding: 12px 16px; text-align: center;
  font-size: 14px; font-weight: 600; letter-spacing: 0.3px;
  transition: background 0.3s;
}
.banner--hidden { display: none; }
.banner--sos { background: var(--color-sos); color: white; }
.banner--share { background: var(--color-share); color: white; }

/* Map */
#map {
  position: fixed; top: 48px; left: 0; right: 0; bottom: 140px;
  z-index: 1;
}

/* Info strip */
.info-strip {
  position: fixed; bottom: 0; left: 0; right: 0; z-index: 1000;
  background: var(--color-surface); padding: 12px 16px;
  border-top: 1px solid rgba(255,255,255,0.08);
  min-height: 100px;
}
.info-strip--hidden { display: none; }

.info-row { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 10px; }
.info-right { text-align: right; }
.info-label { font-size: 10px; text-transform: uppercase; letter-spacing: 1px; color: var(--color-sub); margin-bottom: 2px; }
.info-value { font-size: 15px; font-weight: 600; color: var(--color-text); }
.info-sub { font-size: 12px; color: var(--color-sub); margin-top: 2px; }
#updated-at.stale { color: var(--color-stale); }

/* Emergency button */
.emergency-btn {
  display: block; width: 100%; padding: 10px;
  background: var(--color-sos); color: white;
  border-radius: 8px; text-align: center;
  text-decoration: none; font-weight: 600; font-size: 14px;
}
.emergency-btn--hidden { display: none; }

/* Overlay */
.overlay {
  position: fixed; inset: 0; z-index: 2000;
  background: var(--color-bg);
  display: flex; align-items: center; justify-content: center;
}
.overlay--hidden { display: none; }
.overlay-content { text-align: center; padding: 32px; max-width: 320px; }
.overlay-icon { font-size: 48px; margin-bottom: 16px; }
.overlay-title { font-size: 20px; font-weight: 700; margin-bottom: 8px; }
.overlay-sub { font-size: 14px; color: var(--color-sub); line-height: 1.5; }

/* Leaflet dark overrides */
.leaflet-tile { filter: brightness(0.85) saturate(0.9); }
.leaflet-container { background: var(--color-bg); }
```

- [ ] **Step 3: Commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add public/track/index.html public/track/track.css
git commit -m "feat(phase-c): tracking page HTML shell + CSS"
```

---

## Task 7: Static page — track.js

**Files:**
- Create: `CarpoolBackend/public/track/track.js`

- [ ] **Step 1: Create track.js**

```js
// CarpoolBackend/public/track/track.js
'use strict';

// ── Firebase config (public — not a secret) ──────────────────────────────────
const FIREBASE_CONFIG = {
  apiKey: "AIzaSyDbAbARiyNtsod6VksejudsBE5kRrQHoXY",
  databaseURL: "https://carpool-app-1e668-default-rtdb.firebaseio.com",
  projectId: "carpool-app-1e668",
};

// ── CF endpoint ───────────────────────────────────────────────────────────────
const CF_BASE = "https://us-central1-carpool-app-1e668.cloudfunctions.net/getTrackingData";

// ── State ─────────────────────────────────────────────────────────────────────
const token = location.pathname.split("/track/")[1]?.replace(/\/$/, "") || "";
let map = null;
let marker = null;
let rtdbUnsubscribe = null;
let statusPollTimer = null;
let staleTimer = null;
let lastUpdatedAt = null;

// ── DOM refs ──────────────────────────────────────────────────────────────────
const banner       = document.getElementById("banner");
const mapEl        = document.getElementById("map");
const infoStrip    = document.getElementById("info-strip");
const driverName   = document.getElementById("driver-name");
const vehicleInfo  = document.getElementById("vehicle-info");
const updatedAtEl  = document.getElementById("updated-at");
const emergencyBtn = document.getElementById("emergency-call");
const overlay      = document.getElementById("overlay");
const overlayIcon  = document.getElementById("overlay-icon");
const overlayTitle = document.getElementById("overlay-title");
const overlaySub   = document.getElementById("overlay-sub");

// ── Helpers ───────────────────────────────────────────────────────────────────
function showOverlay(icon, title, sub) {
  overlayIcon.textContent  = icon;
  overlayTitle.textContent = title;
  overlaySub.textContent   = sub;
  overlay.classList.remove("overlay--hidden");
  infoStrip.classList.add("info-strip--hidden");
  banner.classList.add("banner--hidden");
}

function hideOverlay() {
  overlay.classList.add("overlay--hidden");
}

function formatAgo(ms) {
  const s = Math.round((Date.now() - ms) / 1000);
  if (s < 5)  return "just now";
  if (s < 60) return `${s} sec ago`;
  return `${Math.round(s / 60)} min ago`;
}

function initMap(lat, lng) {
  if (map) return;
  map = L.map("map", { zoomControl: true, attributionControl: true })
    .setView([lat, lng], 15);
  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: "© <a href='https://openstreetmap.org'>OpenStreetMap</a>",
    maxZoom: 19,
  }).addTo(map);
}

function updateMarker(lat, lng, isSos) {
  const color = isSos ? "#D93025" : "#1A73E8";
  const html = `<div style="width:16px;height:16px;border-radius:50%;background:${color};
    border:3px solid white;box-shadow:0 0 0 ${isSos ? "6px rgba(217,48,37,0.3)" : "4px rgba(26,115,232,0.3)"};
    animation:${isSos ? "pulse 1.5s infinite" : "none"}"></div>`;
  const icon = L.divIcon({ html, className: "", iconSize: [16, 16], iconAnchor: [8, 8] });

  if (!marker) {
    initMap(lat, lng);
    marker = L.marker([lat, lng], { icon }).addTo(map);
  } else {
    marker.setLatLng([lat, lng]).setIcon(icon);
    map.panTo([lat, lng], { animate: true, duration: 0.5 });
  }
}

function renderMetadata(meta) {
  const isSos = meta.type === "sos";

  // Banner
  banner.textContent = isSos
    ? `🚨 EMERGENCY — ${meta.passengerName} needs help`
    : `🚗 ${meta.passengerName} is sharing their ride`;
  banner.className = `banner ${isSos ? "banner--sos" : "banner--share"}`;

  // Info strip
  driverName.textContent = meta.driverName || "—";
  vehicleInfo.textContent = [meta.vehiclePlate, meta.vehicleModel].filter(Boolean).join(" · ");
  infoStrip.classList.remove("info-strip--hidden");

  // Emergency button (SOS only)
  if (isSos && meta.emergencyNumber) {
    emergencyBtn.href = `tel:${meta.emergencyNumber}`;
    emergencyBtn.textContent = `📞 Call ${meta.emergencyNumber} (Emergency)`;
    emergencyBtn.classList.remove("emergency-btn--hidden");
  }

  // Initial location
  if (meta.location) {
    updateMarker(meta.location.lat, meta.location.lng, isSos);
  }
}

function startStaleCheck(isSos) {
  clearInterval(staleTimer);
  staleTimer = setInterval(() => {
    if (!lastUpdatedAt) return;
    const age = Date.now() - lastUpdatedAt;
    updatedAtEl.textContent = formatAgo(lastUpdatedAt);
    updatedAtEl.classList.toggle("stale", age > 30_000);
    if (age > 120_000) {
      updatedAtEl.textContent = "Location signal lost";
    }
  }, 5_000);
}

function connectRtdb(token, isSos) {
  if (!firebase.apps.length) firebase.initializeApp(FIREBASE_CONFIG);
  const db = firebase.database();
  const ref = db.ref(`liveTracking/${token}`);

  rtdbUnsubscribe = ref.on("value", (snap) => {
    const data = snap.val();
    if (!data) return;
    lastUpdatedAt = data.updatedAt || Date.now();
    updatedAtEl.textContent = formatAgo(lastUpdatedAt);
    updatedAtEl.classList.remove("stale");
    updateMarker(data.lat, data.lng, isSos);
  });

  return () => ref.off("value", rtdbUnsubscribe);
}

function stopTracking() {
  if (rtdbUnsubscribe) { rtdbUnsubscribe(); rtdbUnsubscribe = null; }
  clearInterval(statusPollTimer);
  clearInterval(staleTimer);
}

async function pollStatus(token) {
  try {
    const r = await fetch(`${CF_BASE}?token=${encodeURIComponent(token)}&metadataOnly=true`);
    if (r.status === 410) {
      stopTracking();
      showOverlay("🔒", "This link has expired", "Tracking has ended. If you need help, contact the sender directly.");
    }
  } catch (_) { /* network hiccup — try again next interval */ }
}

// ── Boot ──────────────────────────────────────────────────────────────────────
async function boot() {
  if (!token) {
    showOverlay("❌", "Link not found", "This tracking link is invalid or was never created.");
    return;
  }

  // Check sessionStorage cache for metadata
  const cacheKey = `tracking_meta_${token}`;
  const cached = sessionStorage.getItem(cacheKey);
  if (cached) {
    try {
      const meta = JSON.parse(cached);
      hideOverlay();
      renderMetadata(meta);
      connectRtdb(token, meta.type === "sos");
      startStaleCheck(meta.type === "sos");
      statusPollTimer = setInterval(() => pollStatus(token), 60_000);
      return;
    } catch (_) { sessionStorage.removeItem(cacheKey); }
  }

  // First load: fetch full metadata from CF
  try {
    const r = await fetch(`${CF_BASE}?token=${encodeURIComponent(token)}`);
    const body = await r.json();

    if (r.status === 404) {
      showOverlay("❌", "Link not found", "This tracking link is invalid or was never created.");
      return;
    }
    if (r.status === 410 && body.error === "VIEWER_CAP_EXCEEDED") {
      showOverlay("⚠️", "Too many viewers", "This link has been viewed too many times. Ask the sender for a new link.");
      return;
    }
    if (r.status === 410) {
      showOverlay("🔒", "This link has expired", "Tracking has ended. If you need help, contact the sender directly.");
      return;
    }
    if (!r.ok) {
      showOverlay("⚠️", "Something went wrong", "Please try again in a moment.");
      return;
    }

    // Cache metadata, render, connect RTDB
    sessionStorage.setItem(cacheKey, JSON.stringify(body));
    hideOverlay();
    renderMetadata(body);
    connectRtdb(token, body.type === "sos");
    startStaleCheck(body.type === "sos");
    statusPollTimer = setInterval(() => pollStatus(token), 60_000);

  } catch (_) {
    showOverlay("⚠️", "Connection error", "Check your internet connection and reload the page.");
  }
}

// CSS pulse animation injected inline (avoids extra file)
const style = document.createElement("style");
style.textContent = `@keyframes pulse { 0%,100%{box-shadow:0 0 0 4px rgba(217,48,37,0.4)} 50%{box-shadow:0 0 0 10px rgba(217,48,37,0)} }`;
document.head.appendChild(style);

boot();
```

- [ ] **Step 2: Verify page opens locally (no server needed — just check file is valid JS)**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
node --input-type=module < public/track/track.js 2>&1 | head -5
```

Expected: may error on `document` (browser-only) but NOT on syntax errors. If you see `SyntaxError`, fix it before proceeding.

- [ ] **Step 3: Commit**

```bash
git add public/track/track.js
git commit -m "feat(phase-c): tracking page JS — state machine + RTDB listener"
```

---

## Task 8: Deploy + verify

- [ ] **Step 1: Type-check + lint**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend/functions
npx tsc --noEmit && npm run lint
```

Expected: no errors.

- [ ] **Step 2: Run full unit test suite**

```bash
npx jest tests/unit --no-coverage --runInBand 2>&1 | tail -10
```

Expected: all tests pass.

- [ ] **Step 3: Deploy Firestore indexes**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
firebase deploy --only firestore:indexes
```

Expected: `✔ Deploy complete!`

- [ ] **Step 4: Deploy getTrackingData CF**

```bash
firebase deploy --only functions:getTrackingData
```

Expected: `✔ Deploy complete!` — verify function appears in Firebase Console → Functions.

- [ ] **Step 5: Deploy hosting**

```bash
firebase deploy --only hosting
```

Expected: `✔ Deploy complete!` — hosting URL printed in output.

- [ ] **Step 6: Verify security headers**

```bash
curl -sI "https://carpool-app-1e668.web.app/track/testtoken" | grep -E "X-Robots|Cache-Control|X-Frame"
```

Expected output contains:
```
x-robots-tag: noindex, nofollow
cache-control: no-store, max-age=0
x-frame-options: DENY
```

- [ ] **Step 7: Smoke-test CF with invalid token**

```bash
curl -s "https://us-central1-carpool-app-1e668.cloudfunctions.net/getTrackingData?token=invalid123" | python3 -m json.tool
```

Expected: `{ "error": "NOT_FOUND" }` with HTTP 404.

- [ ] **Step 8: Smoke-test CF with missing token**

```bash
curl -s -w "\nHTTP %{http_code}" "https://us-central1-carpool-app-1e668.cloudfunctions.net/getTrackingData"
```

Expected: `{ "error": "MISSING_TOKEN" }` with HTTP 400.

- [ ] **Step 9: Open tracking page in mobile browser**

Navigate to `https://carpool-app-1e668.web.app/track/testtoken` on iOS Safari or Chrome Android. Expected: overlay shows "Link not found" (correct — no real token). No JS errors in console.

- [ ] **Step 10: Update todo.md Phase C acceptance criteria**

Mark all Phase C acceptance items as `[x]` in `/Users/sairajaggani/Desktop/Space/Project/todo.md`.

- [ ] **Step 11: Final commit**

```bash
cd /Users/sairajaggani/Desktop/Space/Project/CarpoolBackend
git add -A
git commit -m "feat(phase-c): deploy getTrackingData CF + public tracking page"
```
