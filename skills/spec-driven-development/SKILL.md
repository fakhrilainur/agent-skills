---
name: spec-driven-development
description: Creates specs before coding. Use when starting a new project, feature, or significant change and no specification exists yet. Use when drafting a PRD or requirements document with objectives and scope, or when requirements are unclear, ambiguous, or only exist as a vague idea. Use when a single requirement spans several independently testable capabilities and needs decomposing into a capability map of modules before specifying.
---

# Spec-Driven Development

## Overview

Write a structured specification before writing any code. The spec is the shared source of truth between you and the human engineer — it defines what we're building, why, and how we'll know it's done. Code without a spec is guessing.

## When to Use

- Starting a new project or feature
- Requirements are ambiguous or incomplete
- The change touches multiple files or modules
- You're about to make an architectural decision
- The task would take more than 30 minutes to implement

**When NOT to use:** Single-line fixes, typo corrections, or changes where requirements are unambiguous and self-contained.

## The Gated Workflow

Spec-driven development has four phases, preceded by a scope check (Phase 0) that activates only when one request bundles several independently testable capabilities. Do not advance to the next phase until the current one is validated.

```
SPECIFY ──→ PLAN ──→ TASKS ──→ IMPLEMENT
   │          │        │          │
   ▼          ▼        ▼          ▼
 Human      Human    Human      Human
 reviews    reviews  reviews    reviews
```

### Phase 0: Scope Check

Most requests describe one capability. If this one does, skip this phase and go straight to Specify — Phase 0 exists for the exception, not the rule, and it puts no hierarchy on single-capability features.

**Detection.** Decompose before specifying when a single requirement bundles several independently testable capabilities:

- The requirement names distinct capabilities with their own consumers or data (e.g. identity, billing, notifications, reporting)
- Acceptance criteria cluster into groups that could ship and be verified separately
- One capability could be cut or replaced without rewriting the others' requirements

**Propose a capability map before writing any spec.** Small and reviewable — a module table plus a build order, not a project plan:

```markdown
# Capability Map: [Initiative Name]

| Module id | Responsibility | Depends on |
|---|---|---|
| identity | Accounts, sessions, SSO | — |
| billing | Plans, invoices, payments | identity |
| notifications | Email and webhook fan-out | identity |
| reporting | Usage dashboards | billing, notifications |

Build order: identity → billing, notifications → reporting
```

- **Stable module ids.** Kebab-case, chosen once, never renamed mid-initiative. Specs, plans, and downstream commands select work by these ids instead of guessing which spec is active.
- **Dependency direction, no cycles.** Arrows point one way. If two modules each need the other, they are one module.
- **Interfaces live at the boundary.** The map records that `billing` depends on `identity`; the contract between them belongs in the provider module's spec (see `api-and-interface-design` for designing it).

**The map is gated like every phase.** The human reviews module boundaries, dependency direction, and build order before any module spec is written. Getting the map wrong is expensive; reviewing ten lines is not.

**Then recurse per module.** Run Specify → Plan → Tasks → Implement for each module in dependency order. Each module gets its own spec, scoped to that module's objective, boundaries, and success criteria. Save the approved map at the project root and each module's spec alongside it, named by module id (`SPEC-identity.md`, `SPEC-billing.md`) — the map, not filename guessing, is the index of what exists.

### Phase 1: Specify

Start with a high-level vision. Ask the human clarifying questions until requirements are concrete.

**Surface assumptions immediately.** Before writing any spec content, list what you're assuming:

```
ASSUMPTIONS I'M MAKING:
1. This is a web application (not native mobile)
2. Authentication uses session-based cookies (not JWT)
3. The database is PostgreSQL (based on existing Prisma schema)
4. We're targeting modern browsers only (no IE11)
→ Correct me now or I'll proceed with these.
```

Don't silently fill in ambiguous requirements. The spec's entire purpose is to surface misunderstandings *before* code gets written — assumptions are the most dangerous form of misunderstanding.

**Write a spec document covering these six core areas:**

1. **Objective & Context** — What are we building and why? Who is the user? What does current vs. desired behavior look like?

2. **Commands & Toolchain** — Full executable commands with flags, not just tool names.
   ```
   Build: npm run build
   Test: npm test -- --coverage
   Lint: npm run lint --fix
   Dev: npm run dev
   ```

3. **Project Structure & Touchpoints** — Where source code lives and an explicit, locked codebase touchpoints table before editing.

4. **Code Style & Contracts** — Concrete types and interfaces, coding patterns, and environment configuration.

5. **Testing Strategy & Edge Cases** — Test framework, behavioral truth table (table-driven test matrix), and deterministic validation scenarios.

6. **Boundaries & Invariants** — Three-tier system (Always do / Ask first / Never do), plus invariants and non-goals:
   - **Always do:** Run tests before commits, follow naming conventions, validate inputs at system boundaries
   - **Ask first:** Database schema changes, adding dependencies, changing CI config
   - **Never do:** Commit real secrets/API keys, log sensitive tokens/headers, bypass idempotency checks, edit vendor directories, remove failing tests without approval

**Spec template:**

````markdown
# Spec: <Title / Feature Slug>

Path: `SPEC.md` (or `docs/SPEC.md`)

## Frontmatter & Routing Classification
```yaml
---
lane: Lane A (Standard) | Lane B (Heavy)   # Lane B triggers: Auth, DB schema/migrations, public API contracts, financial/ledger, data integrity
verifiability: cheap | expensive          # cheap: unit test / deterministic linter; expensive: manual E2E / external sandbox / staging
production_risk: low | medium | high
estimated_size: small | medium | large
requires_independent_reviewer: bool       # Mandatory true for Lane B
touchpoints_locked: true                  # Agent is prohibited from creating or editing files outside the Touchpoints table
---
```

**Classification Rationale:**
- <Explain why this lane, risk level, and verifiability were selected. List any Lane B triggers explicitly.>

---

## 1. Objective & Codebase Touchpoints (Core Areas 1 & 3)

### Objective & User Story
- **Problem & Context:** <One concise paragraph describing the problem, target outcome, and production relevance.>
- **User Persona / Actor:** <Who is this for and why it matters.>

### Project Layout & Codebase Touchpoints
> **Strict Guardrail:** The agent must inspect the existing codebase first. Modifying or creating files outside this table without explicit human approval is prohibited (`touchpoints_locked: true`).
> *Note: If introducing/modifying `.env` variables, `.env.example` and the config validator MUST be listed here.*

| Operation | Path | Target Symbols / Functions | Notes |
|---|---|---|---|
| `MODIFY` | `src/services/billing.ts` | `calculateInvoice()` | Integrate new discount calculator |
| `CREATE` | `src/types/discount.ts` | `DiscountRule`, `DiscountResult` | Concrete types and error enums |
| `MODIFY` | `tests/billing.test.ts` | `describe('calculateInvoice')` | Add table-driven test matrix |
| `MODIFY` | `.env.example` | — | Document new configuration keys |
| `MODIFY` | `src/config/env.ts` | `EnvSchema` | Add schema validation (Zod/Envalid) |

### Current vs. Desired Behavior
- **Current Behavior** (*Must cite exact `file:line` locations*):
  - `src/services/billing.ts:45-70`: Invoices only compute base rate and fixed tax; any promo input is currently ignored.
- **Desired Behavior**:
  - Valid promo codes apply deterministic discounts before tax calculation.
  - Invoices in `PAID` or `VOID` states reject promo modifications.
  - What **must remain unchanged**: Existing tax calculation formulas and legacy invoice record structures.

*(Optional)* **Bug Reproduction & Root Cause** *(Include only for bug-fix specs)*:
- **Reproduction Command / Log / Trace:** `<CLI command, failing test, or stack trace>`
- **Confirmed Root Cause:** `<Exact code path or race condition causing failure>`
- **Regression Prevention:** `<Specific boundary test to prevent recurrence>`

---

## 2. Boundaries & Invariants (Core Area 6)
> Defining what NOT to do is as critical as defining what to build to prevent scope creep.

- **Goals:**
  - <Clear, observable outcomes achieved by this change>
- **Non-Goals (Out of Scope):**
  - <Explicitly excluded capabilities, related tasks deferred to future MRs, or UI adjustments not covered here>
  - Zero new third-party dependencies unless explicitly authorized.
- **Three-Tier Boundaries:**
  - **Always do:** <Run tests before commits, follow naming conventions, validate inputs at system boundaries>
  - **Ask first:** <Database schema changes, adding dependencies, changing CI/CD config>
  - **Never do:** <Commit real secrets/API keys, log sensitive tokens/headers, bypass idempotency checks, edit vendor directories, remove failing tests without approval>
- **Invariants (Unbreakable System Rules):**
  - <Business or technical invariants that must hold true before, during, and after this change>
  - Example: *Invoice total must never be negative; discounts exceeding subtotal clamp to 0.*
  - **Concurrency & State Invariant:** All mutating operations must be race-condition safe (e.g., use DB-level unique constraints, transactions, or optimistic locking). Concurrent identical requests must not create duplicate state or double-spend.

---

## 3. Architecture & Design Alternatives

- **Chosen Approach:** <Selected design pattern and rationale>
- **Alternatives Considered:**
  - <Alternative 1>: <Why it was rejected>
  - <Alternative 2>: <Why it was rejected>
- **Accepted Tradeoffs:** <Accepted performance costs, operational overhead, or design constraints>

---

## 4. Contracts, Types & Conventions (Core Areas 2 & 4)

### Core Toolchain Commands
```bash
Build: <npm run build / go build ./...>
Test:  <npm test -- --coverage / go test ./...>
Lint:  <npm run lint --fix / golangci-lint run>
Dev:   <npm run dev / go run main.go>
```

### Strict Code & API Contracts
> Prose explanations are prohibited here. Write exact type definitions in the target language.
```typescript
// EXACT CONTRACT: The implementation must match these types and enums verbatim.
export interface ApplyDiscountInput {
  invoiceId: string; // UUID v4
  promoCode: string; // Uppercase alphanumeric, 3-10 chars
}

export type ApplyDiscountResult =
  | { success: true; newTotalCents: number; discountAmountCents: number }
  | { success: false; errorCode: 'EXPIRED' | 'NOT_FOUND' | 'MIN_ORDER_UNMET' | 'INVALID_STATUS' };
```

### Code Style & Conventions
- <Naming conventions, formatting rules, and sample idiomatic snippet>

### Configuration & Environment Variables (.env)
> **Security & Secrets Guardrail:**
> 1. NEVER write real secret values, API keys, or private tokens in this spec, prompts, or committed files.
> 2. NEVER print or log raw `Authorization` headers, bearer tokens, or API keys in application logs, traces, or client error responses.
> 3. Production secrets MUST be loaded strictly via environment variables or a dedicated secrets manager.

| Variable Name | Action | Type / Format | Default (Dev/Test) | Required in Prod? | Description & Secret Source |
|---|---|---|---|---|---|
| `DISCOUNT_SERVICE_URL` | `ADD` | string (URL) | `"http://localhost:8081"` | yes | Internal URL for promo engine |
| `DISCOUNT_MAX_PERCENT` | `ADD` | integer (`1-100`) | `50` | no | Ceiling cap for percentage discounts |

### Database & State Changes
- **Table / Collection:** <Table name or `None`>
- **DDL / Migration:**
  ```sql
  -- Provide exact migration SQL or state "NO SCHEMA CHANGE"
  ALTER TABLE invoices ADD COLUMN discount_cents INT NOT NULL DEFAULT 0;
  ```
- **Backward Compatibility:** <Preserved / Breaking / Expand-Contract approach>

---

## 5. Testing Strategy & Behavioral Truth Table (Core Area 5)

### Testing Strategy
- **Framework & Location:** <e.g. Vitest in `tests/`, PyTest in `tests/`>
- **Test Levels:** <Unit tests for pure logic, integration tests with DB/container, mock policy>

### Behavioral Truth Table (Edge Case Matrix)
> Every single row in this table must be implemented as a distinct unit test case.

| Case ID | Initial State | Input | Expected Output | Error Code / Exception | State Mutated? |
|---|---|---|---|---|---|
| TC-1 | Invoice `DRAFT`, Total: $100 | `code: "SAVE10"` | `success: true`, New Total: $90 | `null` | YES (`discount_cents`) |
| TC-2 | Invoice `DRAFT`, Total: $20 | `code: "MIN50"` (Requires $50) | `success: false` | `MIN_ORDER_UNMET` | NO |
| TC-3 | Invoice `PAID` | `code: "SAVE10"` | `success: false` | `INVALID_STATUS` | NO |
| TC-4 | Invoice `DRAFT` | `code: ""` (Empty) | `success: false` | `VALIDATION_ERROR` | NO |
| TC-5 | Invoice `DRAFT`, Quota = 0 | `code: "SAVE10"` | `success: false` | `EXPIRED` | NO |

---

## 6. Operational Readiness (Production Safety)
*(Mandatory for Lane B; state "N/A for Lane A" if standard change)*

- **Concurrency & Race Conditions:** <Identify potential race conditions, concurrent requests, double-submits, or deadlocks; specify locking/idempotency strategy (e.g., Idempotency-Key header, distributed locks, database row-locks)>
- **Failure Modes & Recovery:** <Potential runtime breakdown (network timeout, rate limit exhausted, DB lock contention) and mitigation/circuit breaker>
- **Security & Privacy:** <Auth/scopes check, PII masking, token redaction in logs, zero plain-text secrets>
- **Performance & Budget:** <Latency budget, query complexity, timeout limits>
- **Observability:** <Structured log fields, Prometheus/StatsD metrics>
---

## 7. Rollout & Rollback Plan
*(Mandatory for Lane B; state "N/A for Lane A" if standard change)*

- **Deployment Sequence:** <1. Config/Secret -> 2. Migration -> 3. App Deploy -> 4. Cleanup>
- **Feature Flag / Gate:** <Flag name or `None`>
- **Rollback Plan:** <Deterministic safe revert procedure>
- **Blast Radius:** <Affected consumers and recovery duration>

---

## 8. Acceptance Criteria & Validation Scenarios Matrix

### Acceptance Criteria
1. [AC-1]: <Unambiguous, observable business or technical condition>
2. [AC-2]: <Unambiguous, observable business or technical condition>

### Validation Scenarios Matrix
> **Deterministic Proof of Done:** All scenarios marked `Required Before MR = yes` must pass via executable commands before code review.

| VS ID | Linked AC | Actor | Type | Deterministic Command / Reproduction Steps | Expected Result | Required Before MR |
|---|---|---|---|---|---|---|
| VS-1 | AC-1 | AI | Unit | `npm test -- tests/billing.test.ts -t "TC-1"` | Exit code 0, PASS | yes |
| VS-2 | AC-1 | AI | Unit | `npm test -- tests/billing.test.ts -t "TC-2"` | Exit code 0, PASS | yes |
| VS-3 | AC-2 | AI | Integration | `npm run test:integration` | Exit code 0, PASS | yes |
| VS-4 | All | AI | Static | `npx tsc --noEmit && npm run lint` | 0 errors, exit code 0 | yes |
| VS-5 | AC-1 | Human | UI QA | Open checkout page, input promo code, observe line item | Total decreases by 10% | no |

---

## 9. Open Questions (Pre-Approval Gate)
> **Strict Gate Blocker:** Every question must reach `decided` or `out-of-scope` before Gate 1 approval. If any item remains `unresolved`, implementation **must not begin**.

1. Status: unresolved | decided | out-of-scope
   - **Question:** <Specific ambiguity regarding requirements, integration, or edge cases>
   - **Decision:** <Final agreed decision, rationale, or explanation why it is deferred>

*(State `None` if there are no open questions.)*
````

**External spec tools:** This workflow is format-agnostic. If the project
already uses OpenSpec or another specification system, keep that system's
artifact format and storage conventions instead of creating a duplicate
`SPEC.md`. This skill owns the clarification, content, and approval gates; the
external tool owns how the approved spec is represented.

**Reframe instructions as success criteria.** When receiving vague requirements, translate them into concrete conditions:

```
REQUIREMENT: "Make the dashboard faster"

REFRAMED SUCCESS CRITERIA:
- Dashboard LCP < 2.5s on 4G connection
- Initial data load completes in < 500ms
- No layout shift during load (CLS < 0.1)
→ Are these the right targets?
```

This lets you loop, retry, and problem-solve toward a clear goal rather than guessing what "faster" means.

### Phase 2: Plan

With the validated spec, generate a technical implementation plan:

1. Identify the major components and their dependencies
2. Determine the implementation order (what must be built first)
3. Note risks and mitigation strategies
4. Identify what can be built in parallel vs. what must be sequential
5. Define verification checkpoints between phases

> Follow `planning-and-task-breakdown` for the dependency-graph mapping and vertical-slicing mechanics behind these steps; it is the canonical source. The bullets above are a lightweight summary; if they ever diverge, `planning-and-task-breakdown` takes precedence.
>
> **Output convention:** Save the plan to `tasks/plan.md` and record the task list in the task list target defined by `planning-and-task-breakdown` (default `tasks/todo.md`; projects may designate an external tracker instead). Create `tasks/` if it does not exist. Downstream commands (`/build`, etc.) expect these defaults.

The plan should be reviewable: the human should be able to read it and say "yes, that's the right approach" or "no, change X."

### Phase 3: Tasks

Break the plan into discrete, implementable tasks:

- Each task should be completable in a single focused session
- Each task has explicit acceptance criteria
- Each task includes a verification step (test, build, manual check)
- Tasks are ordered by dependency, not by perceived importance
- No task should require changing more than ~5 files

> Follow `planning-and-task-breakdown` for the full task-sizing and dependency-ordering mechanics; it is the canonical source. The template below is a lightweight inline form; if they ever diverge, `planning-and-task-breakdown` takes precedence.

**Task template:**
```markdown
- [ ] Task: [Description]
  - Acceptance: [What must be true when done]
  - Verify: [How to confirm — test command, build, manual check]
  - Files: [Which files will be touched]
```

### Phase 4: Implement

Execute tasks one at a time following `skills/incremental-implementation/SKILL.md` (`incremental-implementation`) and `skills/test-driven-development/SKILL.md` (`test-driven-development`). Use `skills/context-engineering/SKILL.md` (`context-engineering`) to load the right spec sections and source files at each step rather than flooding the agent with the entire spec.

## Keeping the Spec Alive

The spec is a living document, not a one-time artifact:

- **Update when decisions change** — If you discover the data model needs to change, update the spec first, then implement.
- **Update when scope changes** — Features added or cut should be reflected in the spec.
- **Commit the spec** — The spec belongs in version control alongside the code.
- **Reference the spec in PRs** — Link back to the spec section that each PR implements.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This is simple, I don't need a spec" | Simple tasks don't need *long* specs, but they still need acceptance criteria. A two-line spec is fine. |
| "I'll write the spec after I code it" | That's documentation, not specification. The spec's value is in forcing clarity *before* code. |
| "The spec will slow us down" | A 15-minute spec prevents hours of rework. Waterfall in 15 minutes beats debugging in 15 hours. |
| "Requirements will change anyway" | That's why the spec is a living document. An outdated spec is still better than no spec. |
| "The user knows what they want" | Even clear requests have implicit assumptions. The spec surfaces those assumptions. |
| "It's one big feature; splitting it is overhead" | If acceptance criteria cluster into independently testable groups, a monolithic spec forces every downstream task to reason over the whole contract. A ten-line capability map is the cheap alternative. |
| "I'll decompose during planning" | Planning slices tasks within a spec. By then the oversized artifact already exists — module boundaries and dependency direction must be decided before the spec is written, not after. |

## Red Flags

- Starting to write code without any written requirements
- Asking "should I just start building?" before clarifying what "done" means
- Implementing features not mentioned in any spec or task list
- Making architectural decisions without documenting them
- Skipping the spec because "it's obvious what to build"
- One spec whose requirements span several independently testable capabilities
- Module boundaries or build order decided implicitly during implementation because no capability map was approved up front

## Verification

Before proceeding to implementation, confirm:

- [ ] The spec covers all six core areas (Objective, Commands, Structure & Touchpoints, Style & Contracts, Testing & Truth Table, Boundaries)
- [ ] Codebase touchpoints are inspected and locked (`touchpoints_locked: true`)
- [ ] The human has reviewed and approved the spec
- [ ] Behavioral truth table covers edge cases and validation scenarios are deterministic
- [ ] Boundaries (Always/Ask First/Never) and invariants (including concurrency & secrets safety) are defined
- [ ] All open questions are marked `decided` or `out-of-scope` (none `unresolved`)
- [ ] The spec is saved to a file in the repository (default: `SPEC.md` or `docs/SPEC.md`)
- [ ] If the request bundles several independently testable capabilities, a capability map (module ids, dependency direction, build order) was approved before any module spec was written
- [ ] Every module spec traces to a module id in the approved map
