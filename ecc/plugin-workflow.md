# Plugin Workflow Rule

Drop this file into `.claude/rules/` of any project that has both Superpowers and ECC installed.

## Why a Rule?

Claude Code skill auto-triggering is unreliable (~20-50% activation). Rules are **always loaded** into context, so putting skill-routing instructions here ensures Claude consistently uses the right plugin at the right time.

## The Rule

Copy the content below into `.claude/rules/plugin-workflow.md`:

---

```markdown
# Plugin Workflow — Superpowers + ECC

## Build Phase (Superpowers)

Use Superpowers skills for design and implementation:

- When brainstorming or designing a feature, use `/brainstorming`
- When creating an implementation plan, use `/writing-plans`
- When executing a plan, use `/executing-plans`
- When writing new code with TDD, use `/tdd`

## Review Phase (ECC)

After implementation, use ECC skills for quality validation:

- After writing auth, API endpoints, or user input handling code, run `/everything-claude-code:security-review`
- After writing TypeScript/React/Node.js code, run `/everything-claude-code:coding-standards`
- After writing React/Next.js components, run `/everything-claude-code:frontend-patterns`
- After writing API endpoints, run `/everything-claude-code:api-design`
- After writing backend services, run `/everything-claude-code:backend-patterns`
- Before creating a PR, run `/everything-claude-code:security-review`

## General

- Do not skip the review phase after significant code changes
- When in doubt about which skill to use, prefer the more specific one
- Both plugins can coexist — use Superpowers for building, ECC for validating
```
