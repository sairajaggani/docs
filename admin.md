# ZygoRIde Admin Panel

## Tech Stack

| Layer | Library |
|---|---|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| UI components | shadcn/ui (Radix primitives + Tailwind) |
| Data fetching | TanStack React Query v5 |
| Firebase client | Firebase JS SDK (Auth + Functions) |
| Toast | Sonner |

All data mutations and reads go through Firebase Cloud Functions. The admin panel **does not** read Firestore directly.

---

## Authentication

**How it works:**

1. Admin visits `/login` and submits email + password.
2. `signInWithEmailAndPassword` from Firebase Auth is called.
3. Immediately after sign-in, `getDashboardStats` Cloud Function is called to verify admin privileges (the function checks `admins/{uid}` Firestore doc).
4. If the call succeeds → `isAdmin = true`, redirect to dashboard.
5. If the call fails → sign out and show "Not authorised" error.

**Session persistence:** `onAuthStateChanged` listener in `AuthContext` re-verifies admin access on every page load (calls `getDashboardStats` unless the `skipNextAuthCheck` flag is set to avoid duplicate verification after fresh sign-in).

**Route protection:** The `(admin)` layout checks `isAdmin` and redirects unauthenticated users to `/login`. Non-admin Firebase users are rejected at the function level.

---

## Pages

### Dashboard — `app/(admin)/page.tsx`

**Purpose:** At-a-glance platform health metrics.

**Displays:**
- 7 stat cards (auto-refreshes every 60 seconds): Total Users, Suspended Users, Open Rides, Active Rides, Pending Verifications, Open Tickets, Bookings (last 7 days)

**Components:** `StatsCards`

**API calls:** `getDashboardStats` (React Query, 60s refetch interval)

---

### Users — `app/(admin)/users/`

**Purpose:** Search and manage user accounts.

**Displays:**
- Search form: query string + field selector (email | phone | uid)
- Results table: name, email, phone, account status (badge), ID verified (badge), created date, actions menu

**Actions per user:**
- **Suspend** — opens `SuspendDialog` → calls `suspendUser({ targetUid, action: 'suspend', reason })`
- **Unsuspend** — calls `suspendUser({ targetUid, action: 'unsuspend' })`
- **Delete** — calls `suspendUser({ targetUid, action: 'delete' })`

**Components:** `SearchForm`, `UserTable`, `SuspendDialog`

**API calls:** `searchUsers({ query, field, limit })`, `suspendUser(...)`

---

### Verify — `app/(admin)/verify/`

**Purpose:** Review and approve or reject user ID verification submissions.

**Displays:**
- Status filter tabs (Pending | Approved | Rejected | All)
- Queue table: name, email, phone, status badge, ID document preview button, created date, action buttons
- **View ID** button — opens `IdViewer` dialog, which calls `decryptIDDocument` to obtain a short-lived signed GCS URL, then renders the document in an `<img>` tag. Requires admin to enter a password (verified server-side).
- **Approve** / **Reject** buttons — open `VerifyDialog` for confirmation + optional rejection reason

**Actions:**
- **Approve** — calls `verifyIDDocument({ targetUid, action: 'approve' })`
- **Reject** — calls `verifyIDDocument({ targetUid, action: 'reject', reason })`

**Components:** `QueueTable`, `VerifyDialog`, `IdViewer`, `DecryptPasswordDialog`

**API calls:** `getVerificationQueue(...)`, `verifyIDDocument(...)`, `decryptIDDocument(...)`

---

### Tickets — `app/(admin)/tickets/`

**Purpose:** View and resolve support tickets submitted by users.

**Displays:**
- Status filter (Open | Resolved | All), category filter
- Tickets table: ticket ID, user name/email, category, description snippet, platform, status badge, created date, resolve button

**Actions:**
- **Resolve** — opens `ResolveDialog`, admin enters resolution text + optional notes → calls `resolveTicket({ ticketId, resolution, adminNotes })`

**Components:** `TicketTable`, `ResolveDialog`

**API calls:** `getSupportTickets(...)`, `resolveTicket(...)`

---

## API Layer — `src/lib/api.ts`

All functions use a `callable<Req, Res>(name)` factory that wraps `httpsCallable` and:
- Strips `undefined` values before serialisation (Firebase rejects `null` on optional Zod fields)
- Unwraps the `CFResponse<T>` wrapper to return `data` directly

| Export | Cloud Function | Parameters | Return |
|---|---|---|---|
| `getDashboardStats` | `getDashboardStats` | `{}` | `DashboardStats` |
| `searchUsers` | `adminSearchUsers` | `{ query, field, limit? }` | `UserResult[]` |
| `getVerificationQueue` | `getVerificationQueue` | `{ status?, cursor?, limit? }` | `CursorPaginated<VerificationItem>` |
| `verifyIDDocument` | `verifyIDDocument` | `{ targetUid, action, reason? }` | `VerifyResult` |
| `suspendUser` | `suspendUser` | `{ targetUid, action, reason? }` | `SuspendResult` |
| `getSupportTickets` | `getSupportTickets` | `{ status?, category?, cursor?, limit? }` | `CursorPaginated<SupportTicket>` |
| `resolveTicket` | `resolveTicket` | `{ ticketId, resolution, adminNotes? }` | `ResolveResult` |
| `decryptIDDocument` | `decryptIDDocument` | `{ userId, adminPassword }` | `DecryptIdResult` |

---

## Key Types — `src/types/admin.ts`

```typescript
DashboardStats          { totalUsers, suspendedUsers, openRides, activeRides,
                          pendingVerifications, openTickets, bookingsLast7Days }

UserResult              { uid, firstName, lastName, email, phone, avatarUrl,
                          accountStatus, idVerified, idVerificationStatus,
                          driverStats, passengerStats, createdAt, updatedAt,
                          suspendedAt, suspensionReason }

VerificationItem        { uid, firstName, lastName, email, phone, avatarUrl,
                          hasIdDocument, idVerified, idVerificationStatus,
                          idRejectionReason, accountStatus, createdAt, updatedAt }

SupportTicket           { ticketId, uid, userName, userEmail, userPhone, category,
                          description, screenshotUrls, platform, appVersion,
                          status, adminNotes, resolution, resolvedBy, resolvedAt,
                          createdAt, updatedAt }

CursorPaginated<T>      { items: T[], nextCursor: string | null }

VerifyResult            { targetUid, action, message }
SuspendResult           { targetUid, action, ridesCancelled?, bookingsCancelled?, message }
ResolveResult           { ticketId, message }
DecryptIdResult         { signedUrl, expiresAt, userName, idVerified }

CFResponse<T>           { success: boolean, data: T }
```

---

## Component Breakdown

### Layout — `src/components/layout/`

| Component | Purpose |
|---|---|
| `header.tsx` | Top bar: title, sign-out button |
| `sidebar.tsx` | Nav links: Dashboard, Users, Verify, Tickets |

### Dashboard — `src/components/dashboard/`

| Component | Purpose |
|---|---|
| `stats-cards.tsx` | Grid of 7 stat cards with React Query auto-refresh |

### Users — `src/components/users/`

| Component | Purpose |
|---|---|
| `search-form.tsx` | Query input + field dropdown |
| `user-table.tsx` | Table with status badges + actions dropdown (suspend/unsuspend/delete) |
| `suspend-dialog.tsx` | Confirmation modal with optional reason input |

### Verify — `src/components/verify/`

| Component | Purpose |
|---|---|
| `queue-table.tsx` | Table with status badges, Eye/Check/X action buttons |
| `verify-dialog.tsx` | Approve/reject confirmation with optional rejection reason |
| `id-viewer.tsx` | Dialog that renders the decrypted ID document image |
| `decrypt-password-dialog.tsx` | Password input before decrypting an ID |

### Tickets — `src/components/tickets/`

| Component | Purpose |
|---|---|
| `ticket-table.tsx` | Table with category/status/description, resolve button |
| `resolve-dialog.tsx` | Resolution text + admin notes input |

### Shared UI — `src/components/ui/`

shadcn/ui primitives: `avatar`, `badge`, `button`, `card`, `dialog`, `dropdown-menu`, `input`, `select`, `skeleton`, `table`, `tabs`, `sonner` (toasts).

---

## Firebase Connection

The admin panel connects to Firebase via `src/lib/firebase.ts` (Firebase JS SDK, web config). It uses:

- **Firebase Auth** — session management, `onAuthStateChanged`
- **Firebase Functions** — all data operations via `httpsCallable`

No direct Firestore reads or writes are made from the admin panel. The `admins/{uid}` collection is the sole access gate, checked by Cloud Functions, not by the frontend.
