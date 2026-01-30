# OpenClaw Cron and Scheduling System: Deep-Dive Technical Analysis

## Executive Summary

OpenClaw features a sophisticated, production-ready cron scheduler that integrates deeply with its agent architecture. Unlike traditional cron, it understands agent sessions, multi-channel delivery, model selection, and thinking levels, making it a first-class automation primitive for conversational AI workflows.

**Key Statistics:**
- ~20 implementation files in `src/cron/`
- 10 comprehensive test suites
- 7+ messaging channels supported
- 3 schedule types (at/every/cron)
- 2 execution modes (main/isolated)
- 8 gateway RPC methods
- Tool exposure for agent self-management

## Core Architecture

### 1. Job Store (`~/.openclaw/cron/jobs.json`)

**File Format:**
```typescript
type CronStoreFile = {
  version: 1;
  jobs: CronJob[];
}
```

**Features:**
- Atomic writes (temp file + rename)
- Automatic backups (.bak)
- In-memory caching
- Automatic migration

**Job Structure:**
```typescript
type CronJob = {
  id: string,                    // UUID
  agentId?: string,              // Agent binding
  name: string,
  description?: string,
  enabled: boolean,
  deleteAfterRun?: boolean,
  schedule: CronSchedule,        // When to run
  sessionTarget: "main" | "isolated",
  wakeMode: "next-heartbeat" | "now",
  payload: CronPayload,          // What to execute
  isolation?: CronIsolation,
  state: CronJobState
}
```

### 2. Service Implementation

**Core Files:**
- `src/cron/service.ts`: Thin wrapper
- `src/cron/service/ops.ts`: All operations
- `src/cron/service/state.ts`: State management
- `src/cron/service/timer.ts`: Execution engine

**Service Methods:**
```typescript
class CronService {
  async start()           // Load store, arm timer
  stop()                  // Clear timer
  async status()          // Scheduler status
  async list(opts?)       // Filter jobs
  async add(input)        // Create job
  async update(id, patch) // Modify job
  async remove(id)        // Delete job
  async run(id, mode?)    // Manual trigger
  wake(opts)              // Immediate wake
}
```

### 3. Schedule Types

**1. At (one-shot):**
```typescript
{ kind: "at", atMs: 1706832000000 }
```
- Runs once at absolute timestamp
- Auto-disables after success
- Optional auto-delete

**2. Every (interval):**
```typescript
{ kind: "every", everyMs: 600000, anchorMs?: 1706832000000 }
```
- Fixed interval (milliseconds)
- Optional anchor for alignment
- Recomputes after each run

**3. Cron (expression):**
```typescript
{ kind: "cron", expr: "0 7 * * *", tz?: "America/Los_Angeles" }
```
- 5-field cron syntax
- IANA timezone support
- Powered by `croner` library

### 4. Timezone Handling

Uses `croner` library with full IANA timezone support:
```bash
openclaw cron add --cron "0 7 * * *" --tz "America/Los_Angeles"
```

**Behaviors:**
- Omitted `tz` → Host system timezone
- ISO timestamps without tz → UTC
- DST transitions handled by croner

## Execution Modes

### Main Session Jobs

**Payload:** `{ kind: "systemEvent", text: string }`

Enqueues system event into main session:

**Execution Flow:**
1. Validate payload (text required, kind=systemEvent)
2. Enqueue via `enqueueSystemEvent(text, { agentId })`
3. If `wakeMode: "now"`:
   - Wait for main lane idle (max 2 min)
   - Execute heartbeat synchronously
   - Report based on heartbeat result
4. If `wakeMode: "next-heartbeat"`:
   - Request heartbeat
   - Mark success immediately

**Use Cases:**
- Reminders needing full context
- Periodic checks in main history
- Wakeup events with normal heartbeat prompt

### Isolated Jobs

**Payload:** `{ kind: "agentTurn", message: string, ...options }`

Runs in dedicated `cron:<jobId>` sessions:

**Execution Flow:**
1. Create fresh session: `agent:<agentId>:main:cron:<jobId>`
2. Resolve agent config (model, thinking, timeout)
3. Build prompt: `[cron:<jobId> <name>] <message>`
4. Execute agent turn
5. Extract output and summary
6. Handle delivery (if configured)
7. Post summary to main session
8. Wake main if requested

**Critical Properties:**
- Fresh session ID each run (no history carryover)
- Stable session key for model/skill persistence
- Main session not polluted
- Summary posted for user awareness

## Advanced Features

### 1. Model and Thinking Overrides

```typescript
payload: {
  kind: "agentTurn",
  message: "Weekly deep analysis",
  model: "anthropic/claude-opus-4-5",
  thinking: "high",
  timeoutSeconds: 300
}
```

**Resolution Cascade:**
1. Job payload override (highest)
2. Hook-specific config
3. Agent config
4. System defaults

**Thinking Levels:** off, minimal, low, medium, high, xhigh

### 2. Multi-Channel Delivery

**Supported:**
- WhatsApp: `+15551234567` (E.164)
- Telegram: `-1001234567890` or `-1001234567890:topic:123`
- Discord: `channel:1234567890` or `user:1234567890`
- Slack: `channel:C1234567890` or `user:U1234567890`
- Signal, iMessage, Mattermost (plugin)
- `last`: Main session's last route

**Telegram Topic Support:**
```bash
--to "-1001234567890:topic:123"
```

**Delivery Modes:**
- `deliver: true` → Deliver to channel
- `deliver: false` → Internal only
- Omitted + `to` set → Auto-deliver
- `bestEffortDeliver: true` → Don't fail job on delivery failure

### 3. Agent Binding

Pin jobs to specific agents:
```bash
openclaw cron add --agent ops --message "Check ops queue"
```

**Use Cases:**
- Different agents for different tasks
- Isolated model/skill configurations
- Workspace segregation

### 4. One-Shot with Auto-Delete

```bash
openclaw cron add \
  --at "2026-01-12T18:00:00Z" \
  --system-event "Submit report" \
  --delete-after-run
```

**Behavior:**
- Normal: Disable after run (stays in store)
- Auto-delete: Remove from store after success
- On error: Remains enabled for retry

### 5. Run History Tracking

**Location:** `~/.openclaw/cron/runs/<jobId>.jsonl`

**Entry Format:**
```typescript
{
  ts: number,
  jobId: string,
  action: "finished",
  status: "ok" | "error" | "skipped",
  error?: string,
  summary?: string,
  durationMs?: number,
  nextRunAtMs?: number
}
```

**Auto-Pruning:** Keeps last 2,000 lines when file exceeds 2MB

## CLI and Gateway Integration

### CLI Commands

```bash
openclaw cron status
openclaw cron list [--all]
openclaw cron add [options]
openclaw cron edit <jobId> [options]
openclaw cron run <jobId> [--force]
openclaw cron runs --id <jobId> [--limit N]
openclaw cron remove <jobId>
```

**Add Options:**
```bash
--name "Job name"
--agent <agentId>
--session main|isolated
--wake now|next-heartbeat
--at <time>|--every <duration>|--cron <expr>
--tz <timezone>
--system-event <text>|--message <text>
--model <model>
--thinking <level>
--deliver
--channel <channel>
--to <target>
--best-effort-deliver
--delete-after-run
```

### Gateway RPC Protocol

**Methods:**
- `wake` → Immediate wake event
- `cron.status` → Scheduler status
- `cron.list` → List jobs
- `cron.add` → Create job
- `cron.update` → Modify job
- `cron.remove` → Delete job
- `cron.run` → Manual trigger
- `cron.runs` → Run history

**Validation:** JSON Schema via @sinclair/typebox

### Tool Exposure

**File:** `src/agents/tools/cron-tool.ts`

Agent can manage its own schedule:
```typescript
{
  name: "cron",
  description: "Manage cron jobs and wake events",
  actions: ["status", "list", "add", "update", "remove", "run", "runs", "wake"]
}
```

**Context Injection:**
`contextMessages` parameter fetches recent chat history:
```typescript
cron({
  action: "add",
  job: {...},
  contextMessages: 5  // Include last 5 messages
})
```

Format: `\n\nRecent context:\n- User: ...\n- Assistant: ...`

## Production Patterns

### 1. Concurrency Control

**Current:** Sequential execution (one job at a time)

**Stuck Job Detection:**
- If `runningAtMs` > 2 hours old → Clear marker
- Prevents permanent deadlock from crashes

### 2. Timer Management

**Single Timer:** One `setTimeout` for next due job

**Features:**
- Timer unref (doesn't block exit)
- Max timeout clamp (2^31-1 ms)
- Rearm on changes
- Duplicate prevention via state lock

### 3. Main Session Coordination

**Synchronous Heartbeat Execution:**
1. Enqueue system event
2. Wait loop (max 2 min):
   - Call `runHeartbeatOnce({ reason: "cron:<jobId>" })`
   - If busy, wait 250ms and retry
   - Otherwise break
3. Report job status based on result

**Why?** Ensures main session processed event before marking job complete

### 4. Post-to-Main Summaries

After isolated job completes:
```
Cron: Inbox has 3 new emails (done)
```

**Modes:**
- `summary`: Job summary (≤2000 chars)
- `full`: Full output (≤8000 chars)

**Prefix:** Configurable (default: "Cron")

### 5. Error Handling

**Execution Errors:**
- Catch all → Mark `status: "error"`
- Store in `job.state.lastError`
- Job remains enabled (retry on next schedule)
- No automatic backoff

**Validation Errors:**
- Main with agentTurn → Skip
- Isolated with systemEvent → Skip
- Empty text → Skip

**Delivery Errors:**
- Normal: Job fails
- Best-effort: Job succeeds, delivery skipped

## Strengths

1. **Deep Agent Integration**: Model/thinking overrides, session isolation
2. **Production Reliability**: Atomic writes, stuck job detection, duplicate prevention
3. **Rich Scheduling**: 3 schedule types, timezones, human-friendly syntax
4. **Multi-Channel Delivery**: 7+ channels, topic support, best-effort mode
5. **Observability**: JSONL history, event stream, detailed status
6. **User Experience**: CLI, agent tool, context injection, auto-delete
7. **Security**: External content sandboxing, suspicious pattern detection

## Weaknesses

1. **Concurrency**: No parallel execution (sequential only)
2. **Distribution**: File-based storage (not distributed)
3. **Retry Logic**: Simple (no backoff, no max retries)
4. **Resource Limits**: No per-job timeout, memory limit, disk quota
5. **Guarantees**: Best-effort only (no at-least-once or exactly-once)

## Comparison to Other Systems

### vs. Unix Cron
- **OpenClaw +**: AI-native, session-aware, multi-channel
- **Unix +**: Battle-tested, parallel, distributed

### vs. Celery
- **OpenClaw +**: Lightweight, no broker, agent integration
- **Celery +**: Distributed, robust retry, parallel

### vs. Kubernetes CronJobs
- **OpenClaw +**: Simpler, faster, session context
- **K8s +**: Distributed, fault-tolerant, isolation

## Future Directions

1. **Parallel Execution**: Add `maxConcurrentRuns` config
2. **Retry Policies**: Exponential backoff, max attempts
3. **Job Dependencies**: `dependsOn` field, propagate failures
4. **Distributed**: Database store, leader election
5. **Delivery Guarantees**: Retry with backoff, dead-letter queue

## Conclusion

OpenClaw's cron system is a **sophisticated, production-ready automation layer** that goes far beyond traditional cron. It deeply integrates with the agent architecture, supporting model overrides, multi-channel delivery, session isolation, and conversational self-management.

**Key Innovations:**
- AI-native scheduling (model/thinking/context)
- Multi-channel delivery with topic support
- Agent self-management via tool
- Security-first (sandboxing, pattern detection)

**Production Gaps:**
- Sequential execution (no parallelism)
- File-based storage (not distributed)
- Simple retry (no backoff)

**Overall Assessment**: **8.5/10** - Enterprise-ready for single-gateway deployments. For high-scale distributed setups, would need evolution (distributed store, parallel execution).

**Files Analyzed:** ~20 files in src/cron/, 10 test suites, CLI/gateway integration
