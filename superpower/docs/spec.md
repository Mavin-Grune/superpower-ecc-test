# 2FA Design Spec

**Date:** 2026-03-11
**Project:** nextjs15-starter-kit (Next.js 15 + NestJS 11 monorepo)
**Status:** Approved

---

## Overview

Add TOTP-based two-factor authentication (2FA) that is **required for all users**. After entering valid credentials, every user must complete a TOTP challenge before receiving a full session. Users who have not yet set up 2FA are forced through an enrollment wizard before accessing the application.

### Decisions made

| Question | Decision |
|---|---|
| Who must use 2FA? | All users — no exceptions |
| Method | TOTP only (Google Authenticator, Authy, etc.) |
| Recovery | Backup codes (10, single-use) + admin reset |
| First-time enrollment | Forced enrollment gate on first login |
| Trusted devices | No — TOTP required on every login |
| Auth approach | Stateless partial-auth JWT |

---

## Auth Flow

```
POST /auth/credentials
        │
        ▼
 Credentials valid?
        │
   ┌────┴─────────────────────────────────────┐
   │ No                                       │ Yes → issue partial JWT (5 min)
   ▼                                          ▼
401 Invalid                          2FA enrolled?
                                              │
                              ┌───────────────┴───────────────┐
                              │ No                            │ Yes
                              ▼                               ▼
                     /2fa/setup (3 steps)           /2fa/verify
                     1. Scan QR code                Enter 6-digit TOTP
                     2. Confirm code                 or backup code
                     3. Save backup codes
                              │                               │
                              └───────────────┬───────────────┘
                                              ▼
                                   Issue full session JWT
                                   (sessionToken → NextAuth)
                                              │
                                              ▼
                                   ✅ Authenticated — access app
```

### Partial JWT

After valid credentials, the backend always issues a short-lived **partial JWT** (5 minutes):

```json
{
  "sub": "userId",
  "twoFactorPending": true,
  "enrollmentRequired": true | false,
  "type": "partial",
  "exp": "now + 5 minutes"
}
```

A `PartialJwtGuard` ensures partial tokens can only access `/auth/2fa/*` endpoints. Full session tokens are rejected on those same endpoints. There is no cross-contamination between token types.

---

## Database Schema

### Changes to `users` table

Two new columns added via a single Drizzle migration:

```typescript
twoFactorSecret:  varchar(255).default(null)  // AES-256-GCM encrypted TOTP secret
twoFactorEnabled: boolean.default(false)       // true after enrollment confirmed
```

`twoFactorSecret` is nullable. `twoFactorEnabled` defaults to `false`. Existing users are unaffected until their next login.

### New `user_backup_codes` table

```typescript
id:        bigint (autoincrement, primary key)
userId:    bigint (foreign key → users.id, cascade delete)
codeHash:  varchar(255)       // bcrypt-hashed — never stored plain
usedAt:    timestamp | null   // null = unused; set atomically on use
createdAt: timestamp (defaultNow)
updatedAt: timestamp (defaultNow)
```

> **Convention note:** This table intentionally omits `deletedAt`. Backup codes are **hard-deleted** on admin reset (not soft-deleted) because keeping invalidated codes serves no audit purpose and poses a security risk. This is a deliberate exception to the project's soft-delete convention.

Ten rows inserted per user at enrollment. Each code is a random 8-character alphanumeric string formatted as `XXXX-XXXX`. Shown to the user once, then hashed and stored.

**Atomic use:** When redeeming a backup code, the repository must perform an atomic `UPDATE user_backup_codes SET usedAt = NOW() WHERE id = ? AND usedAt IS NULL` and verify `rowsAffected === 1` before accepting the code. A two-step SELECT + UPDATE is not acceptable — concurrent requests could both pass a `usedAt IS NULL` check before either writes.

---

## Backend API Endpoints

### Modified

#### `POST /auth/credentials`
- **Auth:** Public, rate-limited 5/min (unchanged)
- **Request:** `{ username, password, rememberMe }` (unchanged)
- **Response (changed):**
  ```json
  {
    "status": "2FA_REQUIRED" | "ENROLLMENT_REQUIRED",
    "partialToken": "eyJ..."
  }
  ```
- No longer returns a full session directly. Always returns a partial JWT.

---

### New — Auth Module

#### `GET /auth/2fa/setup/qr`
- **Auth:** `PartialJwtGuard` (requires `enrollmentRequired: true` in partial JWT)
- **Response:**
  ```json
  {
    "otpauthUrl": "otpauth://totp/...",
    "qrCodeDataUrl": "data:image/png;base64,...",
    "issuer": "AppName",
    "accountName": "user@email.com",
    "encryptedSecret": "<AES-GCM ciphertext hex>"
  }
  ```
- Generates a fresh TOTP secret. AES-256-GCM encrypts it immediately. Returns the `encryptedSecret` to the client — **never the raw secret**. The raw secret is not persisted to the database yet.
- The `encryptedSecret` is opaque to the client. Because AES-GCM is authenticated encryption, any tampering causes decryption to fail server-side. This prevents substitution attacks.
- Calling this endpoint multiple times before confirmation is safe — each call generates a new secret. Nothing is persisted until `POST /auth/2fa/setup` succeeds.

#### `POST /auth/2fa/setup`
- **Auth:** `PartialJwtGuard` (requires `enrollmentRequired: true` in partial JWT)
- **Request:** `{ code: "123456", encryptedSecret: "<AES-GCM ciphertext hex>" }`
- **Response on success:**
  ```json
  {
    "backupCodes": ["XKCD-4827", ...],
    "sessionToken": "eyJ...",
    "user": { ... }
  }
  ```
- **Response on invalid code:** `401 Unauthorized` — generic message, no hint about what failed. Secret is **not** saved. The user may retry or go back to re-scan.
- Server decrypts `encryptedSecret` using the server-side `TWO_FACTOR_ENCRYPTION_KEY`. If decryption fails (tampered ciphertext), returns `400 Bad Request`.
- If the TOTP code is valid: saves the decrypted secret re-encrypted to `users.twoFactorSecret`, sets `twoFactorEnabled = true`, generates 10 backup codes (bcrypt-hashed, stored in `user_backup_codes`), returns plain-text codes **once only**, issues full session JWT.

#### `POST /auth/2fa/verify`
- **Auth:** `PartialJwtGuard` (requires `enrollmentRequired: false` in partial JWT)
- **Request:** `{ code?: "123456", backupCode?: "XKCD-4827" }` (exactly one field present)
- **Response on success:**
  ```json
  {
    "sessionToken": "eyJ...",
    "user": { ... }
  }
  ```
- **Response on failure:** `401 Unauthorized` — generic message regardless of whether the code, backup code, or rate limit was the reason.
- If `code`: verifies TOTP via otplib with ±1 window (30-second grace) for clock drift.
- If `backupCode`: atomic UPDATE as described in the backup codes schema section. Returns 401 if no matching unused code found.
- Rate-limited: 5 attempts/min.

---

### New — Admin / Users Module

#### `POST /admin/users/:id/2fa/reset`
- **Auth:** `SessionTokenGuard`, admin role required
- **Response:** `{ success: true }`
- Clears `twoFactorSecret` (set to null), sets `twoFactorEnabled = false`, **hard-deletes** all `user_backup_codes` rows for the user, and increments `tokenVersion` (immediately invalidating all active sessions). The user's next login triggers forced re-enrollment.

---

## Frontend Pages

### NextAuth Session Integration (read this first)

#### How the partial session works

The `partialToken` (returned from `POST /auth/credentials`) is stored **only inside the NextAuth JWT** — it is **never exposed in the public session object** (i.e., not returned by `useSession()` or `auth()`). This prevents JavaScript running in the browser from accessing the partial token.

The NextAuth `jwt()` callback stores it internally:
```typescript
// In auth.ts jwt() callback
if (token.twoFactorPending) {
  jwt.partialTokenInternal = user.partialToken  // stored in JWT only
}
```

The NextAuth `session()` callback returns only the safe public fields:
```typescript
// In auth.ts session() callback
session.twoFactorPending = token.twoFactorPending
session.enrollmentRequired = token.enrollmentRequired
// partialTokenInternal is NOT copied to session
```

#### How 2FA pages use the partial token

The `/2fa/verify` and `/2fa/setup` pages use **Next.js Server Actions** to make API calls. The server action reads `partialTokenInternal` from the server-side JWT (via `auth()`) and forwards it as the `Authorization: Bearer` header to the NestJS backend. The partial token never reaches the browser.

```typescript
// Example server action in /2fa/verify
'use server'
async function verifyTotpAction(code: string) {
  const session = await auth()  // reads JWT server-side
  const partialToken = session?.partialTokenInternal
  // forward to backend...
}
```

#### Session upgrade after TOTP

After successful TOTP or backup code verification, the server action receives `{ sessionToken, user }` from the backend and calls NextAuth's `update()` to upgrade the session:

```typescript
await update({
  sessionToken,
  user,
  twoFactorPending: false,
  partialTokenInternal: null  // clear the partial token
})
```

NextAuth replaces the pending JWT with a full authenticated session. No second sign-in call needed.

#### NextAuth type extensions (`auth.d.ts`)

```typescript
interface JWT {
  partialTokenInternal?: string  // never in Session — JWT only
}
interface Session {
  twoFactorPending?: boolean
  enrollmentRequired?: boolean
  // no partialToken here
}
```

#### Middleware changes (`middleware.ts`)

New rule added before existing admin route guards:

```
if session.twoFactorPending === true:
  if enrollmentRequired → redirect to /2fa/setup
  else → redirect to /2fa/verify
  (block all other routes)
```

The `/2fa/setup` and `/2fa/verify` routes are accessible with a pending session. All other protected routes are blocked until the session is upgraded.

---

### New: `/2fa/verify`

Shown after credentials are validated when the user already has 2FA enrolled.

- Single `PinInput` (Mantine) for the 6-digit TOTP code
- "Use a backup code instead" link that switches to a text input for the backup code format (`XXXX-XXXX`)
- Submits via Server Action (reads `partialTokenInternal` server-side, calls `POST /auth/2fa/verify`)
- On success: server action calls `update()` and redirects to app

**File locations:**
```
app/2fa/verify/
├── page.tsx
├── _components/
│   ├── VerifyForm.tsx
│   └── BackupCodeForm.tsx
└── actions/
    └── verifyTotp.action.ts   // server action
```

### New: `/2fa/setup`

3-step wizard shown when enrollment is required.

**Step 1 — Scan QR:**
- Calls `GET /auth/2fa/setup/qr` via server action on mount
- Displays QR image
- Stores `encryptedSecret` in a React `useState` within the wizard (never in URL or atoms)
- "Next" advances to Step 2

**Step 2 — Confirm code:**
- User enters the 6-digit TOTP code
- Calls `POST /auth/2fa/setup` via server action (with `code` + `encryptedSecret` from Step 1 state)
- On invalid code: `401` → show error, stay on Step 2, allow retry
- On expired partial token: `401` from guard → server action returns a `SESSION_EXPIRED` signal → client redirects to `/login`
- On success: stores `backupCodes` in wizard state, advances to Step 3

**Step 3 — Save backup codes:**
- Displays all 10 backup codes from wizard state (React `useState`)
- Warning that codes are shown once only
- "Copy all" button
- "I've saved them" triggers session upgrade via `update()` and redirects to app
- If the user navigates away before completing Step 3, the session remains pending and they are redirected back to `/2fa/setup` on next request — but they cannot retrieve the backup codes. A re-setup (new QR + new codes) is required. This is acceptable: the prior codes were never saved to the DB (enrollment was not completed).

**File locations:**
```
app/2fa/setup/
├── page.tsx
├── _components/
│   ├── SetupWizard.tsx      // owns wizard state (useState)
│   ├── QrStep.tsx
│   ├── ConfirmStep.tsx
│   └── BackupCodesStep.tsx
└── actions/
    ├── getSetupQr.action.ts
    └── confirmSetup.action.ts
```

---

## Admin Panel — Reset 2FA

A **"Reset 2FA"** action button added to the user detail/edit page in the admin panel (alongside existing actions like delete/restore). Clicking it opens a confirmation modal. On confirmation: calls `POST /admin/users/:id/2fa/reset` via the existing service hook pattern. After success, displays a notification and refreshes the user record.

---

## New Libraries

| Package | Location | Purpose |
|---|---|---|
| `otplib` | `apps/api` | TOTP generation and verification (RFC 6238) |
| `qrcode` | `apps/api` | QR code PNG generation from otpauth URL |
| `crypto` | `apps/api` | AES-256-GCM encryption (Node built-in, no install) |

**No new frontend packages.** The 2FA pages use Mantine, React Hook Form + Zod (where applicable), and Server Actions — already in the project.

### New env var

```
TWO_FACTOR_ENCRYPTION_KEY=<32-byte hex string>
```

Required in `apps/api/.env`. Must be added to Docker env template and all `.env.*` files. Generated via `openssl rand -hex 32`.

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Invalid TOTP code on `/2fa/verify` | `401` — generic "Invalid code", no hint about what failed |
| Invalid backup code | `401` — same generic message |
| Already-used backup code | `401` — same generic message |
| Partial JWT expired during `/2fa/verify` | `401` from PartialJwtGuard → server action returns `SESSION_EXPIRED` → client redirects to `/login` with "Session expired, please log in again" |
| Partial JWT expired mid-enrollment (during `/2fa/setup`) | Same as above — `401` on `POST /auth/2fa/setup` → server action signals expiry → client redirects to `/login`. The TOTP secret was not saved (nothing to clean up). |
| Invalid TOTP code during setup (Step 2) | `401` — user stays on Step 2, can retry. Secret not saved. |
| Tampered `encryptedSecret` on `POST /auth/2fa/setup` | `400 Bad Request` — AES-GCM decryption failure |
| Accessing app with pending session | Middleware redirects to `/2fa/verify` or `/2fa/setup` |
| Rate limit exceeded | `429` (NestJS Throttler, same as credentials endpoint) |
| User navigates away before completing Step 3 | Pending session persists; user redirected to `/2fa/setup` on next request; backup codes not retrievable; re-setup required |

---

## Security Considerations

- **Raw secret never exposed:** The TOTP secret is AES-encrypted by the server immediately on generation. The client only ever receives the opaque `encryptedSecret` ciphertext. The raw secret is never transmitted to or from the client.
- **Partial token not in public session:** `partialTokenInternal` is stored only in the NextAuth JWT (HttpOnly cookie), never in the public session object returned to the browser.
- **Clock drift:** otplib validates with a ±1 window (30-second grace period)
- **Replay protection:** TOTP codes are time-limited to 30s; otplib prevents reuse within the same window
- **Secret at rest:** AES-256-GCM encrypted in DB — useless without the encryption key
- **Token scope isolation:** `PartialJwtGuard` rejects partial tokens on all non-2FA routes; full tokens are rejected on 2FA endpoints
- **Backup code atomicity:** Redemption uses atomic UPDATE with `rowsAffected === 1` check to prevent concurrent double-use
- **Rate limiting:** 5 attempts/min on both `/auth/credentials` and `/auth/2fa/verify`
- **Admin reset invalidates sessions:** `tokenVersion` incremented on reset, immediately invalidating all active sessions
- **Backup codes:** bcrypt-hashed at storage, shown in plain text once only, hard-deleted on admin reset

---

## Testing Plan

### Backend E2E (`apps/api/test/`)

- Full enrollment flow: credentials → partial token → QR → confirm → backup codes → full session
- TOTP verify happy path
- Invalid TOTP code returns 401 with generic message
- Expired partial JWT returns 401
- Backup code: use once succeeds; use same code again fails
- Concurrent backup code redemption: only one of two simultaneous requests succeeds
- Rate limit enforced after 5 failed attempts
- Admin reset: clears 2FA, increments tokenVersion, next login forces enrollment
- Tampered `encryptedSecret` returns 400

### Backend Unit

- `TotpVerifyUseCase`: valid code, invalid code, clock drift boundary
- `BackupCodeUseCase`: hash matching, already-used detection, atomic update behavior
- `AesEncryptionService`: encrypt/decrypt round-trip; decryption of tampered ciphertext throws
- `PartialJwtGuard`: accepts partial token on 2FA routes; rejects full token; rejects mismatched `enrollmentRequired`

### Browser (`/test-browser`)

- Login → forced to `/2fa/setup` → complete 3-step wizard → land in app
- Log out → login again → `/2fa/verify` → enter TOTP code → land in app
- Verify backup code flow on `/2fa/verify`
- Verify expired partial session (simulate by waiting or manually expiring) redirects to login with message

---

## Implementation Scope

### Backend (`apps/api`)

- `apps/api/src/database/drizzle/schemas/business/users.schema.ts` — add 2 columns
- `apps/api/src/database/drizzle/schemas/business/user-backup-codes.schema.ts` — new table
- `apps/api/src/database/drizzle/schemas/index.ts` — export new schema
- `apps/api/src/modules/auth/` — new use-cases, guards, endpoints
- `apps/api/src/modules/users/` — admin reset endpoint
- `apps/api/src/common/services/aes-encryption.service.ts` — new shared service
- Drizzle migration generated and applied

### Frontend (`apps/web`)

- `apps/web/src/app/2fa/verify/` — new page + server actions
- `apps/web/src/app/2fa/setup/` — new page + server actions
- `apps/web/src/auth.ts` — handle pending session, keep `partialTokenInternal` out of public session
- `apps/web/src/middleware.ts` — pending session redirect logic
- `apps/web/src/auth.d.ts` — type extensions (JWT and Session)
- `apps/web/src/locales/dictionary/ja/` and `en/` — 2FA translations

### Shared packages

- No changes to `@validations`, `@typings`, or `@ui` required (can be added if reuse emerges during implementation)
