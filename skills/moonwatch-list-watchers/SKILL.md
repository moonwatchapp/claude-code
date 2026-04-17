---
name: moonwatch-list-watchers
description: "List and review Moonwatch watchers for the current project's log file."
user_invocable: true
---

# Moonwatch — AI Workflow Guide

This project uses Moonwatch to give you persistent runtime insight. You can place log statements freely — they are sent to a central server and you can query them via MCP at any time. This means you don't need to ask the user to copy-paste logs. You place them, the user reproduces the issue, and you query the results yourself.

## Commands

Check the user's request (or the arguments passed to this skill) to determine which command to execute:

| Command | Action |
|---------|--------|
| `/moonwatch-setup` | Interactive setup wizard — [Setup Wizard](#setup-wizard) |
| `/moonwatch-analyze-logs` | Ad-hoc log analysis — [Analyze Logs](#analyze-logs) |
| `/moonwatch-list-watchers` | List and review watchers — [List Watchers](#list-watchers) |
| No specific command | Loaded contextually — use the relevant workflow section below |

---

## Log File Selection

**Gate rule:** Before ANY Moonwatch MCP tool call (except during setup), you must have log files resolved. Do this once at the start of a conversation, then reuse for all subsequent tool calls.

Log file IDs are stored as environment variables in Claude Code settings (`env` key), which are available to you automatically.

1. Check for `MOONWATCH_LOG_FILE_ID` env var (production file — set in `.claude/settings.json`)
2. Check for `MOONWATCH_LOG_FILE_ID_DEV` env var (dev file — set in `.claude/settings.local.json`)
3. If at least one is set → proceed. Use the production file by default; use the dev file only when the user explicitly mentions local/dev.
4. If neither is set → tell the user: "Moonwatch is not configured for this project. Run `/moonwatch setup` first." and stop.

**Usage rules:**
- Always pass the resolved `logFileId` to `logs_query`, `watchers_create`, and `watchers_list`
- Default to the production file unless the user explicitly mentions local/dev
- For `watchers_list`, pass `logFileId` to filter to the relevant file

---

## The Investigation Workflow

1. **Create a watcher** via MCP (`watchers_create`) describing what you're investigating — you get back a watcher ID (e.g. `watcher-a1b2c3d4-...`)
2. **Place logs liberally** using `logger.debug()` with the watcher ID attached — instrument every code path that could be relevant
3. **Update the watcher** with what you actually instrumented — call `watchers_update` to record the exact files, functions, and what each log captures (see "Update After Instrumentation" below)
4. **Ask the user to reproduce** the issue (or wait for the next occurrence)
5. **Query the logs via MCP** using `logs_query` with `WHERE watcher_id = '<uuid>'` — you see exactly the logs you placed, no noise
6. **Record findings** via `watchers_update` — accumulate your analysis so you don't lose context across conversations
7. **Iterate** — add more logs, query again, update findings
8. **Clean up** — remove all `logger.debug()` calls and watcher-scoped logger setup you added during the investigation
9. **Resolve** — mark the watcher as resolved via `watchers_update` with `status: "resolved"`

You should default to placing logs early and liberally when investigating runtime issues. Don't spend multiple rounds guessing from static code alone — instrument the code, get the data, then fix the bug. Logs persist across runs, so once placed, they keep providing insight. Use `logger.debug()` freely — the watcher ID lets you filter to only your logs, so volume isn't a concern.

**Be generous with logging.** When investigating an issue, don't just log the one line you suspect — log the inputs, the intermediate state, the branch taken, the output. Five `logger.debug()` calls is better than one if it saves a round-trip of "add more logs, reproduce again."

**Cleanup rule:** Use `logger.debug()` for temporary diagnostic logs placed during investigation. These are throwaway — **remove all watcher-related debug logs before marking the watcher as resolved.** This means deleting every `logger.debug()` / `dbg.debug()` call you added, along with any scoped logger setup (`logger.withWatcher(...)`, imports you added solely for logging, etc.). Do this cleanup as part of the resolve step, not as an afterthought. Permanent logs that provide ongoing operational value (errors, warnings, key events) should use the appropriate level and can stay.

---

## Watchers

Watchers are your primary tool for structured investigations. Every time you start debugging a runtime issue, **create a watcher first**. This gives you an ID that tags all your diagnostic logs, so you can query them in isolation.

### Watcher Titles

Every watcher needs a **title** — a short, concise label (under ~60 chars) that makes it easy to identify the watcher at a glance, like a ticket title. The title shows in the watcher list and dashboard header. Keep it scannable and specific.

**Good titles:**
- "Auth token refresh race condition"
- "WebSocket reconnect loop on slow networks"
- "500s on POST /api/orders — gateway timeouts"
- "Electron main process memory leak"

**Bad titles:**
- "Investigating an issue" (too vague)
- "User reports getting logged out from the dashboard frequently, sometimes within minutes of logging in" (too long — this belongs in the description)

### Watcher Descriptions and Findings Must Be Self-Contained

A watcher is the **single source of truth** for an investigation. A future Claude (or you in a new conversation) will read the watcher and need to pick up where you left off with zero prior context. The watcher description and findings together must tell the full story.

**Description** — Write when creating the watcher. Use inline code formatting (backticks) for any code-related references — variable names, function names, file paths, class names, method calls, field names, etc. This makes descriptions scannable and unambiguous. Include:
- **What the user reported:** The symptom in their words (e.g. "User reports getting logged out frequently, sometimes within minutes")
- **Suspicions:** What you think might be causing it after initial code analysis (e.g. "Race condition in `fetchWithAuth` — concurrent 401s may trigger multiple `refreshToken()` calls, revoking each other's tokens")
- **Where logs were placed:** Exact file paths and what each log captures (e.g. "`web/src/lib/fetch.ts`: 401 detection, refresh attempts, success/failure. `web/src/app/api/auth/refresh/route.ts`: missing cookie, invalid token, success")
- **What is being monitored:** What data you expect to see and what would confirm or rule out the suspicion (e.g. "Looking for multiple 'attempting token refresh' entries in quick succession, or 'invalid/expired refresh token' after a successful refresh")

**Findings** — Update as the investigation progresses. Include:
- What the logs revealed
- Root cause (if found)
- What was fixed and how
- Whether the fix needs further monitoring

**Example of a good description:**

> User reports getting logged out from the dashboard frequently, sometimes within minutes of logging in. Initial code review found a race condition in `fetchWithAuth()` (`web/src/lib/fetch.ts`): when multiple concurrent requests get 401, each awaiter resets `refreshPromise = null`, allowing a second refresh to start which revokes the first refresh's token. Fixed the race condition (`refreshPromise` now cleared in `.finally()`). Placed debug logs to monitor: client-side (`web/src/lib/fetch.ts`) — 401 detection, refresh attempts with URL, success/failure, redirect to `/login`. Server-side (`web/src/app/api/auth/refresh/route.ts`) — missing cookie, invalid/expired token with hash prefix, `user` not found, success with `userId`, errors. Looking for: invalid token errors shortly after successful refreshes (would indicate the race condition still exists), or any other pattern explaining the logouts.

### Status Summary

Every watcher has a `statusSummary` field — a short, authoritative headline of the investigation's current state. Think of it as the "subject line" of the investigation. This is what a human glances at in the dashboard, and what a new Claude session reads first to orient itself.

**Update it every time you review new data.** Overwrite the previous summary with a fresh one that reflects the current state. Good status summaries are:
- "Fix deployed, monitoring. 4 successful refreshes, 0 failures since fix."
- "Root cause identified: connection pool exhaustion under concurrent requests. Fix in progress."
- "Inconclusive. Placed additional logs in ws reconnect path. Waiting for reproduction."

**How it differs from findings:** The status summary is the headline; findings are the detailed analysis. A human reading the dashboard sees only the status summary in the collapsed view. The model reads the status summary first when picking up an existing watcher, then dives into findings for full context.

**When to set it:**
- After your first review of a new watcher's data (`watchers_update` with `statusSummary`)
- Every time you review new data flagged by the detection cron
- When the investigation state changes (fix deployed, resolved, waiting for repro, etc.)

### Expected Outcome

When monitoring a fix or waiting for reproduction, always include an **Expected outcome** section in the watcher description. This tells a future session (or yourself) what to look for when new data arrives, and what would confirm or rule out the hypothesis.

Example description ending:
> **Expected outcome:** If the fix works, we should see only `token refresh succeeded` entries with no `invalid/expired refresh token` errors. If the race condition still exists, we'll see `invalid/expired refresh token` errors shortly after successful refreshes.

This turns watcher review from "look at everything and figure it out" into "check whether the expected pattern matches."

### Update After Instrumentation

The watcher description is written at creation time — before you've actually placed any logs. Once you finish instrumenting the code, **immediately update the watcher** so it reflects what was actually done. This is critical because the next session will read the watcher and needs to know exactly what's instrumented and where, without having to scan the codebase.

Call `watchers_update` with an updated `description` that includes:
- **Every file you touched** and what logs you placed there
- **What each log captures** (inputs, branch decisions, timing, state, etc.)
- **The groups you used** so the next session knows how to query by subsystem

If you also have initial observations from reading the code (e.g. suspected root cause), put those in `findings`. The description is "what exists and where"; findings is "what we think and know."

**Example:** You create a watcher for intermittent WebSocket disconnects, then instrument 3 files. After placing all logs, update:

```
watchers_update({
  id: "a1b2c3d4-...",
  description: "Investigating intermittent WebSocket disconnects reported by user. Suspected cause: reconnect timer not being cleared on manual disconnect.\n\nInstrumentation placed:\n- `src/ws/connection.ts`: connect() entry with params, onopen/onclose/onerror events with codes and readyState, reconnect timer scheduling with delay value\n- `src/ws/reconnect.ts`: reconnect attempt counter, backoff calculation, timer clear on manual disconnect\n- `src/api/handler.ts`: request handler entry with traceId, WS send with payload size, send failures with error\n\nAll logs use group `ws/connection` or `ws/reconnect`. Looking for: onclose events without a preceding manual disconnect call, or reconnect attempts that overlap with an existing connection.",
  statusSummary: "Debug logs placed across 3 files. Awaiting reproduction.",
  findings: "Code review shows `reconnectTimer` is set in onclose but never cleared in `disconnect()` — likely cause. Logs placed to confirm."
})
```

**Why this matters:** Without this update, a future session sees only the original description ("investigating WebSocket disconnects") and has to grep the codebase to find what was instrumented. With it, the next session can immediately query the right groups, understand the coverage, and decide whether more logs are needed — or jump straight to analysis when data arrives.

### Creating and Using a Watcher

```
1. watchers_create({ title: "<short label>", logFileId: "<configured log file ID>", description: "<detailed description as above>" })
   -> returns ID: "watcher-a1b2c3d4-..."

2. Place logs in code with that watcher ID:
```

```ts
// Scoped logger — all logs from this get the watcher ID automatically
const dbg = logger.withWatcher("a1b2c3d4-...");
dbg.debug("ws reconnect attempt", { attempt: this.wsReconnectAttempts, delay });
dbg.debug("ws state before connect", { state: this.wsState, hasSocket: !!this.ws });
dbg.debug("ws onclose fired", { code: event.code, reason: event.reason });

// Or inline for one-off logs
logger.debug({ message: "auth header value", watcherId: "a1b2c3d4-...", metadata: { header } });
```

```
3. User reproduces -> query YOUR logs only:
   logs_query({ query: "SELECT timestamp, message, metadata FROM logs.entries WHERE watcher_id = 'a1b2c3d4-...' ORDER BY timestamp" })

   Then check what ELSE was happening around the same timestamps:
   logs_query({ query: "SELECT timestamp, level, group, message FROM logs.entries WHERE timestamp BETWEEN '2026-02-18 13:42:00' AND '2026-02-18 13:43:00' ORDER BY timestamp LIMIT 50" })

   Your watcher logs tell you what YOUR instrumented code paths did. The surrounding logs tell you what the rest of the system was doing at the same time — errors in other subsystems, concurrent requests, background jobs firing. This context often reveals the actual cause (e.g. your watcher shows a retry, but the surrounding logs show a database connection pool exhaustion that triggered it).

4. Record what you learned AND report to the user:
   watchers_update({ id: "a1b2c3d4-...", findings: "The reconnect fails because wsState is still 'connecting' when onclose fires — race condition between...", statusSummary: "Root cause: wsState race condition in onclose handler. Fix in progress." })
   Always tell the user what you found — the watcher records findings for future sessions, but the user needs to hear it now.

5. Fix the bug, verify, then clean up and resolve:
   - Remove all `logger.debug()` / `dbg.debug()` calls you added, plus any `withWatcher()` setup and imports added solely for the investigation
   - Then mark resolved:
   watchers_update({ id: "a1b2c3d4-...", status: "resolved", findings: "Fixed by checking wsState in onclose handler. Verified reconnect works after 5 attempts. Removed all diagnostic debug logs.", statusSummary: "Resolved. wsState race condition fixed, verified with 5 reconnect cycles." })
```

### ID Prefixes

The dashboard and MCP tools display IDs with human-readable prefixes: `watcher-<uuid>` for watchers and `log-<uuid>` for log files. These prefixes are **display-only** — the database stores raw UUIDs. Users may copy-paste prefixed IDs from the dashboard. You can safely pass either form (prefixed or raw UUID) to any MCP tool — the prefix is automatically stripped. However, when using IDs in **SDK code** (e.g. `withWatcher()`) or **SQL queries** (e.g. `WHERE watcher_id = '...'`), always use the **raw UUID without the prefix**.

### Watcher MCP Tools

- **`watchers_create`** — Start an investigation. Requires `logFileId` (use configured file), a concise title, and a detailed description.
- **`watchers_list`** — See all your watchers and which have new data.
- **`watchers_get`** — Read full details + findings. Resets the "new data" flag.
- **`watchers_update`** — Record findings, change status, update title, description, or status summary.

### When to Create a Watcher

- **Always** when investigating a runtime bug — even if you think it'll be quick
- When monitoring a fix to verify it works in production
- When you need to understand the flow through unfamiliar code at runtime
- When correlating behavior across multiple components

### Logging Generously with Watchers

Because watcher IDs let you filter to only your logs, don't be stingy. For a typical investigation, you might place 5-15 `logger.debug()` calls across the relevant code path:

```ts
const dbg = logger.withWatcher("a1b2c3d4-...");

// Log inputs
dbg.debug("handleRequest called", { method, path, headers: Object.keys(headers) });

// Log branch decisions
dbg.debug("auth check result", { authenticated, userId, tokenExpiry });

// Log intermediate state
dbg.debug("query built", { sql: query.substring(0, 200), paramCount: params.length });

// Log the thing you actually suspect
dbg.debug("response timing", { dbMs, renderMs, totalMs });

// Log the output
dbg.debug("response sent", { status, bodyLength: body.length });
```

This is much more effective than placing one log and having to do multiple rounds of "add more logs, reproduce again."

---

## Example: User Asks You to Analyze Logs

When the user asks you to look into a problem visible in logs (e.g. "why are we getting 500s on the billing endpoint?"), follow this pattern:

1. **Query the logs first** — use `logs_query` to understand the problem
2. **Report your findings** to the user
3. **Check for existing watchers** — call `watchers_list` to see if there's already a watcher tracking this issue
4. **If a related watcher exists** — read it with `watchers_get`, append your new findings via `watchers_update`
5. **If no watcher exists** — offer to create one:

> "I found the issue — the billing endpoint is throwing a null reference when `subscription.plan` is missing. I can set up a watcher to track this. It'll place debug logs around the billing flow so we automatically capture the full context next time it happens. Want me to do that?"

If the user agrees:
- `watchers_create({ title: "Null ref in billing endpoint", description: "Tracking null ref in billing endpoint when subscription.plan is missing", logFileId: "..." })`
- Place `logger.debug()` calls with the watcher ID around the relevant code paths
- The cron job will flag new matching logs automatically, so next time it happens you'll have full context

**This matters because investigations span multiple conversations.** A watcher persists your findings and keeps collecting data between sessions. Without one, the next conversation starts from scratch. With one, you (or a future Claude) can call `watchers_list`, see there's an active watcher with findings, and pick up where you left off.

### The full flow looks like:

```
User: "We're seeing intermittent 500s on POST /api/orders"

You:
  1. logs_query -> find the errors, identify the pattern
  2. Report: "These are timeout errors when the payment gateway takes >10s"
  3. watchers_list -> no existing watcher for this
  4. Offer: "Want me to set up a watcher? I'll instrument the order flow
     so we capture timing data automatically next time."

User: "Yes"

You:
  5. watchers_create({ title: "500s on POST /api/orders — gateway timeouts", logFileId: "<configured log file ID>", description: "Intermittent 500s on POST /api/orders — payment gateway timeouts >10s" })
  6. Place debug logs:
     - Before the gateway call (log request params, timestamp)
     - After the gateway call (log response time, status)
     - In the error handler (log the full error context)
     - At the retry logic (log attempt count, backoff delay)
  7. watchers_update({
     description: "Intermittent 500s on POST /api/orders — payment gateway timeouts >10s.\n\nInstrumentation placed:\n- `src/services/orderService.ts`: createOrder() entry with orderId and items, pre-gateway timestamp, post-gateway response time and status\n- `src/services/paymentGateway.ts`: charge() entry with amount and customerId, HTTP request timing, response status and body\n- `src/services/orderService.ts`: error handler with full error context and elapsed time\n- `src/services/orderService.ts`: retry logic with attempt count, backoff delay, previous error\n\nAll logs use group `api/orders`. Looking for: gateway response times >10s, retry patterns, and whether timeouts correlate with specific request params.",
     findings: "Initial analysis: gateway timeouts when response >10s. Code review shows no timeout configured on the HTTP client — defaults to Node's global timeout. Logs placed to capture timing data on next occurrence.",
     statusSummary: "Debug logs placed across 2 files. Awaiting reproduction." })
  8. Report back to user with what you placed and what to expect

--- later, in a new conversation ---

User: "The 500s happened again"

You:
  1. watchers_list -> see the active watcher with new_data = true
  2. watchers_get -> read previous findings
  3. logs_query WHERE watcher_id = '...' -> see the detailed debug logs
  4. Now you have full context without starting over
```

---

## When to Place Logs

- Debugging a runtime bug that isn't obvious from code alone
- Investigating timing issues, race conditions, or intermittent failures
- Understanding the flow through async code, event handlers, or callbacks
- Correlating behavior across multiple services or processes (use `traceId`)
- Monitoring a specific code path after a fix to verify it works

## Best Practices

**Use `withGroup()` to categorize logs — don't embed context in the message.**

The `group` field is a queryable column in ClickHouse. When you prefix messages manually, you lose the ability to filter and aggregate by subsystem. Groups should reflect the domain/feature area using slash-separated paths, like a file system:

```ts
// Bad — context is buried in the message string, not queryable
logger.info("[StorageService] Getting user");

// Good — group is a structured, queryable field
const log = logger.withGroup("storage");
log.info("Getting user");

// Also good — inline object form for one-off logs
logger.info({ message: "Getting user", group: "storage" });
```

**Use hierarchical slash-separated groups** so you can filter at any level of granularity:

```ts
// Specific subsystem groups — lets you zoom in or out when querying
logger.withGroup("api/billing").debug("charge initiated", { amount, customerId });
logger.withGroup("api/billing/webhooks").debug("stripe event received", { type, eventId });
logger.withGroup("api/orders").debug("order created", { orderId, items: items.length });
logger.withGroup("api/orders/autodispatch").debug("driver assignment", { orderId, driverId, distance });
logger.withGroup("ws/connections").debug("client connected", { clientId, protocol });
logger.withGroup("auth/oauth").debug("token refresh", { userId, expiresIn });
logger.withGroup("cron/cleanup").debug("retention sweep", { partitionsScanned, deleted });
```

This lets you query at different levels:
```sql
-- Everything in the billing subsystem
WHERE group LIKE 'api/billing%'

-- Just webhook processing
WHERE group = 'api/billing/webhooks'

-- All API logs
WHERE group LIKE 'api/%'

-- Log volume by subsystem
SELECT group, count(*) FROM logs.entries GROUP BY group ORDER BY count() DESC
```

When placing logs across multiple files or modules during an investigation, create a scoped logger per module so logs are naturally organized. Combine `withGroup()` and `withWatcher()` for maximum queryability — the group tells you *where* in the system, the watcher tells you *which investigation*.

```ts
const dbg = logger.withWatcher("a1b2c3d4-...").withGroup("api/orders/autodispatch");
dbg.debug("dispatch started", { orderId, availableDrivers: drivers.length });
dbg.debug("driver scored", { driverId, score, distance, eta });
dbg.debug("driver assigned", { orderId, driverId, dispatchMs: Date.now() - start });
```

Use `withTraceId()` to correlate logs across a single request or operation, especially across services.

---

## CLI for Non-JS Environments

When you need to log from shell scripts, cron jobs, build pipelines, or any non-JavaScript context, use the `moonwatch` CLI instead of the JS SDK. It's part of the same `@moonwatch/js` package.

**Prerequisites:** The CLI reads `MOONWATCH_API_KEY` (workspace key) and `MOONWATCH_LOG_FILE_ID` from environment variables, both set automatically during `/moonwatch setup`. No extra configuration needed — just use the commands below.

**First use:** Before using the CLI for the first time, check if `moonwatch` is available on PATH by running `which moonwatch`. If it's not installed globally, ask the user: "I'd like to install `@moonwatch/js` globally so I can use the `moonwatch` CLI to log from shell scripts. OK to run `npm install -g @moonwatch/js`?" Only proceed if they confirm. Once installed, it's available for all future sessions.

### Single log entry

```bash
moonwatch log "deploy started" --level INFO --group deploy
moonwatch log "backup complete" -l INFO -g cron/backup -m '{"duration":42}'
```

### Pipe mode — stream lines from any process

```bash
./build.sh 2>&1 | moonwatch pipe --group build
tail -f /var/log/app.log | moonwatch pipe --detect-level
```

### Watcher-tagged logging from scripts

```bash
moonwatch log "script checkpoint reached" --watcher-id <uuid> --group deploy
```

### Monitoring events from scripts

```bash
# Record a deploy event
moonwatch event deploy -t service=api -t commit=$(git rev-parse --short HEAD) -t version=1.2.3

# Record a server restart
moonwatch event server_restart -t reason=oom -t host=prod-1

# Requires MOONWATCH_PROJECT_ID and MOONWATCH_API_KEY env vars (or --project-id / --api-key flags)
```

### When to use the CLI vs the JS SDK

- **JS/TS code** → use the SDK (`logger.debug(...)` or `metrics.event(...)`)
- **Shell scripts, Makefiles, cron jobs, Docker entrypoints, CI pipelines, non-JS languages** → use the CLI (`moonwatch log ...` or `moonwatch event ...`)
- **Piping output from any process** → use `| moonwatch pipe`

The CLI supports all the same fields as the SDK: `--level`, `--group`, `--trace-id`, `--watcher-id`, `--metadata` (JSON string), `--tag` / `-t` (for events). Run `moonwatch --help` for full usage.

---

## SDK Quick Reference

### Setup (already done if the project uses this SDK)

**Important:** Look for an existing logger instance in the project before creating a new one. Projects typically have a shared logger in a file like `logger.ts` or `lib/logger.ts` — import from there instead of calling `createLogger()` again.

```ts
import { createLogger } from '@moonwatch/js';

const logger = createLogger({
  logId: 'uuid-from-dashboard',   // required — identifies the log stream
  apiKey: 'rl_...',               // required — ingestion auth
  group: 'api',                   // optional: default group for all entries
  traceId: 'req-123',            // optional: default trace ID
  level: 'DEBUG',                 // optional: minimum level (default: 'DEBUG')
  silent: false,                  // optional: suppress console echo (default: false)
  flush: 'auto',                  // optional: 'auto' (default), 'immediate', or 'manual'
});
```

### Flush Modes

- **`'auto'`** (default) — buffers logs and flushes every 1 second or when the buffer is full. Best for long-running servers (Express, Koa, etc.).
- **`'immediate'`** — flushes on the next microtask after each log call, batching all synchronous logs into one request. Use this for **serverless/Lambda** environments where the runtime may freeze before the 1-second timer fires.
- **`'manual'`** — no automatic flushing. You must call `await logger.flush()` yourself. Use when you need full control over when network requests happen.

**When setting up the SDK, choose the flush mode based on the runtime:**
- AWS Lambda, Vercel Functions, Cloudflare Workers, or any short-lived serverless → `flush: 'immediate'`
- Express, Koa, Fastify, Next.js server, or any long-running process → `flush: 'auto'` (default, can be omitted)
- Performance-critical hot paths where you want zero overhead → `flush: 'manual'`

```ts
// Example: Lambda
const logger = createLogger({
  logId: '...',
  apiKey: '...',
  flush: 'immediate',
});
```

### Logging

```ts
logger.debug("cache hit", { key: "user:42" });
logger.info("request handled", { method: "GET", path: "/api/users", ms: 12 });
logger.warn("slow query", { duration: 1200 });
logger.error(err as Error, { orderId: order.id });  // Error objects extract stack + error_type
logger.fatal(new Error("database unreachable"));
```

### Console Interception

```ts
logger.interceptConsole();        // captures console.debug/log/info/warn + unhandled errors
logger.interceptConsole("app");   // with a custom group name
logger.restoreConsole();          // undo
```

Use this to capture logs from third-party code or existing console.log statements without modifying them.

---

## Reading Logs via MCP

You have two MCP tools available:

- **`logs_list_log_files`** — Lists all log files with data. Call this first to find the log file ID.
- **`logs_query`** — Execute SQL (ClickHouse) against a log file. Tenant/log filtering is automatic.

### Common Queries

```sql
-- Recent errors
SELECT timestamp, level, message, stack FROM logs.entries WHERE level = 'ERROR' ORDER BY timestamp DESC LIMIT 20

-- Errors by type
SELECT error_type, count(*) as cnt FROM logs.entries WHERE error_type IS NOT NULL GROUP BY error_type ORDER BY cnt DESC

-- Search for a specific message pattern
SELECT timestamp, level, group, message FROM logs.entries WHERE message LIKE '%ECONNREFUSED%' ORDER BY timestamp DESC LIMIT 20

-- Error rate over time
SELECT toStartOfMinute(timestamp) as minute, count(*) FROM logs.entries WHERE level = 'ERROR' GROUP BY minute ORDER BY minute

-- Logs by group
SELECT group, count(*) FROM logs.entries GROUP BY group ORDER BY count() DESC

-- Trace a specific request across services
SELECT timestamp, group, level, message FROM logs.entries WHERE trace_id = 'req-abc-123' ORDER BY timestamp

-- Recent logs from a specific group
SELECT timestamp, level, message FROM logs.entries WHERE group = 'auth' ORDER BY timestamp DESC LIMIT 50

-- Error with source context (see the original code around the error)
SELECT timestamp, message, stack, source_context FROM logs.entries WHERE level = 'ERROR' AND source_context IS NOT NULL ORDER BY timestamp DESC LIMIT 5

-- All logs for a watcher investigation
SELECT timestamp, message, metadata FROM logs.entries WHERE watcher_id = 'a1b2c3d4-...' ORDER BY timestamp

-- Watcher logs filtered by time window
SELECT timestamp, message, metadata FROM logs.entries WHERE watcher_id = 'a1b2c3d4-...' AND timestamp >= '2026-02-16 10:00:00' ORDER BY timestamp
```

**Tip:** When investigating production errors, include `source_context` in your SELECT. It contains the original source code around the error line (resolved from source maps at ingestion time), so you can see exactly what code triggered the error without needing to open the file.

### Table Schema

```
seq UInt64              -- sequence number
timestamp DateTime64(3) -- e.g., '2025-01-22 14:30:00.123'
level String            -- DEBUG, INFO, WARN, ERROR, FATAL
trace_id String         -- request correlation
group String            -- categorization (e.g., 'api/billing', 'main', 'renderer')
message String          -- log content
stack Nullable(String)  -- stack trace (resolved via source maps if available)
error_type String       -- exception class name
release_id String       -- build release ID (for source map resolution)
source_context String   -- JSON: { file, line, code: { "lineNum": "source line" } } — original source around the error
watcher_id String       -- watcher ID (empty string if not tagged)
metadata JSON           -- arbitrary structured data
```

---

## Setup Wizard

*Triggered by `/moonwatch setup`.*

This guides the user through connecting Claude Code to Moonwatch. Follow these steps exactly in order.

### Step 1 — Personal API Key

1. **Check if a key was passed as an argument.** If the user ran `/moonwatch setup <key>` (e.g. `/moonwatch setup mw_personal_abc123`), use that key directly — skip to step 6.

2. **Check for the old `MOONWATCH_API_KEY` env var.** This is a deprecated name from an earlier version. Look for it in:
   - `~/.claude/settings.json` (user scope) under `env.MOONWATCH_API_KEY`
   - `.claude/settings.json` (project scope) under `env.MOONWATCH_API_KEY`
   - `.claude/settings.local.json` (project local scope) under `env.MOONWATCH_API_KEY`

   If found in **any** of these files, **remove the `MOONWATCH_API_KEY` key** from that file's `env` object and save. Tell the user:
   > Found the old `MOONWATCH_API_KEY` env var — this has been renamed to `MOONWATCH_PERSONAL_KEY`. Cleaning it up.

   If the old key's value looks like a valid personal key (starts with `mw_personal_`) and `MOONWATCH_PERSONAL_KEY` is not already set, reuse the value as the new key — save it as `MOONWATCH_PERSONAL_KEY` in `~/.claude/settings.json` and skip to step 6.

3. Check if the `MOONWATCH_PERSONAL_KEY` environment variable is already set (it's available as `process.env.MOONWATCH_PERSONAL_KEY` / accessible via the MCP server config).

4. **If set:** Verify it works by calling `logs_list_log_files`. If the call succeeds, tell the user their key is valid and skip to Step 2. If auth fails (401/403), tell the user their key is stale and continue to the key setup flow below.

5. **If not set or stale:** Tell the user:
   > Your Moonwatch personal key is not configured. You can find your key at:
   > **https://moonwatch.dev/app/setup**
   >
   > Paste your personal API key in the text input below.

   Use `AskUserQuestion` to collect the key. The user will paste their key via the "Other" free-text option. Provide these options for non-key-entry paths:
   - "I don't have an account" (description: "Sign up at moonwatch.dev first")
   - "I need help finding my key" (description: "I have an account but can't find the key")

6. Once you have the key (from argument, migrated old key, or user input), **save it to `~/.claude/settings.json`** under `env.MOONWATCH_PERSONAL_KEY`:
   - Read the existing file at `~/.claude/settings.json` (it may not exist — that's fine)
   - Parse as JSON (or start with `{}` if missing/invalid)
   - Set `settings.env.MOONWATCH_PERSONAL_KEY = "<the key>"`
   - Write the file back with `JSON.stringify(settings, null, 2)`
   - Use the user's home directory: on macOS/Linux `~/.claude/settings.json`, on Windows `%USERPROFILE%\.claude\settings.json`

7. Tell the user the key has been saved and that **they need to restart Claude Code** (or start a new session) for the env var to take effect. This is because env vars from settings are loaded at session start.

### Step 2 — SDK Check

Check if the SDK is already set up. If it is, we can reuse the existing `logId` and skip straight to file selection.

1. **Check for `@moonwatch/js` dependency** — look in `package.json` for `@moonwatch/js` in `dependencies` or `devDependencies`.

2. **Check for an existing logger setup** — search for `import { createLogger } from '@moonwatch/js'` (or similar) in the codebase. Look for a shared logger instance.

3. **If the SDK is installed and a logger exists with both a valid `apiKey` and `logId`:** The SDK is fully set up. Note the `logId` from the logger — it will be used as the production log file in Step 3. Skip to Step 3.

4. **If the SDK is not set up or is missing config:** That's fine — we'll handle SDK setup in Step 4 after log file selection. Proceed to Step 3.

### Step 3 — Log File Selection

1. Check if `MOONWATCH_LOG_FILE_ID` and/or `MOONWATCH_LOG_FILE_ID_DEV` environment variables are already set.

2. **If both are set:** Tell the user their log files are already configured and skip to Step 4.

3. **If `MOONWATCH_LOG_FILE_ID` is not set but a `logId` was found in the existing SDK logger (Step 2.3):** Use that `logId` as the production file — no need to ask. Tell the user:
   > Found your production log file in the existing SDK config — using the same file for Claude analysis.

   Still ask about the dev file if `MOONWATCH_LOG_FILE_ID_DEV` is missing — see step 4b below.

4. **If no existing `logId` and no env vars:** Run the log file selection flow.

   a. Call `logs_list_log_files` to discover available files.

   b. **Try to auto-detect the best match.** Look at the project name (from `package.json` `name` field, or the git repo name / directory name) and compare it against the available log file names and paths. Consider partial matches, common abbreviations, and workspace context. If you find a likely match, use it as the recommended first option.

   c. **Present 3 choices via `AskUserQuestion`:**
      - **Option 1 — Best match** *(only if you found a likely match in step b)*: `"Use <workspace>/<path>/<name>"` with description `"Looks like the right file for this project"`. If no match was detected, omit this option and only show options 2 and 3.
      - **Option 2 — Create a new log file**: `"Create a new log file"` with description `"Create a fresh log file for this project"`
      - **Option 3 — Enter a log file ID manually**: `"Enter a log file ID"` with description `"I know the ID of the file I want to use"`

      If there are other existing files that aren't the auto-detected match, you can include them as additional options (up to 4 total) so the user can pick a different existing file.

   d. **Handle each choice:**
      - **Best match / existing file selected:** Use that file's ID.
      - **Create a new log file:** Ask the user for a name (suggest the project name as default via `AskUserQuestion`). Then call `logs_create_log_file` with the name and the user's workspace ID. Use the returned file ID.
      - **Enter manually:** Use `AskUserQuestion` to collect the log file ID via the "Other" free-text option.

   e. **Dev log file** (optional) — after selecting the production file, ask if they also want a dev file. Call `logs_list_log_files` with `onlyOwned: true` to show only files from workspaces the user owns. Offer the same 3-choice pattern (auto-detect, create new, enter manually).

5. Write the selections:
   - Production file ID → `.claude/settings.json` in the **current project directory** under `env.MOONWATCH_LOG_FILE_ID`
   - Dev file ID → `.claude/settings.local.json` in the **current project directory** under `env.MOONWATCH_LOG_FILE_ID_DEV`

   For each file, read existing JSON, merge the env key, write back — same pattern as Step 1.

6. **Save the workspace API key.** Call `logs_get_workspace_key` with the production log file ID. If it succeeds, save the returned key to `.claude/settings.local.json` in the **current project directory** under `env.MOONWATCH_API_KEY`. This key is used by the CLI tool (`moonwatch log`, `moonwatch pipe`) and should not be committed — it's workspace-specific and lives in the local config only. If the call fails (insufficient permissions), skip this — the user can pass `--api-key` manually when using the CLI.

7. Note: `.claude/settings.local.json` is gitignored and specific to the local machine. `.claude/settings.json` is project-scoped and can be committed.

### Step 4 — SDK Setup

**Skip this step if the SDK was already fully set up (Step 2.3).**

1. **If the SDK is already installed and a logger exists with valid `apiKey` and `logId`:** Tell the user the SDK is already set up and skip to Step 5.

2. **If the SDK is not set up (missing dependency, no logger, or missing config):** Offer to set it up.

   Use `AskUserQuestion`:
   - "Yes, set up the SDK" (description: "Install @moonwatch/js and create a logger")
   - "Skip for now" (description: "I'll set up the SDK later")

   If the user wants to set it up:
   a. Install the dependency: `npm install @moonwatch/js`
   b. **Fetch the workspace API key yourself** — call `logs_get_workspace_key` with the production log file ID from Step 3. This works for any user with `write`, `manager`, or `owner` role. **Do not ask the user for the workspace key — fetch it automatically.** The only key the user should ever need to provide manually is their personal key (Step 1). If `logs_get_workspace_key` fails due to insufficient permissions (user has `read`-only role), then fall back to directing the user to ask a workspace manager for the key or copy it from workspace settings at https://moonwatch.dev/app.
   c. Create a logger file (e.g. `src/lib/logger.ts` or wherever makes sense for the project):
      ```ts
      import { createLogger } from '@moonwatch/js';

      export const logger = createLogger({
        apiKey: '<workspace-api-key-from-logs_get_workspace_key>',
        logId: '<the-production-log-file-id-from-step-3>',
      });
      ```
      Use the production log file ID from Step 3 as the `logId`, and the key returned by `logs_get_workspace_key` as the `apiKey`. Both values should be inlined directly — do not use placeholders or ask the user to fill them in.
      If the project runs in a serverless environment (AWS Lambda, Vercel Functions, Cloudflare Workers, etc.), add `flush: 'immediate'` to the config.
   d. Tell the user the SDK is set up and they can start logging with `logger.info(...)`, `logger.error(...)`, etc.

### Step 5 — Confirmation

Summarize what was configured:

> **Moonwatch setup complete!**
>
> - Personal API key: saved to `~/.claude/settings.json`
> - Production log file: `<file name>` → `.claude/settings.json`
> - Dev log file: `<file name>` → `.claude/settings.local.json` *(if selected)*
> - Workspace API key: saved to `.claude/settings.local.json` *(if fetched)*
> - SDK: `@moonwatch/js` installed, logger at `<path>` *(if set up)*
>
> You can now use:
> - `/moonwatch-analyze-logs` — Run a log analysis
> - `/moonwatch-list-watchers` — Check your active watchers
> - `moonwatch log "message"` — Log from shell scripts (CLI)

### Important Notes

- The personal API key goes in `~/.claude/settings.json` (user scope, not project scope) because it's tied to the user's Moonwatch account, not a specific project.
- Log file IDs go in project-scoped settings because different projects use different log files.
- The workspace API key (`MOONWATCH_API_KEY`) is for SDK ingestion and the CLI tool (writing logs). The personal API key is for MCP (reading logs via Claude). They are different keys. The workspace key goes in `.claude/settings.local.json` (project local, gitignored) because it's tied to a specific workspace.
- If the user says they don't have an account, direct them to https://moonwatch.dev to sign up via Google OAuth.
- Never display or log the full API key after saving — just confirm it was saved.

---

## Analyze Logs

*Triggered by `/moonwatch analyze-logs`.*

### Fetch Existing Context

Before running analysis, get context on what's already being tracked:

1. Call `watchers_list` with the resolved `logFileId`.
2. Note any active watchers — these represent ongoing investigations. You'll use this to avoid creating duplicate watchers for known issues.

### Run Analysis Queries

Execute these queries against the configured log file (adjust time window based on data volume):

1. **Level distribution** — overall health check:
   ```sql
   SELECT level, count(*) as cnt FROM logs.entries GROUP BY level ORDER BY cnt DESC
   ```

2. **Recent errors** — what's breaking:
   ```sql
   SELECT timestamp, level, group, message, error_type FROM logs.entries WHERE level IN ('ERROR', 'FATAL') ORDER BY timestamp DESC LIMIT 20
   ```

3. **Error patterns** — recurring issues:
   ```sql
   SELECT error_type, count(*) as cnt FROM logs.entries WHERE error_type IS NOT NULL GROUP BY error_type ORDER BY cnt DESC LIMIT 10
   ```

4. **Volume trends** — is anything spiking:
   ```sql
   SELECT toStartOfHour(timestamp) as hour, level, count(*) as cnt FROM logs.entries WHERE timestamp >= now() - INTERVAL 24 HOUR GROUP BY hour, level ORDER BY hour
   ```

5. **Active groups** — where are logs coming from:
   ```sql
   SELECT group, count(*) as cnt FROM logs.entries WHERE group != '' GROUP BY group ORDER BY cnt DESC LIMIT 15
   ```

Skip queries that don't apply (e.g., if there are very few logs, skip the volume trends query).

### Synthesize Findings

Present a clear summary to the user:

1. **Overall health** — total log volume, error rate, any concerning trends
2. **Top issues** — the most significant errors or patterns, with specific examples (timestamps, messages, stack traces if available)
3. **Anomalies** — anything unusual (sudden spikes, new error types, silent groups)

For each significant issue found:

1. **Check if an existing watcher already covers it** — compare with the active watchers fetched earlier. Match by error type, affected group, or similar description.
2. **If a related watcher exists:** Mention it. Offer to update its findings with the new data instead of creating a duplicate.
3. **If no watcher covers it:** Offer to create a new watcher to track and investigate the issue further. Follow the watcher creation workflow from the [Watchers](#watchers) section above.

### Report Format

Keep the report concise and actionable. Example:

> **Log Analysis Summary**
>
> **Volume:** 12,450 entries in the last 24h. Error rate: 2.3% (287 errors).
>
> **Top Issues:**
> 1. `TypeError: Cannot read property 'plan' of null` (142 occurrences) in `api/billing` — this is the most frequent error. No existing watcher covers this.
> 2. `ECONNREFUSED 127.0.0.1:6379` (38 occurrences) in `cache` — Valkey connection failures, clustered around 14:00-14:30 UTC. Likely a restart.
>
> **Active watchers covering known issues:**
> - "Auth token refresh race condition" — still active, monitoring
>
> Want me to create a watcher for the billing `TypeError`?

---

## List Watchers

*Triggered by `/moonwatch list-watchers`.*

### List and Display

1. Call `watchers_list` with the resolved `logFileId`.
2. Present watchers grouped by status:

**Active watchers first**, then resolved. For each watcher show:
- Title
- Status summary (if any)
- Whether it has new data (highlight these)
- Time since last update

Example output format:

> **Active Watchers**
>
> 1. **Auth token refresh race condition** — *Fix deployed, monitoring. 4 successful refreshes, 0 failures since fix.* (updated 2h ago) **[NEW DATA]**
> 2. **WebSocket reconnect loop** — *Inconclusive. Placed additional logs. Waiting for reproduction.* (updated 1d ago)
>
> **Resolved Watchers**
>
> 3. **500s on POST /api/orders** — *Resolved. Gateway timeout fixed by adding 15s client timeout.* (resolved 3d ago)

3. If there are no watchers, tell the user and suggest `/moonwatch analyze-logs` to start a log analysis.

### Handle New Data

If any active watcher has `new_data = true`, offer to investigate it:

1. Call `watchers_get` for the watcher (this resets the new_data flag).
2. Read the existing findings and description to understand context.
3. Query the watcher's tagged logs: `SELECT timestamp, message, group, metadata FROM logs.entries WHERE watcher_id = '<id>' ORDER BY timestamp DESC LIMIT 50`
4. Analyze the results in context of the watcher's description and previous findings.
5. Report findings to the user.
6. Update the watcher: call `watchers_update` with updated `findings` (append to existing), and a fresh `statusSummary`.

If multiple watchers have new data, work through them one at a time, starting with the most recently updated.

---

## Do NOT

- Do not pass `batchSize`, `flushInterval`, `httpEndpoint`, `wsEndpoint`, or `transport` — these do not exist. The only flush-related option is `flush: 'auto' | 'immediate' | 'manual'`.
- Do not call `errorWithStack()` — it does not exist. Use `logger.error(err)` instead.
- Do not call `logger.shutdown()` unless explicitly tearing down the logger mid-process. It is not required.

---

## Monitoring

Moonwatch also provides real-time monitoring dashboards. You can set up metrics collection in the user's code, create dashboards, and add visualization cards — all via MCP.

### Monitoring Concepts

- **Project** — a data container (like a log file for logs). The SDK pushes data to a project via `projectId`.
- **Dashboard** — a visual arrangement of cards within a project. Multiple dashboards can show different views of the same data.
- **Card** — a single visualization: graph (time series), table (aggregated events or snapshot), or key/value (snapshot).
- **Poll** — SDK calls a handler every 5s and pushes the result. Numbers go to ClickHouse (plottable), objects/arrays go to Valkey (snapshots).
- **Event** — SDK records a tagged occurrence (e.g. API request). Stored in ClickHouse for aggregation/grouping.

### Monitoring MCP Tools

- **`monitoring_list_projects`** — List all monitoring projects the user can access.
- **`monitoring_create_project`** — Create a new monitoring project. Requires write access. Returns the project ID for SDK setup.
- **`monitoring_list_dashboards`** — List dashboards within a project.
- **`monitoring_create_dashboard`** — Create a new dashboard. Requires write access.
- **`monitoring_list_metrics`** — Discover available poll metrics, events, and snapshots for a project. **Call this first** to see what data exists before creating cards.
- **`monitoring_query`** — Execute SQL against monitoring data (polls or events). Similar to `logs_query` but for monitoring tables.
- **`monitoring_create_card`** — Create a card on a dashboard with full config.
- **`monitoring_list_cards`** — List cards on a dashboard (id, type, title, position, source summary).
- **`monitoring_get_card`** — Get a single card's full settings, including complete config JSON.
- **`monitoring_update_card`** — Update an existing card's title, type, config, or position.
- **`monitoring_delete_card`** — Delete a card from a dashboard.

### Setting Up Monitoring in User Code

When the user wants to monitor their application, set up the SDK's monitoring API:

```ts
import { createMonitor } from '@moonwatch/js';

const metrics = createMonitor({
  projectId: '<uuid>',  // from monitoring_list_projects
  apiKey: '<workspace-api-key>',  // same key used for logging
});

// Numeric polls — plottable as line charts
metrics.poll('cpu.load', () => os.loadavg()[0]);
metrics.poll('memory.rss', () => process.memoryUsage().rss);

// Object polls — numeric fields extracted to ClickHouse + full snapshot to Valkey
metrics.poll('db', () => ({
  connections: pool.totalCount,
  active: pool.activeCount,
  idle: pool.idleCount,
}));

// Array polls — snapshot to Valkey (rendered as table)
metrics.poll('processes', () => pm2List());

// Async polls — for database queries, API calls, etc.
metrics.poll('db.stats', async () => {
  const result = await pool.query('SELECT count(*) as cnt, sum(active) as active FROM pg_stat_activity');
  return { connections: parseInt(result.rows[0].cnt), active: parseInt(result.rows[0].active) };
});

// Events — tagged occurrences for aggregation
metrics.event('api_request', {
  endpoint: req.path,
  method: req.method,
  status: res.statusCode,
  duration: elapsed,
});
```

### Creating Dashboard Cards via MCP

After data is flowing, create visualizations:

```
// Graph card — single poll metric
monitoring_create_card({
  dashboardId: "<id>",
  type: "graph",
  title: "CPU Load",
  config: {
    "series": [{ "source": "poll", "metric": "cpu.load", "aggregation": "avg", "color": "#007acc" }]
  }
})

// Graph card — event with aggregation
monitoring_create_card({
  dashboardId: "<id>",
  type: "graph",
  title: "Request Rate",
  config: {
    "series": [{ "source": "event", "event": "api_request", "aggregation": "rate", "color": "#3fb950" }]
  }
})

// Multi-series graph — correlate metrics
monitoring_create_card({
  dashboardId: "<id>",
  type: "graph",
  title: "Requests vs Response Time",
  config: {
    "series": [
      { "source": "event", "event": "api_request", "aggregation": "count", "color": "#007acc" },
      { "source": "event", "event": "api_request", "aggregation": "avg", "valueTag": "duration", "color": "#f0883e" }
    ],
    "yAxisMode": "independent"
  },
  width: 6
})

// Event table — top endpoints
monitoring_create_card({
  dashboardId: "<id>",
  type: "table",
  title: "Top Endpoints",
  config: {
    "source": "event",
    "event": "api_request",
    "groupBy": "endpoint",
    "aggregation": "count",
    "period": 3600,
    "limit": 10
  }
})

// Key/value card — snapshot
monitoring_create_card({
  dashboardId: "<id>",
  type: "keyval",
  title: "System Info",
  config: { "source": "snapshot", "snapshot": "system" }
})
```

### Monitoring Query Examples

```sql
-- Recent poll values
SELECT timestamp, name, value FROM monitoring.polls_realtime ORDER BY timestamp DESC LIMIT 20

-- Average CPU over last hour (from historical rollups)
SELECT toStartOfMinute(timestamp) as minute, sum(sum_value)/sum(count) as avg_value
FROM monitoring.polls_historical WHERE name = 'cpu.load' GROUP BY minute ORDER BY minute

-- Event count by endpoint
SELECT tags['endpoint'] as endpoint, count() as cnt
FROM monitoring.events WHERE name = 'api_request' GROUP BY endpoint ORDER BY cnt DESC

-- Average response time by endpoint
SELECT tags['endpoint'] as endpoint, avg(toFloat64OrNull(tags['duration'])) as avg_duration
FROM monitoring.events WHERE name = 'api_request' GROUP BY endpoint ORDER BY avg_duration DESC

-- Error rate (status >= 500)
SELECT toStartOfMinute(timestamp) as minute, count() as errors
FROM monitoring.events WHERE name = 'api_request' AND toFloat64OrNull(tags['status']) >= 500
GROUP BY minute ORDER BY minute
```

### Monitoring Table Schema

**monitoring.polls_realtime** — 5s precision, 10min TTL:
```
timestamp DateTime64(3), tenant_id UUID, project_id UUID, name LowCardinality(String), value Float64
```

**monitoring.polls_historical** — 1min rollups:
```
timestamp DateTime, tenant_id UUID, project_id UUID, name LowCardinality(String),
min_value Float64, max_value Float64, sum_value Float64, count UInt64
```

**monitoring.events** — tagged events:
```
timestamp DateTime64(3), tenant_id UUID, project_id UUID, name LowCardinality(String),
tags Map(String, String)
```

### Monitoring Workflow

When a user asks to set up monitoring:

1. **Check existing projects** — `monitoring_list_projects`
2. **If no project exists** — create one with `monitoring_create_project` (needs workspace ID from `logs_list_log_files` output)
3. **Get the workspace API key** — `logs_get_workspace_key` (same key used for logging)
4. **Set up SDK code** — add `createMonitor({ projectId, apiKey })` with polls and events for the metrics they want
5. **Create a dashboard** — `monitoring_create_dashboard`
6. **Add cards** — `monitoring_create_card` for each visualization
7. **Check available data** — `monitoring_list_metrics` once data starts flowing
8. **Verify** — `monitoring_query` to confirm data is being stored
