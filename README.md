# 2FA Design Spec, Plan & Implementation

Designing and building a complete TOTP-based Two-Factor Authentication system for a **Next.js 15 + NestJS 11** monorepo — from brainstorming to a 17-task implementation plan to working code — using [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with the [Superpowers](https://github.com/obra/superpowers) plugin.

## What's Inside

| Document | Description |
|----------|-------------|
| [Design Spec](docs/spec.md) | Full 2FA design — auth flow, database schema, API endpoints, frontend pages, security considerations |
| [Implementation Plan](docs/plan.md) | 17-task plan across 3 chunks (backend foundation, backend endpoints, frontend) with exact code |
| [Implementation Summary](docs/implementation.md) | Full implementation walkthrough — session-by-session progress, issues resolved, verification results |

## The Process

The entire design was produced through an interactive brainstorming session. Superpowers' **brainstorming skill** guided the conversation through clarifying questions, approach selection, and iterative design — presenting each section visually in a companion browser window for review before moving on.

After the spec was approved, the **writing-plans skill** generated a detailed implementation plan, which was then reviewed by parallel code-review agents that caught critical bugs (like `auth()` vs `getToken()` for reading JWT internals in Server Actions) before a single line of code was written.

### Brainstorming Flow

**1. Auth Flow Design** — Interactive flowchart showing the partial JWT to full session upgrade path

![Auth Flow](screenshots/01-auth-flow.png)

**2. Database Schema** — Schema changes to `users` table + new `user_backup_codes` table

![Database Schema](screenshots/02-database-schema.png)

**3. Backend API Endpoints** — Modified credentials endpoint + 3 new auth + 1 admin endpoint

![API Endpoints](screenshots/03-api-endpoints.png)

**4. Frontend Pages & Session Integration** — 4 new pages, middleware changes, NextAuth session handling

![Frontend Pages](screenshots/04-frontend-pages.png)

**5. Libraries, Error Handling & Testing** — Dependencies, error cases, security notes, test plan

![Libraries and Testing](screenshots/05-libraries-and-testing.png)

## Tools Used

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** (CLI) — Anthropic's agentic coding tool
- **[Superpowers](https://github.com/obra/superpowers)** — Claude Code plugin adding structured skills for brainstorming, planning, TDD, and more
- **Skills used:** `brainstorming`, `writing-plans`, `executing-plans`, `code-reviewer` (parallel agents)

## Implementation

The plan was executed across two Claude Code sessions using the `executing-plans` skill, which loaded the plan, tracked progress through a task checklist, and ran verifications at each checkpoint. All 17 tasks completed with 8/8 unit tests passing.

**Session 1: Backend** — Tasks 1–10 (Chunks 1 & 2). Built the complete backend: AES encryption service, database schema + migration, PartialJwtGuard, auth repository, and all API endpoints. Hit an otplib v13 API breaking change at the end.

**Session 2: Frontend + Fix** — Tasks 11–17 (Chunk 3). Fixed otplib v13 migration, then built the full frontend: NextAuth session handling, middleware redirects, 2FA verify page, 3-step setup wizard, i18n (JA/EN), and admin Reset 2FA with confirmation modal.

### Implementation Screenshots

**TOTP Verification Page** — 6-digit PinInput with auto-submit

![2FA Verify - TOTP](screenshots/06-verify-totp.png)

**Backup Code Entry** — Alternative recovery method

![2FA Verify - Backup Code](screenshots/07-verify-backup.png)

**Setup Wizard — Step 1: Scan QR Code**

![Setup QR Code](screenshots/10-setup-qr.png)

**Setup Wizard — Step 2: Confirm TOTP Code**

![Setup Confirm](screenshots/11-setup-confirm.png)

**Setup Wizard — Step 3: Save Backup Codes**

![Setup Backup Codes](screenshots/12-setup-backup-codes.png)

**Admin User Edit with Reset 2FA** — Orange outline button on user edit form

![Admin Reset 2FA](screenshots/08-admin-reset-button.png)

**Reset 2FA Confirmation Modal** — Warning dialog before resetting

![Reset 2FA Modal](screenshots/09-reset-modal.png)

### Issue Highlight: otplib v12 → v13

The plan was written against otplib v12's API (`authenticator.generateSecret()`), but v13.3.0 was installed, which completely removed the `authenticator` object in favor of a functional API (`generateSecret()`, `verifySync()`). The plan review agents caught several issues pre-implementation, but library version breaks remain a risk. Resolution was straightforward: read the actual TypeScript declarations and migrate to the new API.

## Key Design Decisions

- **TOTP-only** (no SMS/email) via `otplib` — simpler, more secure
- **Stateless partial JWT** — 5-minute token after credentials, upgraded to full session after TOTP
- **AES-256-GCM** encryption for TOTP secrets at rest
- **10 bcrypt-hashed backup codes** with atomic single-use redemption
- **Server Actions** keep the partial token server-side (never exposed to browser JS)
- **Forced enrollment** — all users must set up 2FA, no opt-out
