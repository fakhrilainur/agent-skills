---
name: live-smoke-testing
description: Use when conducting live smoke tests, pre-merge verification, or real end-to-end service checks against local runtime dependencies such as Docker, databases, Redis, or message queues. Drives real running service daemons using type-safe disposable Go test harnesses, dynamic cache and queue state injection, and guaranteed defer teardown without polluting databases or leaking alerts.
---

# Live Smoke Testing

Live smoke testing verifies that a service or pipeline works end-to-end against real runtime dependencies (MySQL, Redis, NATS JetStream, WebSockets) before merging or deployment.

Instead of relying solely on in-memory unit test mocks or resorting to fragile shell scripts with broken escaping, this skill enforces a **type-safe, black-box driver pattern**: snapshotting working-tree state, running the real compiled service daemon in the background, driving inputs and observing outputs via a disposable Go harness, and enforcing guaranteed state rollback (database, cache, and repository code via temporary commit and soft reset) upon completion.

## Overview

Unit tests prove that individual functions behave according to assumptions; live smoke tests prove that the **real compiled binary** connects to real dependencies, handles real scheduler loops, honors cache time-to-live thresholds, and responds to real network events without panicking or leaking state.

### The Code-First Rule (No Fragile Shell Escaping)

Multi-line shell scripts attempting to escape JSON inside `docker exec`, calculate millisecond timestamps with `date +%s`, or grep through raw logs introduce high cognitive overhead and subtle formatting bugs.

This skill dictates:
1. Run the real service daemon as a supervised background process (`run.sh run` or `make <service>`).
2. Write a single, disposable Go driver script (`scripts/tmp_smoke_driver.go`) importing native project structs (`frame.Price`, `entity.Pair`).
3. Register all database and cache restoration inside a guaranteed Go `defer` block.
4. Execute via `go run scripts/tmp_smoke_driver.go` and clean up the temporary script upon completion.

### The Working-Tree State Machine (Temporary Commit & Soft Reset)

Live smoke testing often requires temporary source adjustments for runtime observability (e.g. adding structured log statements before external RPC calls, or updating run scripts to load test configurations).

The agent must treat the repository code as a state machine:
- **Pre-Test State Snapshot:** Before modifying any file for testing, if uncommitted changes exist, create a temporary snapshot commit:
  ```bash
  git add -A && git commit -m "wip: pre-smoke-test snapshot"
  ```
- **Post-Test State Rollback:** When testing finishes, discard all test-only edits and soft-reset the snapshot commit:
  ```bash
  git reset --hard HEAD && git clean -fd
  git reset --soft HEAD~1
  ```
  This restores the developer's exact uncommitted changes without merge conflicts or dirty git states.

---

## When to Use

- When verifying a feature branch against real local databases, Redis, or NATS before opening or merging a PR/MR.
- When validating state machine transitions, alert triggers, cooldown intervals, or cache expiration in running daemons.
- When testing schedule-aware behaviors (e.g., market hours open/closed bypass).
- When validating cross-service message propagation (e.g., Fetcher $\rightarrow$ Aggregator $\rightarrow$ Broadcaster).
- When conducting pre-merge smoke tests on financial or price-sensitive systems where mocks hide concurrency or serialization flaws.

## When NOT to Use

- Do not use for testing pure, stateless business logic that can be deterministically tested with table-driven unit tests. Use `test-driven-development` instead.
- Do not use for investigating unknown system crashes or root-cause reproduction. Use `debugging-and-error-recovery` instead.
- Do not use directly against shared production or staging environments. Live smoke tests require an isolated local sandbox or dedicated testcontainer.

---

## Process

```
[Phase 1: Preflight] ──> [Phase 2: Temporary Commit Snapshot] ──> [Phase 3: Launch Daemon]
                                                                            │
[Phase 5: Soft Reset Rollback] <── [Phase 4: Run Go Driver] <───────────────┘
```

### Phase 1: Preflight & Archetype Discovery
1. Check running local containers (`docker ps`) to verify database, Redis, and message queue availability.
2. Determine project archetype:
   - **Cron Scheduler:** Services running periodic ticker loops (e.g., `services/monitoring`).
   - **Event Pipeline:** Services consuming and publishing events (e.g., `aggregator`, `broadcaster`).
   - **Stream Provider:** Real-time WebSocket or feed connectors (e.g., `fetcher`, `oracle-provider`).
   - **Engine Library:** Core calculation and serialization libraries (e.g., `price-oracle`, `go-sdk`).

### Phase 2: Working-Tree Snapshot & State Isolation
Before modifying any repository file or launching any background daemon:
1. **Snapshot Working-Tree Code State (Temporary Commit Pattern):**
   - Run `git status --porcelain`.
   - **If there are uncommitted working-tree changes:** Create a temporary snapshot commit:
     ```bash
     git add -A && git commit -m "wip: pre-smoke-test snapshot"
     ```
     This safely anchors all in-flight modifications in git history and avoids stash-pop conflicts.
   - **If the working tree is already clean:** Note that no snapshot commit was required.
2. **Isolate Outbound Webhooks & Sinks:**
   - Create a local `.env.test` override:
     - Clear all alert webhooks: `DISCORD_WEBHOOK=""`, `TELEGRAM_TOKEN=""`, `SENTRY_DSN=""`.
     - Set high-frequency loop intervals (e.g., `5s` instead of 15m) for instant test feedback.
     - Disable unrelated background workers or scrapers.

### Phase 3: Launch Service Daemon
Start the real service daemon as a supervised background process using the project's native runner:
```bash
./services/<service>/run.sh run # or make <service>
```
Verify the daemon boots and logs its health server / ready banner before proceeding.

### Phase 4: Generate and Execute Type-Safe Go Driver
Generate a disposable Go driver script (`scripts/tmp_smoke_driver.go`). The driver must follow this exact anatomy:

```go
package main

import (
	"context"
	"fmt"
	"time"
	// Import native project models and storage packages
)

func main() {
	ctx := context.Background()

	// 1. Snapshot original database / cache state
	// snapshot := queryOriginalState(ctx)

	// 2. Register mandatory cleanup
	defer func() {
		fmt.Println("🧹 Restoring database rows and deleting test keys...")
		// restoreState(ctx, snapshot)
		// flushTestKeys(ctx)
		fmt.Println("✅ Environment 100% restored.")
	}()

	// 3. Arrange: Seed test data using native structs
	// testPrice := frame.Price{ Pair: "TEST:USD/JPY", LastUpdatedAtMs: time.Now().Add(-25 * time.Second).UnixMilli() }
	// cache.Set(ctx, "master:TEST:USD/JPY", testPrice)

	// 4. Act & Assert: Poll daemon output or observe state transitions
	// assertCondition(...)

	fmt.Println("🎉 SMOKE TEST PASSED")
}
```

Run the driver:
```bash
go run scripts/tmp_smoke_driver.go
```

### Phase 5: Mandatory Code Rollback, Teardown & Reporting
1. The driver's `defer` block restores modified database rows to their pre-test values and deletes seeded cache keys.
2. Stop the background service daemon.
3. **Explicit Working-Tree Code Rollback (Soft Reset Pattern):**
   - **If a temporary snapshot commit was created in Phase 2:**
     ```bash
     # Discard all smoke-test modifications and untracked files
     git reset --hard HEAD
     git clean -fd

     # Soft-reset the temporary commit to restore original uncommitted changes
     git reset --soft HEAD~1
     ```
   - **If the working tree was originally clean:**
     ```bash
     git checkout -- .
     git clean -fd
     ```
   - Verify `git status` matches the pre-test snapshot exactly.
4. Remove temporary standalone files (`scripts/tmp_smoke_driver.go`, `.env.test`).
5. Re-run `make test` to verify unit test parity remains 100% clean on the restored working tree.
6. Format observed evidence into a structured Markdown matrix and attach to the MR/PR.

---

## Common Rationalizations

| Rationalization | Reality |
| :--- | :--- |
| "Unit tests with mocks are enough; we don't need a live test." | Mocks do not test real network timeouts, SQL JSON projections, cache TTLs, or process schedulers. Live smoke tests prove real runtime behavior. |
| "I can just run quick bash commands to curl and update Redis." | Shell escaping breaks on JSON, date math varies across platforms (`date +%s%N`), and failed bash commands leave dirty state. Go scripts provide type safety and guaranteed `defer` cleanup. |
| "Writing a temporary Go script is slower than manual CLI." | A typed Go script catches compilation errors immediately, eliminates quoting bugs, and runs in seconds without trial-and-error shell debugging. |
| "We are in a rush; skip the cleanup step." | Uncleaned state pollutes local databases, causes flaky tests for colleagues, and invalidates subsequent smoke runs. Teardown must be automatic via `defer`. |
| "Leaving temporary log calls or runner script edits in git is harmless." | Test-only scaffolding pollutes the feature branch diff, fails code reviews, and can leak debug logs into production. Always roll back code changes via soft reset after smoke tests. |

---

## Red Flags

- The agent starts crafting multi-line `docker exec mysql -e "UPDATE ... '{\"json\":...}'"` commands with nested quotes.
- The agent runs live smoke tests without clearing `DISCORD_WEBHOOK` or external alert sinks.
- The agent tests a service by importing its internal private functions into a test file rather than running the compiled daemon.
- The agent exits after a test failure without restoring database rows or deleting test keys.
- The agent leaves temporary code edits, logger calls, or modified runner scripts in `git status` after completing the smoke test.
- The agent reports "smoke test passed" without providing exact timestamps, log snippets, or assertion deltas.

---

## Verification

Before declaring the smoke test complete, verify all items:

- [ ] Working tree state was snapshotted before any file edits (via temporary commit if uncommitted changes existed).
- [ ] Outbound alert sinks were verified empty (`DISCORD_WEBHOOK=""`).
- [ ] Real service daemon was running during test execution.
- [ ] Test state was generated using native Go structs and project models.
- [ ] Assertions verified positive, negative, and edge-case behaviors (e.g. cooldown, schedule bypass).
- [ ] Pre-test database snapshot matches post-test database state 100%.
- [ ] All temporary Redis keys and message queues were flushed.
- [ ] All temporary source code and runner script modifications were rolled back (via `git reset --soft HEAD~1` if temporary commit was used).
- [ ] Temporary driver script and `.env.test` files were deleted.
- [ ] Standard unit test suite (`make test`) passes with zero regression.
- [ ] Structured evidence report with log snippets and timestamps is generated.

---

## Anti-patterns

- **Don't test against production or shared staging** — always isolate dependencies to local Docker or dedicated test schemas.
- **Don't leave cleanup to manual terminal commands** — always embed restoration logic inside a Go `defer` statement so it executes even if assertions panic.
- **Don't leave test scaffolding in source files** — always use the temporary commit and soft reset pattern to guarantee zero lingering test edits.
- **Don't hardcode absolute timestamps** — use relative offsets (`time.Now().Add(-20 * time.Second)`) to prevent flaky time boundaries.
- **Don't leak alert spam** — always verify that webhook sinks are silenced in the test configuration.
