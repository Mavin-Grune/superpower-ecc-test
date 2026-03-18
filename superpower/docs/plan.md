# 2FA Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add TOTP-based 2FA required for all users, with backup codes and admin reset.

**Architecture:** Stateless partial-auth JWT issued after credentials, exchanged for a full session JWT after TOTP verification. Backend handles TOTP generation/verification via `otplib`. Frontend uses Server Actions to keep the partial token server-side only.

**Tech Stack:** NestJS 11, Drizzle ORM (MySQL), otplib, qrcode, Next.js 15 (App Router), NextAuth v5, Mantine 8, Jotai

**Spec:** `docs/superpowers/specs/2026-03-11-2fa-design.md`

---

## File Structure

### Backend — New files
| File | Responsibility |
|---|---|
| `apps/api/src/common/services/aes-encryption.service.ts` | AES-256-GCM encrypt/decrypt |
| `apps/api/src/common/services/aes-encryption.service.spec.ts` | Unit tests for AES service |
| `apps/api/src/database/drizzle/schemas/business/user-backup-codes.schema.ts` | Backup codes table schema |
| `apps/api/src/modules/auth/guards/partial-jwt.guard.ts` | Guards for partial JWT validation |
| `apps/api/src/modules/auth/guards/partial-jwt.guard.spec.ts` | Unit tests for partial JWT guard |
| `apps/api/src/modules/auth/dto/two-factor-verify.dto.ts` | DTO for TOTP verify endpoint |
| `apps/api/src/modules/auth/dto/two-factor-setup.dto.ts` | DTO for TOTP setup confirm endpoint |
| `apps/api/src/modules/auth/use-cases/two-factor-setup-qr.usecase.ts` | Generate TOTP secret + QR code |
| `apps/api/src/modules/auth/use-cases/two-factor-confirm.usecase.ts` | Confirm setup + generate backup codes |
| `apps/api/src/modules/auth/use-cases/two-factor-verify.usecase.ts` | Verify TOTP or backup code |
| `apps/api/src/modules/auth/use-cases/reset-user-two-factor.usecase.ts` | Admin reset use-case |
| `apps/api/src/modules/auth/use-cases/index.ts` | Barrel export for all use-cases |

### Backend — Modified files
| File | Change |
|---|---|
| `apps/api/src/database/drizzle/schemas/business/users.schema.ts` | Add `twoFactorSecret`, `twoFactorEnabled` columns |
| `apps/api/src/database/drizzle/schemas/business/index.ts` | Export new schema + types |
| `apps/api/src/modules/auth/auth.module.ts` | Register new providers |
| `apps/api/src/modules/auth/auth.controller.ts` | Modify credentials, add 2FA endpoints |
| `apps/api/src/modules/auth/repositories/auth.repository.ts` | Add 2FA query methods |
| `apps/api/src/modules/auth/dto/index.ts` | Export new DTOs |
| `apps/api/src/modules/user/user.controller.ts` | Add admin reset endpoint |
| `apps/api/src/modules/user/user.module.ts` | Import AuthModule |
| `apps/api/package.json` | Add `otplib`, `qrcode`, `@types/qrcode` |

### Frontend — New files
| File | Responsibility |
|---|---|
| `apps/web/src/auth.d.ts` | NextAuth module augmentation (JWT + Session types) |
| `apps/web/src/lib/auth/get-partial-token.ts` | Server-side helper to read `partialTokenInternal` from JWT cookie |
| `apps/web/src/app/2fa/verify/page.tsx` | 2FA verify page entry |
| `apps/web/src/app/2fa/verify/_components/VerifyForm.tsx` | TOTP code input form |
| `apps/web/src/app/2fa/verify/_components/BackupCodeForm.tsx` | Backup code input form |
| `apps/web/src/app/2fa/verify/actions/verifyTotp.action.ts` | Server action for TOTP verify |
| `apps/web/src/app/2fa/setup/page.tsx` | 2FA setup page entry |
| `apps/web/src/app/2fa/setup/_components/SetupWizard.tsx` | Wizard state machine |
| `apps/web/src/app/2fa/setup/_components/QrStep.tsx` | Step 1: scan QR |
| `apps/web/src/app/2fa/setup/_components/ConfirmStep.tsx` | Step 2: confirm code |
| `apps/web/src/app/2fa/setup/_components/BackupCodesStep.tsx` | Step 3: save backup codes |
| `apps/web/src/app/2fa/setup/actions/getSetupQr.action.ts` | Server action: fetch QR |
| `apps/web/src/app/2fa/setup/actions/confirmSetup.action.ts` | Server action: confirm setup |
| `apps/web/src/app/2fa/setup/actions/completeSetup.action.ts` | Server action: upgrade session |
| `apps/web/src/locales/dictionary/ja/two-factor.ts` | Japanese translations |
| `apps/web/src/locales/dictionary/en/two-factor.ts` | English translations |

### Frontend — Modified files
| File | Change |
|---|---|
| `apps/web/src/auth.config.ts` | Handle partial JWT response, 2FA session state |
| `apps/web/src/middleware.ts` | Redirect pending sessions to 2FA pages |
| `apps/web/src/locales/dictionary/ja/index.ts` | Export two-factor dictionary |
| `apps/web/src/locales/dictionary/en/index.ts` | Export two-factor dictionary |

---

## Chunk 1: Backend Foundation

### Task 1: Install packages and configure env

**Files:**
- Modify: `apps/api/package.json`
- Modify: `apps/api/.env` (and Docker env template)

- [ ] **Step 1: Install backend dependencies**

```bash
cd apps/api && pnpm add otplib qrcode && pnpm add -D @types/qrcode
```

- [ ] **Step 2: Add env var to `.env` files**

Add to `apps/api/.env`:
```
TWO_FACTOR_ENCRYPTION_KEY=
```

Generate a key for local dev:
```bash
openssl rand -hex 32
```

Copy the output into `.env.local` as `TWO_FACTOR_ENCRYPTION_KEY=<value>`.

Add to `docker/dev/.env`:
```
TWO_FACTOR_ENCRYPTION_KEY=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

Also add it (empty) to `apps/api/.env.example`, `apps/api/.env.local.example`, and any database-specific example files (`apps/api/.env.mysql.example`, etc.).

- [ ] **Step 3: Commit**

```bash
git add apps/api/package.json pnpm-lock.yaml docker/dev/.env
git commit -m "chore: add otplib and qrcode packages for 2FA"
```

---

### Task 2: AES Encryption Service (TDD)

**Files:**
- Create: `apps/api/src/common/services/aes-encryption.service.ts`
- Create: `apps/api/src/common/services/aes-encryption.service.spec.ts`
- Modify: `apps/api/src/common/services/index.ts`

- [ ] **Step 1: Write the failing tests**

Create `apps/api/src/common/services/aes-encryption.service.spec.ts`:

```typescript
import { ConfigService } from '@nestjs/config';
import { Test } from '@nestjs/testing';
import { AesEncryptionService } from './aes-encryption.service';

// Valid 32-byte hex key for testing
const TEST_KEY = 'a'.repeat(64);

describe('AesEncryptionService', () => {
  let service: AesEncryptionService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        AesEncryptionService,
        {
          provide: ConfigService,
          useValue: {
            getOrThrow: (key: string) => {
              if (key === 'TWO_FACTOR_ENCRYPTION_KEY') return TEST_KEY;
              throw new Error(`Unknown key: ${key}`);
            },
          },
        },
      ],
    }).compile();

    service = module.get(AesEncryptionService);
  });

  it('should encrypt and decrypt a string round-trip', () => {
    const plaintext = 'JBSWY3DPEHPK3PXP';
    const encrypted = service.encrypt(plaintext);
    expect(encrypted).not.toBe(plaintext);
    expect(service.decrypt(encrypted)).toBe(plaintext);
  });

  it('should produce different ciphertexts for the same plaintext (random IV)', () => {
    const plaintext = 'JBSWY3DPEHPK3PXP';
    const a = service.encrypt(plaintext);
    const b = service.encrypt(plaintext);
    expect(a).not.toBe(b);
  });

  it('should throw on tampered ciphertext', () => {
    const encrypted = service.encrypt('test');
    const parts = encrypted.split(':');
    // Flip a byte in the ciphertext portion
    parts[2] = 'ff' + parts[2].slice(2);
    expect(() => service.decrypt(parts.join(':'))).toThrow();
  });

  it('should throw on invalid format', () => {
    expect(() => service.decrypt('not-valid')).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd apps/api && npx jest --testPathPattern=aes-encryption --no-coverage
```

Expected: FAIL — `Cannot find module './aes-encryption.service'`

- [ ] **Step 3: Implement the service**

Create `apps/api/src/common/services/aes-encryption.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class AesEncryptionService {
  private readonly algorithm = 'aes-256-gcm' as const;
  private readonly key: Buffer;

  constructor(private readonly configService: ConfigService) {
    const keyHex = this.configService.getOrThrow<string>(
      'TWO_FACTOR_ENCRYPTION_KEY',
    );
    this.key = Buffer.from(keyHex, 'hex');
    if (this.key.length !== 32) {
      throw new Error(
        'TWO_FACTOR_ENCRYPTION_KEY must be a 64-character hex string (32 bytes)',
      );
    }
  }

  encrypt(plaintext: string): string {
    const iv = crypto.randomBytes(12);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    const encrypted = Buffer.concat([
      cipher.update(plaintext, 'utf8'),
      cipher.final(),
    ]);
    const authTag = cipher.getAuthTag();
    return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted.toString('hex')}`;
  }

  decrypt(ciphertext: string): string {
    const parts = ciphertext.split(':');
    if (parts.length !== 3) {
      throw new Error('Invalid ciphertext format');
    }
    const [ivHex, authTagHex, encryptedHex] = parts;
    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    const encrypted = Buffer.from(encryptedHex, 'hex');
    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv);
    decipher.setAuthTag(authTag);
    const decrypted = Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),
    ]);
    return decrypted.toString('utf8');
  }
}
```

- [ ] **Step 4: Export from barrel**

Add to `apps/api/src/common/services/index.ts`:

```typescript
export { AesEncryptionService } from './aes-encryption.service';
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd apps/api && npx jest --testPathPattern=aes-encryption --no-coverage
```

Expected: 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/common/services/aes-encryption.service.ts apps/api/src/common/services/aes-encryption.service.spec.ts apps/api/src/common/services/index.ts
git commit -m "feat(auth): add AES-256-GCM encryption service for 2FA secrets"
```

---

### Task 3: Database schema changes

**Files:**
- Modify: `apps/api/src/database/drizzle/schemas/business/users.schema.ts`
- Create: `apps/api/src/database/drizzle/schemas/business/user-backup-codes.schema.ts`
- Modify: `apps/api/src/database/drizzle/schemas/business/index.ts`

- [ ] **Step 1: Add 2FA columns to users schema**

In `apps/api/src/database/drizzle/schemas/business/users.schema.ts`:

First, update the import from `drizzle-orm/mysql-core` to include `boolean`:

```typescript
import {
  bigint,
  boolean,
  foreignKey,
  index,
  int,
  mysqlTable,
  timestamp,
  unique,
  varchar,
} from 'drizzle-orm/mysql-core';
```

Then add two columns after `tokenVersion`:

```typescript
twoFactorSecret: varchar('two_factor_secret', { length: 255 }),
twoFactorEnabled: boolean('two_factor_enabled').notNull().default(false),
```

- [ ] **Step 2: Create backup codes schema**

> **Convention exception:** This table intentionally omits `deletedAt`, `createdBy`, `updatedBy` columns and uses hard-delete instead of soft-delete. Backup codes are security-sensitive single-use tokens that must be fully purged on admin reset to prevent recovery. This is documented in the design spec as an intentional deviation from the project's soft-delete convention.

Create `apps/api/src/database/drizzle/schemas/business/user-backup-codes.schema.ts`:

```typescript
import { relations } from 'drizzle-orm';
import {
  bigint,
  index,
  mysqlTable,
  timestamp,
  varchar,
} from 'drizzle-orm/mysql-core';
import { user } from './users.schema';

export const userBackupCode = mysqlTable(
  'user_backup_codes',
  {
    id: bigint('id', { mode: 'number' }).autoincrement().primaryKey(),
    userId: bigint('user_id', { mode: 'number' })
      .notNull()
      .references(() => user.id, { onDelete: 'cascade' }),
    codeHash: varchar('code_hash', { length: 255 }).notNull(),
    usedAt: timestamp('used_at', { mode: 'date' }),
    createdAt: timestamp('created_at', { mode: 'date' })
      .notNull()
      .defaultNow(),
    updatedAt: timestamp('updated_at', { mode: 'date' })
      .notNull()
      .defaultNow(),
  },
  (table) => ({
    userIdIdx: index('user_backup_codes_user_id_idx').on(table.userId),
  }),
);

export const userBackupCodeRelations = relations(
  userBackupCode,
  ({ one }) => ({
    user: one(user, {
      fields: [userBackupCode.userId],
      references: [user.id],
    }),
  }),
);

export type UserBackupCode = typeof userBackupCode.$inferSelect;
export type NewUserBackupCode = typeof userBackupCode.$inferInsert;
```

- [ ] **Step 3: Update business barrel**

In `apps/api/src/database/drizzle/schemas/business/index.ts`, add:

Import at top:
```typescript
import {
  userBackupCode,
  userBackupCodeRelations,
} from './user-backup-codes.schema';
```

Add export block:
```typescript
// User Backup Codes (2FA)
export {
  userBackupCode,
  userBackupCodeRelations,
  type UserBackupCode,
  type NewUserBackupCode,
} from './user-backup-codes.schema';
```

Add to `businessTables`:
```typescript
export const businessTables = {
  user,
  passwordResetToken,
  userBackupCode,
} as const;
```

Add to `businessTableRelations`:
```typescript
export const businessTableRelations = {
  userRelations,
  passwordResetTokenRelations,
  userBackupCodeRelations,
} as const;
```

- [ ] **Step 4: Generate Drizzle migration**

```bash
pnpm api:db:generate
```

Review generated SQL in `apps/api/src/database/drizzle/migrations/`. It should contain:
- `ALTER TABLE users ADD two_factor_secret varchar(255)`
- `ALTER TABLE users ADD two_factor_enabled boolean NOT NULL DEFAULT false`
- `CREATE TABLE user_backup_codes (...)`

- [ ] **Step 5: Apply migration**

```bash
pnpm api:db:migrate
```

- [ ] **Step 6: Verify**

```bash
pnpm api:db:check
```

Expected: No drift detected.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/database/drizzle/schemas/ apps/api/src/database/drizzle/migrations/
git commit -m "feat(db): add 2FA columns to users and user_backup_codes table"
```

---

### Task 4: Auth Repository — 2FA methods

**Files:**
- Modify: `apps/api/src/modules/auth/repositories/auth.repository.ts`

- [ ] **Step 1: Update existing imports**

In `auth.repository.ts`, update the existing `drizzle-orm` import to add `and` and `isNull`:

```typescript
import { and, eq, isNull, sql } from 'drizzle-orm';
```

Add `userBackupCode` and `getAffectedRowCount` to the existing imports:

```typescript
import {
  user,
  User,
  userRole,
  userBackupCode,
} from '../../../database/drizzle/schemas';
import { getAffectedRowCount } from '../../../database/utils/delete-result.util';
```

- [ ] **Step 1b: Update `findByUsername` select to include 2FA fields**

After Task 3 adds `twoFactorSecret` and `twoFactorEnabled` to the `user` schema, the inferred `User` type will include these fields. The existing `findByUsername` method uses an explicit `.select({...})` that doesn't include them. Without this fix, `foundUser.twoFactorEnabled` from `findByUsername` results would be `undefined` at runtime despite TypeScript thinking it's `boolean` (type lie).

Add these two fields to the `.select({...})` in `findByUsername`:

```typescript
twoFactorSecret: user.twoFactorSecret,
twoFactorEnabled: user.twoFactorEnabled,
```

- [ ] **Step 2: Add `findByIdWithTwoFactor` method**

```typescript
/**
 * Find user by ID including 2FA fields and role value
 */
async findByIdWithTwoFactor(
  id: number,
): Promise<
  | (User & { roleValue: string | number | null })
  | null
> {
  const db = this.drizzle.getDb() as DatabaseClient;
  const users = await db
    .select({
      id: user.id,
      username: user.username,
      email: user.email,
      fullname: user.fullname,
      password: user.password,
      userRoleId: user.userRoleId,
      tokenVersion: user.tokenVersion,
      twoFactorSecret: user.twoFactorSecret,
      twoFactorEnabled: user.twoFactorEnabled,
      createdAt: user.createdAt,
      updatedAt: user.updatedAt,
      deletedAt: user.deletedAt,
      createdByType: user.createdByType,
      updatedByType: user.updatedByType,
      createdBy: user.createdBy,
      updatedBy: user.updatedBy,
      roleValue: userRole.value,
    })
    .from(user)
    .leftJoin(userRole, eq(user.userRoleId, userRole.id))
    .where(eq(user.id, id))
    .limit(1);

  return (users[0] as any) || null;
}
```

- [ ] **Step 3: Add `saveTwoFactorSecret` method**

```typescript
/**
 * Save encrypted 2FA secret and enable 2FA for a user
 */
async saveTwoFactorSecret(
  userId: number,
  encryptedSecret: string,
): Promise<void> {
  const db = this.drizzle.getDb() as DatabaseClient;
  await db
    .update(user)
    .set({
      twoFactorSecret: encryptedSecret,
      twoFactorEnabled: true,
      updatedAt: new Date(),
    })
    .where(eq(user.id, userId));
}
```

- [ ] **Step 4: Add `clearTwoFactor` method**

```typescript
/**
 * Clear 2FA secret, disable 2FA, and hard-delete all backup codes
 */
async clearTwoFactor(userId: number): Promise<void> {
  const db = this.drizzle.getDb() as DatabaseClient;
  await db
    .update(user)
    .set({
      twoFactorSecret: null,
      twoFactorEnabled: false,
      updatedAt: new Date(),
    })
    .where(eq(user.id, userId));
  await db
    .delete(userBackupCode)
    .where(eq(userBackupCode.userId, userId));
}
```

- [ ] **Step 5: Add backup code methods**

```typescript
/**
 * Insert hashed backup codes for a user
 */
async insertBackupCodes(
  userId: number,
  codeHashes: string[],
): Promise<void> {
  const db = this.drizzle.getDb() as DatabaseClient;
  const now = new Date();
  await db.insert(userBackupCode).values(
    codeHashes.map((hash) => ({
      userId,
      codeHash: hash,
      createdAt: now,
      updatedAt: now,
    })),
  );
}

/**
 * Find all unused backup codes for a user
 */
async findUnusedBackupCodes(
  userId: number,
): Promise<{ id: number; codeHash: string }[]> {
  const db = this.drizzle.getDb() as DatabaseClient;
  return db
    .select({ id: userBackupCode.id, codeHash: userBackupCode.codeHash })
    .from(userBackupCode)
    .where(
      and(
        eq(userBackupCode.userId, userId),
        isNull(userBackupCode.usedAt),
      ),
    );
}

/**
 * Atomically redeem a backup code. Returns true if redeemed, false if already used.
 */
async redeemBackupCode(codeId: number): Promise<boolean> {
  const db = this.drizzle.getDb() as DatabaseClient;
  const result = await db
    .update(userBackupCode)
    .set({ usedAt: new Date(), updatedAt: new Date() })
    .where(
      and(
        eq(userBackupCode.id, codeId),
        isNull(userBackupCode.usedAt),
      ),
    );
  // Use the project's existing DB-agnostic utility (supports MySQL, PostgreSQL, etc.)
  const count = getAffectedRowCount(Array.isArray(result) ? result[0] : result);
  return count === 1;
}

- [ ] **Step 6: Verify build**

```bash
cd apps/api && pnpm build
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/modules/auth/repositories/auth.repository.ts
git commit -m "feat(auth): add 2FA repository methods for secrets and backup codes"
```

---

### Task 5: Modify POST /auth/credentials to return partial JWT

**Files:**
- Modify: `apps/api/src/modules/auth/use-cases/validate-credentials.usecase.ts`
- Modify: `apps/api/src/modules/auth/auth.controller.ts`

> **Note:** This is a breaking change — existing frontend login will stop working until Chunk 3 (Task 11) is implemented. Do not test login end-to-end between Chunk 1 and Chunk 3.

- [ ] **Step 1: Update ValidateCredentialsUseCase to return 2FA status**

The existing use-case returns a `PublicUser`. Modify it to also return `twoFactorEnabled` so the controller doesn't need to directly access the repository (preserving the use-case architecture pattern).

In `apps/api/src/modules/auth/use-cases/validate-credentials.usecase.ts`, after the user is found and password is validated, add the 2FA enabled flag to the return:

```typescript
// Add to the return object:
return {
  ...publicUser,
  twoFactorEnabled: foundUser.twoFactorEnabled ?? false,
};
```

The `foundUser` from `findByUsername` will now include `twoFactorEnabled` thanks to Step 1b in Task 4.

- [ ] **Step 2: Modify `validateCredentials` controller method**

Replace the existing method body (after the `try {` and the `validateCredentialsUseCase.execute` call). The credential validation stays the same. Change the response section:

```typescript
// After: const user = await this.validateCredentialsUseCase.execute(...)

// Check if user has 2FA enrolled (returned by the use-case)
const enrollmentRequired = !user.twoFactorEnabled;

// Issue a short-lived partial JWT (5 minutes)
const partialToken = this.jwtService.sign(
  {
    sub: user.id.toString(),
    twoFactorPending: true,
    enrollmentRequired,
    type: 'partial',
    rememberMe: credentials.rememberMe === true,
  },
  { expiresIn: '5m' },
);

this.activityLogService.logByCode('AUTH001_LOGIN', 'success', {
  actorId: user.id,
  actorType: resolveActorType(user),
  replacements: { username: user.username },
  req,
});

return {
  status: enrollmentRequired ? 'ENROLLMENT_REQUIRED' : '2FA_REQUIRED',
  partialToken,
};
```

- [ ] **Step 3: Verify build compiles**

```bash
cd apps/api && pnpm build
```

Expected: Success

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/modules/auth/auth.controller.ts
git commit -m "feat(auth): modify credentials endpoint to return partial JWT for 2FA"
```

---

## Chunk 2: Backend 2FA Endpoints

### Task 6: PartialJwtGuard

**Files:**
- Create: `apps/api/src/modules/auth/guards/partial-jwt.guard.ts`
- Create: `apps/api/src/modules/auth/guards/partial-jwt.guard.spec.ts`

- [ ] **Step 1: Write the failing tests**

Create `apps/api/src/modules/auth/guards/partial-jwt.guard.spec.ts`:

```typescript
import { ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { PartialJwtGuard } from './partial-jwt.guard';

function mockContext(authHeader?: string): ExecutionContext {
  return {
    switchToHttp: () => ({
      getRequest: () => ({
        headers: { authorization: authHeader },
        partialJwtPayload: undefined,
      }),
    }),
  } as any;
}

describe('PartialJwtGuard', () => {
  let guard: PartialJwtGuard;
  let jwtService: JwtService;

  beforeEach(() => {
    jwtService = {
      verifyAsync: jest.fn(),
    } as any;
    guard = new PartialJwtGuard(jwtService);
  });

  it('should reject when no token provided', async () => {
    await expect(guard.canActivate(mockContext())).rejects.toThrow(
      UnauthorizedException,
    );
  });

  it('should reject when token type is not partial', async () => {
    (jwtService.verifyAsync as jest.Mock).mockResolvedValue({
      sub: '1',
      type: 'session',
    });
    await expect(
      guard.canActivate(mockContext('Bearer tok')),
    ).rejects.toThrow(UnauthorizedException);
  });

  it('should accept valid partial token and attach payload', async () => {
    const payload = {
      sub: '1',
      twoFactorPending: true,
      enrollmentRequired: false,
      type: 'partial',
    };
    (jwtService.verifyAsync as jest.Mock).mockResolvedValue(payload);
    const ctx = mockContext('Bearer tok');
    const result = await guard.canActivate(ctx);
    expect(result).toBe(true);
    const req = ctx.switchToHttp().getRequest();
    expect(req.partialJwtPayload).toEqual(payload);
  });

  it('should reject expired token', async () => {
    (jwtService.verifyAsync as jest.Mock).mockRejectedValue(
      new Error('jwt expired'),
    );
    await expect(
      guard.canActivate(mockContext('Bearer tok')),
    ).rejects.toThrow(UnauthorizedException);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd apps/api && npx jest --testPathPattern=partial-jwt.guard --no-coverage
```

Expected: FAIL — cannot find module

- [ ] **Step 3: Implement the guard**

Create `apps/api/src/modules/auth/guards/partial-jwt.guard.ts`:

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

export type PartialJwtPayload = {
  sub: string;
  twoFactorPending: boolean;
  enrollmentRequired: boolean;
  type: string;
  rememberMe?: boolean;
};

@Injectable()
export class PartialJwtGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      throw new UnauthorizedException('No authorization token provided');
    }

    let payload: PartialJwtPayload;
    try {
      payload = await this.jwtService.verifyAsync<PartialJwtPayload>(token);
    } catch {
      throw new UnauthorizedException('Invalid or expired token');
    }

    if (payload.type !== 'partial' || !payload.twoFactorPending) {
      throw new UnauthorizedException('Invalid token type');
    }

    (request as any).partialJwtPayload = payload;
    return true;
  }

  private extractTokenFromHeader(request: Request): string | null {
    const authHeader = request.headers?.authorization;
    if (!authHeader) return null;
    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' && token ? token : null;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd apps/api && npx jest --testPathPattern=partial-jwt.guard --no-coverage
```

Expected: 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/modules/auth/guards/partial-jwt.guard.ts apps/api/src/modules/auth/guards/partial-jwt.guard.spec.ts
git commit -m "feat(auth): add PartialJwtGuard for 2FA token validation"
```

---

### Task 7: DTOs for 2FA endpoints

**Files:**
- Create: `apps/api/src/modules/auth/dto/two-factor-setup.dto.ts`
- Create: `apps/api/src/modules/auth/dto/two-factor-verify.dto.ts`
- Modify: `apps/api/src/modules/auth/dto/index.ts`

- [ ] **Step 1: Create setup DTO**

Create `apps/api/src/modules/auth/dto/two-factor-setup.dto.ts`:

```typescript
import { IsString, Length } from 'class-validator';

export class TwoFactorSetupDto {
  @IsString()
  @Length(6, 6, { message: 'Code must be exactly 6 digits' })
  code: string;

  @IsString()
  encryptedSecret: string;
}
```

- [ ] **Step 2: Create verify DTO**

Create `apps/api/src/modules/auth/dto/two-factor-verify.dto.ts`:

```typescript
import { IsOptional, IsString, Length, ValidateIf } from 'class-validator';

export class TwoFactorVerifyDto {
  @ValidateIf((o) => !o.backupCode)
  @IsString()
  @Length(6, 6, { message: 'Code must be exactly 6 digits' })
  code?: string;

  @ValidateIf((o) => !o.code)
  @IsString()
  @IsOptional()
  backupCode?: string;
}
```

- [ ] **Step 3: Update DTO barrel**

In `apps/api/src/modules/auth/dto/index.ts`, add:

```typescript
export { TwoFactorSetupDto } from './two-factor-setup.dto';
export { TwoFactorVerifyDto } from './two-factor-verify.dto';
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/modules/auth/dto/
git commit -m "feat(auth): add DTOs for 2FA setup and verify endpoints"
```

---

### Task 8: 2FA Use-Cases

**Files:**
- Create: `apps/api/src/modules/auth/use-cases/two-factor-setup-qr.usecase.ts`
- Create: `apps/api/src/modules/auth/use-cases/two-factor-confirm.usecase.ts`
- Create: `apps/api/src/modules/auth/use-cases/two-factor-verify.usecase.ts`
- Create: `apps/api/src/modules/auth/use-cases/reset-user-two-factor.usecase.ts`

- [ ] **Step 1: Create TwoFactorSetupQrUseCase**

Create `apps/api/src/modules/auth/use-cases/two-factor-setup-qr.usecase.ts`:

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { authenticator } from 'otplib';
import * as QRCode from 'qrcode';
import { AesEncryptionService } from '../../../common/services/aes-encryption.service';
import { AuthRepository } from '../repositories/auth.repository';

// Allow ±1 window (30-second grace) for clock drift
authenticator.options = { window: 1 };

@Injectable()
export class TwoFactorSetupQrUseCase {
  constructor(
    private readonly authRepo: AuthRepository,
    private readonly aesEncryptionService: AesEncryptionService,
    private readonly configService: ConfigService,
  ) {}

  async execute(userId: number) {
    const foundUser = await this.authRepo.findById(userId);
    if (!foundUser) {
      throw new UnauthorizedException('User not found');
    }

    const secret = authenticator.generateSecret();
    const issuer =
      this.configService.get<string>('APP_NAME') || 'App';
    const accountName = foundUser.email;
    const otpauthUrl = authenticator.keyuri(accountName, issuer, secret);
    const qrCodeDataUrl = await QRCode.toDataURL(otpauthUrl);
    const encryptedSecret = this.aesEncryptionService.encrypt(secret);

    return {
      otpauthUrl,
      qrCodeDataUrl,
      issuer,
      accountName,
      encryptedSecret,
    };
  }
}
```

- [ ] **Step 2: Create TwoFactorConfirmUseCase**

Create `apps/api/src/modules/auth/use-cases/two-factor-confirm.usecase.ts`:

```typescript
import {
  BadRequestException,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { authenticator } from 'otplib';
import { hash } from 'bcrypt';
import * as crypto from 'crypto';
import { AesEncryptionService } from '../../../common/services/aes-encryption.service';
import { toPublicUser } from '../mappers/auth.mapper';
import { AuthRepository } from '../repositories/auth.repository';

function generateBackupCode(): string {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  const part = (len: number) =>
    Array.from(crypto.randomBytes(len))
      .map((b) => chars[b % chars.length])
      .join('');
  return `${part(4)}-${part(4)}`;
}

@Injectable()
export class TwoFactorConfirmUseCase {
  constructor(
    private readonly authRepo: AuthRepository,
    private readonly aesEncryptionService: AesEncryptionService,
    private readonly jwtService: JwtService,
  ) {}

  async execute(
    userId: number,
    code: string,
    encryptedSecret: string,
    rememberMe: boolean,
  ) {
    // 1. Decrypt the secret
    let rawSecret: string;
    try {
      rawSecret = this.aesEncryptionService.decrypt(encryptedSecret);
    } catch {
      throw new BadRequestException('Invalid encrypted secret');
    }

    // 2. Verify TOTP code
    const isValid = authenticator.verify({
      token: code,
      secret: rawSecret,
    });
    if (!isValid) {
      throw new UnauthorizedException('Invalid verification code');
    }

    // 3. Save encrypted secret and enable 2FA
    const storedSecret = this.aesEncryptionService.encrypt(rawSecret);
    await this.authRepo.saveTwoFactorSecret(userId, storedSecret);

    // 4. Generate 10 backup codes
    const plainCodes = Array.from({ length: 10 }, () => generateBackupCode());
    const codeHashes = await Promise.all(
      plainCodes.map((c) => hash(c, 10)),
    );
    await this.authRepo.insertBackupCodes(userId, codeHashes);

    // 5. Issue full session JWT
    const foundUser = await this.authRepo.findByIdWithTwoFactor(userId);
    if (!foundUser) {
      throw new UnauthorizedException('User not found');
    }
    const tokenVersion = await this.authRepo.findTokenVersion(userId);
    const sessionToken = this.jwtService.sign(
      {
        sub: userId.toString(),
        id: userId.toString(),
        email: foundUser.email,
        username: foundUser.username,
        userRole: foundUser.userRoleId,
        tokenVersion: tokenVersion?.tokenVersion ?? 0,
      },
      { expiresIn: rememberMe ? '7d' : '1d' },
    );

    return {
      backupCodes: plainCodes,
      sessionToken,
      user: toPublicUser(foundUser as any),
    };
  }
}
```

- [ ] **Step 3: Create TwoFactorVerifyUseCase**

Create `apps/api/src/modules/auth/use-cases/two-factor-verify.usecase.ts`:

```typescript
import {
  BadRequestException,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { authenticator } from 'otplib';
import { compare } from 'bcrypt';
import { AesEncryptionService } from '../../../common/services/aes-encryption.service';
import { toPublicUser } from '../mappers/auth.mapper';
import { AuthRepository } from '../repositories/auth.repository';

@Injectable()
export class TwoFactorVerifyUseCase {
  constructor(
    private readonly authRepo: AuthRepository,
    private readonly aesEncryptionService: AesEncryptionService,
    private readonly jwtService: JwtService,
  ) {}

  async execute(
    userId: number,
    rememberMe: boolean,
    code?: string,
    backupCode?: string,
  ) {
    if (!code && !backupCode) {
      throw new BadRequestException('code or backupCode is required');
    }

    const foundUser = await this.authRepo.findByIdWithTwoFactor(userId);
    if (
      !foundUser ||
      !foundUser.twoFactorSecret ||
      !foundUser.twoFactorEnabled
    ) {
      throw new UnauthorizedException('Invalid code');
    }

    if (code) {
      // Verify TOTP
      const rawSecret = this.aesEncryptionService.decrypt(
        foundUser.twoFactorSecret,
      );
      const isValid = authenticator.verify({ token: code, secret: rawSecret });
      if (!isValid) {
        throw new UnauthorizedException('Invalid code');
      }
    } else if (backupCode) {
      // Verify backup code
      const unusedCodes = await this.authRepo.findUnusedBackupCodes(userId);
      let matchedId: number | null = null;

      for (const stored of unusedCodes) {
        const isMatch = await compare(backupCode, stored.codeHash);
        if (isMatch) {
          matchedId = stored.id;
          break;
        }
      }

      if (!matchedId) {
        throw new UnauthorizedException('Invalid code');
      }

      // Atomic redeem
      const redeemed = await this.authRepo.redeemBackupCode(matchedId);
      if (!redeemed) {
        throw new UnauthorizedException('Invalid code');
      }
    }

    // Issue full session JWT
    const tokenVersion = await this.authRepo.findTokenVersion(userId);
    const sessionToken = this.jwtService.sign(
      {
        sub: userId.toString(),
        id: userId.toString(),
        email: foundUser.email,
        username: foundUser.username,
        userRole: foundUser.userRoleId,
        tokenVersion: tokenVersion?.tokenVersion ?? 0,
      },
      { expiresIn: rememberMe ? '7d' : '1d' },
    );

    return {
      sessionToken,
      user: toPublicUser(foundUser as any),
    };
  }
}
```

- [ ] **Step 4: Create ResetUserTwoFactorUseCase**

Create `apps/api/src/modules/auth/use-cases/reset-user-two-factor.usecase.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { AuthRepository } from '../repositories/auth.repository';

@Injectable()
export class ResetUserTwoFactorUseCase {
  constructor(private readonly authRepo: AuthRepository) {}

  async execute(userId: number): Promise<void> {
    await this.authRepo.clearTwoFactor(userId);
    await this.authRepo.incrementTokenVersion(userId);
  }
}
```

- [ ] **Step 5: Create/update use-cases barrel file**

Create or update `apps/api/src/modules/auth/use-cases/index.ts`:

```typescript
export { ValidateCredentialsUseCase } from './validate-credentials.usecase';
export { FindUserByUsernameUseCase } from './find-user-by-username.usecase';
export { GetTokenVersionUseCase } from './get-token-version.usecase';
export { InvalidateUserSessionsUseCase } from './invalidate-user-sessions.usecase';
export { ResetTokenVersionUseCase } from './reset-token-version.usecase';
export { TwoFactorSetupQrUseCase } from './two-factor-setup-qr.usecase';
export { TwoFactorConfirmUseCase } from './two-factor-confirm.usecase';
export { TwoFactorVerifyUseCase } from './two-factor-verify.usecase';
export { ResetUserTwoFactorUseCase } from './reset-user-two-factor.usecase';
```

Note: If this barrel file already exists, only add the four new exports. If it doesn't exist, check which existing use-cases need to be exported and include them.

- [ ] **Step 6: Verify build**

```bash
cd apps/api && pnpm build
```

Expected: Success (or minor import issues to fix)

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/modules/auth/use-cases/
git commit -m "feat(auth): add 2FA use-cases for setup, verify, and admin reset"
```

---

### Task 9: Auth Controller — 2FA endpoints + module wiring

**Files:**
- Modify: `apps/api/src/modules/auth/auth.controller.ts`
- Modify: `apps/api/src/modules/auth/auth.module.ts`

- [ ] **Step 1: Add new endpoint imports to controller**

Add imports at top of `auth.controller.ts`:

```typescript
import { UseGuards } from '@nestjs/common';
import { TwoFactorSetupDto, TwoFactorVerifyDto } from './dto';
import {
  PartialJwtGuard,
  PartialJwtPayload,
} from './guards/partial-jwt.guard';
import { TwoFactorSetupQrUseCase } from './use-cases/two-factor-setup-qr.usecase';
import { TwoFactorConfirmUseCase } from './use-cases/two-factor-confirm.usecase';
import { TwoFactorVerifyUseCase } from './use-cases/two-factor-verify.usecase';
```

- [ ] **Step 2: Inject new use-cases in constructor**

Add to the constructor parameters:

```typescript
private readonly twoFactorSetupQrUseCase: TwoFactorSetupQrUseCase,
private readonly twoFactorConfirmUseCase: TwoFactorConfirmUseCase,
private readonly twoFactorVerifyUseCase: TwoFactorVerifyUseCase,
```

And `private readonly authRepo: AuthRepository` (if not added in Task 5).

- [ ] **Step 3: Add GET /auth/2fa/setup/qr endpoint**

```typescript
@Public()
@UseGuards(PartialJwtGuard)
@Get('2fa/setup/qr')
@HttpCode(HttpStatus.OK)
async getTwoFactorSetupQr(@Request() req: ExpressRequest) {
  const payload = (req as any).partialJwtPayload as PartialJwtPayload;
  if (!payload.enrollmentRequired) {
    throw new BadRequestException('2FA already enrolled');
  }
  const userId = parseInt(payload.sub, 10);
  return this.twoFactorSetupQrUseCase.execute(userId);
}
```

- [ ] **Step 4: Add POST /auth/2fa/setup endpoint**

```typescript
@Public()
@UseGuards(PartialJwtGuard)
@Throttle({ default: { limit: 5, ttl: 60000 } })
@Post('2fa/setup')
@HttpCode(HttpStatus.OK)
async confirmTwoFactorSetup(
  @Body() dto: TwoFactorSetupDto,
  @Request() req: ExpressRequest,
) {
  const payload = (req as any).partialJwtPayload as PartialJwtPayload;
  if (!payload.enrollmentRequired) {
    throw new BadRequestException('2FA already enrolled');
  }
  const userId = parseInt(payload.sub, 10);
  return this.twoFactorConfirmUseCase.execute(
    userId,
    dto.code,
    dto.encryptedSecret,
    payload.rememberMe ?? false,
  );
}
```

- [ ] **Step 5: Add POST /auth/2fa/verify endpoint**

```typescript
@Public()
@UseGuards(PartialJwtGuard)
@Throttle({ default: { limit: 5, ttl: 60000 } })
@Post('2fa/verify')
@HttpCode(HttpStatus.OK)
async verifyTwoFactor(
  @Body() dto: TwoFactorVerifyDto,
  @Request() req: ExpressRequest,
) {
  const payload = (req as any).partialJwtPayload as PartialJwtPayload;
  if (payload.enrollmentRequired) {
    throw new BadRequestException('2FA enrollment required first');
  }
  const userId = parseInt(payload.sub, 10);
  return this.twoFactorVerifyUseCase.execute(
    userId,
    payload.rememberMe ?? false,
    dto.code,
    dto.backupCode,
  );
}
```

- [ ] **Step 6: Update auth.module.ts providers**

Add to imports at top of `auth.module.ts`:

```typescript
import { ConfigModule } from '@nestjs/config';
import { AesEncryptionService } from '../../common/services/aes-encryption.service';
import { PartialJwtGuard } from './guards/partial-jwt.guard';
import { TwoFactorSetupQrUseCase } from './use-cases/two-factor-setup-qr.usecase';
import { TwoFactorConfirmUseCase } from './use-cases/two-factor-confirm.usecase';
import { TwoFactorVerifyUseCase } from './use-cases/two-factor-verify.usecase';
import { ResetUserTwoFactorUseCase } from './use-cases/reset-user-two-factor.usecase';
```

Add `ConfigModule` to `imports` array (if not already there).

Add to `providers` array:

```typescript
AesEncryptionService,
PartialJwtGuard,
TwoFactorSetupQrUseCase,
TwoFactorConfirmUseCase,
TwoFactorVerifyUseCase,
ResetUserTwoFactorUseCase,
```

Add to `exports` array:

```typescript
ResetUserTwoFactorUseCase,
AesEncryptionService,
```

- [ ] **Step 7: Verify build**

```bash
cd apps/api && pnpm build
```

Expected: Success

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/modules/auth/
git commit -m "feat(auth): add 2FA controller endpoints and wire up auth module"
```

---

### Task 10: Admin reset endpoint

**Files:**
- Modify: `apps/api/src/modules/user/user.controller.ts`
- Modify: `apps/api/src/modules/user/user.module.ts`

- [ ] **Step 1: Import and inject in UserController**

Add to imports in `user.controller.ts`:

```typescript
import { HttpCode, HttpStatus } from '@nestjs/common'; // Add these if not already imported
import { ResetUserTwoFactorUseCase } from '../auth/use-cases/reset-user-two-factor.usecase';
```

Add to constructor: `private readonly resetUserTwoFactorUseCase: ResetUserTwoFactorUseCase`

- [ ] **Step 2: Add reset endpoint**

Follow the existing activity logging pattern in `UserController` — use `this.auditService.getCurrentUserFromRequest(req)` to get the actor, and pass the user object to `resolveActorType()`. The activity log code is `USER003_USER_UPDATE` (not `USER003_UPDATE`).

```typescript
@Post(':id/2fa/reset')
@Roles('ADMIN')
@HttpCode(HttpStatus.OK)
async resetTwoFactor(
  @Param('id', ParseIntPipe) id: number,
  @Request() req: ExpressRequest,
) {
  await this.resetUserTwoFactorUseCase.execute(id);

  const currentUser = await this.auditService.getCurrentUserFromRequest(req);
  this.activityLogService.logByCode('USER003_USER_UPDATE', 'success', {
    actorId: currentUser?.id ?? null,
    actorType: resolveActorType(currentUser),
    entityId: id,
    entityType: 'user',
    metadata: { action: '2fa_reset' },
    req,
  });

  return { success: true };
}
```

- [ ] **Step 3: Verify AuthModule import in UserModule**

`AuthModule` is already imported in `apps/api/src/modules/user/user.module.ts`. Since Task 9 exports `ResetUserTwoFactorUseCase` from `AuthModule`, it will be available for injection in `UserController` without any changes to `user.module.ts`. Verify this is the case — if `AuthModule` is NOT in the imports, add it.

- [ ] **Step 4: Verify build**

```bash
cd apps/api && pnpm build
```

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/modules/user/user.controller.ts apps/api/src/modules/user/user.module.ts
git commit -m "feat(admin): add POST /users/:id/2fa/reset endpoint for admin 2FA reset"
```

---

## Chunk 3: Frontend

### Task 11: NextAuth type extensions + auth.config.ts

**Files:**
- Create: `apps/web/src/auth.d.ts`
- Create: `apps/web/src/lib/auth/get-partial-token.ts`
- Modify: `apps/web/src/auth.config.ts`

- [ ] **Step 1: Create NextAuth module augmentation**

Create `apps/web/src/auth.d.ts`:

```typescript
import type { DefaultSession } from 'next-auth';

declare module 'next-auth' {
  interface Session extends DefaultSession {
    sessionToken?: string;
    twoFactorPending?: boolean;
    enrollmentRequired?: boolean;
    user?: DefaultSession['user'] & {
      id?: string;
      username?: string;
      userRole?: string;
      role?: string;
      companyId?: string | number;
    };
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    id?: string | number;
    username?: string;
    userRole?: string;
    role?: string;
    companyId?: string | number;
    sessionToken?: string;
    rememberMe?: boolean;
    twoFactorPending?: boolean;
    enrollmentRequired?: boolean;
    partialTokenInternal?: string;
  }
}
```

- [ ] **Step 2: Create server-side partial token helper**

The `partialTokenInternal` field is stored in the NextAuth JWT (HttpOnly cookie), NOT in the session object. Server actions cannot use `auth()` to read it — `auth()` returns the session (which strips `partialTokenInternal`). Instead, use `getToken()` from `next-auth/jwt` to decode the raw JWT cookie.

Create `apps/web/src/lib/auth/get-partial-token.ts`:

```typescript
import { cookies } from 'next/headers';
import { decode } from 'next-auth/jwt';
import type { JWT } from 'next-auth/jwt';

/**
 * Reads the partialTokenInternal from the NextAuth JWT cookie.
 * This field is NOT available via auth() / useSession() — it lives only in the JWT.
 */
export async function getPartialToken(): Promise<string | null> {
  const cookieStore = await cookies();
  const sessionCookie =
    cookieStore.get('__Secure-authjs.session-token') ??
    cookieStore.get('authjs.session-token');

  if (!sessionCookie?.value) return null;

  try {
    const token = await decode({
      token: sessionCookie.value,
      salt: sessionCookie.name,
      secret: process.env.AUTH_SECRET || '',
    });
    // JWT type is augmented in auth.d.ts to include partialTokenInternal
    return (token as JWT)?.partialTokenInternal ?? null;
  } catch {
    return null;
  }
}
```

- [ ] **Step 3: Add 2FA types to auth.config.ts**

Add to the `AuthUser` type:

```typescript
type AuthUser = User & {
  username?: string;
  userRole?: string;
  userRoleId?: string;
  role?: string;
  companyId?: string | number;
  sessionToken?: string;
  rememberMe?: boolean;
  // 2FA fields
  twoFactorPending?: boolean;
  enrollmentRequired?: boolean;
  partialToken?: string;
};
```

Add to `ExtendedToken` type:

```typescript
type ExtendedToken = AppJWT & {
  // ... existing fields ...
  twoFactorPending?: boolean;
  enrollmentRequired?: boolean;
  partialTokenInternal?: string;
};
```

Add to `SessionWithToken` type:

```typescript
type SessionWithToken = Session & {
  sessionToken?: string;
  twoFactorPending?: boolean;
  enrollmentRequired?: boolean;
  user?: Session['user'] & {
    id?: string;
    username?: string;
    userRole?: string;
    role?: string;
    companyId?: string | number;
  };
};
```

- [ ] **Step 4: Modify the `authorize` function in Credentials provider**

Replace the section after `const user = payload?.data ?? payload;`:

```typescript
const user = payload?.data ?? payload;

// Handle 2FA response (status + partialToken)
if (
  user?.status === '2FA_REQUIRED' ||
  user?.status === 'ENROLLMENT_REQUIRED'
) {
  return {
    id: 'pending-2fa',
    twoFactorPending: true,
    enrollmentRequired: user.status === 'ENROLLMENT_REQUIRED',
    partialToken: user.partialToken,
    rememberMe,
  };
}

// Legacy: direct session (should not happen with 2FA enforced)
if (!user || !user.id) {
  return null;
}

return {
  ...user,
  rememberMe,
};
```

- [ ] **Step 5: Modify the `jwt` callback**

Replace the `if (user) {` block:

```typescript
if (user) {
  const authUser = user as AuthUser;
  const extendedToken = token as ExtendedToken;

  // Handle 2FA pending session
  if (authUser.twoFactorPending) {
    extendedToken.twoFactorPending = true;
    extendedToken.enrollmentRequired = authUser.enrollmentRequired;
    extendedToken.partialTokenInternal = authUser.partialToken;
    extendedToken.id = 'pending-2fa';
    extendedToken.rememberMe = authUser.rememberMe;
    // 5-minute expiry for the pending session
    token.exp = Math.floor(Date.now() / 1000) + 300;
    return token;
  }

  // Full session (existing logic)
  extendedToken.id = user.id;
  extendedToken.username =
    authUser.username || authUser.email || undefined;
  extendedToken.name =
    authUser.name || authUser.username || authUser.email;
  extendedToken.userRole = authUser.userRole ?? authUser.userRoleId;
  extendedToken.role = authUser.role;
  extendedToken.companyId = authUser.companyId;
  extendedToken.sessionToken = authUser.sessionToken;

  const rememberMe = authUser.rememberMe;
  extendedToken.rememberMe = rememberMe;
  if (rememberMe) {
    token.exp = Math.floor(Date.now() / 1000) + SEVEN_DAYS_SECONDS;
  } else {
    token.exp = Math.floor(Date.now() / 1000) + ONE_DAY_SECONDS;
  }
}
```

Modify the `if (trigger === 'update' && session) {` block to handle 2FA completion:

```typescript
if (trigger === 'update' && session) {
  const extendedToken = token as ExtendedToken;

  // Handle 2FA completion — upgrade pending session to full session
  if (session.twoFactorPending === false && session.sessionToken) {
    extendedToken.twoFactorPending = false;
    extendedToken.enrollmentRequired = undefined;
    extendedToken.partialTokenInternal = undefined;
    extendedToken.sessionToken = session.sessionToken;
    extendedToken.id = session.user?.id;
    extendedToken.username = session.user?.username;
    extendedToken.name = session.user?.name;
    extendedToken.userRole = session.user?.userRole ?? session.user?.userRoleId;
    extendedToken.role = session.user?.role;
    extendedToken.companyId = session.user?.companyId;
    // Set proper expiry
    const rememberMe = extendedToken.rememberMe;
    token.exp = Math.floor(Date.now() / 1000) +
      (rememberMe ? SEVEN_DAYS_SECONDS : ONE_DAY_SECONDS);
    return token;
  }

  // Existing update handling
  extendedToken.userRole = session.user?.userRole;
  extendedToken.role = session.user?.role;
  extendedToken.companyId = session.user?.companyId;
}
```

- [ ] **Step 6: Modify the `session` callback**

Replace the body:

```typescript
async session({ session, token }) {
  const extendedToken = token as ExtendedToken;
  const sessionWithToken = session as SessionWithToken;

  // Pending 2FA session — return minimal data, NO partialToken
  if (extendedToken.twoFactorPending) {
    sessionWithToken.twoFactorPending = true;
    sessionWithToken.enrollmentRequired =
      extendedToken.enrollmentRequired;
    return sessionWithToken;
  }

  // Full session (existing logic)
  if (session.user) {
    const sessionUser = session.user as NonNullable<
      SessionWithToken['user']
    >;
    sessionUser.id = (extendedToken.id ?? '') as string;
    sessionUser.username =
      (extendedToken.username as string | undefined) ??
      sessionUser?.email ??
      '';
    sessionUser.name =
      (extendedToken as { name?: string }).name || sessionUser?.name;
    sessionUser.userRole = extendedToken.userRole;
    sessionUser.role = extendedToken.role;
    sessionUser.companyId = extendedToken.companyId;
  }

  sessionWithToken.sessionToken = extendedToken.sessionToken;
  return sessionWithToken;
},
```

- [ ] **Step 7: Verify build**

```bash
cd apps/web && pnpm build
```

- [ ] **Step 8: Commit**

```bash
git add apps/web/src/auth.d.ts apps/web/src/lib/auth/get-partial-token.ts apps/web/src/auth.config.ts
git commit -m "feat(auth): handle 2FA pending session in NextAuth config"
```

---

### Task 12: Middleware — 2FA redirect

**Files:**
- Modify: `apps/web/src/middleware.ts`

> **Note:** The `/2fa/*` routes do NOT need to be added to any public routes list. They are accessed by authenticated users who have a pending 2FA session — the middleware handles routing them correctly.

- [ ] **Step 1: Add 2FA redirect logic**

Add after the `const isAuthenticated = !!session?.user;` line, before the role-based check. With the `auth.d.ts` module augmentation from Task 11, these fields are typed properly:

```typescript
// 2FA pending session — redirect to appropriate 2FA page
if (isAuthenticated && session?.twoFactorPending) {
  const twoFaPath = session.enrollmentRequired ? '/2fa/setup' : '/2fa/verify';

  // Allow access to the correct 2FA page, auth API routes, and escape routes
  if (
    pathname.startsWith(twoFaPath) ||
    pathname.startsWith('/api/auth') ||
    pathname === '/login' ||
    pathname === '/logout'
  ) {
    return NextResponse.next();
  }

  // Block all other routes — redirect to 2FA
  return NextResponse.redirect(new URL(twoFaPath, request.url));
}
```

- [ ] **Step 2: Verify build**

```bash
cd apps/web && pnpm build
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/middleware.ts
git commit -m "feat(auth): redirect pending 2FA sessions in middleware"
```

---

### Task 13: i18n translations

**Files:**
- Create: `apps/web/src/locales/dictionary/ja/two-factor.ts`
- Create: `apps/web/src/locales/dictionary/en/two-factor.ts`
- Modify: `apps/web/src/locales/dictionary/ja/index.ts`
- Modify: `apps/web/src/locales/dictionary/en/index.ts`

- [ ] **Step 1: Create Japanese translations**

Create `apps/web/src/locales/dictionary/ja/two-factor.ts`:

```typescript
export default {
  verify: {
    title: '二要素認証',
    subtitle: '認証アプリの6桁コードを入力してください',
    codeLabel: '認証コード',
    codePlaceholder: '6桁コードを入力',
    verifyButton: '認証する',
    useBackupCode: 'バックアップコードを使用する',
    backupCodeLabel: 'バックアップコード',
    backupCodePlaceholder: 'XXXX-XXXX',
    backupCodeSubmit: 'バックアップコードで認証',
    backToCode: '認証コードに戻る',
    error: {
      invalidCode: '無効なコードです。もう一度お試しください。',
      sessionExpired:
        'セッションが期限切れです。再度ログインしてください。',
      unknown: 'エラーが発生しました。もう一度お試しください。',
    },
  },

  setup: {
    title: '二要素認証の設定',
    step1: {
      title: '認証アプリの設定',
      subtitle:
        'Google Authenticator、Authy などの認証アプリでこのQRコードをスキャンしてください',
      nextButton: '次へ — スキャンしました',
    },
    step2: {
      title: 'コードの確認',
      subtitle:
        '認証アプリに表示される6桁コードを入力して設定を確認してください',
      confirmButton: '確認して2FAを有効にする',
      backButton: '戻る',
    },
    step3: {
      title: 'バックアップコードを保存',
      warning:
        'これらのコードを安全な場所に保存してください。各コードは1回のみ使用可能で、再表示されません。',
      copyButton: 'すべてコピー',
      copiedMessage: 'コピーしました',
      doneButton: '保存しました — アプリに進む',
    },
    error: {
      invalidCode: '無効なコードです。もう一度お試しください。',
      sessionExpired:
        'セッションが期限切れです。再度ログインしてください。',
      tamperedSecret: '無効なリクエストです。最初からやり直してください。',
      unknown: 'エラーが発生しました。もう一度お試しください。',
    },
  },
  admin: {
    resetButton: '2FAリセット',
    resetConfirmTitle: '二要素認証のリセット',
    resetConfirmMessage:
      'このユーザーの2FAを無効にします。次回ログイン時に2FAの再設定が必要になります。既存のセッションはすべて無効化されます。',
    resetSuccess: '2FAがリセットされました',
  },
} as const;
```

- [ ] **Step 2: Create English translations**

Create `apps/web/src/locales/dictionary/en/two-factor.ts`:

```typescript
export default {
  verify: {
    title: 'Two-Factor Authentication',
    subtitle: 'Enter the 6-digit code from your authenticator app',
    codeLabel: 'Authentication code',
    codePlaceholder: 'Enter 6-digit code',
    verifyButton: 'Verify',
    useBackupCode: 'Use a backup code instead',
    backupCodeLabel: 'Backup code',
    backupCodePlaceholder: 'XXXX-XXXX',
    backupCodeSubmit: 'Verify with backup code',
    backToCode: 'Back to authentication code',
    error: {
      invalidCode: 'Invalid code. Please try again.',
      sessionExpired: 'Session expired. Please log in again.',
      unknown: 'An error occurred. Please try again.',
    },
  },

  setup: {
    title: 'Set Up Two-Factor Authentication',
    step1: {
      title: 'Set Up Authenticator',
      subtitle:
        'Scan this QR code with Google Authenticator, Authy, or a similar app',
      nextButton: "Next — I've scanned it",
    },
    step2: {
      title: 'Confirm Your Code',
      subtitle:
        'Enter the 6-digit code your app shows to verify setup',
      confirmButton: 'Confirm & Enable 2FA',
      backButton: 'Back',
    },
    step3: {
      title: 'Save Your Backup Codes',
      warning:
        'Store these somewhere safe. Each code works once and will not be shown again.',
      copyButton: 'Copy all',
      copiedMessage: 'Copied!',
      doneButton: "I've saved them — Enter app",
    },
    error: {
      invalidCode: 'Invalid code. Please try again.',
      sessionExpired: 'Session expired. Please log in again.',
      tamperedSecret: 'Invalid request. Please start over.',
      unknown: 'An error occurred. Please try again.',
    },
  },

  admin: {
    resetButton: 'Reset 2FA',
    resetConfirmTitle: 'Reset Two-Factor Authentication',
    resetConfirmMessage:
      'This will disable 2FA for this user. They will need to set up 2FA again on their next login. All existing sessions will be invalidated.',
    resetSuccess: '2FA has been reset',
  },
} as const;
```

- [ ] **Step 3: Update per-language barrel files**

Add to `apps/web/src/locales/dictionary/ja/index.ts`:

```typescript
export { default as twoFactor } from './two-factor';
```

Add the same line to `apps/web/src/locales/dictionary/en/index.ts`.

- [ ] **Step 4: Register in central dictionary index**

**IMPORTANT:** Without this step, translations will silently fail at runtime — `useTranslation()` will return key strings instead of translations.

Update `apps/web/src/locales/dictionary/index.ts`:

1. Add a type export near the other dictionary types:
```typescript
export type TwoFactorDictionary = DeepStringify<typeof ja.twoFactor>;
```

2. Add to the `Dictionary` type:
```typescript
twoFactor: TwoFactorDictionary;
```

3. Add to both language entries in the `dictionaries` record:
```typescript
// In the 'en' entry:
twoFactor: en.twoFactor,

// In the 'ja' entry:
twoFactor: ja.twoFactor,
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/locales/
git commit -m "feat(i18n): add Japanese and English translations for 2FA"
```

---

### Task 14: /2fa/verify page — server actions + components

**Files:**
- Create: `apps/web/src/app/2fa/verify/actions/verifyTotp.action.ts`
- Create: `apps/web/src/app/2fa/verify/_components/VerifyForm.tsx`
- Create: `apps/web/src/app/2fa/verify/_components/BackupCodeForm.tsx`
- Create: `apps/web/src/app/2fa/verify/page.tsx`

- [ ] **Step 1: Create verifyTotp server action**

Create `apps/web/src/app/2fa/verify/actions/verifyTotp.action.ts`:

**IMPORTANT:** `partialTokenInternal` is stored only in the NextAuth JWT, NOT in the session object returned by `auth()`. Server actions must use the `getPartialToken()` helper (which decodes the JWT cookie directly) to read it.

```typescript
'use server';

import { unstable_update } from '@/auth';
import { getPartialToken } from '@/lib/auth/get-partial-token';

export type VerifyTotpResult =
  | { success: true }
  | {
      success: false;
      error: 'INVALID_CODE' | 'SESSION_EXPIRED' | 'UNKNOWN';
    };

export async function verifyTotpAction(
  code: string,
): Promise<VerifyTotpResult> {
  const partialToken = await getPartialToken();

  if (!partialToken) {
    return { success: false, error: 'SESSION_EXPIRED' };
  }

  const apiUrl = (
    process.env.API_URL || process.env.NEXT_PUBLIC_API_URL
  )?.trim();
  if (!apiUrl) return { success: false, error: 'UNKNOWN' };

  try {
    const res = await fetch(`${apiUrl}/auth/2fa/verify`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${partialToken}`,
      },
      body: JSON.stringify({ code }),
    });

    if (!res.ok) {
      return {
        success: false,
        error: res.status === 401 ? 'INVALID_CODE' : 'UNKNOWN',
      };
    }

    const json = await res.json();
    const data = json?.data ?? json;

    await unstable_update({
      sessionToken: data.sessionToken,
      user: data.user,
      twoFactorPending: false,
    });

    return { success: true };
  } catch {
    return { success: false, error: 'UNKNOWN' };
  }
}

export async function verifyBackupCodeAction(
  backupCode: string,
): Promise<VerifyTotpResult> {
  const partialToken = await getPartialToken();

  if (!partialToken) {
    return { success: false, error: 'SESSION_EXPIRED' };
  }

  const apiUrl = (
    process.env.API_URL || process.env.NEXT_PUBLIC_API_URL
  )?.trim();
  if (!apiUrl) return { success: false, error: 'UNKNOWN' };

  try {
    const res = await fetch(`${apiUrl}/auth/2fa/verify`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${partialToken}`,
      },
      body: JSON.stringify({ backupCode }),
    });

    if (!res.ok) {
      return {
        success: false,
        error: res.status === 401 ? 'INVALID_CODE' : 'UNKNOWN',
      };
    }

    const json = await res.json();
    const data = json?.data ?? json;

    await unstable_update({
      sessionToken: data.sessionToken,
      user: data.user,
      twoFactorPending: false,
    });

    return { success: true };
  } catch {
    return { success: false, error: 'UNKNOWN' };
  }
}
```

- [ ] **Step 2: Create VerifyForm component**

Create `apps/web/src/app/2fa/verify/_components/VerifyForm.tsx`:

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { Alert, Button, PinInput, Stack, Text } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { verifyTotpAction } from '../actions/verifyTotp.action';

type Props = {
  onSwitchToBackup: () => void;
};

export const VerifyForm: React.FC<Props> = ({ onSwitchToBackup }) => {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const router = useRouter();
  const [code, setCode] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    if (code.length !== 6) return;
    setError(null);

    startTransition(async () => {
      const result = await verifyTotpAction(code);
      if (result.success) {
        router.replace('/admin');
        router.refresh();
      } else if (result.error === 'SESSION_EXPIRED') {
        router.replace('/login?error=session_expired');
      } else {
        setError(dict.verify.error.invalidCode);
        setCode('');
      }
    });
  };

  return (
    <Stack gap="md">
      <Text ta="center" size="sm" c="dimmed">
        {dict.verify.subtitle}
      </Text>

      {error && (
        <Alert color="red" variant="light">
          {error}
        </Alert>
      )}

      <PinInput
        length={6}
        type="number"
        value={code}
        onChange={setCode}
        onComplete={handleSubmit}
        size="lg"
        style={{ justifyContent: 'center' }}
        autoFocus
      />

      <Button
        onClick={handleSubmit}
        loading={isPending}
        disabled={code.length !== 6}
        fullWidth
      >
        {dict.verify.verifyButton}
      </Button>

      <Button
        variant="subtle"
        size="sm"
        onClick={onSwitchToBackup}
      >
        {dict.verify.useBackupCode}
      </Button>
    </Stack>
  );
};
```

- [ ] **Step 3: Create BackupCodeForm component**

Create `apps/web/src/app/2fa/verify/_components/BackupCodeForm.tsx`:

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { Alert, Button, Stack, Text, TextInput } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { verifyBackupCodeAction } from '../actions/verifyTotp.action';

type Props = {
  onSwitchToCode: () => void;
};

export const BackupCodeForm: React.FC<Props> = ({ onSwitchToCode }) => {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const router = useRouter();
  const [backupCode, setBackupCode] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    if (!backupCode.trim()) return;
    setError(null);

    startTransition(async () => {
      const result = await verifyBackupCodeAction(backupCode.trim());
      if (result.success) {
        router.replace('/admin');
        router.refresh();
      } else if (result.error === 'SESSION_EXPIRED') {
        router.replace('/login?error=session_expired');
      } else {
        setError(dict.verify.error.invalidCode);
        setBackupCode('');
      }
    });
  };

  return (
    <Stack gap="md">
      {error && (
        <Alert color="red" variant="light">
          {error}
        </Alert>
      )}

      <TextInput
        label={dict.verify.backupCodeLabel}
        placeholder={dict.verify.backupCodePlaceholder}
        value={backupCode}
        onChange={(e) =>
          setBackupCode(e.currentTarget.value.toUpperCase())
        }
        autoFocus
      />

      <Button
        onClick={handleSubmit}
        loading={isPending}
        disabled={!backupCode.trim()}
        fullWidth
      >
        {dict.verify.backupCodeSubmit}
      </Button>

      <Button variant="subtle" size="sm" onClick={onSwitchToCode}>
        {dict.verify.backToCode}
      </Button>
    </Stack>
  );
};
```

- [ ] **Step 4: Create page.tsx**

Create `apps/web/src/app/2fa/verify/page.tsx`:

The project uses a `TranslationProvider` context + `useTranslation()` hook for i18n. Each component calls `useTranslation()` directly — no prop drilling of `dict` needed. The `twoFactor` key must be registered in the central dictionary (Task 13 Step 4) for this to work.

```tsx
'use client';

import { useState } from 'react';
import { Card, Container, Title } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { BackupCodeForm } from './_components/BackupCodeForm';
import { VerifyForm } from './_components/VerifyForm';

export default function TwoFactorVerifyPage() {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const [mode, setMode] = useState<'code' | 'backup'>('code');

  return (
    <Container size={420} py={40}>
      <Title ta="center" order={2} mb="lg">
        {dict.verify.title}
      </Title>
      <Card shadow="sm" padding="lg" radius="md" withBorder>
        {mode === 'code' ? (
          <VerifyForm onSwitchToBackup={() => setMode('backup')} />
        ) : (
          <BackupCodeForm onSwitchToCode={() => setMode('code')} />
        )}
      </Card>
    </Container>
  );
}
```

- [ ] **Step 5: Verify build**

```bash
cd apps/web && pnpm build
```

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/app/2fa/verify/
git commit -m "feat(2fa): add /2fa/verify page with TOTP and backup code forms"
```

---

### Task 15: /2fa/setup page — server actions + wizard

**Files:**
- Create: `apps/web/src/app/2fa/setup/actions/getSetupQr.action.ts`
- Create: `apps/web/src/app/2fa/setup/actions/confirmSetup.action.ts`
- Create: `apps/web/src/app/2fa/setup/actions/completeSetup.action.ts`
- Create: `apps/web/src/app/2fa/setup/_components/QrStep.tsx`
- Create: `apps/web/src/app/2fa/setup/_components/ConfirmStep.tsx`
- Create: `apps/web/src/app/2fa/setup/_components/BackupCodesStep.tsx`
- Create: `apps/web/src/app/2fa/setup/_components/SetupWizard.tsx`
- Create: `apps/web/src/app/2fa/setup/page.tsx`

- [ ] **Step 1: Create getSetupQr server action**

Create `apps/web/src/app/2fa/setup/actions/getSetupQr.action.ts`:

**IMPORTANT:** `partialTokenInternal` is stored only in the NextAuth JWT, NOT in the session object returned by `auth()`. Server actions must use the `getPartialToken()` helper (which decodes the JWT cookie directly) to read it.

```typescript
'use server';

import { getPartialToken } from '@/lib/auth/get-partial-token';

export type SetupQrResult =
  | {
      success: true;
      qrCodeDataUrl: string;
      encryptedSecret: string;
      accountName: string;
    }
  | { success: false; error: 'SESSION_EXPIRED' | 'UNKNOWN' };

export async function getSetupQrAction(): Promise<SetupQrResult> {
  const partialToken = await getPartialToken();

  if (!partialToken) {
    return { success: false, error: 'SESSION_EXPIRED' };
  }

  const apiUrl = (
    process.env.API_URL || process.env.NEXT_PUBLIC_API_URL
  )?.trim();
  if (!apiUrl) return { success: false, error: 'UNKNOWN' };

  try {
    const res = await fetch(`${apiUrl}/auth/2fa/setup/qr`, {
      headers: { Authorization: `Bearer ${partialToken}` },
    });

    if (res.status === 401) {
      return { success: false, error: 'SESSION_EXPIRED' };
    }
    if (!res.ok) return { success: false, error: 'UNKNOWN' };

    const json = await res.json();
    const data = json?.data ?? json;

    return {
      success: true,
      qrCodeDataUrl: data.qrCodeDataUrl,
      encryptedSecret: data.encryptedSecret,
      accountName: data.accountName,
    };
  } catch {
    return { success: false, error: 'UNKNOWN' };
  }
}
```

- [ ] **Step 2: Create confirmSetup server action**

Create `apps/web/src/app/2fa/setup/actions/confirmSetup.action.ts`:

**IMPORTANT:** Same as `getSetupQr.action.ts` — must use `getPartialToken()` instead of `auth()`.

```typescript
'use server';

import { getPartialToken } from '@/lib/auth/get-partial-token';

export type ConfirmSetupResult =
  | {
      success: true;
      backupCodes: string[];
      sessionToken: string;
      user: Record<string, any>;
    }
  | {
      success: false;
      error:
        | 'INVALID_CODE'
        | 'SESSION_EXPIRED'
        | 'TAMPERED_SECRET'
        | 'UNKNOWN';
    };

export async function confirmSetupAction(
  code: string,
  encryptedSecret: string,
): Promise<ConfirmSetupResult> {
  const partialToken = await getPartialToken();

  if (!partialToken) {
    return { success: false, error: 'SESSION_EXPIRED' };
  }

  const apiUrl = (
    process.env.API_URL || process.env.NEXT_PUBLIC_API_URL
  )?.trim();
  if (!apiUrl) return { success: false, error: 'UNKNOWN' };

  try {
    const res = await fetch(`${apiUrl}/auth/2fa/setup`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${partialToken}`,
      },
      body: JSON.stringify({ code, encryptedSecret }),
    });

    if (res.status === 401) {
      return { success: false, error: 'INVALID_CODE' };
    }
    if (res.status === 400) {
      return { success: false, error: 'TAMPERED_SECRET' };
    }
    if (!res.ok) return { success: false, error: 'UNKNOWN' };

    const json = await res.json();
    const data = json?.data ?? json;

    return {
      success: true,
      backupCodes: data.backupCodes,
      sessionToken: data.sessionToken,
      user: data.user,
    };
  } catch {
    return { success: false, error: 'UNKNOWN' };
  }
}
```

- [ ] **Step 3: Create completeSetup server action**

Create `apps/web/src/app/2fa/setup/actions/completeSetup.action.ts`:

```typescript
'use server';

import { unstable_update } from '@/auth';

export async function completeSetupAction(
  sessionToken: string,
  user: Record<string, any>,
): Promise<{ success: true }> {
  await unstable_update({
    sessionToken,
    user,
    twoFactorPending: false,
  });
  return { success: true };
}
```

- [ ] **Step 4: Create QrStep component**

Create `apps/web/src/app/2fa/setup/_components/QrStep.tsx`:

```tsx
'use client';

import { useEffect, useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { Button, Center, Image, Loader, Stack, Text } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { getSetupQrAction } from '../actions/getSetupQr.action';

type Props = {
  onNext: (encryptedSecret: string) => void;
};

export const QrStep: React.FC<Props> = ({ onNext }) => {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const router = useRouter();
  const [qrUrl, setQrUrl] = useState<string | null>(null);
  const [encryptedSecret, setEncryptedSecret] = useState<string | null>(
    null,
  );
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getSetupQrAction().then((result) => {
      if (result.success) {
        setQrUrl(result.qrCodeDataUrl);
        setEncryptedSecret(result.encryptedSecret);
      } else if (result.error === 'SESSION_EXPIRED') {
        router.replace('/login?error=session_expired');
      }
      setLoading(false);
    });
  }, []);

  if (loading) {
    return (
      <Center py="xl">
        <Loader />
      </Center>
    );
  }

  return (
    <Stack gap="md" align="center">
      <Text size="sm" c="dimmed" ta="center">
        {dict.setup.step1.subtitle}
      </Text>

      {qrUrl && (
        <Image
          src={qrUrl}
          alt="QR Code"
          w={200}
          h={200}
          fit="contain"
        />
      )}

      <Button
        fullWidth
        onClick={() => encryptedSecret && onNext(encryptedSecret)}
        disabled={!encryptedSecret}
      >
        {dict.setup.step1.nextButton}
      </Button>
    </Stack>
  );
};
```

- [ ] **Step 5: Create ConfirmStep component**

Create `apps/web/src/app/2fa/setup/_components/ConfirmStep.tsx`:

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { Alert, Button, Group, PinInput, Stack, Text } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { confirmSetupAction } from '../actions/confirmSetup.action';

type Props = {
  encryptedSecret: string;
  onBack: () => void;
  onNext: (
    backupCodes: string[],
    sessionToken: string,
    user: Record<string, unknown>,
  ) => void;
};

export const ConfirmStep: React.FC<Props> = ({
  encryptedSecret,
  onBack,
  onNext,
}) => {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const router = useRouter();
  const [code, setCode] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    if (code.length !== 6) return;
    setError(null);

    startTransition(async () => {
      const result = await confirmSetupAction(code, encryptedSecret);
      if (result.success) {
        onNext(result.backupCodes, result.sessionToken, result.user);
      } else if (result.error === 'SESSION_EXPIRED') {
        router.replace('/login?error=session_expired');
      } else if (result.error === 'TAMPERED_SECRET') {
        setError(dict.setup.error.tamperedSecret);
      } else {
        setError(dict.setup.error.invalidCode);
        setCode('');
      }
    });
  };

  return (
    <Stack gap="md">
      <Text size="sm" c="dimmed" ta="center">
        {dict.setup.step2.subtitle}
      </Text>

      {error && (
        <Alert color="red" variant="light">
          {error}
        </Alert>
      )}

      <PinInput
        length={6}
        type="number"
        value={code}
        onChange={setCode}
        onComplete={handleSubmit}
        size="lg"
        style={{ justifyContent: 'center' }}
        autoFocus
      />

      <Button
        onClick={handleSubmit}
        loading={isPending}
        disabled={code.length !== 6}
        fullWidth
      >
        {dict.setup.step2.confirmButton}
      </Button>

      <Button variant="subtle" size="sm" onClick={onBack}>
        {dict.setup.step2.backButton}
      </Button>
    </Stack>
  );
};
```

- [ ] **Step 6: Create BackupCodesStep component**

Create `apps/web/src/app/2fa/setup/_components/BackupCodesStep.tsx`:

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import {
  Alert,
  Button,
  CopyButton,
  SimpleGrid,
  Stack,
  Text,
} from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { completeSetupAction } from '../actions/completeSetup.action';

type Props = {
  backupCodes: string[];
  sessionToken: string;
  user: Record<string, unknown>;
};

export const BackupCodesStep: React.FC<Props> = ({
  backupCodes,
  sessionToken,
  user,
}) => {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  const handleDone = () => {
    startTransition(async () => {
      await completeSetupAction(sessionToken, user);
      router.replace('/admin');
      router.refresh();
    });
  };

  return (
    <Stack gap="md">
      <Alert color="yellow" variant="light">
        {dict.setup.step3.warning}
      </Alert>

      <SimpleGrid cols={2} spacing="xs">
        {backupCodes.map((code) => (
          <Text
            key={code}
            ta="center"
            ff="monospace"
            size="sm"
            p={6}
            bg="dark.7"
            style={{ borderRadius: 4 }}
          >
            {code}
          </Text>
        ))}
      </SimpleGrid>

      <CopyButton value={backupCodes.join('\n')}>
        {({ copied, copy }) => (
          <Button variant="light" onClick={copy} fullWidth>
            {copied
              ? dict.setup.step3.copiedMessage
              : dict.setup.step3.copyButton}
          </Button>
        )}
      </CopyButton>

      <Button onClick={handleDone} loading={isPending} fullWidth>
        {dict.setup.step3.doneButton}
      </Button>
    </Stack>
  );
};
```

- [ ] **Step 7: Create SetupWizard component**

Create `apps/web/src/app/2fa/setup/_components/SetupWizard.tsx`:

```tsx
'use client';

import { useState } from 'react';
import { Stepper } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { BackupCodesStep } from './BackupCodesStep';
import { ConfirmStep } from './ConfirmStep';
import { QrStep } from './QrStep';

type WizardState = {
  step: number;
  encryptedSecret: string | null;
  backupCodes: string[] | null;
  sessionToken: string | null;
  user: Record<string, unknown> | null;
};

export const SetupWizard: React.FC = () => {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;
  const [state, setState] = useState<WizardState>({
    step: 0,
    encryptedSecret: null,
    backupCodes: null,
    sessionToken: null,
    user: null,
  });

  return (
    <>
      <Stepper active={state.step} size="sm" mb="lg">
        <Stepper.Step label={dict.setup.step1.title} />
        <Stepper.Step label={dict.setup.step2.title} />
        <Stepper.Step label={dict.setup.step3.title} />
      </Stepper>

      {state.step === 0 && (
        <QrStep
          onNext={(encryptedSecret) =>
            setState((s) => ({ ...s, step: 1, encryptedSecret }))
          }
        />
      )}

      {state.step === 1 && state.encryptedSecret && (
        <ConfirmStep
          encryptedSecret={state.encryptedSecret}
          onBack={() => setState((s) => ({ ...s, step: 0 }))}
          onNext={(backupCodes, sessionToken, user) =>
            setState((s) => ({
              ...s,
              step: 2,
              backupCodes,
              sessionToken,
              user,
            }))
          }
        />
      )}

      {state.step === 2 &&
        state.backupCodes &&
        state.sessionToken &&
        state.user && (
          <BackupCodesStep
            backupCodes={state.backupCodes}
            sessionToken={state.sessionToken}
            user={state.user}
          />
        )}
    </>
  );
};
```

- [ ] **Step 8: Create page.tsx**

Create `apps/web/src/app/2fa/setup/page.tsx`:

Same `useTranslation()` pattern as Task 14:

```tsx
'use client';

import { Card, Container, Title } from '@mantine/core';
import { useTranslation } from '@/contexts/translation-context';
import { SetupWizard } from './_components/SetupWizard';

export default function TwoFactorSetupPage() {
  const { dictionary } = useTranslation();
  const dict = dictionary.twoFactor;

  return (
    <Container size={480} py={40}>
      <Title ta="center" order={2} mb="lg">
        {dict.setup.title}
      </Title>
      <Card shadow="sm" padding="lg" radius="md" withBorder>
        <SetupWizard />
      </Card>
    </Container>
  );
}
```

- [ ] **Step 9: Verify build**

```bash
cd apps/web && pnpm build
```

- [ ] **Step 10: Commit**

```bash
git add apps/web/src/app/2fa/setup/
git commit -m "feat(2fa): add /2fa/setup page with 3-step enrollment wizard"
```

---

### Task 16: Admin reset UI

**Files:**
- Modify: `apps/web/src/app/(authenticated)/admin/users/list/_hooks/services/useUserServices.ts` (or similar service hook)
- Modify: `apps/web/src/app/(authenticated)/admin/users/[id]/edit/page.tsx` (user edit page)

- [ ] **Step 1: Add reset 2FA function**

Check the existing pattern in `apps/web/src/app/(authenticated)/admin/users/list/_hooks/services/useUserServices.ts`. The project uses custom Jotai-based service hooks (NOT React Query `useMutation`). Follow the same pattern as `deleteUser` or similar existing functions:

```typescript
// Add a function following the existing pattern (e.g., similar to deleteUser)
const resetTwoFactor = async (userId: number) => {
  const response = await apiClient.post(`/users/${userId}/2fa/reset`);
  return response.data;
};
```

Expose it from the hook's return value alongside existing methods.

- [ ] **Step 2: Add "Reset 2FA" button to user detail page**

Add a button in the actions area of the user edit/detail page. Use the i18n strings from the `twoFactor.admin` dictionary (added in Task 13):

```tsx
<Button
  variant="outline"
  color="orange"
  onClick={() => {
    openConfirmModal({
      title: dict.twoFactor.admin.resetConfirmTitle,
      children: (
        <Text size="sm">
          {dict.twoFactor.admin.resetConfirmMessage}
        </Text>
      ),
      confirmProps: { color: 'orange' },
      onConfirm: () => resetTwoFactor.mutate(userId),
    });
  }}
>
  {dict.twoFactor.admin.resetButton}
</Button>
```

Note: Adjust the `dict` access pattern and modal API to match the project's existing modal and notification patterns. The API endpoint path is `POST /users/:id/2fa/reset` (under the user controller, not the auth controller).

- [ ] **Step 3: Verify build**

```bash
cd apps/web && pnpm build
```

- [ ] **Step 4: Commit**

```bash
git add "apps/web/src/app/(authenticated)/admin/users/"
git commit -m "feat(admin): add Reset 2FA button to user management"
```

---

### Task 17: Quality checks

- [ ] **Step 1: Format, lint, and type-check**

```bash
pnpm format && pnpm lint && pnpm check-types
```

Fix any errors before proceeding.

- [ ] **Step 2: Final commit if any fixes were needed**

Stage only the files that were fixed (do NOT use `git add -A` — use specific file paths):

```bash
git add <specific files that needed fixes>
git commit -m "fix: resolve formatting and type errors from 2FA implementation"
```
