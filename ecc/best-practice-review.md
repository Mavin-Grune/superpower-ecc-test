# Best Practice Review — nextjs15-starter-kit

Automated review of the **Next.js 15 + NestJS 11** Turborepo monorepo using Everything Claude Code (ECC) agents.

**Date:** 2026-03-18
**Target:** `nextjs15-starter-kit` (Turborepo monorepo)
**Method:** 3 parallel ECC Explore agents (frontend, backend, security)

---

## Frontend

**Stack:** Next.js 15 (App Router) + Mantine 8 + Jotai + TanStack Query + AG Grid

### What's Good

- Clean App Router structure with error boundaries and 404 pages
- Jotai provider correctly configured with stable store
- Well-structured hooks (useUserServices, useTableColumns, useTablePagination)
- Centralized i18n-aware metadata system
- API client with proper error handling, timeout, and FormData support
- QueryClient configured with staleTime: 60s

### Issues Found

| Priority | Issue | Detail |
|----------|-------|--------|
| High | No `loading.tsx` files | Manual LoadingOverlay instead of App Router's built-in loading states |
| High | No `next/dynamic` | Modals, tables, forms not code-split — everything loads synchronously |
| High | No `next/Image` | All images use `<img>` tags, missing optimization |
| Medium | No per-page metadata | Only root layout has `generateMetadata`, bad for SEO |
| Medium | Atom proliferation | `tableStore.ts` has 24 atoms — consider atom families |
| Medium | No refresh token | Session 401 triggers full logout, no token refresh mechanism |
| Low | No filter persistence | Table sorting/filtering lost on refresh |
| Low | Inconsistent loading states | `create/page.tsx` uses `<div>Loading...</div>` while others use LoadingOverlay |

---

## Backend

**Stack:** NestJS 11 + Drizzle ORM + Passport JWT + Winston + Helmet

### What's Good

- Clean modular structure with domain-driven layout
- Use-case pattern (CreateUserUseCase, UpdateUserUseCase, etc.)
- Repository pattern isolates data access
- Comprehensive global exception filter with activity logging
- Multi-database support (PostgreSQL, MySQL, MariaDB)
- Transaction support defined in DrizzleService
- Bcrypt hashing (10 rounds) for passwords

### Issues Found

| Priority | Issue | Detail |
|----------|-------|--------|
| High | POST for reads | `POST /users/list` should be `GET /users` with query params |
| High | No transactions used | Transactions defined but not used in critical operations (user creation, bulk ops) |
| High | DB queries in guards | `RolesGuard` and `TokenVersionGuard` hit DB on every authenticated request |
| Medium | No API versioning | No `/api/v1` prefix — breaking changes affect all clients |
| Medium | Missing input sanitization | No `@Trim()`, `@Lowercase()` on email/username fields |
| Medium | No query logging | Drizzle configured without query logging |
| Medium | Date validation weak | Date fields use `@IsString()` without `@IsISO8601()` format validation |
| Low | Activity logging in controllers | Should be a cross-cutting concern (interceptor) |
| Low | SSL cert path hardcoded | `/app/apps/api/certs/global-bundle.pem` should be configurable |

---

## Security

### What's Good

- Drizzle ORM = zero SQL injection risk (all parameterized)
- JWT auth with token versioning for forced logout
- Rate limiting: login (5/min), password reset (3/hr), global (300/min)
- Helmet CSP with restrictive `defaultSrc: ["'self'"]`
- React escaping = low XSS risk
- No `dangerouslySetInnerHTML` or `eval()` usage
- `.env` files properly gitignored
- Email enumeration prevention in password reset

### Issues Found

| Severity | Issue | Detail |
|----------|-------|--------|
| **Critical** | Default AUTH_SECRET | Placeholder in `.env.example` — JWT tokens forgeable if not changed |
| **Critical** | Default MinIO credentials | `minioadmin`/`minioadmin123` as fallback in `storage.config.ts` |
| Medium | MIME-only file validation | No magic number check — file type easily spoofed |
| Medium | CORS allows no-origin | Requests without origin header accepted |
| Medium | CSP relaxed for dev | `crossOriginEmbedderPolicy` disabled, `unsafe-inline` for styles |
| Medium | No FileUpload defaults | If no maxSize/allowedMimeTypes configured, no validation occurs |
| Low | No CSRF protection | Missing CSRF middleware |
| Low | Rate limit headers | Not verified in responses |

---

## Summary

| Area | Status | Critical | High | Medium | Low |
|------|--------|----------|------|--------|-----|
| Frontend | Good | 0 | 3 | 3 | 2 |
| Backend | Good | 0 | 3 | 4 | 2 |
| Security | Good | 2 | 0 | 4 | 2 |
| **Total** | | **2** | **6** | **11** | **6** |

### Top 3 Actions

1. **Add startup validation for secrets** — Reject default AUTH_SECRET and MinIO creds in production
2. **Fix REST conventions** — Use GET for reads, wrap critical operations in transactions
3. **Add code-splitting** — `next/dynamic` for modals/tables, `next/Image` for images
