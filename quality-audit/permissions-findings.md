## Permissions — Audit Findings

### Bugs (must fix)

- [BUG-01] Ratings Firestore rule `allow create` permits direct client writes, bypassing ALL CF validation (ride COMPLETED check, 7-day window, duplicate check, role verification). A client can write a rating document directly for any ride/user without restrictions | `CarpoolBackend/firestore.rules:52-53` | Severity: high

- [BUG-02] Chat `allow update` rule allows a participant to zero out the OTHER participant's unread count. `hasOnly(['unreadCounts'])` restricts the top-level key but does not prevent modifying sibling keys within the `unreadCounts` map | `CarpoolBackend/firestore.rules:110-113` | Severity: medium

- [BUG-03] RTDB `driverLocation.write` rule contains `|| !newData.exists()`, which allows ANY authenticated user to delete a driver's location from any active ride. Should be restricted to the driver only | `CarpoolBackend/database.rules.json` (driverLocation write) | Severity: low

- [BUG-04] `chatId` max-length inconsistency across validation schemas: `sendMessage` and `blockUser` cap at 100; `markRead` and `deleteChat` cap at 200. Actual chatId (`${rideId}_${passengerId}`) is ~55 chars — pick one limit (100) and enforce it consistently | `CarpoolBackend/functions/src/middleware/validation.middleware.ts:257,274,279,284` | Severity: low

### Improvements (should fix)

- [IMP-01] `notifications allow update` permits updating ANY field of a user's own notification. Intent is mark-as-read only — restrict to `isRead` field via `affectedKeys().hasOnly(['isRead'])` | `CarpoolBackend/firestore.rules:62` | Impact: data integrity

### Notes

- Firestore collections without rules (disputes, adminActions, mapsCache, supportTickets, idempotencyKeys) correctly default to deny-all — admin SDK bypasses rules so CFs work fine.
- `authenticateUser` calls `auth.getUser(uid)` on every CF invocation — intentional design choice for immediate revocation support (vs. stale JWT claims). Tradeoff is acceptable at carpool scale.
- `checkAdminStatus` hits Firestore per call — same rationale; allows real-time admin management without token refresh.
- RTDB `liveTracking` read rule allows authenticated users to read expired tokens — in practice harmless since expired tracking data is cleaned up by scheduler.
- Profile phone allows optional `+` (`/^\+?[1-9]\d{1,14}$/`) while trusted contacts enforce strict E.164. Profile phone is display-only (not used for WhatsApp/SOS). Inconsistency is acceptable.
