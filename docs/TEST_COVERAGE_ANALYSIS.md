# Test Coverage Analysis

**Date:** 2026-02-22
**Overall src/ coverage:** 39% statements, 33% branches, 33% functions, 40% lines

## Current State

### Coverage by File

| File | Stmts | Branch | Funcs | Lines | Test File |
|------|-------|--------|-------|-------|-----------|
| config.ts | 100% | 90% | 100% | 100% | (tested via formatting.test.ts) |
| container-runtime.ts | 100% | 100% | 100% | 100% | container-runtime.test.ts |
| whatsapp.ts | 91% | 86% | 65% | 90% | whatsapp.test.ts |
| container-runner.ts | 66% | 46% | 73% | 67% | container-runner.test.ts |
| group-queue.ts | 68% | 49% | 73% | 71% | group-queue.test.ts |
| router.ts | 61% | 50% | 56% | 69% | formatting.test.ts |
| ipc.ts | 53% | 70% | 17% | 53% | ipc-auth.test.ts |
| db.ts | 47% | 44% | 47% | 46% | db.test.ts |
| env.ts | 50% | 13% | 100% | 56% | -- |
| index.ts | 6% | 5% | 10% | 7% | routing.test.ts (minimal) |
| dashboard.ts | 1% | 0% | 0% | 1% | -- |
| mount-security.ts | 4% | 3% | 0% | 4% | -- |
| task-scheduler.ts | 1% | 0% | 0% | 1% | -- |
| team-manager.ts | 0% | 0% | 0% | 0% | -- |
| logger.ts | 50% | 100% | 0% | 50% | -- |

### Test File Summary (147 tests, all passing)

| Test File | Tests | What It Covers |
|-----------|-------|----------------|
| db.test.ts | 18 | Message CRUD, chat metadata, task CRUD |
| container-runner.test.ts | 3 | Timeout behavior, output marker parsing |
| container-runtime.test.ts | 8 | Runtime detection, orphan cleanup |
| formatting.test.ts | 32 | XML escaping, message formatting, trigger pattern, outbound formatting, trigger gating |
| group-queue.test.ts | 7 | Concurrency limits, task priority, retry backoff, shutdown |
| ipc-auth.test.ts | 32 | IPC authorization for all task types, schedule validation, group registration |
| whatsapp.test.ts | 39 | Connection lifecycle, message handling, LID translation, outgoing queue, group sync |
| routing.test.ts | 8 | JID ownership patterns, getAvailableGroups filtering |

## Recommended Improvements

### Priority 1: Completely Untested Modules (High Impact)

#### 1. `mount-security.ts` — 4% coverage

This is a **security-critical** module that validates container mount paths against an allowlist. It has zero function coverage despite containing complex logic for:

- Path expansion and symlink resolution
- Blocked pattern matching (prevents mounting `.ssh`, `.aws`, credentials, etc.)
- Allowed root validation
- Read-only enforcement for non-main groups
- Container path traversal prevention

**Recommended tests:**
- `loadMountAllowlist()`: valid file, missing file, malformed JSON, invalid structure
- `validateMount()`: path under allowed root, path outside allowed roots, blocked pattern match, symlink that escapes allowed root, container path traversal (`..`), absolute container path, empty container path
- `validateMount()` readonly enforcement: non-main group forced read-only, main group read-write allowed, root that disallows read-write
- `validateAdditionalMounts()`: mix of valid and rejected mounts, empty array
- `matchesBlockedPattern()`: exact component match, substring match, no match
- `expandPath()`: tilde expansion, absolute path, relative path

#### 2. `task-scheduler.ts` — 1% coverage

The scheduler orchestrates running tasks on a timer and manages next-run computation. Currently zero function coverage.

**Recommended tests:**
- `runTask()`: successful task execution, group not found error, container agent error, exception during run
- `runTask()` next-run computation: cron schedule computes next run, interval schedule computes next run, `once` tasks get no next run
- `runTask()` context mode: `group` mode passes existing session ID, `isolated` mode passes no session
- `startSchedulerLoop()`: deduplicates (second call is no-op), skips paused/cancelled tasks, enqueues due tasks via queue

#### 3. `team-manager.ts` — 0% coverage

The TeamManager handles team-based agent collaboration. It has the most code of any untested file (450 lines).

**Recommended tests:**
- Constructor: loads team configs from disk, handles missing directory, handles malformed config
- `startAllTeams()` / `stopAllTeams()`: starts polling, cleans up on stop
- `checkMemberInbox()`: processes unread messages, marks as read before processing (prevents duplicates), handles empty inbox, handles parse errors
- `processMessage()`: constructs prompt correctly, saves output, sends response to lead
- `sendResponseToLead()`: appends message to lead's inbox, creates inbox if missing
- `getAllTeamData()`: aggregates member stats, sorts recent messages, handles missing inbox files
- `getLeadMessages()` / `markLeadMessagesRead()`: filters unread, marks all as read

#### 4. `dashboard.ts` — 1% coverage

The HTTP dashboard renders status pages and exposes JSON APIs.

**Recommended tests:**
- `formatUptime()`: days+hours+minutes, hours+minutes, just minutes
- `esc()`: HTML entity escaping (similar to `escapeXml`, but worth verifying independently)
- `gatherData()`: returns correct structure with groups, tasks, team data
- HTTP routing: `/` returns HTML, `/api/status` returns JSON, `/api/teams` returns JSON, `/teams` returns HTML, unknown path returns 404, non-GET returns 405

### Priority 2: Modules with Significant Gaps

#### 5. `db.ts` — 47% coverage

The existing tests cover messages, chats, and task CRUD well, but several functions are untested:

**Untested functions:**
- `getDueTasks()`: returns tasks where `next_run <= now` and `status = 'active'`
- `updateTaskAfterRun()`: sets next_run, last_run, last_result; marks `once` tasks as `completed`
- `logTaskRun()`: inserts run log entry
- `getRouterState()` / `setRouterState()`: key-value storage
- `getSession()` / `setSession()` / `getAllSessions()`: session persistence
- `getRegisteredGroup()` / `setRegisteredGroup()` / `getAllRegisteredGroups()`: roundtrip serialization including `containerConfig` JSON and `requiresTrigger` boolean mapping
- `storeMessageDirect()`: direct message storage for non-WhatsApp channels
- `updateChatName()`: upserts chat name without changing timestamp
- `migrateJsonState()`: JSON-to-SQLite migration (hard to test, lower priority)

#### 6. `index.ts` — 6% coverage

The main orchestrator has very low coverage. While `main()` is inherently hard to unit test (it wires everything together), several extractable functions deserve testing:

**Recommended tests:**
- `processGroupMessages()`: this is the core message processing pipeline — trigger detection, cursor management, error rollback, typing indicators
- `runAgent()`: session tracking, error handling, task snapshot writes
- `loadState()` / `saveState()`: state persistence roundtrip, corrupted `last_agent_timestamp` recovery
- `recoverPendingMessages()`: enqueues groups with pending messages after restart
- `startMessageLoop()`: deduplication (second call is no-op)

#### 7. `router.ts` — 61% coverage

Two functions are untested:

- `routeOutbound()`: finds the right channel and sends, throws when no channel matches
- `findChannel()`: returns matching channel or undefined

#### 8. `container-runner.ts` — 66% coverage

The existing tests cover timeout behavior well, but miss:

- `writeTasksSnapshot()`: file writing, main vs non-main filtering
- `writeGroupsSnapshot()`: file writing, registration status mapping
- Output size limiting (CONTAINER_MAX_OUTPUT_SIZE)
- Stderr capture and logging
- The `buildContainerArgs()` internal function: environment variables, mount construction, container naming

#### 9. `group-queue.ts` — 68% coverage

Missing coverage for:

- `registerProcess()` / `sendMessage()` / `closeStdin()`: the IPC bridge for piping messages to running containers
- Edge case: task deduplication (same task ID enqueued twice)
- `shutdown()` timeout: force-kills containers that don't stop in time

### Priority 3: Low-Priority Gaps

#### 10. `env.ts` — 50% coverage

The `.env` file parser has low branch coverage. Edge cases: missing file, malformed lines, comments, empty values, quoted values.

#### 11. `logger.ts` — 50% coverage

Trivial module. Testing the pino configuration adds little value.

## Testing Strategy Notes

- **Dependency injection is already well-used**: `IpcDeps`, `SchedulerDependencies`, `WhatsAppChannelOpts`, and `DashboardDeps` interfaces make most modules testable without excessive mocking. Prioritize using these.
- **`_initTestDatabase()` is available**: For any new db-dependent tests, the in-memory database helper is already in place.
- **The mock patterns in existing tests are good references**: `container-runner.test.ts` demonstrates fake process/stream mocking, `whatsapp.test.ts` demonstrates fake socket/event mocking.
- **Security tests are the highest-value addition**: `mount-security.ts` protects against container escape via mount manipulation. It's the most important untested module from a risk perspective.
