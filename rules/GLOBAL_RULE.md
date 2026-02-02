# GLOBAL_RULE.md

> **Engineering standards for AI IDE agents**

---

## 🎯 Identity & Style

**Treat dev as senior engineer. Zero hand-holding.**

- **Terse.** No "Here's how you can…" — just deliver.
- Answer **first**, explain after (if needed).
- If ambiguous, ask **one** question.
- No moral lectures, AI disclosures, or cutoff caveats.

---

## 📦 Code Delivery

**Return deltas, not full pastes.**

- ❌ **Don't:** Return 200 lines with one change buried.
- ✅ **Do:** Return 5 lines (2 before, 1 changed, 2 after).
- Respect Prettier/formatting. Match existing style.
- Strip dead code (unused imports, vars, types).
- Flag bugs/smells outside scope. Don't silently fix.

---

## 💬 Comments

**Trigger:** `always_on`

- Explain **why**, not what. Code shows *what*.
- Comment tricky logic, non-obvious decisions.
- `TODO`/`FIXME`/`HACK` for tech debt.

**JSDoc example:**

```typescript
/**
 * Calculates total points including bonuses
 * @param userId - User identifier
 * @returns Total points earned
 * @throws {NotFoundError} If user doesn't exist
 */
async function calculatePoints(userId: string): Promise<number>
```

**Don't comment obvious code.** If `getUserById(id)` needs a comment, rename it.

---

## ⚙️ Principles

1. **SOLID** — SRP, OCP, LSP, ISP, DIP
2. **KISS** — Simple beats clever
3. **YAGNI** — Solve current problem, not hypothetical
4. **Composition > Inheritance**
5. **Fail fast** — Validate at boundaries. `if (!user) throw new NotFoundError('User', userId)`

---

## 🛡️ Error Handling

**Never swallow errors.** Catch to: recover, log+rethrow, or transform.

**TypeScript:** Use discriminated unions for typed errors.

**React:** Error boundaries for component failures.

**Node:** Unhandled rejections crash in prod. Structured errors: `{message, code, statusCode, meta?}`

---

## 🧪 Testing

**Integration > Unit.**

- Test: API contracts, DB queries, critical flows
- Don't test: trivial logic, framework internals, implementation details
- **React:** Testing Library, query by role/label
- **Mock at boundary** (API, DB), not internals
- **TDD** for well-defined problems, prototype-then-test for exploration

---

## ⚡ Performance

- **Bundle:** <200KB gzipped first-load
- **React:** Lazy-load routes, memo only when profiling shows need
- **DB:** No N+1 queries, index foreign keys, paginate
- **Images:** WebP/AVIF with fallback, lazy-load below fold

---

## 🔒 Security

- Validate server-side always. Never trust client input.
- Secrets in env vars, never code.
- Parameterized queries only: `db.query('SELECT * WHERE id = $1', [id])`
- Run `npm audit`/`pip-audit` in CI.
- bcrypt/argon2 for passwords.

---

## ♿ Accessibility

- Semantic HTML (`<button>` not `<div onClick>`)
- Keyboard navigable, labels on inputs
- WCAG AA contrast (4.5:1)
- Images need `alt`, empty for decorative

---

## 🛠️ Tech Stack

### React/TypeScript

- Functional only. Props via `interface`.
- Hooks order: state → memo → effects → handlers → render
- Avoid `useEffect` for derivable state
- Co-locate state, lift when needed

### Node.js

- Async-first. Structured logging (`pino`).
- Validate at boundary. ESM over CJS.

### Python

- Type hints + `mypy --strict`
- `asyncio` + `uvloop` for I/O
- `ruff` for lint/format
- Virtual envs via `uv`

### Astro

- Islands architecture. Static default, hydrate selectively.
- `.astro` for static, JSX only for client state.

### Eleventy

- Nunjucks or Liquid, pick one
- Plugins over custom filters
- Pre-process in JS, keep templates dumb

### Tailwind

- Utility-first. `@apply` sparingly.
- Extend theme, no arbitrary values.
- `tailwind-merge` for class conflicts.

**Tailwind vs SCSS:**
- Tailwind: 95% of cases
- SCSS: Global themes, complex animations, print
- Never mix on same element

### ShadCN

- Code generator, not library. Treat as owned code.
- `npx shadcn@latest add <component>`
- Customize via `cn()` + Tailwind
- Pair with `tailwind-merge`

### SCSS

- BEM or CSS Modules
- Max 2 levels nesting
- `@use`/`@forward`, never `@import`
- `@mixin` for patterns, avoid `@extend`

---

## 🧠 Problem Solving

**Anticipate needs.** Flag if you see:
- Repeated code → DRY
- Manual sync → Derive
- Hard-coded values → Config
- Missing errors → Add handling
- Unvalidated input → Schema

Don't auto-fix. Just flag.

**Favor contrarian approaches** when better. Speculation OK, flag `[speculative]`.

---

## 📝 Git

**Conventional Commits:** `type(scope): subject`

Types: `feat|fix|refactor|docs|test|chore|perf`

- Imperative mood, max 50 chars
- Body explains why, wrap 72 chars
- Breaking: `BREAKING CHANGE: desc`

---

## 🚫 Don't

- ❌ "Here's how you can…"
- ❌ High-level summaries for fix requests
- ❌ Repeat user's code (delta only)
- ❌ Comment obvious code
- ❌ Over-abstract
- ❌ Swallow errors silently
- ❌ Use `any` (use `unknown`)
- ❌ Skip error/loading/empty states

---

**Updated:** 2026-02-02
