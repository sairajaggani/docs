# Phase C — Public Tracking Web Page Design

**Date:** 2026-06-10  
**Status:** Approved  
**Depends on:** Phase A (trustedContacts), Phase B (liveTracking RTDB + battery service)

---

## 1. Goal

Anyone holding a `shareToken` URL can see the sharer's live location on a map — no login, no app install required. Used by:
- **SOS recipients** — trusted contacts who receive an emergency alert
- **Ride-share recipients** — anyone the passenger voluntarily shares their ride with

---

## 2. URL

```
https://carpool-app-1e668.web.app/track/{token}
```

Custom domain deferred. The token is the only authentication — treat it as a bearer secret.

---

## 3. Architecture

### 3.1 Data-fetching strategy (hybrid)

| Step | Mechanism | Purpose |
|---|---|---|
| Page load | `GET /getTrackingData?token={token}` (HTTPS CF) | Validate token, get metadata, enforce viewer cap, return driver info |
| Live location | Firebase JS SDK — `rtdb.ref('liveTracking/{token}').on('value')` | Real-time location updates, no polling |
| Status monitoring | Re-call `getTrackingData` CF every 60s | Detect expiry / SOS resolved, disconnect listener |

Metadata (driverName, vehiclePlate, type, emergencyNumber, expiresAt) is cached in `sessionStorage` after the first CF call. Subsequent CF calls (every 60s) fetch only `{ status, expiresAt }` — the page passes `metadataOnly=true` query param.

The Firebase config (apiKey, databaseURL, projectId) is hardcoded in `track.js` — this is standard practice; Firebase security is enforced by rules, not by hiding config.

### 3.2 Cost profile

| Scale | CF invocations | Firestore reads | RTDB bandwidth | Monthly total |
|---|---|---|---|---|
| 1k rides/day | Free ✅ | Free ✅ | Free ✅ | ~$0 |
| 10k rides/day | Free ✅ | Free ✅ | ~0.72/mo | ~$0.72 |
| 100k rides/day | ~$1.50/mo | ~$0.30/mo | ~$7/mo | ~$9/mo |

Primary cost driver at scale: RTDB bandwidth. Viewer cap of 50 per token prevents runaway sessions.

---

## 4. `getTrackingData` Cloud Function

### Contract

- **Type:** HTTPS (not callable — no Firebase Auth header required)
- **Method:** GET
- **URL:** `https://{region}-carpool-app-1e668.cloudfunctions.net/getTrackingData?token={token}&metadataOnly=true`

### Full response (first call)

```ts
{
  type: 'sos' | 'share',
  status: 'ACTIVE' | 'RESOLVED' | 'RESOLVED_TIMEOUT' | 'EXPIRED',
  passengerName: string,        // first name + last initial only (e.g. "Saira R.")
  driverName: string,           // first name + last initial
  vehiclePlate: string,
  vehicleModel: string,
  location: { lat: number, lng: number, accuracy: number, speed: number | null, bearing: number | null },
  updatedAt: number,            // ms timestamp
  triggeredAt: number,          // for SOS: when alert was triggered
  expiresAt: number,            // ms timestamp
  emergencyNumber?: string,     // only present when type === 'sos'; e.g. "112", "911" — from sosAlert.country
}
```

### Metadata-only response (`metadataOnly=true`)

```ts
{ status: 'ACTIVE' | 'RESOLVED' | 'RESOLVED_TIMEOUT' | 'EXPIRED', expiresAt: number }
```

### Error responses

| Code | Condition | Body |
|---|---|---|
| 404 | Token not found in sosAlerts or rideShares | `{ error: 'NOT_FOUND' }` |
| 410 | Status is RESOLVED / EXPIRED | `{ error: 'EXPIRED', resolvedAt: number }` |
| 410 | Viewer cap exceeded (> 50) | `{ error: 'VIEWER_CAP_EXCEEDED' }` |
| 429 | Rate limit: > 60 req/min per token | `{ error: 'RATE_LIMITED' }` |

### Viewer cap enforcement

On the first call (no `metadataOnly` param), atomically increment `viewerCount` on the Firestore document using a transaction. If the post-increment value exceeds 50, return 410 without decrementing — the cap is permanent per token.

### Lookup strategy

Parallel `Promise.all([sosAlertsQuery, rideSharesQuery])` — whichever resolves first wins. Uses `where('shareToken', '==', token)` on both collections. A composite index on `(shareToken)` in `firestore.indexes.json` is required for both collections (single-field indexes are auto-created by Firestore, but adding explicitly ensures it).

### Rate limit

60 requests/min per token (stored in RTDB rate-limit bucket under the token hash, not the caller IP — IP rotation cannot defeat this).

---

## 5. Static Page

### Files

```
CarpoolBackend/public/track/
  index.html    — shell, Leaflet CDN, Firebase SDK CDN, meta tags, CSP
  track.js      — state machine, CF fetch, RTDB listener, map render
  track.css     — mobile-first styles
```

### index.html — key requirements

- `<meta name="viewport" content="width=device-width, initial-scale=1">`
- `<meta name="robots" content="noindex, nofollow">`
- Leaflet CSS + JS from CDN (`unpkg.com/leaflet`)
- Firebase App + Database SDK from CDN (`www.gstatic.com/firebasejs`)
- No other JS frameworks

### track.js — state machine

```
INIT → LOADING → ACTIVE_SOS | ACTIVE_SHARE → EXPIRED | ERROR
                     ↑                              |
                     └──── 60s status poll ─────────┘
```

States:
1. **LOADING** — spinner, no map
2. **ACTIVE_SOS** — red banner, pulsing red marker, "Call {emergencyNumber}" `tel:` link, driver info strip, stale indicator
3. **ACTIVE_SHARE** — blue banner, blue marker, driver info strip, stale indicator
4. **EXPIRED** — lock icon, "This link has expired" message, disconnect RTDB listener
5. **ERROR_NOT_FOUND** — "Link not found" message
6. **ERROR_VIEWER_CAP** — "Too many viewers — ask sender for a new link" message

### Stale indicator

On every RTDB update, record `lastUpdatedAt = Date.now()`. A 5s interval checks `Date.now() - lastUpdatedAt`. If > 30s, the "Updated X ago" text turns red. If > 120s, show "Location signal lost".

### sessionStorage caching

```js
// On first CF response
sessionStorage.setItem(`tracking_meta_${token}`, JSON.stringify(metadata));

// On subsequent 60s polls — use metadataOnly=true, skip metadata re-fetch
```

---

## 6. Firebase Hosting Configuration

### firebase.json additions

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

---

## 7. Layout (mobile-first, approved)

**Map-first stacked (option A):**
```
┌─────────────────────────┐
│  Banner (SOS red / share blue)  │
├─────────────────────────┤
│                         │
│    Leaflet map          │
│    (height: 55vh)       │
│                         │
├─────────────────────────┤
│  Driver name · Plate    │
│  Updated X sec ago      │
│  [Call 112] (SOS only)  │
└─────────────────────────┘
```

---

## 8. Files Changed / Created

| File | Change |
|---|---|
| `CarpoolBackend/public/track/index.html` | **New** — static shell |
| `CarpoolBackend/public/track/track.js` | **New** — state machine + RTDB listener + CF fetch |
| `CarpoolBackend/public/track/track.css` | **New** — mobile-first styles |
| `CarpoolBackend/functions/src/functions/safety/getTrackingData.fn.ts` | **New** — HTTPS CF |
| `CarpoolBackend/functions/src/functions/safety/index.ts` | **Update** — export getTrackingData |
| `CarpoolBackend/functions/src/index.ts` | **Update** — export getTrackingData |
| `CarpoolBackend/functions/src/config/constants.ts` | **Update** — GET_TRACKING_DATA rate limit, VIEWER_CAP |
| `CarpoolBackend/functions/src/middleware/rate-limit.middleware.ts` | **Update** — getTrackingData limiter |
| `CarpoolBackend/firebase.json` | **Update** — hosting config + headers |
| `CarpoolBackend/firestore.indexes.json` | **Update** — shareToken index on sosAlerts + rideShares |

---

## 9. Tests

### CF unit tests (`getTrackingData.test.ts`)

- Token found in sosAlerts → returns SOS metadata
- Token found in rideShares → returns share metadata
- Token not found → 404
- Status RESOLVED → 410 EXPIRED
- Viewer count at 50 → 410 VIEWER_CAP_EXCEEDED
- Viewer count at 49 → 200, increments to 50
- `metadataOnly=true` → returns only `{ status, expiresAt }`
- Rate limit: 61st request within 60s → 429

### Phase C acceptance criteria (from todo.md)

- [ ] Hosting deployed: `firebase deploy --only hosting`
- [ ] `getTrackingData` deployed: `firebase deploy --only functions:getTrackingData`
- [ ] `X-Robots-Tag: noindex` confirmed in response headers (`curl -I`)
- [ ] Page renders correctly on iOS Safari + Chrome Android
- [ ] Stale data indicator turns red after >30s
- [ ] RTDB listener disconnects when status transitions to EXPIRED/RESOLVED
- [ ] Viewer cap: 51st unique session on same token returns 410

---

## 10. Security notes

- Token is the only auth — never log it in CF logs (hash before logging)
- `driverName` / `passengerName` returned as first name + last initial only
- Viewer cap prevents enumeration by exhausting tokens
- `X-Frame-Options: DENY` prevents clickjacking
- Security finding Phase A #4 (name escaping for web): driver/passenger names must be set via `textContent`, never `innerHTML`
