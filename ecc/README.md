# Everything Claude Code (ECC) — Test Results

Testing [Everything Claude Code](https://github.com/affaan-m/everything-claude-code) plugin capabilities on our **Next.js 15 + NestJS 11** monorepo (`nextjs15-starter-kit`).

## What We Tested

### Best Practice Review

Ran 3 parallel codebase exploration agents to review the codebase. These agents performed deep file analysis — reading source files, tracing patterns, and compiling findings.

| Review | Focus Areas | Duration |
|--------|-------------|----------|
| [Frontend Review](best-practice-review.md#frontend) | App Router, state management, performance, auth | ~90s |
| [Backend Review](best-practice-review.md#backend) | Module organization, API design, DB patterns, validation | ~77s |
| [Security Review](best-practice-review.md#security) | Auth, injection, CORS, secrets, file uploads, rate limiting | ~118s |

### ECC Skills That Could Be Used

For a more targeted review, these ECC skills can be invoked directly:

| Skill | Purpose |
|-------|---------|
| `/everything-claude-code:coding-standards` | TypeScript/React/Node.js best practices |
| `/everything-claude-code:frontend-patterns` | React/Next.js patterns, state management, performance |
| `/everything-claude-code:backend-patterns` | API design, DB optimization, server-side patterns |
| `/everything-claude-code:security-review` | OWASP top 10, auth, input validation, secrets |
| `/everything-claude-code:api-design` | REST conventions, status codes, pagination, error responses |
| `/everything-claude-code:postgres-patterns` | DB schema, indexing, query optimization |

### Key Findings Summary

- **2 Critical** — Default secrets (AUTH_SECRET, MinIO creds) not validated at startup
- **6 High** — No code-splitting, no `next/Image`, POST-for-reads, no transactions
- **8 Medium** — Missing input sanitization, CORS gaps, MIME-only file validation
- **4 Low** — No filter persistence, no per-page metadata

Full results: [best-practice-review.md](best-practice-review.md)

## ECC vs Superpower

| Aspect | Superpower | ECC |
|--------|-----------|-----|
| Focus | Structured workflows (brainstorming, planning, TDD) | Engineering quality gates (review, security, best practices) |
| Strength | Interactive skills with visual output | Deep automated analysis with parallel agents |
| Best for | Design & implementation | Code review & quality assurance |

**Conclusion:** They complement each other — Superpower for building, ECC for validating.

## Recommended Workflow

**Use separately (Option B):** Build with Superpower first, then review with ECC.

Auto-triggering of skills is unreliable (~20-50% activation rate by default). While Claude Code reads all skill descriptions and *can* pick them automatically, it doesn't always fire — especially when the prompt is large or multiple plugins compete for context.

**To improve reliability**, add a `.claude/rules/` file that explicitly tells Claude when to use each plugin's skills. See [plugin-workflow.md](plugin-workflow.md) for a ready-to-use rule.

## Tools Used

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** (CLI)
- **[Everything Claude Code](https://github.com/affaan-m/everything-claude-code)** — Plugin with 100+ skills for engineering workflows
- **Method used:** Parallel Explore agents for deep codebase analysis (not specific ECC skills)
