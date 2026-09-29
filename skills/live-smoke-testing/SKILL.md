---
name: live-smoke-testing
description: Use when conducting live smoke tests, pre-merge verification, or real end-to-end service checks against local runtime dependencies such as Docker, databases, caches, or message queues. Drives real running service daemons using type-safe disposable native test harnesses, dynamic state injection, and guaranteed teardown without polluting databases or leaking alerts.
---

# Live Smoke Testing

Live smoke testing verifies that a service or pipeline works end-to-end against real runtime dependencies (databases, key-value caches, message brokers, streaming transports) before merging or deployment.

Instead of relying solely on in-memory unit test mocks or resorting to fragile shell scripts with broken escaping, this skill enforces a **type-safe, black-box driver pattern**: snapshotting working-tree state, running the real compiled service daemon in the background, driving inputs and observing outputs via a disposable native harness, and enforcing guaranteed state rollback (database, cache, and repository code via temporary commit and soft reset) upon completion.

## Overview

Unit tests prove that individual functions behave according to assumptions; live smoke tests prove that the **real compiled binary** connects to real dependencies, handles real scheduler loops, honors cache time-to-live thresholds, and responds to real network events without panicking or leaking state.

### 1. The Code-First Rule (No Fragile Shell Escaping)

Multi-line shell scripts attempting to escape JSON inside `docker exec`, calculate millisecond timestamps with platform-divergent `date` flags, or grep through raw logs introduce high cognitive overhead and subtle formatting bugs.

This skill dictates:
1. Run the real service daemon as a supervised background process (e.g. `npm start`, `go run main.go`, `make run`, or `./run.sh`).
2. Write a single, disposable driver script in the project's native language (`scripts/tmp_smoke_driver.ts`, `.go`, or `.py`) importing native project domain types and models.
3. Register all database and cache restoration inside a guaranteed cleanup block (`defer` in Go, `try ... finally` in TypeScript/Python).
4. Execute the driver, observe behavior, and clean up the temporary script upon completion.

### 2. The Working-Tree State Machine (Temporary Commit & Soft Reset)

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

- **Ephemeral Worktree Alternative (Zero-Contamination):**
  If the working tree has sensitive partial staging (`git add -p`), uncommitted submodules, or high risk of untracked file generation, prefer an ephemeral worktree instead of mutating the primary checkout:
  ```bash
  WORKTREE_DIR=$(mktemp -d /tmp/smoke-worktree-XXXXXX)
  git worktree add --detach "$WORKTREE_DIR" HEAD
  # Execute smoke test and driver inside $WORKTREE_DIR...
  git worktree remove --force "$WORKTREE_DIR"
  ```

### 3. Universal Dependency Taxonomy & Isolation Boundaries

Regardless of specific technology (PostgreSQL or MySQL; Redis, Dragonfly, or KeyDB; NATS, NSQ, RabbitMQ, or Kafka), enforce these isolation boundaries:

- **State Stores (Databases):** Record created/modified IDs or use transactions so changes can be precisely reverted during teardown.
- **Key-Value & Cache Stores:** Always use an isolated namespace prefix (e.g. `test:smoke:<uuid>:*`) for all injected keys to avoid polluting real keys.
- **Message Brokers & Event Buses:** Use ephemeral channels/topics or distinct correlation IDs (`x-smoke-test-id: <uuid>`) to isolate test traffic from standard consumers.
- **Outbound Sink Neutralization:** Always silence real webhook targets (Slack, Discord, Telegram, PagerDuty, Sentry) in the test configuration so local runs never leak alert spam.

---

## When to Use

- When verifying a feature branch against real local databases, caches, or message brokers before opening or merging a PR/MR.
- When validating state machine transitions, event consumer pipelines, cooldown intervals, or cache expiration in running daemons.
- When testing time-windowed behaviors (e.g. TTL expiration, cron ticker intervals, quiet-hours bypass).
- When validating cross-service message propagation (e.g. Producer $\rightarrow$ Message Broker $\rightarrow$ Consumer Worker).
- When conducting pre-merge verification on data-integrity or event-driven systems where mocks hide concurrency or serialization flaws.

## When NOT to Use

- Do not use for testing pure, stateless business logic that can be deterministically tested with table-driven unit tests. Use `test-driven-development` instead.
- Do not use for investigating unknown system crashes or root-cause reproduction. Use `debugging-and-error-recovery` instead.
- Do not use directly against shared production or staging environments. Live smoke tests require an isolated local sandbox or dedicated testcontainer.

---

## Process

```
[Phase 1: Discovery] ──> [Phase 2: Code Snapshot] ──> [Phase 3: Launch Daemon]
                                                                  │
[Phase 5: Rollback & Teardown] <── [Phase 4: Run Native Driver] <──┘
```

### Phase 1: Dependency Discovery & Archetype Identification

1. **Inspect Local Dependencies:**
   Detect running containers or local daemons without assuming fixed ports:
   ```bash
   docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
   grep -E "PORT|HOST|URL|ADDR" .env .env.local 2>/dev/null
   ```
2. **Determine Service Archetype:**
   - **Periodic Scheduler / Worker:** Daemons running periodic ticker loops or polling queues.
   - **Event Pipeline / Consumer:** Services consuming from a broker, transforming data, and publishing results.
   - **Ingress / Gateway Transport:** Real-time WebSocket, SSE, or gRPC stream connectors.
   - **Data Store Synchronizer:** Background workers batching memory/cache changes into durable databases.

### Phase 2: Working-Tree Snapshot & Sink Isolation

Before modifying any repository file or launching any background daemon:

1. **Snapshot Working-Tree Code State (Temporary Commit):**
   - Run `git status --porcelain`.
   - **If there are uncommitted changes:** Create a temporary snapshot commit:
     ```bash
     git add -A && git commit -m "wip: pre-smoke-test snapshot"
     ```
     This safely anchors all in-flight modifications in git history.
   - **If the working tree is clean:** Proceed directly.

2. **Isolate Outbound Webhooks & Sinks:**
   Create a local `.env.test` or test configuration override:
   - Clear all notification webhook URLs (`DISCORD_WEBHOOK=""`, `SLACK_WEBHOOK=""`, `ALERT_ENDPOINT=""`).
   - Shorten ticker intervals (e.g. `2s` or `5s` instead of 15m) for fast deterministic feedback.
   - Disable non-essential external third-party sync workers.

### Phase 3: Launch Service Daemon (Black-Box)

Start the real service daemon as a supervised background process using the project's native runner:
```bash
# Launch in a distinct process group to prevent orphaned background processes
set -m
npm run dev & # or go run cmd/server/main.go or python -m service.main
DAEMON_PID=$!
set +m
echo $DAEMON_PID > /tmp/smoke_daemon.pid
```
**Readiness Gate:** Poll for port binding or wait for the startup ready banner in the logs (timeout max 15 seconds) before triggering tests.

### Phase 4: Generate and Execute Disposable Native Driver

Create a single disposable driver script in `scripts/tmp_smoke_driver.<ext>` using the project's native language.

#### Example A: Go Driver Pattern
```go
package main

import (
	"context"
	"fmt"
	"time"
	// Import native project entity types and client packages
)

func main() {
	ctx := context.Background()

	// 1. Snapshot initial state if modifying existing records
	origState := captureInitialState(ctx)

	// 2. Register guaranteed cleanup
	defer func() {
		fmt.Println("🧹 Restoring state and deleting test keys...")
		restoreState(ctx, origState)
		purgePrefix(ctx, "test:smoke:*")
		fmt.Println("✅ Environment restored.")
	}()

	// 3. Arrange: Seed namespaced test data using native structs
	// testItem := entity.Item{ID: "smoke-test-1", Status: "PENDING", CreatedAt: time.Now().Add(-10 * time.Minute)}
	// store.Insert(ctx, testItem)

	// 4. Act & Assert: Observe daemon state transition or broker message
	// assertCondition(...)
	fmt.Println("🎉 SMOKE TEST PASSED")
}
```

#### Example B: TypeScript Driver Pattern
```typescript
import { db } from "../src/db";
import { broker } from "../src/broker";

async function main() {
  const origRecord = await db.findItem("smoke-test-1");

  try {
    // Arrange: Publish test event with unique correlation ID
    await broker.publish("events:topic", {
      id: "smoke-test-1",
      action: "PROCESS_EXPIRATION",
      timestamp: Date.now() - 600_000,
    });

    // Act & Assert: Poll database or consumer state for expected transition
    await pollUntilAssert(() => db.findItem("smoke-test-1"), (item) => item.status === "EXPIRED");
    console.log("🎉 SMOKE TEST PASSED");
  } finally {
    console.log("🧹 Restoring initial state...");
    if (origRecord) await db.updateItem("smoke-test-1", origRecord);
    else await db.deleteItem("smoke-test-1");
  }
}

main().catch((err) => {
  console.error("❌ Smoke test failed:", err);
  process.exit(1);
});
```

Run the driver:
```bash
# Go
go run scripts/tmp_smoke_driver.go

# TypeScript / Node
npx tsx scripts/tmp_smoke_driver.ts

# Python
python scripts/tmp_smoke_driver.py
```

### Phase 5: Code Rollback, Teardown & Reporting

1. **Driver Teardown:** The driver's `defer` or `try ... finally` block automatically cleans up seeded keys and restores modified records.
2. **Stop Daemon Process:** Terminate the background daemon cleanly (`kill $(cat /tmp/smoke_daemon.pid)`).
3. **Working-Tree Code Rollback (Soft Reset Pattern):**
   - **If a temporary snapshot commit was created:**
     ```bash
     git reset --hard HEAD
     git clean -fd
     git reset --soft HEAD~1
     ```
   - **If the tree was originally clean:**
     ```bash
     git checkout -- .
     git clean -fd
     ```
   - Verify `git status` matches pre-test state exactly.
4. **Remove Temporary Standalone Files:** Delete `scripts/tmp_smoke_driver.*`, `.env.test`, and `/tmp/smoke_daemon.pid`.
5. **Verify Regressions:** Run standard unit tests (`npm test`, `go test ./...`, `pytest`) to verify the tree is 100% healthy.
6. **Format Evidence:** Compile observed timestamps, log lines, and state assertions into a structured markdown table for review evidence.

---

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "Unit tests with mocks are enough; we don't need a live test." | Mocks do not test real network timeouts, database query parsing, broker reconnect loops, or cache TTL boundaries. Live smoke tests prove real runtime behavior. |
| "I can just run quick bash commands to curl and update the database." | Shell escaping breaks on JSON payloads, timestamp math varies across operating systems, and aborted bash commands leave dirty state behind. Native language scripts provide type safety and guaranteed cleanup blocks. |
| "Writing a temporary script takes too long." | A typed script in the project's native language catches schema errors at compile/run time, reuses existing models, and eliminates manual CLI debugging. |
| "We are in a rush; skip the cleanup step." | Uncleaned state pollutes local databases, causes flaky tests for team members, and invalidates subsequent test runs. Teardown must be automated via `defer` or `finally`. |
| "Leaving temporary log calls or runner script edits in git is harmless." | Test-only scaffolding pollutes git diffs, fails PR code reviews, and risks leaking debug logs to production. Always roll back code changes via soft reset after testing. |

---

## Red Flags

- The agent crafts multi-line `docker exec <container> -e "..."` commands with nested escaping instead of writing a typed driver.
- The agent runs live smoke tests without clearing outbound alert webhooks (`DISCORD_WEBHOOK`, `SLACK_WEBHOOK`).
- The agent tests a service by importing internal private package functions into a test file rather than running the compiled daemon as a black-box.
- The agent exits after a failed assertion without restoring modified database records or clearing test keys.
- The agent leaves temporary code edits, logger calls, or modified configurations visible in `git status` after completing the test.
- The agent claims "smoke test passed" without providing concrete timestamps, log lines, or assertion evidence.

---

## Verification

Before declaring the smoke test complete, confirm:

- [ ] Working tree was snapshotted before any file edits (via temporary commit if uncommitted changes existed).
- [ ] Outbound alert sinks were verified neutralized or empty.
- [ ] Real service daemon was running as a supervised black-box process during testing.
- [ ] Test state was generated using native models and project types with isolated namespacing.
- [ ] Assertions verified positive cases, edge cases, and deduplication/cooldown behavior.
- [ ] Pre-test database and cache state matches post-test state (guaranteed via `defer` / `finally`).
- [ ] Temporary driver script and `.env.test` overrides were deleted.
- [ ] All temporary source code and runner modifications were rolled back (via `git reset --soft HEAD~1`).
- [ ] Standard unit test suite passes with zero regressions on the restored working tree.
- [ ] Structured evidence report with log snippets and timestamps is generated for the PR/MR.
