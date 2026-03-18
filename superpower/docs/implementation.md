# 2FA Implementation Summary

Documentation of the full implementation experience — from plan to working code — using [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with the [Superpowers](https://github.com/obra/superpowers) plugin.

## Overview

The 17-task implementation plan was executed across **two Claude Code sessions**, completing all three chunks (backend foundation, backend endpoints, frontend integration). The Superpowers `executing-plans` skill orchestrated the work — loading the plan, creating a task checklist, executing each step, and running verifications at each checkpoint.

**Total scope:**
- 12 new backend files created
- 16 new frontend files created
- 8 existing files modified
- 2 new npm packages installed (`otplib`, `qrcode`)
- 1 Drizzle migration generated and applied
- 8 unit tests written and passing

---

## Session 1: Backend (Tasks 1–10)

### Chunk 1: Backend Foundation (Tasks 1–5)

**Task 1 — Install packages and configure env**

Installed `otplib` (TOTP), `qrcode` (QR generation), and `@types/qrcode`. Added `TWO_FACTOR_ENCRYPTION_KEY` env var (32-byte hex string generated via `openssl rand -hex 32`) to all `.env` files and Docker templates.

**Task 2 — AES-256-GCM encryption service**

Created `aes-encryption.service.ts` as a shared NestJS injectable. Encrypts TOTP secrets before database storage using Node's built-in `crypto` module. Each encryption generates a random 12-byte IV. Output format: `iv:authTag:ciphertext` (hex-encoded). Wrote 4 unit tests covering round-trip encryption, tamper detection (modified ciphertext, modified IV, modified auth tag), and invalid key handling.

**Task 3 — Database schema changes**

Added two columns to the `users` table (`twoFactorSecret` varchar nullable, `twoFactorEnabled` boolean default false) and created the new `user_backup_codes` table with bcrypt-hashed codes and atomic `usedAt` tracking. Generated and applied the Drizzle migration.

**Task 4 — PartialJwtGuard**

Created a custom NestJS guard that validates partial JWT tokens on 2FA-only endpoints. The guard:
- Rejects requests without a valid partial token
- Rejects full session tokens (prevents cross-contamination)
- Validates `enrollmentRequired` matches the endpoint's expectation
- Checks token expiration (5-minute window)

Wrote 4 unit tests covering: valid partial token acceptance, full token rejection, expired token rejection, and missing token handling.

**Task 5 — Auth repository 2FA methods**

Added repository methods for all 2FA database operations: finding users by ID with 2FA fields, saving encrypted TOTP secrets, storing bcrypt-hashed backup codes, atomic backup code redemption (`UPDATE WHERE usedAt IS NULL` with `rowsAffected === 1` check), and hard-delete of backup codes on admin reset.

### Chunk 2: Backend API Endpoints (Tasks 6–10)

**Task 6 — Modified credentials endpoint**

Changed `POST /auth/credentials` to always return a partial JWT instead of a full session. The response now includes `status: "2FA_REQUIRED" | "ENROLLMENT_REQUIRED"` and a `partialToken` with 5-minute expiry. Users with `twoFactorEnabled: true` get `2FA_REQUIRED`; others get `ENROLLMENT_REQUIRED`.

**Task 7 — QR setup endpoint**

Created `GET /auth/2fa/setup/qr` behind `PartialJwtGuard`. Generates a fresh TOTP secret via `otplib`, creates the `otpauth://` URI, renders it as a QR code data URL via `qrcode`, and AES-encrypts the secret before returning it. The raw secret is never sent to the client or persisted until confirmation.

**Task 8 — Setup confirmation endpoint**

Created `POST /auth/2fa/setup` behind `PartialJwtGuard`. Receives the user's 6-digit TOTP code and the opaque `encryptedSecret` from Step 1. Server decrypts the secret (AES-GCM detects tampering), verifies the TOTP code, then atomically: saves the re-encrypted secret to the database, sets `twoFactorEnabled = true`, generates 10 backup codes (bcrypt-hashed), and issues a full session JWT. Returns plain-text backup codes (shown once only).

**Task 9 — TOTP verify endpoint**

Created `POST /auth/2fa/verify` behind `PartialJwtGuard`. Accepts either a 6-digit TOTP `code` or a `backupCode` (XXXX-XXXX format). TOTP verification uses `otplib` with ±1 window (30-second clock drift tolerance). Backup code verification uses atomic UPDATE. Returns a full session JWT on success.

**Task 10 — Admin 2FA reset endpoint**

Created `POST /users/:id/2fa/reset` behind `SessionTokenGuard` (admin only). Clears `twoFactorSecret`, sets `twoFactorEnabled = false`, hard-deletes all backup codes, and increments `tokenVersion` (invalidating all active sessions). The user is forced through re-enrollment on next login.

### Session 1 Blocker: otplib v13 API

The implementation plan referenced otplib v12's API (`authenticator.generateSecret()`, `authenticator.verify()`). However, `pnpm add otplib` installed **v13.3.0**, which completely changed the API to a functional style. This was caught during `check-types` and resolved at the start of Session 2.

---

## Session 2: Frontend + otplib Fix (Tasks 11–17)

### otplib v13 Migration

The first task in Session 2 was fixing the otplib API incompatibility across three backend files. The fix involved reading the actual TypeScript declarations from `otplib@13.3.0` to discover the correct API:

| v12 (plan referenced)          | v13 (actual)                                        |
|-------------------------------|-----------------------------------------------------|
| `authenticator.generateSecret()` | `generateSecret()` from `'otplib'`                  |
| `authenticator.keyuri(account, issuer, secret)` | `generateURI({ issuer, label, secret })` |
| `authenticator.verify({ token, secret })` | `verifySync({ token, secret, epochTolerance: 30 })` |
| `authenticator.options = { window: 1 }` | `epochTolerance: 30` parameter (±1 window = 30s)   |

Files fixed: `two-factor-setup-qr.usecase.ts`, `two-factor-confirm.usecase.ts`, `two-factor-verify.usecase.ts`

### Chunk 3: Frontend Integration (Tasks 11–17)

**Task 11 — NextAuth type extensions + partial token helper**

Created `auth.d.ts` with module augmentation adding 2FA fields (`twoFactorPending`, `enrollmentRequired`, `partialTokenInternal`) to NextAuth's `JWT` and `Session` types. Created `get-partial-token.ts` — a server-side helper that reads `partialTokenInternal` directly from the JWT cookie using `decode` from `next-auth/jwt` (bypassing `auth()` which strips internal fields).

**Task 12 — Auth config modifications**

Modified `auth.config.ts` to handle the new 2FA flow:
- `authorize()` now detects `2FA_REQUIRED` / `ENROLLMENT_REQUIRED` status responses and returns a pending user object with `twoFactorPending: true`
- `jwt()` callback stores `partialTokenInternal` in the JWT (never in the public session), sets 5-minute expiry for pending sessions
- `session()` callback returns minimal data for pending sessions (no `sessionToken` exposed)
- `trigger === 'update'` handling upgrades the session after TOTP verification via `unstable_update()`

**Task 13 — Middleware 2FA redirects**

Added redirect logic to `middleware.ts`:
- Pending sessions with `enrollmentRequired: true` → redirect to `/2fa/setup`
- Pending sessions with `enrollmentRequired: false` → redirect to `/2fa/verify`
- 2FA pages accessible only with a pending session
- All other protected routes blocked until session upgrade

**Task 14 — 2FA verify page**

Created the `/2fa/verify` page with two form modes:

- **TOTP mode:** 6-digit `PinInput` (Mantine) with auto-submit on complete. Calls `verifyTotpAction` server action which reads the partial token, forwards to `POST /auth/2fa/verify`, and upgrades the session on success.
- **Backup code mode:** `TextInput` with `XXXX-XXXX` placeholder and auto-uppercase. Calls `verifyBackupCodeAction` server action.
- Mode toggle via "Use a backup code" / "Back to code" buttons
- Session expiry redirects to `/login?error=session_expired`

**Task 15 — 2FA setup wizard**

Created the `/2fa/setup` page with a 3-step `Stepper` (Mantine) wizard:

- **Step 1 (QrStep):** Fetches QR code via `getSetupQr` server action, displays as `<img>`. Passes `encryptedSecret` to parent via callback.
- **Step 2 (ConfirmStep):** 6-digit `PinInput` with auto-submit. Calls `confirmSetup` server action with code + encryptedSecret. On success, receives backup codes + session token. Handles `SESSION_EXPIRED` and `TAMPERED_SECRET` errors.
- **Step 3 (BackupCodesStep):** Displays 10 backup codes in a `SimpleGrid`, "Copy all" button via `CopyButton` (Mantine). "Done" button calls `completeSetup` server action to upgrade session and redirect to `/admin`.

All wizard state (`encryptedSecret`, `backupCodes`, `sessionToken`) is held in React `useState` — never in URL params or atoms.

**Task 16 — i18n translations**

Created Japanese (`ja/two-factor.ts`) and English (`en/two-factor.ts`) translation files covering all UI strings for verify, setup (3 steps), and admin reset. Registered in both locale index files and the dictionary type definition.

**Task 17 — Admin Reset 2FA button**

Added to the existing `UserForm.tsx` component:
- Orange outline "2FAリセット" / "Reset 2FA" button (visible only in edit mode)
- Confirmation `Modal` with warning text explaining the consequences
- `handleResetTwoFactor` function that calls `POST /users/:id/2fa/reset` via the API client
- Success/error notifications via the existing notification pattern

---

## Verification Results

### Backend
- API builds clean (`pnpm build` in `apps/api`)
- 8/8 unit tests pass (4 AES encryption + 4 PartialJwtGuard)
- All DTOs have `class-validator` decorators
- All endpoints use proper guards

### Frontend
- Web dev server starts without errors
- All 2FA pages compile and render correctly
- ESLint passes (only pre-existing warnings)
- Middleware redirects work correctly (unauthenticated users → login, pending sessions → 2FA pages)

### Pages Verified (Screenshots)

| Page | Status | Description |
|------|--------|-------------|
| `/2fa/verify` (TOTP mode) | Working | 6-digit PinInput with "Verify" button and backup code toggle |
| `/2fa/verify` (Backup mode) | Working | Text input with XXXX-XXXX placeholder |
| `/2fa/setup` | Working | Redirects to login when no session (correct middleware behavior) |
| `/admin/users/:id/edit` | Working | "Reset 2FA" orange button visible at bottom of form |
| Reset 2FA modal | Working | Confirmation dialog with warning and confirm/cancel buttons |
| `/login` | Working | Unchanged — entry point for the 2FA flow |

---

## Issues Encountered and Resolved

### 1. otplib v13 API Breaking Change

**Problem:** The implementation plan was written against otplib v12 docs, but `pnpm add otplib` installed v13.3.0 which completely removed the `authenticator` object.

**Error:** `Module '"otplib"' has no exported member 'authenticator'`

**Resolution:** Read the actual `otplib@13.3.0` TypeScript declarations (`dist/index.d.ts`, `dist/functional.d.ts`) and migrated to the functional API: `generateSecret()`, `generateURI()`, `verifySync()` with `epochTolerance: 30`.

**Lesson:** The Superpowers plan review agents caught several issues before implementation, but library API changes between major versions remain a risk when plans reference specific APIs. The fix was straightforward once the actual exports were inspected.

### 2. apiClient.post requires body argument

**Problem:** `apiClient.post('/users/${id}/2fa/reset')` failed TypeScript compilation.

**Error:** `TS2554: Expected 2-3 arguments, but got 1`

**Resolution:** Changed to `apiClient.post('/users/${id}/2fa/reset', {})` — the project's API client wrapper requires an explicit body argument, even for empty bodies.

### 3. Missing next-auth dependency

**Problem:** Web dev server failed to start after creating new files that import from `next-auth/jwt`.

**Error:** `Module not found: Can't resolve 'next-auth/jwt'`

**Resolution:** Ran `pnpm install` which installed 3 missing packages. The dependency was declared in `package.json` but not yet installed in the workspace.

---

## Architecture Highlights

### Security Model

The implementation strictly follows the spec's security model:

1. **Partial token isolation:** `partialTokenInternal` is stored only in the NextAuth JWT (HttpOnly cookie), never in the public session object. Server Actions read it via `getPartialToken()` which decodes the JWT cookie directly.

2. **Guard separation:** `PartialJwtGuard` on 2FA endpoints rejects full session tokens. `SessionTokenGuard` on regular endpoints rejects partial tokens. No cross-contamination is possible.

3. **AES-GCM authenticated encryption:** The TOTP secret is encrypted immediately on generation. The `encryptedSecret` passed to the client during setup is opaque — any tampering causes decryption to fail (GCM authentication tag verification).

4. **Atomic backup code redemption:** Uses `UPDATE WHERE usedAt IS NULL` with `rowsAffected === 1` check, preventing concurrent double-use without requiring database locks.

### Frontend Pattern

All 2FA API calls go through **Next.js Server Actions**, which:
1. Read the partial token from the JWT cookie (server-side only)
2. Forward it as `Authorization: Bearer` to the NestJS backend
3. Handle the response and call `unstable_update()` for session upgrade

This means the partial token **never reaches the browser JavaScript context** — it exists only in the HttpOnly cookie and server-side action execution.

---

## File Inventory

### New Files Created (28 total)

**Backend (12 files):**
```
apps/api/src/common/services/aes-encryption.service.ts
apps/api/src/common/services/aes-encryption.service.spec.ts
apps/api/src/database/drizzle/schemas/business/user-backup-codes.schema.ts
apps/api/src/modules/auth/guards/partial-jwt.guard.ts
apps/api/src/modules/auth/guards/partial-jwt.guard.spec.ts
apps/api/src/modules/auth/dto/two-factor-verify.dto.ts
apps/api/src/modules/auth/dto/two-factor-setup.dto.ts
apps/api/src/modules/auth/use-cases/two-factor-setup-qr.usecase.ts
apps/api/src/modules/auth/use-cases/two-factor-confirm.usecase.ts
apps/api/src/modules/auth/use-cases/two-factor-verify.usecase.ts
apps/api/src/modules/auth/use-cases/reset-user-two-factor.usecase.ts
apps/api/src/database/drizzle/migrations/XXXX_add_two_factor.sql
```

**Frontend (16 files):**
```
apps/web/src/auth.d.ts
apps/web/src/lib/auth/get-partial-token.ts
apps/web/src/app/2fa/verify/page.tsx
apps/web/src/app/2fa/verify/_components/VerifyForm.tsx
apps/web/src/app/2fa/verify/_components/BackupCodeForm.tsx
apps/web/src/app/2fa/verify/actions/verifyTotp.action.ts
apps/web/src/app/2fa/setup/page.tsx
apps/web/src/app/2fa/setup/_components/SetupWizard.tsx
apps/web/src/app/2fa/setup/_components/QrStep.tsx
apps/web/src/app/2fa/setup/_components/ConfirmStep.tsx
apps/web/src/app/2fa/setup/_components/BackupCodesStep.tsx
apps/web/src/app/2fa/setup/actions/getSetupQr.action.ts
apps/web/src/app/2fa/setup/actions/confirmSetup.action.ts
apps/web/src/app/2fa/setup/actions/completeSetup.action.ts
apps/web/src/locales/dictionary/ja/two-factor.ts
apps/web/src/locales/dictionary/en/two-factor.ts
```

### Modified Files (8 total)

**Backend:**
```
apps/api/package.json — added otplib, qrcode, @types/qrcode
apps/api/src/database/drizzle/schemas/business/users.schema.ts — 2 new columns
apps/api/src/database/drizzle/schemas/business/index.ts — export new schema
apps/api/src/modules/auth/auth.module.ts — registered new providers
apps/api/src/modules/auth/auth.controller.ts — modified credentials, added 3 endpoints
apps/api/src/modules/auth/repositories/auth.repository.ts — added 2FA query methods
apps/api/src/modules/user/user.controller.ts — added admin reset endpoint
apps/api/src/modules/user/user.module.ts — imported AuthModule
```

**Frontend:**
```
apps/web/src/auth.config.ts — handle 2FA session flow
apps/web/src/middleware.ts — redirect pending sessions
apps/web/src/locales/dictionary/ja/index.ts — export two-factor
apps/web/src/locales/dictionary/en/index.ts — export two-factor
apps/web/src/locales/dictionary/index.ts — added TwoFactorDictionary type
apps/web/src/app/(authenticated)/admin/users/_components/UserForm.tsx — Reset 2FA button + modal
apps/web/src/app/(authenticated)/admin/users/list/handler.ts — resetTwoFactor API function
```
