# User Bio — Design Spec

**Date:** 2026-06-15
**Status:** Approved
**Scope:** Add a short user-authored bio to the profile, editable from Edit Profile and visible to other users via the public profile.

---

## 1. Goal

Let a user write a short "About me" blurb (up to 200 characters) that appears on their public profile. The bio humanizes profiles in a trust-driven carpool context — a rider/driver can read a couple sentences before booking.

## 2. Non-Goals (v1)

- No rich text / no links / no profanity filter (rely on the existing report-user flow).
- No bio on `DriverSnapshot` / `PassengerSnapshot` denormalized on rides/bookings.
- No bio on ride-details / booked-ride-details driver cards (users can tap through to public profile).
- No localization of the bio content (user types in their own language; we don't translate).
- No migration script for legacy profiles — defensive read handles missing field.

## 3. Data Model

**File:** `Carpool/src/types/models/profile.types.ts`

Add to `Profile`:

```ts
bio: string;  // 0–200 chars; '' = unset
```

Rules:
- **Required** in the type (no `?`) — backend guarantees an empty string when missing.
- Empty string `''` is the canonical "unset" state.
- Trimmed by backend before storage; max 200 chars **after** trim.
- Legacy profile docs (no `bio` field) are tolerated: frontend uses `profile.bio?.trim()` defensively.

## 4. Backend Changes

### 4.1 Zod schema — `CarpoolBackend/functions/src/middleware/validation.middleware.ts`

Add `bio` to the existing `updateProfile` schema:

```ts
updateProfile: z.object({
  firstName: z.string().min(1).max(50).optional(),
  lastName: z.string().min(1).max(50).optional(),
  phone: commonSchemas.phone.optional(),
  email: z.string().email().optional(),
  bio: z.string().trim().max(200).optional(),
}),
```

`.trim()` runs before `.max(200)` — whitespace can't bypass the limit.
`.optional()` allows partial updates (existing pattern for this CF).
Empty string is explicitly allowed (lets a user clear their bio).

### 4.2 Update CF — `CarpoolBackend/functions/src/functions/profiles/updateProfile.fn.ts`

Mirror the existing conditional pattern:

```ts
if (validatedData.bio !== undefined) {
  updates.bio = validatedData.bio;
}
```

No new rate limiter, no new auth check — bio rides on the existing `updateProfile` flow.

### 4.3 New-profile defaults — three entry points

All three places that produce a fresh profile doc must set `bio: ''`:

1. **`CarpoolBackend/functions/src/services/profile.service.ts`** → `ProfileService.createProfileData()` — the canonical factory. Add `bio: ''` to the returned object.
2. **`CarpoolBackend/functions/src/functions/profiles/createProfile.fn.ts`** — already calls the helper, so no separate change needed once the helper is updated.
3. **`CarpoolBackend/functions/src/functions/auth/onCreate.fn.ts`** — inlines `profileData` directly (does not use the helper). Add `bio: ''` to that inline object.

Existing pre-feature profiles are not backfilled — defensive frontend read (`profile.bio?.trim()`) handles missing fields.

### 4.4 Cloud Functions to redeploy

```
firebase deploy --only functions:updateProfile,functions:createProfile,functions:onUserCreate
```

## 5. Frontend — Edit Profile

**File:** `Carpool/app/update-profile.tsx`

Add to the existing **Basic Information** card, below the Phone input:

- Local state: `const [bio, setBio] = useState('')`.
- Seed in the existing `useEffect`: `setBio(profile.bio || '')`.
- UI:

```tsx
<Text style={styles.label}>About you</Text>
<TextInput
  style={[styles.input, styles.bioInput]}
  value={bio}
  onChangeText={setBio}
  multiline
  numberOfLines={3}
  maxLength={200}
  textAlignVertical="top"
  placeholder="Tell other riders a bit about yourself"
  placeholderTextColor={theme.colors.text.secondary}
/>
<Text style={styles.bioCounter}>{bio.length} / 200</Text>
```

- Styles:
  - `bioInput`: `height: 96`, `paddingTop: theme.spacing.md` (so multiline doesn't center).
  - `bioCounter`: `fontSize: theme.fontSize.xs`, `color: theme.colors.text.secondary`, `alignSelf: 'flex-end'`, `marginTop: theme.spacing.xs`.

- In `handleSave`, include `bio: bio.trim()` in the existing `functionsService.updateProfile({...})` payload. After the CF succeeds, call `updateProfileLocally({ bio: bio.trim() })` so local cache reflects the change immediately.
- Bio is **optional** — empty string is a valid save. Do not block save when `bio` is empty.

## 6. Frontend — Public Profile

**File:** `Carpool/app/public-profile.tsx`

Extend the local `ProfileDoc` interface with `bio?: string`.

In the `ListHeaderComponent`, between the `profileSection` (name) and the `statsRow`, render the bio only when it has content:

```tsx
{profile.bio?.trim() ? (
  <View style={styles.bioSection}>
    <Text style={styles.bioText}>{profile.bio.trim()}</Text>
  </View>
) : null}
```

Styles (added to the existing `StyleSheet.create`):

```ts
bioSection: {
  backgroundColor: theme.colors.surface,
  paddingHorizontal: theme.spacing.xl,
  paddingBottom: theme.spacing.xl,
  marginBottom: theme.spacing.md,
},
bioText: {
  fontSize: theme.fontSize.base,
  color: theme.colors.text.primary,
  lineHeight: 22,
},
```

Notes:
- No section header label — placement under the name is self-explanatory.
- No "Read more" — 200 chars fits in ~3 lines on a phone, no truncation needed.
- When `bio` is empty/missing, the section is fully omitted — no placeholder, no empty card.

## 7. Other Touchpoints

| Surface | Decision | Reason |
|---|---|---|
| `app/tabs/profile.tsx` (own profile tab) | Out of scope | Settings hub, not a preview surface. |
| `DriverSnapshot` / `PassengerSnapshot` on rides/bookings | Out | Snapshots are minimal trust badges. Bumping every ride doc is expensive; users can tap through. |
| `ride-details.tsx` / `booked-ride-details.tsx` driver cards | Out v1 | "View Profile" already routes to public profile. |
| `ProfileContext` | No code change | Passes the full `Profile` through; the type-level change is enough. |
| `updateProfileLocally` call after save | Pass `{ bio: bio.trim() }` | Keeps local cache in sync with the write. |

## 8. Testing

| Layer | Test | Notes |
|---|---|---|
| Backend unit (`updateProfile.fn.test.ts`) | Happy path with `bio`; 200-char boundary accepted; 201-char rejected; empty string clears bio; whitespace-only trims to empty | Extend existing test file. |
| Backend Zod schema | `bio` accepts `''`, accepts 200 chars, rejects 201 chars after trim, rejects non-string | New test or add to existing schema test. |
| Backend integration | Round-trip: write bio → read via `getProfile` returns it | Cover in existing profile integration suite. |
| Frontend | None new — UI test value is low; manual QA on Edit + Public profile is sufficient. |
| Migration | Read a legacy profile doc without `bio` → public profile renders without error | Implicit in defensive `profile.bio?.trim()`. |

## 9. Rollout

1. Type change + backend Zod + CFs — deploy.
2. Frontend (Edit + Public profile) — ship.
3. No feature flag — the field is additive, legacy reads tolerate missing `bio`, empty bios hide cleanly.
4. No comms needed; surface is self-discoverable in Edit Profile.

## 10. Risks

- **Abuse content (PII, slurs, contact info):** Mitigated by 200-char cap (reduces grooming surface) + existing report-user flow. Re-evaluate if abuse reports trend up.
- **Whitespace-only bios bypassing the "empty" check:** Mitigated by `.trim()` both server-side (Zod) and client-side (`bio.trim()` in save + `profile.bio?.trim()` in public render).
- **Stale local cache after save:** Mitigated by the `updateProfileLocally({ bio })` call.

## 11. Files Touched

```
Carpool/src/types/models/profile.types.ts                                    (add bio field)
Carpool/app/update-profile.tsx                                               (bio input + counter)
Carpool/app/public-profile.tsx                                               (bio section + ProfileDoc type)

CarpoolBackend/functions/src/middleware/validation.middleware.ts             (bio in updateProfile schema)
CarpoolBackend/functions/src/functions/profiles/updateProfile.fn.ts          (bio in updates)
CarpoolBackend/functions/src/services/profile.service.ts                     (default bio: '' in createProfileData)
CarpoolBackend/functions/src/functions/auth/onCreate.fn.ts                   (default bio: '' in inline profileData)

CarpoolBackend/functions/tests/unit/profiles/updateProfile.fn.test.ts        (new test cases)
```
